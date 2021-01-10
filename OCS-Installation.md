
# OCS on Openshift Bare Metal and Troubleshooting for some Expected Problems
**Openshift Container Storage(OCS)**  is a software defined storage tool that allows us to provide **Persistent Storage** in Openshift Cluster.

- In this documentation OCS will be deployed in Cluster, so provide prerequisites down below.
- # Prerequisites
- At least there **worker node**.
- Add two new disks to each worker node. Two 50GB new disk is added each nodes in this example. These disks will serve as different components of OCS. These are;
  1. **OCS mon**
  2. **OCS osd**
> Two different disks are for different types of storage.
---
To use these disks in Openshift, **Local Volume** should be defined. These **Local Volumes** will serve for **OCS** to define software defined storages.
## [Defining Local Volume](https://docs.openshift.com/container-platform/4.6/storage/persistent_storage/persistent-storage-local.html)
Although **Local Volume** is **Red Hat's Operator**, it does not come with default. You need to install it.
- To do that you should first create **local-storage** namespace(project).
```bash
oc create ns local-storage
```
- Install Local Storage Operator from OperatorHub

![1](https://user-images.githubusercontent.com/59168275/104120231-3233be00-5346-11eb-961a-fe28a7f84297.png)
- Install it on **local-storage namespace(project)**

![2](https://user-images.githubusercontent.com/59168275/104120229-319b2780-5346-11eb-9379-c71eb3656f85.png)

- Check it is installed completely on **Installed Operator** tab.

![3](https://user-images.githubusercontent.com/59168275/104120228-31029100-5346-11eb-8145-5307e6efd29d.png)
- Check on each worker node that you have newly added disks as devices. In my case all of them were added as **/dev/sdb** and **/dev/sdc**
```bash
sudo fdisk -l | grep Disk
```
![4](https://user-images.githubusercontent.com/59168275/104120227-31029100-5346-11eb-935f-747e663e7c44.png)
- With these informations, YAML file for creating **Local Volume**'s can be created as down below.
  - In this YAML file, two disks will referance two **storageClass**es on **OCS Cluster**
- The hostnames of the worker nodes should be entered in YAML file as below.
```yaml
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-disks-mon"
  namespace: "local-storage" 
spec:
  nodeSelector: 
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-01 #worker node hostname should be entered
          - worker-02 #worker node hostname should be entered
          - worker-03 #worker node hostname should be entered
  storageClassDevices:
    - storageClassName: "local-sc-mon"
      volumeMode: Filesystem 
      fsType: xfs 
      devicePaths: 
        - /dev/sdb
---
apiVersion: "local.storage.openshift.io/v1"
kind: "LocalVolume"
metadata:
  name: "local-disks-osd"
  namespace: "local-storage" 
spec:
  nodeSelector: 
    nodeSelectorTerms:
    - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker-01 #worker node hostname should be entered
          - worker-02 #worker node hostname should be entered
          - worker-03 #worker node hostname should be entered
  storageClassDevices:
    - storageClassName: "local-sc-osd"
      volumeMode: Block
      devicePaths: 
        - /dev/sdc
```
- Apply the YAML file from **oc client** or **Openshift Console**.
```bash
oc apply -f localvolumes.yaml
```
## OCS Cluster Deployment
- To deploy it, **Openshift Container Storage Operator** should be installed first.
- In Openshift 4.3, there were prerequisities like below to fulfill, it might have changed with the version upgrades.
```yaml
cat <<EOF > rhocs-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    openshift.io/cluster-monitoring: "true"
  name: openshift-storage
spec: {}
EOF
```
```bash
oc create -f rhocs-namespace.yaml
```
```yaml
cat <<EOF > rhocs-operatorgroup.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-storage-operatorgroup
  namespace: openshift-storage
spec:
  serviceAccount:
    metadata:
      creationTimestamp: null
  targetNamespaces:
  - openshift-storage
EOF
```
```bash
oc create -f rhocs-operatorgroup.yaml
```
- After these prerequisites, OCS can be installed on **openshift-storage** namespace(project).

![6](https://user-images.githubusercontent.com/59168275/104120224-2f38cd80-5346-11eb-875b-5792910c99d0.png)

![7](https://user-images.githubusercontent.com/59168275/104120222-2e07a080-5346-11eb-84eb-c38e1218b5e3.png)
- After the installation is done, OCS Cluster can be deployed with YAML configuration.
- Before applying YAML configuration some labels should be added to each worker node to make it available for OCS Cluster use.
```bash
oc label node worker-01 cluster.ocs.openshift.io/openshift-storage="" && oc label node worker-01 topology.rook.io/rack="rack0"
oc label node worker-02 cluster.ocs.openshift.io/openshift-storage="" && oc label node worker-02 topology.rook.io/rack="rack1"
oc label node worker-03 cluster.ocs.openshift.io/openshift-storage="" && oc label node worker-03 topology.rook.io/rack="rack2"
```
- Check if they correctly labeled with the command.
```bash
oc get node -l  beta.kubernetes.io/arch=amd64 -o yaml | egrep 'kubernetes.io/hostname: worker|cluster.ocs.openshift.io/openshift-storage|topology.rook.io/rack'
```
```bash
      cluster.ocs.openshift.io/openshift-storage: ""
      kubernetes.io/hostname: worker-01
      topology.rook.io/rack: rack0

      cluster.ocs.openshift.io/openshift-storage: ""
      kubernetes.io/hostname: worker-02
      topology.rook.io/rack: rack1

      cluster.ocs.openshift.io/openshift-storage: ""
      kubernetes.io/hostname: worker-03
      topology.rook.io/rack: rack2
```
- After these YAML configurations can be prepared and applied to **cluster**.
```yaml


apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:    
  namespace: openshift-storage
  name: ocs-storagecluster
spec:
  manageNodes: false
  monPVCTemplate:
    spec:    
      storageClassName: local-sc-mon
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 50Gi
  resources:
    mon:
      requests: {}
      limits: {}
    mds:
      requests: {}
      limits: {}
    rgw:      
      requests: {}
      limits: {}
    mgr:
      requests: {}
      limits: {}
    noobaa-core:
      requests: {}
      limits: {}
    noobaa-db:
      requests: {}
      limits: {}
  storageDeviceSets:
  - name: ocs-deviceset
    count: 1
    resources: {}
    placement: {}
    dataPVCTemplate:
      spec:
        storageClassName: local-sc-osd
        accessModes:
        - ReadWriteOnce
        volumeMode: Block
        resources:
          requests:
            storage: 50Gi
    portable: true
    replica: 3
```
- After these, watch every component is deployed or not from Topology and Developer Tab section. And it is done.

![8](https://user-images.githubusercontent.com/59168275/104120234-33fd8180-5346-11eb-9e80-c18224c9dcda.png)
- **storageClass**es will be created without any warning. 

![9](https://user-images.githubusercontent.com/59168275/104120233-3364eb00-5346-11eb-80b5-e9c6b66bb0b5.png)


# Some troubleshootings
## Behind Proxy Noobaa Troubleshooting
- OCS does not add proxies to its components. So it should be given by manually.
```bash
oc set env sts/noobaa-core HTTP_PROXY=<value from cluster/proxy>
oc set env sts/noobaa-core HTTPS_PROXY=<value from cluster/proxy>
oc set env sts/noobaa-core NO_PROXY=<value from cluster/proxy>,rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage
```
```bash
oc set env deployment/noobaa-operator NO_PROXY=<value from cluster/proxy>,rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage
```
## Storage Degraded Warning Troubleshooting (clock skew)
> **After this troubleshooting all Nodes will be restarted because NTP settings will be changed. Do it your own RISK**
- This kind of warning is because NTP Service is not proper.
- It can be solved after doing NTP settings.
- In RHCOS servers **NTP Server** settings are tied with **chronyd service**. To make adjustments chronyd.conf file should be created and encode with base64 like down below.
```bash
cat << EOF | base64
server NTPSERVERADDRESS iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
```
- Command output will be two lines, but it needs to be only one line. So merge them together to add it on YAML. 
```bash
c2VydmVyIE5UUFNFUlZFUkFERFJFU1MgaWJ1cnN0CmRyaWZ0ZmlsZSAvdmFyL2xpYi9jaHJvbnkv
ZHJpZnQKbWFrZXN0ZXAgMS4wIDMKcnRjc3luYwpsb2dkaXIgL3Zhci9sb2cvY2hyb255Cg==
```
- Encoded information should be added after **base64** on **spec.storage.files.contents.source** section.
> After its creation on cluster, whole cluster will be restarted, even pods. So it should take about minimum of 20 minutes.
```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
    machineconfiguration.openshift.io/role: worker
  name: nodes-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,c2VydmVyIE5UUFNFUlZFUkFERFJFU1MgaWJ1cnN0CmRyaWZ0ZmlsZSAvdmFyL2xpYi9jaHJvbnkvZHJpZnQKbWFrZXN0ZXAgMS4wIDMKcnRjc3luYwpsb2dkaXIgL3Zhci9sb2cvY2hyb255Cg==
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
```
- After everything is working correctly and done, you can check on **Storage** section is healthy.
![10](https://user-images.githubusercontent.com/59168275/104120232-32cc5480-5346-11eb-9ef0-3b95d9b6a2d5.png)

### Referances
[1] [https://www.techbeatly.com/2020/01/red-hat-openshift-container-storage-4-2-installation.html](https://www.techbeatly.com/2020/01/red-hat-openshift-container-storage-4-2-installation.html)  
[2] [https://access.redhat.com/solutions/4776841](https://access.redhat.com/solutions/4776841)  
[3] [https://access.redhat.com/solutions/4828941](https://access.redhat.com/solutions/4828941)  
[4] [https://docs.openshift.com/container-platform/4.3/installing/install_config/installing-customizing.html#installation-special-config-crony_installing-customizing](https://docs.openshift.com/container-platform/4.3/installing/install_config/installing-customizing.html#installation-special-config-crony_installing-customizing)
