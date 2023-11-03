# Upgrade IBM Maximo to MAS on VMware vSphere - Lessons Learned

With the [September 2025 end of support for Maximo 7.6.1](https://www.ibm.com/support/pages/end-support-announcement-eos-maximo-761) timeline approaching, IBM Maximo customers are interested to learn what is required to upgrade their 7.6.1.x environments, many deployed and hosted on-premises, to the new releases, Maximo Application Suite (MAS). Some customers take one step further by setting up a standalone environment to evaluate the new infrastructure and of course the MAS out of the box functionality. 

IBM offers several MAS deployment options that customers can choose to meet their business requirements - on-premises (customer managed), public clouds or hyperscalers (customer managed), SaaS (IBM managed) and dedicated (IBM managed). 

From the infrastructure perspective, MAS requires Red Hat OpenShift, which is a hybrid cloud application platform powered by Kubernetes. Depending on the deployment option, either customers or IBM manage OpenShift and MAS deployment and provide ongoing support.

This document covers MAS upgrade for a customer running Maximo 7.6.1.x on premises in VMware vSphere, in particular, the image registry issue related to OpenShift storage, the network slowness issue related to OpenShift network type, and the login issue related to MAS Manage activation with industry solutions and add-ons. 

Note that our test environment is based on Red Hat OpenShift version 4.12, MAS core 8.11 and MAS Manage 8.7, and the Oracle database, version 19c, that is hosted separated from the OpenShift cluster. All master and worker nodes, plus the Bastion machine, are VMs hosted in vSphere.

## Install OpenShift with Customization

Red Hat has provided detailed documents on how to [install an OpenShift cluster on vSphere with customizations](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/installing-vsphere-installer-provisioned-customizations.html). 

There are four [OpenShift installation](https://docs.openshift.com/container-platform/4.12/installing/index.html) options - interactive, local agent-based, automated or installed provisioned infrastructure (IPI), and full control or user provisioned infrastructure (UPI). When possible, IPI is recommended.

Alternatively, use tools such as [Daffy](https://ibm.github.io/daffy/Deploying-OCP/), an IBM open source project. 

In our project, we download the OpenShift installer from the Red Hat portal, create a configuration file, custom the configuration, and then install the OpenShift cluster. For more details, check [Preparing to install on vSphere](https://docs.openshift.com/container-platform/4.12/installing/installing_vsphere/preparing-to-install-on-vsphere.html) and related documents. 

It's worth noting that you can run the command below to delete an existing OpenShift cluster. Additional cleanup may be required on vSphere, for example,  release the IP addresses consumed by VMs.

```
./openshift-install destroy cluster --dir <installation_directory> --log-level info 
```

### Worker node customization

By default, 3 master nodes and 3 worker nodes are created. However, you can change the number of worker nodes, and specify cpu and memory for each worker node in the install-config.yaml file.

### Network customization

For networking, you can change the machineNetwork cidr value so that all master nodes and worker nodes obtain IP addresses from DHCP and are accessible from your local network.

By default, network type "OVNKubernetes" is used. We change it to "OpenShiftSDN", to optimize the network performance in the test environment. More details to follow later in the document.

### Sample install-config.yaml file

```
additionalTrustBundlePolicy: Proxyonly
apiVersion: v1
baseDomain: xxx.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: 
    vsphere:
        cpus: 16
        coresPerSocket: 4
        memoryMB: 65536  
        osDisk: 
            diskSizeGB: 180    
  replicas: 2    
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform: {}
  replicas: 3
metadata:
  creationTimestamp: null
  name: xxx
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 192.168.1.0/24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  vsphere:
    apiVIPs:
    - 192.168.1.101
    cluster: xxx
    datacenter: xxx
    defaultDatastore: xxx
    ingressVIPs:
    - 192.168.1.102
    network: xxx
    password: xxx
    username: xxx
    vCenter: xxx
publish: External
pullSecret: 'xxx'

```

## Install Maximo Application Suite (MAS) and Activate MAS Manage

In our project we install MAS by running Ansible playbooks in a Docker container session on a Bastion machine. For more details, check [Install Maximo On Microsoft Azure and Amazon AWS](https://github.com/zxue/install-maximo-on-azure-and-aws)

By default, only two storage classes, "thin-csi" and "thin", are available in the OpenShift cluster on vSphere. We make a few changes to the environment variables, and specify the storage class, "thin-csi", for several services.

```
export MAS_INSTANCE_ID=poc1
export IBM_ENTITLEMENT_KEY=xxx
export MAS_CONFIG_DIR=/mascli/masconfig
export SLS_LICENSE_ID=xxx
export SLS_LICENSE_FILE=/mascli/masconfig/license.dat
export UDS_CONTACT_EMAIL=xxx@xxx.com
export UDS_CONTACT_FIRSTNAME=firstname
export UDS_CONTACT_LASTNAME=lastname
export MAS_APP_ID=manage 
# Specify storage class
export PROMETHEUS_STORAGE_CLASS = thin-csi
export PROMETHEUS_USERWORKLOAD_STORAGE_CLASS= thin-csi
export GRAFANA_INSTANCE_STORAGE_CLASS= thin-csi
export MONGODB_STORAGE_CLASS= thin-csi
export UDS_STORAGE_CLASS= thin-csi
export PROMETHEUS_ALERTMGR_STORAGE_CLASS=thin-csi
```

It's recommended that you go with automatic approval of channel subscription during MAS Manage activation, unless you need to stay with a specific version. This ensures that you get a latest version of MAS Manage. You can change the setting later from the MAS admin portal.

After successful installation of MAS core and MAS Manage activation, you can see a number of pods created in OpenShift.

Pods in MAS core namespace

```
NAME                                                     READY   STATUS      RESTARTS   AGE
ibm-mas-operator-5fc7f5df55-hptst                        1/1     Running     0          24h
ibm-truststore-mgr-controller-manager-5c67c5f8fb-gghp2   1/1     Running     0          17d
poc1-accapppoints-28317180-2t67c                         0/1     Completed   0          24m
poc1-admin-dashboard-749f4cbf48-v6ft8                    1/1     Running     0          24h
poc1-adoptionusage-reporter-28316281-rsqzv               0/1     Completed   0          15h
poc1-adoptionusageapi-864569c986-4jmtb                   1/1     Running     0          24h
poc1-catalogapi-6fddd76485-sxtzh                         1/1     Running     0          24h
poc1-catalogmgr-85ff5b95dd-cvj86                         1/1     Running     0          24h
poc1-coreapi-74b448f948-hv2mw                            1/1     Running     0          24h
poc1-coreapi-74b448f948-lhwfw                            1/1     Running     0          24h
poc1-coreapi-74b448f948-mqx7n                            1/1     Running     0          24h
poc1-coreidp-86895cffcf-bk8kl                            1/1     Running     0          24h
poc1-coreidp-login-7ccf687bc9-rmgtz                      1/1     Running     0          24h
poc1-coreidp-truststore-worker-jhkh6                     0/1     Completed   0          17d
poc1-entitymgr-addons-6d5bcdcf58-g9kgm                   1/1     Running     0          24h
poc1-entitymgr-bascfg-59c44c6698-ckr8w                   1/1     Running     0          24h
poc1-entitymgr-coreidp-6fb4864d97-6btvc                  1/1     Running     0          24h
poc1-entitymgr-idpcfg-5d9cbcf664-p5cb5                   1/1     Running     0          24h
poc1-entitymgr-jdbccfg-5cdd66496f-twb9q                  1/1     Running     0          24h
poc1-entitymgr-kafkacfg-5f79567cc6-8qvsn                 1/1     Running     0          24h
poc1-entitymgr-mongocfg-6c9f9949ff-8zwwn                 1/1     Running     0          24h
poc1-entitymgr-objectstorage-79dd66bccf-4wvjm            1/1     Running     0          24h
poc1-entitymgr-pushnotificationcfg-6bbd444f5d-n64dw      1/1     Running     0          24h
poc1-entitymgr-scimcfg-7bdbd4cf8d-l9x2s                  1/1     Running     0          24h
poc1-entitymgr-slscfg-fdf49ff5-8mcxv                     1/1     Running     0          24h
poc1-entitymgr-smtpcfg-7c6fdc46d9-6k8r6                  1/1     Running     0          24h
poc1-entitymgr-suite-fd5db7c48-jbpkq                     1/1     Running     0          24h
poc1-entitymgr-watsonstudiocfg-6d7d77b44-28mxs           1/1     Running     0          24h
poc1-entitymgr-ws-5fb96655d4-84nxd                       1/1     Running     0          24h
poc1-groupsync-coordinator-84894d45df-7mdxn              1/1     Running     0          24h
poc1-homepage-868487dd9b-pb2zz                           1/1     Running     0          24h
poc1-internalapi-79c6654795-2b8jc                        1/1     Running     0          24h
poc1-licensing-mediator-55cf7cf589-n6g99                 1/1     Running     0          24h
poc1-ltpakeygenerator-rfx5t                              0/1     Completed   0          17d
poc1-milestonesapi-7f94fdd7d9-s67wr                      1/1     Running     0          24h
poc1-mobileapi-56d559fbfc-c5txn                          1/1     Running     0          24h
poc1-monagent-mas-69df65b74b-7jhjh                       1/1     Running     0          24h
poc1-navigator-57bb547cbc-tnb7j                          1/1     Running     0          24h
poc1-oidcclientreg-cxlvk                                 0/1     Completed   0          17d
poc1-truststore-worker-j6r55                             0/1     Completed   0          17d
poc1-usage-daily-28317200-q5br8                          0/1     Completed   0          4m15s
poc1-usage-historical-28317150-mhn9q                     0/1     Completed   0          54m
poc1-usage-hourly-28317190-rvl2w                         0/1     Completed   0          14m
poc1-usersync-coordinator-76f8f9cd58-ww8gs               1/1     Running     0          24h
poc1-workspace-coordinator-b9fbd48c4-rxbtq               1/1     Running     0          24h
```

Pods in MAS Manage namespace

```
NAME                                                     READY   STATUS      RESTARTS   AGE
admin-build-config-1-build                               0/1     Completed   0          17d
admin-build-config-2-build                               0/1     Completed   0          24h
all-build-config-1-build                                 0/1     Completed   0          17d
all-build-config-2-build                                 0/1     Completed   0          23h
ibm-mas-imagestitching-operator-6467757856-xgzpf         1/1     Running     0          17d
ibm-mas-manage-operator-55b48cd6d5-26lzs                 2/2     Running     0          24h
ibm-truststore-mgr-controller-manager-779cfbf557-8q47l   1/1     Running     0          17d
poc1-entitymgr-appstatus-7ffcfc46c7-k9q8k                1/1     Running     0          24h
poc1-entitymgr-bdi-5bcffb647d-xnh4n                      1/1     Running     0          24h
poc1-entitymgr-primary-entity-5fd75f4759-9f6fp           1/1     Running     0          24h
poc1-entitymgr-ws-599475ff89-bt9fs                       1/1     Running     0          24h
poc1-groupsyncagent-76bcd87847-kmzfx                     1/1     Running     0          17d
poc1-healthext-entitymgr-ws-5868c5ddf-zsm7k              1/1     Running     0          24h
poc1-masdev-all-8f99964c5-nnjnt                          2/2     Running     0          23h
poc1-masdev-manage-maxinst-77d77f548b-dlksk              1/1     Running     0          23h
poc1-masdev-truststore-worker-dgqdm                      0/1     Completed   0          17d
poc1-monitoragent-5d7d9989d9-8m6wc                       1/1     Running     0          24h
poc1-usersyncagent-689458968-qm47s                       1/1     Running     0          17d
```

## Troubleshoot OpenShift and Maximo Issues

We encounter and resolve a few technical issues in the project, one related to image registry, one related to network performance and one related to MAS Manage activation with industry solutions and add-ons.

Check out the details on [Monitoring Maximo Manage activation by using the Maximo Application Suite interface](https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=manage-monitoring-maximo-activation)

### OpenShift image registry issue

During MAS Manage activation, we notice the error from the log of the admin-build-config pod. 

```
Warning: Push failed, retrying in 5s ...
Registry server Address:
Registry server User Name: serviceaccount
Registry server Email: serviceaccount@example.org
Registry server Password: <<non-empty>>
error: build error: Failed to push image: trying to reuse blob sha256:5d39cd8b7ac1444c015a4531ff371860726ed47224dd5de877429ca388214019 at destination: pinging container registry image-registry.openshift-image-registry.svc:5000: Get "https://image-registry.openshift-image-registry.svc:5000/v2/": dial tcp 172.30.154.142:5000: connect: connection refused
```

Upon further investigation, we find that only one deployment, "image-registry", is available in the openshift-image-registry namespace in OpenShift. The "cluster-image-registry-operator" deployment is missing. The root cause of the issue is that MAS requires a "read write many" (RWX) storage (or PVC volume) which the storage class "thin" does not support. As noted in the OpenShift documentation, "The Image Registry Operator is not initially available for platforms that do not provide default storage. After installation, you must configure your registry to use storage so that the Registry Operator is made available."

The workaround is to change the storage requirement to "read write once" (RWO). This should not be an issue for MAS since only RWX is only required for documents, which we do not use in the project. To address the issue, we take the following steps:

- fix the image registry storage issue by issuing the patch command: 
```
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
```

- change managementState Image Registry Operator configuration from Removed to Managed.
```
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

- change registry configuration by specifying pvc and claim. More details on [Configuring registry storage for VMware vSphere](https://docs.openshift.com/container-platform/4.12/registry/configuring_registry_storage/configuring-registry-storage-vsphere.html#configuring-registry-storage-vsphere)
```
oc edit configs.imageregistry.operator.openshift.io
```

The yaml file looks like
```
storage:
  pvc:
    claim: 
```

You can verify that the claim value, which is updated to "image-registry-storage" automatically, from the "cluster" instance of the "config" custom resource in the imageregistry.operator.openshift.io group. 

- Delete the pvc named "image-registry-storage" and re-create it manually from the OpenShift admin portal, using the "thin" storage class, with 100GB storage size.

The pvc should be bounded immediately, and the image push failure should be resolved.

### Network slowness issue

For reasons initially unknown to us, MAS Manage activation takes a long time and does not complete as expected. On the MAS admin portal, we see errors with DeploymentCR or "Deployment Ready". On the OpenShift admin portal, we see errors with the  masdev-manage-maxinst pod in the MAS Manage instance namespace. 
 
We notice "Oslc servlet.... Server did not start yet. Will wait 30 sec." in the log. Upon further investigation, we discover that the query that fetches records from the maxattribute table, `select * from maximo.maxattribute`, takes longer time than expected. 

By running an internal database connection testing tool, we find that it takes 20+ seconds to fetch 100 records from the maxinst pod to the Oracle database. Interestingly, the select query runs much faster on the Bastion machine: it takes less one second, which are expected network speeds. So the question is, why it takes so much time for the query to reach the database and return results to the pod?

We start looking at our OpenShift network settings. We suspect that the OpenShift nodes may be getting to the database host using a different networking layer or route, which slows down the database communications. By default, the networktype is set to OVNKubernetes in the install-config.yaml file. We decide to re-build the OpenShift cluster using the networktype, "OpenShiftSDN". The change is effective, and the MAS Manage action completes within less than 3 hours.

It is worth noting that you can change networktype using the patch command, without re-building the cluster.

```
oc patch network.operator cluster -p '{"spec": {"defaultNetwork": {"ovnKubernetesConfig": {"gatewayConfig": {"routingViaHost": true}}}}}' --type merge
```

The patch command ensures that the packets leaving the pods use the same routing tables as the host VM that they are on by setting this value. With OpenShiftSDN networktype, this works automatically. For more details, check [Cluster Network Operator in OpenShift Container Platform](https://docs.openshift.com/container-platform/4.12/networking/cluster-network-operator.html)

### Database encryption issue and re-activation

Users are not expected to have any database encryption issues during MAS upgrade. However, if some properties in Maximo 7.6.1.x and prior versions are encrypted using custom encryption keys instead of system generated keys, or if the database upgrade is completed but MAS Manage activation does not complete, the the encryption keys and encryption algorithms must be specified. 

For more details, check [Database encryption scenarios](https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=de-database-encryption-scenarios)

To provide database encryption keys, select MAS manage. Select "Update configuration" and "Advanced Settings". 

- Deselect "System managed" in the Database section. 
- Specify schema, table space and index space to match the values in the existing Maximo database.
- Specify two properties, MXE_SECURITY_OLD_CRYPTO_KEY and MXE_SECURITY_OLD_CRYPTOX_KEY. Their values are custom keys used in Maximo 7.6.1.x, or values from two properties stored in MXE_SECURITY_CRYPTOX_KEY and MXE_SECURITY_CRYPTO_KEY from a previous MAS upgrade. The reason is that the some Maximo properties such as the password fields have been re-encrypted by the encryptions. 
- Specify MXE_SECURITY_CRYPTOX_KEY and MXE_SECURITY_CRYPTO_KEY and their values. They can be set to the same values as the old encryption keys accordingly, or different values than the old encryptions. For the latter option, the table properties are decrypted with the old keys and re-encrypted with the new keys.

You can verify the maximo properties from the terminal session of the maxinst pod. The maximo.propeties file is stored in the  `/opt/IBM/SMP/maximo/applications/maximo/properties` folder.

```
# cat maximo.properties

mxe.name=MXServer
mxe.db.url=xxx
mxe.db.driver=xxx
mxe.db.user=xxx
mxe.db.password=xxx
mxe.db.schemaowner=maximo
mxe.security.crypto.key=HkckmIQmNQWbjNKVJZtJJHjR
mxe.security.cryptox.key=PSdPefRQuwNSTfSDFiwJMmkU
mxe.security.old.crypto.key=
mxe.security.old.cryptox.key=
```

After updating encryption keys from the MAS admin portal, you can re-activate MAS Manage. Before re-activation, delete the secret named `<mas workspace e.g. masdev>-manage-encryptionsecret-operator` in the MAS Manage namespace. This ensures that the changes are reloaded during activation.

### MAS admin log in failure

If some industry solutions and add-ons are enabled in Maximo, they must be activated or enabled when MAS Manage is activated, or afterwards. Otherwise, MAS admin login fails with the following message.

```
The app has experienced a problem that is preventing it from loading/rendering.

A major exception has occurred. Check the system log to see if there are any companion errors logged. Report this error to your system administrator.

...
```

### Deactivate MAS Manage

You never deactivate your production Manage system once activated. However, for dev and test environment, you may need to deactivate MAS Manage.

Deactivate means uninstall the Manage system, including configuration settings, in the MAS workspace. Before you deactivate a system, make a copy of the encryption keys. Once deactivated, the encryption keys will be removed. If you cannot recover the encryption keys used for your database, the database cannot be used by Maximo system anymore.

While MAS Manage activation is still going, the Deactivate command from the MAS admin portal may not be responsive. To force deactivation manually, find the instance in the ManageWorkspace custom resource. Open the yaml file and remove the following lines.

```
  finalizers:
    - manageworkspace.apps.mas.ibm.com/finalizer
```

### Uninstall MAS core and delete MAS namespace in OpenShift

You can run an Ansibile playbook to remove MAS. It is suggested that you delete the MAS Manage project or namespace and load the environment variable specifying the MAS instance, before running the playbook. Uninstalling MAS core takes a few minutes.

```
#docker run -ti --rm --pull always -v ~/masconfig:/mascli/masconfig quay.io/ibmmas/cli
#ansible-playbook ibm.mas_devops.oneclick_core
ansible-playbook ibm.mas_devops.uninstall_core
```

Note that you may find that deleting a MAS Manage project changes its status to "Terminating" indefinitely, and `oc delete project dev --force --grace-period=0` does not completely delete a project. Take the following steps.
- Make sure that all MAS Manage CRs including ManageApp, ManageBuild, ManageDeployment, ManageOfflineUpdateRequest, ManageServerBundle, ManageStatusCchecker,ManageWorkspace are deleted.
- oc login and run the command, `kill-ns mas-proc-manage` 

## Project Team from IBM

The following people from IBM contributed to the project.

- Ahmed AlSadiq, Technology Engineer, IBM
- Alex Cravalho, Technology Engineer, IBM
- Ryan Menossi, Solution Architect, IBM
- Benjamin Xue,	Solution Architect, IBM

Special thanks to Brian Zhu, Senior Software Engineer, Maximo, IBM, Jenny Wang, Technical Architect, Maximo, IBM, and our Red Hat friend, Cedric Anderson, for their deep expertise on Red Hat OpenShift and IBM Maximo and guidance on how to troubleshoot several critical issues.

