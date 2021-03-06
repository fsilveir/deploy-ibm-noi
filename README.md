# Deploy IBM Netcool Operations Insight

## Overview

The following steps will cover how to install IBM Netcool Operations Insight (NOI) on IBM Cloud Private (ICP) Community Edition on SoftLayer VM's deployed with the Ansible playbooks provided by IBM.

The following will be covered by this guide:

- What is ICP / NOI?
- Installing ICP CE (v3.1.1) on SoftLayer VM's with Ansible
- Deploying Heketi GlusterFS on ICP 3.1.1
- Installing NOI

### What is IBM Cloud Private?

More information can be found at the following:

- [IBM Cloud Private Overview](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.1/getting_started/introduction.html)
- [IBM Cloud Private Deployment GitHub Repository](https://github.com/IBM/deploy-ibm-cloud-private/)

### What is Netcool Operations Insight (NOI)?

More information can be found at the following:

- [Netcool Operations Insight - Product Overview](https://www-01.ibm.com/common/ssi/ShowDoc.wss?docURL=/common/ssi/rep_ca/9/897/ENUS217-009/index.html&lang=en&request_locale=en)
- [Netcool Operations Insight - Demo](https://www.ibm.com/cloud/garage/dte/producttour/ibm-netcool-operations-insight)

### How to Deploy ICP?

In case you don't have ICP already deployed, there are multiple automated ways of doing so, please check the following repository for instructions:

- [IBM Cloud Private Deployment GitHub Repository](https://github.com/IBM/deploy-ibm-cloud-private/)

In order to simplify the instructions of this guide, we'll consider the Ansible playbooks as base for the instructions, however the same knowledge can be adapted for other solutions as well.

We won't reproduce the exact deployment steps from the deployment repository, instead we'll try making a few remarks on points that might be more prone to errors and are relevant for deploying Netcool Operations Insight (NOI) and GlusterFS, such as hardware requirements.

The first thing you need to do is to clone this the IBM deployment repository, set up to use SoftLayer CLI and Ansible, by executing the following:

```bash
$ git clone https://github.com/IBM/deploy-ibm-cloud-private.git
$ cd deploy-ibm-cloud-private
$ sudo pip install -r requirements.txt
```
The deployment repository will provide you basically with 2 Ansible playbooks:

- [create_sl_vms.yml](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/playbooks/create_sl_vms.yml): To create the VM's on SoftLayer according to the specifications defined on `clusters/config.yaml` file.
- [prepare_sl_vms.yml](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/playbooks/prepare_sl_vms.yml): Prepare and install required packages for ICP

Please take the requirements listed below in consideration before starting ICP deployment:

#### Hardware Requirements

Before starting, please get familiar with the ICP requirements for deploying NOI 1.5.0 on ICP Community Edition 3.1.1:

- [Netcool Operations Insight 1.5.0 requirements for an Installation on IBM Cloud Private](https://www.ibm.com/support/knowledgecenter/en/SSTPTP_1.5.0/com.ibm.netcool_ops.doc/soc/integration/reference/soc_int_reqsforcloudinstallation.html)

On this document we're suggesting the following architecture: 

- 1 Master/Management/Boot Node
- 1 Proxy Node
- 3 Worker Nodes 

_Note: The number of workers is specially important, since we'll use Heketi Chart for GlusterFS storage -- which requires a minimum of 3 worker nodes to function._

You can use the following definitions as a template for your master node, by updating the following at your local copy of the [create_sl_vms.yml](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/playbooks/create_sl_vms.yml) playbook:

Since NOI takes a reasonable amount of disk, its important that you increase the default settings provided in the playbook to meet the requirements. 

Originally, the idea was to have one big disk and slice it with multiple filesystems, which we soon discovered would not be possible without using a custom template/image for the deployment.

When creating VM's with IBM's Ansible playbook, you'll have to provision the VM's with the exact disk sizes as defined at the SoftLayer template/catalog. What this means in practical terms is:

**Primary disk can have from 25 GB to 100 GB, and additional disks may have from 25 GB to 2 TB, however the values should match the catalog available sizes, meaning if you try creating a VM with 275 GB, your provisioning will fail because this disk size is not available in the catalog.**

Check the documentation below for a reference table with the sizes of disk currently supported:

- [IBM Cloud Provider (Argument Reference)](https://console.bluemix.net/docs/infrastructure/BlockStorage/index.html#provisioning-with-performance)


For a complete reference on the available arguments, check the following links:

- [IBM Cloud Infrastructure (SoftLayer) API docs](http://sldn.softlayer.com/reference/services/SoftLayer_Virtual_Guest)
- [IBM Cloud Provider (Argument Reference)](https://ibm-cloud.github.io/tf-ibm-docs/v0.11.1/r/compute_vm_instance.html)

##### > Master Node Requirements

Here are some hardware definitions for the master Node to serve as reference/example when creating your ICP environment, nas noted below we've used a bit more than the minimum hardware recommended by NOI documentation, and decided to use 8 CPU's with 32 GB of Memory and 500 GB of disk:

```yaml
    - name: create master
      sl_vm:
        hostname: "{{ item }}"
        domain: ibm.local
        datacenter: "{{ sl_datacenter|default('ams01') }}"
        tags: "{{ item }},icp,master"
        hourly: False
        private: False
        dedicated: False
        local_disk: False
        cpus: 8
        memory: 32768
        disks: ['100','400']
        os_code: UBUNTU_LATEST
        private_vlan: "{{ sl_vlan|default(omit) }}"
        wait: no
        ssh_keys: "{{ sl_ssh_key|default(omit) }}"
      with_items: " {{ sl_icp_masters }}"
```


##### > Worker Nodes Requirements

We also had performance problems when trying to deploy worker nodes with less than 32 GB of memory, therefore we've repeated the same settings with an additional disk of 200 GB to be used by GlusterFS.

```yaml
    - name: create workers
      sl_vm:
        hostname: "{{ item }}"
        domain: ibm.local
        datacenter: "{{ sl_datacenter|default('ams01') }}"
        tags: "{{ item }},icp,worker"
        hourly: False
        private: False
        dedicated: False
        local_disk: False
        cpus: 8
        memory: 32768
        disks: ['100','200']
        os_code: UBUNTU_LATEST
        private_vlan: "{{ sl_vlan|default(omit) }}"
        wait: no
        ssh_keys: "{{ sl_ssh_key|default(omit) }}"
      with_items: " {{ sl_icp_workers }}"

```

##### > Proxy Requirements

For the proxy, we used 4 CPU's, with 8 GB of memory and a single disk of 100 GB.

```yaml
    - name: create proxies
      sl_vm:
        hostname: "{{ item }}"
        domain: ibm.local
        datacenter: "{{ sl_datacenter|default('ams01') }}"
        tags: "{{ item }},icp,proxy"
        hourly: False
        private: False
        dedicated: False
        local_disk: False
        cpus: 4
        memory: 8192
        disks: ['100']
        os_code: UBUNTU_LATEST
        private_vlan: "{{ sl_vlan|default(omit) }}"
        wait: no
        ssh_keys: "{{ sl_ssh_key|default(omit) }}"
      with_items: " {{ sl_icp_proxies }}"

```

##### > Choosing the latest ICP Release

In order to use the latest ICP version (3.1.1) you must also update the image tag at the [DockerFile](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/cluster/Dockerfile), which currently uses `2.1.0` by default, as shown right below:

```yaml
FROM ibmcom/icp-inception:3.1.1 
RUN apk add --update python python-dev py-pip &&\
    pip install SoftLayer
```

##### > Deploying ICP

As mentioned by IBM's deployment guide, if a similar message is displayed, it means you have successfully deployed ICP with the Ansible Playbook:

```
$ ssh root@169.46.198.XXX docker run -e SL_USERNAME=XXXXX -e SL_API_KEY=YYYY -e LICENSE=accept --net=host \
   --rm -t -v /root/cluster:/installer/cluster icp-on-sl install
...
...
PLAY RECAP *********************************************************************
169.46.198.XXX             : ok=44   changed=22   unreachable=0    failed=0   
169.46.198.YYY             : ok=69   changed=39   unreachable=0    failed=0   
169.46.198.ZZZ             : ok=45   changed=22   unreachable=0    failed=0   


POST DEPLOY MESSAGE ************************************************************

UI URL is https://169.46.198.XXX:8443 , default username/password is admin/admin
```

See [Accessing IBM Cloud Private](https://github.com/IBM/deploy-ibm-cloud-private/blob/master/README.md#accessing-ibm-cloud-private) for more details on how access the tool, and make sure you [change the super administrator password](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.1/user_management/change_admin_passwd.html) as soon as possible.


### How to Deploy GlusterFS on your ICP Cluster?

As noted at the documentation, GlusterFS is the recommended storage of choice when deploying NOI on ICP. We've used [IBM GlusterFS Storage Cluster chart](https://github.com/IBM/charts/tree/master/stable/ibm-glusterfs) for this implementation.

To start the install process, access the ICP UI URL, (https://192.168.0.XXX:8443), select the Catalog, and them search for Heketi GlusterFS chart provided by IBM for the install.

We won't reproduce the exact deployment steps from the chart repository, instead we'll try making a few remarks regarding some of points that might be more prone to errors and are relevant for Netcool Operations Insight (NOI) deployment.

Though the complete list of requirements can be found at the [Heketi/GlusterFS Chart repository](https://github.com/IBM/charts/tree/master/stable/ibm-glusterfs), the following should be highlighted before starting the deployment:

- The Heketi GlusterFS Chart provided by IBM **will only work with IBM Cloud Private Version 3.1.0 or later**.
- You must use **at least three storage nodes to configure GlusterFS storage cluster**.
- **The storage devices that you use for GlusterFS must be raw disks. They must not be formatted, partitioned, or used for file system storage needs**. You must use the symbolic link (symlink) to identify the GlusterFS storage device (and if using VSI's or SUSE Linux, you must take some additional steps, as described at the next section).
- The Heketi GlusterFS Chart must be deployed on `kube-system` namespace.
- **Your cluster default storageClass must be GlusterFS in order for the NOI deployment to work**, set the parameter `storageClass.isDefault` to `true` during GlusterFS deployment.
- You must also define before the deployment which **ReclaimPolicy** will be used on your default storageClass, For GlusterFS, the accepted values include `Retain`, and `Delete`. The volume reclaim policy `Retain` indicates that the volume will be preserved after the pods accessing it terminates.

#### Preparing the Disks on SoftLayer VSI's

In some environments, such as IBM Cloud VSI or SUSE Linux Enterprise Server (SLES), no symlinks are automatically generated for the devices. You must manually create symlinks by writing custom udev (userspace /dev) rules. When you create the symlink, use attributes that are unique to the device, check the following page for the complete steps:

- [Preparing the disks (Use manually created symlinks)](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.1/manage_cluster/prepare_disks.html)

#### Increasing Gluster CPU & Memory Limits

We've experienced some performance issues when using IBM's recommended requirements, therefore we've had better results when using the following requirements for the Gluster configuration parameters:

##### > Gluster Configuration Parameters

```yaml
###############################################################################
## Gluster Configuration Parameters
###############################################################################
gluster:
  image:
    repository: "ibmcom/gluster"
    tag: "v4.0.2"
    pullPolicy: "IfNotPresent"
  installType: "Fresh"
  resources:
    requests:
      cpu: "1000m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "2Gi"
```

#### Confirm if the chart was successfully deployed

If all the steps were followed accordingly you should see a similar output after starting the deployment:

```bash
[fsilveir@fsilveir ibm-glusterfs]$ helm install --name icp-gluster --namespace kube-system -f values.yaml . --tls
NAME:   icp-gluster
E1204 17:04:31.531695    8543 portforward.go:303] error copying from remote stream to local connection: readfrom tcp4 127.0.0.1:43451->127.0.0.1:41224: write tcp4 127.0.0.1:43451->127.0.0.1:41224: write: broken pipe
LAST DEPLOYED: Tue Dec  4 17:04:21 2018
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                                   DATA  AGE
icp-gluster-glusterfs-heketi-config    1     6s
icp-gluster-glusterfs-heketi-topology  3     6s

==> v1/Service
NAME                                  TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)   AGE
icp-gluster-glusterfs-heketi-service  ClusterIP  192.168.0.22  <none>       8080/TCP  6s

==> v1beta2/DaemonSet
NAME                             DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR          AGE
icp-gluster-glusterfs-daemonset  3        3        0      3           0          storagenode=glusterfs  6s

==> v1beta2/Deployment
NAME                                     DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
icp-gluster-glusterfs-heketi-deployment  1        1        1           0          6s

==> v1/Pod(related)
NAME                                                     READY  STATUS             RESTARTS  AGE
icp-gluster-glusterfs-daemonset-d7ngm                    0/1    ContainerCreating  0         6s
icp-gluster-glusterfs-daemonset-mhsfj                    0/1    ContainerCreating  0         6s
icp-gluster-glusterfs-daemonset-mhv85                    0/1    ContainerCreating  0         6s
icp-gluster-glusterfs-heketi-deployment-977f486d7-6r2tk  0/1    Init:0/2           0         6s


NOTES:
Installation of GlusterFS is successful.

1. Heketi Service is created

   kubectl --namespace kube-system get service -l  glusterfs=heketi-service

2. Heketi Deployment is successful

   kubectl --namespace kube-system get deployment -l  glusterfs=heketi-deployment

3. Storage class glusterfs can be used to create GlusterFS volumes.

   kubectl get storageclasses glusterfs

[fsilveir@fsilveir ibm-glusterfs]$ 
```

Wait for all the related pods to be on `Running` state before considering the deployment as successfull, you can follow the instruction notes displayed at the end of the deployment output summary to confirm the status of each component individually. 

In some cases, the `icp-gluster-glusterfs-heketi-deployment` pod takes too long to start, even after the daemonset pods are started, try delete the pod in this case and check if it them get's properly recreated, if it still fails troubleshoot it by checking the Deployment and ReplicaSet logs.


### Installing NOI

For the install to work, we'll need to perform the following steps:

1. Copying and loading the installation package to the ICP catalog.
2. Create Persistent Volumes & Claims for NOI.
3. Deploy the NOI chart from ICP Catalog.
4. Confirm that everything is running as expected.

#### 1. Loading the Installation Package to the catalog


Download or ask your IBM vendor to download the **IBM Netcool Operations Insight 1.5 Operations Management for IBM Cloud Private English (part number CNX04EN)** from one of the following links:

- [IBM Passport Advantage](http://www-01.ibm.com/software/howtobuy/passportadvantage/pao_customers.htm)
- [IBM Software Sellers Workplace - Software Downloads](http://w3.ibm.com/software/xl/download/ticket.do)

And perform the steps described below: 


##### 1.1. Perform login on ICP Master Node:

```bash
root@icp-master1:~# cloudctl login -a https://169.46.198.XXX:8443 --skip-ssl-validation 

Username> admin

Password> 
Authenticating...
OK

Targeted account mycluster Account (id-mycluster-account)

Select a namespace:
1. cert-manager
2. default
3. istio-system
4. kube-public
5. kube-system
6. platform
7. services
Enter a number> 2
Targeted namespace default

Configuring kubectl ...
Property "clusters.mycluster" unset.
Property "users.mycluster-user" unset.
Property "contexts.mycluster-context" unset.
Cluster "mycluster" set.
User "mycluster-user" set.
Context "mycluster-context" created.
Switched to context "mycluster-context".
OK

Configuring helm: /root/.helm
OK

```

##### 1.2. Perform Docker login on the ICP Master Node

In order to upload the installation package to the Docker repository, first you must perform Docker login, as shown below:

```bash
root@icp-master1:~# docker login mycluster.icp:8500
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
root@icp-master1:~#
```

For more instructions on how to configure the docker CLI before the login, check the following link:

- [Configuring Docker CLI](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_2.1.0/manage_images/configuring_docker_cli.html)

##### 1.3. Load the installation package (archive) to the Docker repository

Be sure you have set the context to `default` namespace with the following command before loading the installation package with the following command:

```bash
root@icp-master1:~# kubectl config set-context $(kubectl config current-context) --namespace default
Context "mycluster-context" modified.
```

Them load the installation package to the catalog as shown below:

```bash
root@icp-master1:~# cloudctl catalog load-archive --archive NOI_V1.5_OM_FOR_ICP.tar.gz
```

For more details, check the following page, ***but please be aware that the documentation is not updated, and is still informing to use `bx pr` utility instead of the new `cloudctl` utility***:

- [Loading the archive into IBM Cloud Private](https://www.ibm.com/support/knowledgecenter/en/SSTPTP_1.5.0/com.ibm.netcool_ops.doc/soc/integration/task/int_loading-into-icp.html)


#### 2. Creating Persistent Volumes & Claims

Before you can start the install, you must create the Persistent Volume and Volume Claims required by NOI deployment to be able to start. To do so copy the [noi-gluster-pv.yaml](./volume-claims/noi-gluster-pv.yaml) file to your master node and execute the following command:

```bash
root@icp-master1:~# kubectl create -f noi-gluster-pv.yaml
```

Them confirm if the Persistent volume was properly created:

```bash
root@icp-master1:~# kubectl get pv noi-gluster-pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
noi-gluster-pv   500Gi      RWX            Retain           Available                                      29d
```

If the PersistentVolume is displaying `Available` status, download the following Persistent Volume Claim configuration files to your Master node:

- [noi-db2ese-pvc.yaml](./volume-claims/noi-db2ese-pvc.yaml)
- [noi-impactgui-pvc.yaml](./volume-claims/noi-impactgui-pvc.yaml)
- [noi-ncoprimary-pvc.yaml](./volume-claims/noi-ncoprimary-pvc.yaml)
- [noi-ncobackup-pvc.yaml](./volume-claims/noi-ncobackup-pvc.yaml)
- [noi-nciprimary-pvc.yaml](./volume-claims/noi-nciprimary-pvc.yaml)
- [noi-ncibackup-pvc.yaml](./volume-claims/noi-ncibackup-pvc.yaml)
- [noi-openldap-pvc.yaml](./volume-claims/noi-openldap-pvc.yaml)
- [noi-scala-pvc.yaml](./volume-claims/noi-scala-pvc.yaml)
- [noi-share-pvc.yaml](./volume-claims/noi-share-pvc.yaml)


... Be sure you have set the context to `default` namespace with the following command:

```bash
root@icp-master1:~# kubectl config set-context $(kubectl config current-context) --namespace default
Context "mycluster-context" modified.
```

... Execute the following for each for of the downloaded Persistent Volume CLaim files:

```bash
root@icp-master1:~# kubectl create -f <pvc_file_name>
```

Confirm all Persistent Volume Claims have been properly created and are displaying `Bound` status, as shown below:

```bash
 root@icp-master1:~# kubectl get pvc
NAME                     STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
icp-noi-ncibackup-pvc    Bound     pvc-4b55e1fd-0541-11e9-9e37-066eda578dc1   5Gi        RWO            glusterfs      29d
icp-noi-nciprimary-pvc   Bound     pvc-4b564247-0541-11e9-9e37-066eda578dc1   5Gi        RWO            glusterfs      29d
icp-noi-share-pvc        Bound     pvc-4b56b36c-0541-11e9-9e37-066eda578dc1   1Gi        RWX            glusterfs      29d
noi-db2ese-pvc           Bound     pvc-c692b071-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-impactgui-pvc        Bound     pvc-c6ce096a-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-ncibackup-pvc        Bound     pvc-c7079387-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-nciprimary-pvc       Bound     pvc-c73db470-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-ncobackup-pvc        Bound     pvc-c770ecd3-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-ncoprimary-pvc       Bound     pvc-c7ac8a00-0531-11e9-9e37-066eda578dc1   5Gi        RWX            glusterfs      29d
noi-openldap-pvc         Bound     pvc-c7e1b4f4-0531-11e9-9e37-066eda578dc1   1Gi        RWX            glusterfs      29d
noi-scala-pvc            Bound     pvc-c8150d9e-0531-11e9-9e37-066eda578dc1   20Gi       RWX            glusterfs      29d
root@icp-master1:~#
```


#### 3. Deploying NOI from the ICP Catalog


After loading the installation package to your catalog, check the chart instructions for the complete step-by-step instructions for deployment:

- [ibm-netcool-prod-1.500.29](ibm-netcool-prod-1.500.29/README.md)


#### (Optional) Creating a Kubernetes secret for the Netcool/OMNIbus password

More details on the following link:

- https://www.ibm.com/support/knowledgecenter/en/SSTPTP_1.5.0/com.ibm.netcool_ops.doc/soc/integration/task/int_installingbaseprivatecloud-gui.html


#### 4. Confirming if NOI was Successfully Deployed


If your deployment was completed successfully, you should be able to access the URL's with its respective credentials:


- **WebGUI**: https://netcool.icp-noi.mycluster.icp/ibm/console (**credentials**: icpadmin/XXXXXXX)
- **WAS Console**: https://was.icp-noi.mycluster.icp/ibm/console (**credentials**: admin/XXXXXXX)
- **Log Analysis**: https://scala.icp-noi.mycluster.icp/Unity (**credentials**: unityadmin/XXXXXXX)
- **Impact GUI**: https://impact.icp-noi.mycluster.icp/ibm/console (**credentials**: impactadmin/XXXXXXX)

You might have to add the IP addresses displayed at the installation summary output to your local `/etc/hosts` in order to properly access the consoles, as shown below:


<img src="ibm-netcool-prod-1.500.29/image_webgui.png" width="95%" height="350"></td>

### Known Problems


#### Internal Service Error after loading the NOI installation package into the catalog.

Depending on your securty settings, you might receive the following **Internal service error** when trying to load the installation package:

```
Internal service error : rpc error: code = Unknown desc = release noi failed: Internal error occurred: admission webhook "trust.hooks.securityenforcement.admission.cloud.ibm.com" denied the request: Deny "mycluster.icp:8500/noi/ibmcom/netcool-db2ese-ee:1.4.1-16", trust is not supported for images from this registry: mycluster.icp:8500
```
To solve the problem, execute the following with the administrator user:

```
kubectl delete --ignore-not-found=true MutatingWebhookConfiguration image-admission-config
kubectl delete --ignore-not-found=true ValidatingWebhookConfiguration image-admission-config
```

Check the link below for more information:

- [Removing Container Image Security Enforcement](https://console.bluemix.net/docs/services/Registry/registry_security_enforce.html#remove)


#### "Failed to pull image" and "Error: ImagePullBackOff" when trying to install NOI

If the NOI installation images were not loaded to the `default` namespace, NOI won't be able to pull the individual images during the install process.

The best way to fix the issue is to remove from package from the catalog and load it again, however more information on possible ways to fix the issue can be found at the following (though the document below is related to a different tool, the same logic can be used to fix the problem when using NOI):

- [Install does not complete successfully on IBM Cloud Private (ICP)](https://www-01.ibm.com/support/docview.wss?uid=ibm10737075)
