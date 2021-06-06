# Install-Kube-Prometheus-to-Kubernetes
**Requirements:**
We need 2 linux (could be any linux variants) instances on cloud or premise. For this excercise, we use on 2 on premise ubuntu 18.04 lts with assumption that 
the default remote ansible user being use is "ubuntu" and already has public key upload to remove hosts.

1.Directory layout:
```
    ├── ansible.cfg                 #Ansible config
     ── deploy-kbmaster.yml         #Master playbook
    ├── deploy-kbnodes.yml          #Node Playbook
    ├── fetch-kubercerts.yml
    ├── group_vars                  #Group var
    │   ├── all
    │   │   ├── vars.yml
    │   │   └── vault.yml           #Vault file
    │   └── windowsagents.yml
    ├── ingress-nginx.yml               #Ingress controller
    ├── kube-prometheus-ingress.yml     #Ingress for Kube-Prometheus stack
    ├── kube-prometheus-stack           #Kube-Prometheus helm chart
    ├── host_vars
    ├── log
    ├── nfs-pv.yml                  #Persistent volume
    ├── production
    │   └── static_hosts
    ├── README.md
    ├── roles
    │   ├── docker
    │   │   ├── files
    │   │   │   └── daemon.json
    │   │   ├── handlers
    │   │   │   └── main.yml
    │   │   └── tasks
    │   │       └── main.yml
    │   ├── kbmaster
    │   │   ├── handlers
    │   │   │   └── main.yml
    │   │   ├── meta
    │   │   │   └── main.yml
    │   │   └── tasks
    │   │       └── main.yml
    │   └── kubernetes
    │       ├── handlers
    │       │   └── main.yml
    │       ├── meta
    │       │   └── main.yml
    │       └── tasks
    │           └── main.yml
    ├── staging
    │   └── static_hosts
    ├── tcp-ingress-config.yml
    └── ubuntu.pem                #privare key
```
2. Ansible inventory - we use host in staging folder
```
ansible-inventory -i staging --graph
@all:
  |--@stage_jenkins:
  |  |--jenkins-host
  |--@stage_kubernetes:
  |  |--master-node       #This will be our master node
  |  |--node1             #This will be worker node
  |--@stage_windows:
  |  |--windows-host
  |--@ungrouped:@all:
```
3. Deploy kubernetes master
```
ansible-playbook -i staging/ --limit master-node deploy-kbmaster.yml --ask-become

TASK [kbmaster : debug] ************************************************************************************************
ok: [master-node] => {
    "out.stdout_lines": [
        "kubeadm join xxxxxxxx:6443 --token xxxxxxxxxxxxx     --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxx "
    ]
}
``` 
Copy the join token output and update the variable "vault_join_command" in group_vars/all/vault.yml and encrypt the vault.yml
``` 
ansible-vault encrypt group_vars/all/vault.yml
New Vault password:
Confirm New Vault password:
Encryption successful
```
4 Deploy worker node.
``` 
ansible-playbook -i staging/ --limit node1 deploy-kbnodes.yml --ask-become --ask-vault-pass
BECOME password:
Vault password:
```
fetch the kubenetes certs to ansible work station
```
ansible-playbook -i staging/ --limit master-node fetch-kubercerts.yml --ask-become --ask-vault-pass
BECOME password:
Vault password:
```
check the cluster
```
kubectl get node
master-node      Ready    master   3m   v1.18.3
node1            Ready    <none>   2m   v1.18.3
```
5. Deploy ingress controller <br>
  download ingress-inginx controller yaml file
```
curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.46.0/deploy/static/provider/baremetal/deploy.yaml -o ingress-nginx.yml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 18224  100 18224    0     0  63720      0 --:--:-- --:--:-- --:--:-- 63720
```
update ingress-nginx.yml to support tcp redirect for prometheus and alert manager then deploy it
```
kubectl apply -f ingress-nginx.yml
kubectl apply -f tcp-ingress-config.yml
```
5. Install helm and deploy kube-prometheus
install helm
```
curl -O https://get.helm.sh/helm-v3.6.0-linux-amd64.tar.g
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 13.5M  100 13.5M    0     0  56.0M      0 --:--:-- --:--:-- --:--:-- 56.0M
tar xzvf helm-v3.6.0-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin
helm version
version.BuildInfo{Version:"v3.5.4", GitCommit:"1b5edb69df3d3a08df77c9902dc17af864ff05d1", GitTreeState:"clean", GoVersion:"go1.15.11"}
```
add kube-prometheus repo
```
 $helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
 "prometheus-community" has been added to your repositories
 $helm repo update
 ...Successfully got an update from the "prometheus-community" chart repository
 $helm pull prometheus-community/kube-prometheus-stack
 $tar xzvf kube-prometheus-stack-11.1.1.tgz
```
Update the value.yaml so we monitor outside of the stack by changing the value of podMonitorSelectorNilUsesHelmValues and serviceMonitorSelectorNilUsesHelmValues<br>
we also need to create persistent storage class add the into the AlertManger global setting for sending email alert to us. Please check the vaule.yaml for all change <br>
install kube-prometheus stack
```
$kubectl create namespace monitoring
$kubectl create nfs-pv.yml
persistentvolume/prometheuss-pv created
$helm install monitoring -f kube-prometheus-stack/values.yaml ./kube-prometheus-stack -n monitoring
NAME: monitoring
LAST DEPLOYED: Sun Jun  6 10:33:01 2021
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitoring"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager                      and Prometheus instances using the Operator
```
apply ingress for stacks
```
$kubectl apply -f kube-prometheus-ingress.yml
ingress.networking.k8s.io/ingress-grafana configured
ingress.networking.k8s.io/ingress-alertmanager configured
ingress.networking.k8s.io/ingress-prometheus configured
```
If everything correct, you can access Grafana, Prometheus, Alertmanager at the url have been setup in kube-prometheus-ingress.yml.<br>
and alert email should send to the email address configure in the value.yaml of the Kube-Prometheus stack.
