## <font color='red'>Hitachi Vantara Foundry 2.2.1 Platform</font>  

Ansible playbooks install and configures Hitachi Vantara Foundry Platform.

Prerequisites for the CentOS 7 machines:
* A public key generated on your Ansible Controller
* SSH passwordless access on Nodes with root permissions
* Completed 01 Infrastructure section
* Completed 02 Pre-flight section

The following playbooks are run:  

#### cluster.yml
* Installs k8s 18.10

#### pre-flight_foundry.yml
* Update packages
* Ensure Map Max count > 262144
* Install Helm - all Nodes
* Prepare kubeconfig
* Install jq
* Install kubectl
* Configure kubectl for 'installer' access
* Install Docker
* Configure a Docker insecure Registry - Ansible Controller
* Copy over certs to 'installer'
* Install OpenEBS storage class


#### install_foundry.yml
* Creates a Log directory
* Creates a Foundry directory
* Unarchives Foundry Control Plane 2.2.1
* Creates a Metrics directory
* Unarchives Metrics 1.0.0
* Installs Cluster Services
* Run Hitachi CRDs
* Install Foundry Control Plane
* Upload Foundry charts & images
* Install Metrics Addon
* Upload Metrics image

---


<em>Run the playbook - kubespray-release-2.14/cluster.yml</em>   
Its is important that you explicitly include all the parameters when running the kubespray-release-2.14/cluster.yml playbook. 
Kubespray release 2.14 installs and configures the Foundry Platform supported version: kubernetes 1.18.10

Pre-requistes:
* Firewalls are not managed by kubespray. You'll need to implement appropriate rules as needed. You should disable your firewall in order to avoid any issues during deployment.  
* If kubespray is ran from a non-root user account, correct privilege escalation method should be configured in the target servers and the ansible_become flag or command parameters --become or -b should be specified. 

``run the cluster.yml playbook:``
```
cd /installers/kubespray-release-2.14
ansible-playbook -i hosts-skytap.yml --extra-vars "@extra-vars.yml"  -b -v cluster.yml
```
Note: this is going to take about 5-7 mins..

<font color='green'>The following section is for Reference only.</font>

``if you need to reset the k8s deployment:``
```
cd /installers/kubespray-release-2.14
ansible-playbook -i hosts-skytap.yml --extra-vars="@extra-vars.yml" reset.yml -b -v --become-user=root
```
Note: This will still keep some residual config files, IP routing tables, etc

``rest kubernetes cluster using kubeadm:``
```
kubeadm reset -f
```
``remove all the data from all below locations:``
```
sudo rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/*
```
``flush all the firewall (iptables) rules (as root):``
```
sudo -i
iptables -F && iptables -X
iptables -t nat -F && iptables -t nat -X
iptables -t raw -F && iptables -t raw -X
iptables -t mangle -F && iptables -t mangle -X
```
``restart the Docker service:``
```
systemctl restart docker
```

---

<em>Run the playbook - pre-flight_foundry.yml</em>      
This will update, install and configure the various required packages for the Foudry Platform.
 

``run the playbook - pre-flight_foundry.yml:`` 
```
cd /etc/ansible/playbooks
ansible-playbook -i hosts-skytap.yml --extra-vars="@extra-vars.yml" -b -v pre-flight_foundry.yml
```
Note: Run in OS Terminal. Do not run playbook in VSC Terminal. 

---

<em>Configure Registry</em>  
Notice that the last few playbooks haven't run.  To complete the playbook tasks:

``restart Docker:``
```
systemctl status docker
systemctl restart docker
```
Note: This is really just a check of the docker service.

``to 'log' the 'installer' user out and in:`` 
```
sudo su - installer 
```
``re-run the playbook - pre-flight_foundry.yml:`` 
```
cd /etc/ansible/playbooks
ansible-playbook -i hosts-skytap.yml --extra-vars="@extra-vars.yml" -b -v pre-flight_foundry.yml -t continue
```
Note:  This will pick up the playbook from the continue tag onwards.

---

<em>Run the playbook - install_foundry.yml</em> 

``run the playbook - install_foundry.yml:`` 
```
cd /etc/ansible/playbooks
ansible-playbook -i hosts-skytap.yml --extra-vars="@extra-vars.yml" -b -v install_foundry.yml
```
Note: It will take about 10mins to unachive the Founadry Platform package.  

you should have some logs appearing.  
``tail install-cluster-services.log: (new terminal)``
```
cd /installers/logs
ls
tail -f install-cluster-services.log
```
``check namespaces:``
```
kubectl get ns
```
Note: wait until all the cluster services have been installed, otherwise not all the namespaces will appear.  
``check the pods:``
```
kubectl get pods -A
```
``to access the Foundry Solutions Control Plane:``
```
username: foundry
echo $(kubectl get keycloakusers -n hitachi-solutions keycloak-user -o jsonpath="{.spec.user.credentials[0].value}")
```
or if you have configured .kubectl_aliases, just type ``foundry`` at command prompt.

---

<em>.kubectl_aliases</em>  
To save typing out the kubectl commands, in the resources folder there's a kubectl_aliases file which you copy over to your $HOME directory.

<font color='green'>The .kubectl_alias has been configured.</font>

``add the following to your .bashrc/.zshrc file:``
```
[ -f ~/.kubectl_aliases ] && source ~/.kubectl_aliases
```

If you want to print the full command before running it.   

``add this to your .bashrc or .zshrc file:``
```
function kubectl() { echo "+ kubectl $@">&2; command kubectl $@; }
```

For further information:

> browse to: https://github.com/ahmetb/kubectl-aliases

---