## <font color='red'>Lumada DataOps Suite 1.2.0</font>

DataOps unlocks business value by operationalizing data management with automation and collaboration. Lumada DataOps Suite helps you build your DataOps practice for more business agility with an intelligent data operations platform.

#### <font color='red'>Prerequisites for LDOS:</font>

* Foundry Platform:   
LDOS is installed over an existing k8s cluster running cluster services (a default storage class, istio and  cert-manager) and the Hitachi Vantara Solution Control Plane.  

* Metrics-addon: 
The Metrics Add-On is a generic extension for Foundry. It will be installed as a common solution in Foundry that will silently and transparently collect generic kubernetes metrics about any resource in the cluster.

* NFS Server:     
The Dataflow engine needs access to the Pentaho ETL files, plug-ins and kettle.properties. These will be mounted, into the containers, via NFS from an external NFS Server mount point. 

* Object Storage Service:   
 The Catalog leverages an object store to manage it's fingerprinting and Spark logs.  The default internal minio pod is not sufficient to handle production workloads (as it will fill up the clusters filesystem).   A third party object store, such as S3 or an external mino cluster should be used in production installations.

  - The LDOS installation requires configuraiton for the local environment.  Many of these are already set/used to install the control-plane.   Others can be discovered from the cluster.
    - The NFS mount point needs to be specified in the extra-vars.yml.  If it is missing the pods will fail to attach to it at startup and the install process will not complete.
    - The Object Store needs to be configured at install.   The default configuration for minio can be used and then changed after install, or the HELM chart for catalog can be modified prior to installation.
    - The default installation is: LDOS.  To change this, manually edit the "install_mode" in env.properties.
    - Additional editing of the default helm charts or install.sh itself may be needed for a custom installation of the product components.

Please read the documentation: [LDOS 1.2.0 Installation & Configuration](resources/LDOS-1.2.0_Installation_and_Configuration_Guide.pdf)

All files required for installation are available in the release folder and can be found in the link below.  
https://hcpanywhere.hitachivantara.com/a/PWPVYtZj1UovY9VO/e52a0db2-ad14-4673-941b-c304c2b108b2?l

You’ll need your Hitachi Vantara credentials or ask Customer Success.

The following playbooks are run:

#### preflight_nfs.yml
* Install NFS
* Create exports configuration file
* Start NFS
* Check NFS mounts
* Show mounts   

#### install_ldos-1.2.0.yml
* Install NFS utils
* Create directories
* Prepare env.properties
* Get foundry password
* Populate env.properties template
* Update Hostnames in Helm Charts
* Install LDOS
* Check Pods

---

#### <font color='red'>Pre-requisties</font>

* Check the Health of the Foundry Platform
* Install Metrics-addon 1.0.0 (Optional)
* Install NFS Server

<em>Check Foundry Platform</em> 

Before you start the LDOS installation, check that the Foundry Platform is healthy.

``check namespaces (Ansible Controller box):``
```
kubectl get namespaces -o wide (alias: kgns -o wide)
```
``check nodes:``
```
kubectl get namespaces -o wide (alias: kgno -o wide)
```
``check Pods in hitachi-solutions:``
```
kubectl get pods -n hitachi-solutions -o wide (alias: kgpo -n hitachi-solutions -o wide)
```

<font color='teal'>Please ensure that the Pentaho EE license has been copied into the correct mount path</font> <font color='red'>before</font><font color='teal'> you install LDOS.</font>  

Pentaho EE License path: /data/pdi/licenses/.installedLicenses.xml on Pentaho server.

---

<em>Download and unpack the Metrics Add-On (Optional)</em>  

If you have completed the Installation & Configuration of the Foundry Platform, the Metrics-addon 1.0.0. image and chart has been uploaded into the Registry.  

Recommended to let the LDOS install script install the Metrics-addon.

Please refer to: Lab - LDOS Pre-flight

Please refer to the documentation to manually upload: [LDOS 1.2.0 Installation & Configuration](resources/LDOS-1.2.0_Installation_and_Configuration_Guide.pdf)  


Please refer to the official Metrics Add-On documentation for details and additional troubleshooting: 
http://docs.foundry.wal.hds.com/addons/metricsaddon/docs/1.0.0/UserManuals/InstallingMetricsAddonSolutionAtControlPlane/


``verfify Metrics-addon CRDs:``
```
kubectl get CrdPackage -n foundry-crds | grep "metrics"
```
``verfify Metrics-addon:``
```
kubectl get solutionpackage -n hitachi-solutions | grep "metrics"
```

---

<em>Install NFS Server - preflight_nfs.yml</em>  

Installs a NFS server that is required by the DataFlow Engine and DataFlow Importer.

<font color='green'>The NFS server has been installed and configured.</font>

``run the playbook - pre-flight_nfs.yml:``
```
cd /etc/ansible/playbooks
ansible-playbook pre-flight_nfs.yml
```
The /etc/exports file controls which file systems are exported to remote hosts and specifies options.  
``verify the export configuration file (HA Proxy Server):``
```
sudo nano /etc/exports
```

---

#### <font color='red'>Install LDOS 1.2.0</font>

The install-ldos-1.2.0.yml playbook performs the following tasks.
- Install NFS utilities on all hosts. Again, this is needed to be able to mount the shared directory for KTR, KJB and additional content.
- Run update-hostname.sh to update the hostnames within the Helm chart templates.
- Run upload-solutions.sh to load the modified Helm charts into the Solution Control Plane, to make them available for installation.
- Configure env.properties values based on the local environment (see env.properties.template for additonal context)
    
    ####
    |Variable|Value|From|
    |-|-|-|
    | hostname|{{ apiserver_loadbalancer_domain_name }}|                            from extra-vars.yml|
    | registry|{{ registry_domain }}:{{ registry_port }}|  from extra-vars.yml|
    | foundry_client_secret|{{ client_secret }}        |                            extracted from the installation|
    | username|foundry                                 |                            hardcoded|
    | password|{{ foundry_password }}                  |                            extracted from the installation|
    | volume_host|{{ nfs_host }}                       |                            from extra-vars.yml|
    | volume_path|{{ nfs_path }}                       |                            from extra-vars.yml|


``run the playbook - install_ldos-1.2.0.yml:``
```
cd /etc/ansible/playbooks
ansible-playbook -i hosts-skytap.yml --extra-vars="@extra-vars.yml" -b -v install_ldos-1.2.0.yml
```
Note: This will take about 65mins to complete. 

<font color='teal'>The current installation will fail to install the Data Transformation Editor. Allow the tokens to be reset.</font>

If you wish to install without troubleshooting:  
Do not unpack the tar.gz. Just double-click on the file. 
* browse to: /lumada-dataops-suite/charts/data-transformation-editor-0.9.5.tgz/templates
* double-click on: rabac.yml
* change:  <font color='teal'>automountServiceAccountToken:</font><font color='red'> false</font> to <font color='green'> true</font>
* update & save

---

#### <font color='red'>Troubleshooting DTE Pod</font>
So where do you start..?

* Take a look at the Pod status

``list the Pods in hitachi-solutions namespace:``
```
kgp -n hitachi-solutions
```
As suspected the status of data-transformation-editor pod:  CrashLoopBackOff

If the container can't start, then Kubernetes shows the CrashLoopBackOff message as a status.
Usually, a container can't start when:
* There's an error in the application that prevents it from starting.
* The container is misconfigured.
* The Liveness probe failed too many times.
You should try and retrieve the logs from that container to investigate why it failed.  

``check the Pod:``
```
kdpo data-transformation-editor-xxxx -n hitachi-solutions
```
Note: looks like theres an issue with the istio-proxy.  

``list containers:``
```
kgpo data-transformation-editor-xxxx -n hitachi-solutions -o jsonpath='{.spec.containers[*].name}'
```
``check the log for data-transformation-editor istio-proxy:``
```
kubectl logs data-transformation-editor-xxxx istio-proxy -n hitachi-solutions
```
Note: ``fatal	Missing JWT, can't authenticate with control plane.``  The problem is authentication..  so take a look at the templates/RBAC.yml

<font color='teal'>automountServiceAccountToken:</font><font color='red'> false</font>

You can access the API from inside a Pod using automatically mounted service account credentials, as described in Accessing the Cluster. The API permissions of the service account depend on the authorization plugin (JWT) and policy in use.

In version 1.6+, you can opt out of automounting API credentials for a service account by setting: <font color='teal'>automountServiceAccountToken:</font><font color='red'> false</font> on the service account:

Solution: 
* open:  /lumada-dataops-suite/charts/data-transformation-editor-0.9.5.tgz
* edit templates/RBAC.yml - <font color='teal'>automountServiceAccountToken:</font> <font color='green'>true </font>  
* uninstall current data-transformation-editor
* re-run playbook

``verfify solution packages:``
```
kubectl get solutionpackage -n hitachi-solutions
```

Please refer to documentation.  You may want to remove .ansible/tmp folder to free up space and ensure a clean installation.

---

