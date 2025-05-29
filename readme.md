# QNAP CSI Driver for Kubernetes  
This is the official [Container Storage Interface](https://github.com/container-storage-interface) (CSI) driver for QNAP NAS devices.  

## Table of Contents
* [Introduction](#Introduction)
* [Software Prerequisites](#Software-Prerequisites)
* [Supported Features](#Supported-Features)
* [Installation](#Installation)
* [CSI Driver Configuration](#CSI-Driver-Configuration)
* [Deployment](#Deployment)
* [Operations](#Operations)

<a name="Introduction"></a> 
## Introduction
The QNAP CSI driver is a storage service designed to provide volumes from local or remote NAS devices to Kubernetes. It is based on [NetApp Trident](http://github.com/NetApp/Trident), providing dynamic volume provisioning. The driver supports quality of service (QoS) assignment to ensure optimal performance tailored to application needs.

The key concepts in this driver are `backend` and `pools`. A backend represents the NAS storage used for [PersistentVolumeClaim (PVC)](https://reurl.cc/DlYzZe) requests. Each backend contains virtual pools, which are logical groupings of storage that dynamically select an appropriate physical pool based on specific requirements. PVCs request storage through a [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/), which binds to the virtual pool, ensuring the volume is created from the correct physical resources. The process for creating a volume using this driver is as follows (also illustrated in Figure 1):

1. The user claims a PVC request specifying a StorageClass.
2. The system binds the request to a suitable virtual pool configured in the backend.
3. The system filters the available physical pools based on the required features.
4. A suitable physical pool is selected from the filtered options.
5. The requested volume is created on the selected physical pool.
  > [!Tip]
  > The system prioritizes selecting the backend with the lowest current volume usage to ensure load balancing.

[![create.png](https://i.postimg.cc/cC6YjCNq/create.png)](https://postimg.cc/YLKhLp98)
Figure 1. Process of creating a volume with the QNAP CSI driver.

<a name="Software-Prerequisites"></a> 
## Software Prerequisites  
### CSI Driver Version and Compatibility  
| **Driver Version** | **Supported Kubernetes Versions** | **Supported QNAP NAS Operating Systems**                |  
|------------------- | --------------------------------- | ------------------------------------- | 
| v1.5.0             | v1.24 to v1.32                    | QTS 5.0.0 or later<br>QuTS hero h5.0.0 or later|
 
### Supported Host Operating Systems  
- Debian 8 or later  
- Ubuntu 16.04 or later  
- CentOS 7.0 or later  
- RHEL 7.0 or later  
- CoreOS 1353.8.0 or later
- Talos 1.8 or later

### Supported Platforms
- AMD64
- ARM64 and ARMv7

<a name="Supported-Features"></a> 
## Supported Features 

* Protocol: iSCSI, Samba
* Access mode: ReadWriteOnce, ReadWriteMany
* Cloning  
* Snapshots
* Expansion
* Pool level: RAID levels, SSD cache, Tiering
* Volume level: Threshold, ThinAllocate, Compression, Deduplication

<a name="Installation"></a> 
## Installation
### Preinstallation Checklist

* Ensure that both the Kubernetes and the QNAP NAS operating system versions are supported.
> [!NOTE] 
> Minikube is not supported.
* If the iSCSI protocol is utilized. Install open-iscsi in Kubernetes by running the following command in both the master and worker nodes: 
  ``` 
  sudo apt install open-iscsi 
  ``` 
* Verify that your NAS has at least one available storage pool and iSCSI service is enabled. 
   - To check storage pools on your NAS, open Storage & Snapshots and navigate to "Storage > Storage/Snapshots".
   - If the iSCSI protocol is utilized. To check iSCSI service on your NAS, open iSCSI & Fibre Channel and verify that the "Service" toggle switch is on. 
<details>  <summary>Verify your Kubernetes cluster.</summary>

### Verifying Your Kubernetes Cluster  
* Make sure `kubectl` is installed and working.
    ``` 
    kubectl get pods -A
    ``` 
    ``` 
    kubectl version 
    ``` 
    Verify that both commands return expected results without errors.

* Make sure that you are logged in as a Kubernetes cluster administrator.
    ``` 
    kubectl auth can-i '*' '*' --all-namespaces 
    ``` 
    The command should return `yes`. 

* Verify that you can launch a pod using an image from Docker Hub and check connectivity to the storage system over the pod network.
    ``` 
    kubectl run -i --tty ping --image=busybox --restart=Never --rm -- \ping <NAS management IP> 
    ``` 
    For example: `kubectl run -i --tty ping --image=busybox --restart=Never --rm -- \ping 8.8.8.8`  
    A successful test will display packet responses, indicating connectivity. The pod will automatically delete itself after completion.
</details>

### Install the QNAP CSI Plugin 
1. Clone the git repository.  
    ``` 
    sudo git clone https://github.com/qnap-dev/QNAP-CSI-PlugIn.git 
    ``` 
2. Navigate to the directory. 
    ``` 
    cd QNAP-CSI-PlugIn 
    ``` 
3. Choose one of the following installation methods. 
 
    * Installing via kubectl 

      Execute the following commands in order: 
      ``` 
      kubectl apply -f Deploy/Trident/namespace.yaml 
      ``` 
      ``` 
      kubectl apply -f Deploy/crds/tridentorchestrator_crd.yaml 
      ``` 
      ``` 
      kubectl apply -f Deploy/Trident/bundle.yaml 
      ``` 
      ``` 
      kubectl apply -f Deploy/Trident/tridentorchestrator.yaml 
      ``` 
 
    * Installing via Kustomize  
      
      Execute the following commands in order:
      ``` 
      kubectl apply -k Deploy/crds 
      ``` 
      ``` 
      kubectl apply -k Deploy/Trident 
      ``` 
 
    * Installing via Helm   
      1. Install Helm. 

          Please refer to the [Helm Installation Guide](https://helm.sh/docs/intro/install/) for instructions.

      2. Install the CSI plugin. 
          ``` 
          helm install qnap-trident ./Helm/trident -n trident --create-namespace 
          ``` 

       3. Upgrade the plugin.
          ```
          helm upgrade qnap-trident Helm/trident/ -n trident
          ```

### Install VolumeSnapshot (Optional) 
A VolumeSnapshot is required to take snapshots.
``` 
kubectl apply -k VolumeSnapshot 
``` 
 
### Check that Trident is Ready
1. Check the status of Trident.

    ``` 
    kubectl get deployment -n trident 
    ``` 
    The output should display both `trident-controller` and `trident-operator` as being ready.<br>Example output:
  [![trident-ready.png](https://i.postimg.cc/zGx1qkC6/trident-ready.png)](https://postimg.cc/75JRV0Qn)
2. Verify the Trident service.

    ``` 
    kubectl get service -n trident 
    ``` 
    The output should include `trident-csi`.<br>Example output:
  [![trident-ready2.png](https://i.postimg.cc/PrhGZ7K0/trident-ready2.png)](https://postimg.cc/7bBQrmFN)

<a name="CSI-Driver-Configuration"></a>  
## CSI Driver Configuration  
To ensure the CSI driver functions correctly, it is important to configure the backend, secert (Optional), StorageClass, and PVC with matching labels and parameters to enable proper binding. The backend defines the virtual storage pools and connects the underlying physical resources, the secret is required when the Samba protocol is utilized for connection, while the StorageClass specifies the QoS that the PVC will request. 

For binding to succeed, the PVC must reference a StorageClass that has the correct labels and selectors, which correspond to the virtual pools defined in the backend. Accurate configuration of names, labels, and selectors in each component is essential to ensure the PVC is correctly bound to the appropriate storage resources through the backend and StorageClass.

### Backend
There are two methods for configuring the backend: 
* Custom resource (`TridentBackendConfig`) 
* CLI (`tridentctl`).

> [!TIP] 
> We recommend using the newer custom resource method over the legacy CLI method.

1. Create or edit a YAML file (for the custom resource method) or a JSON file (for the CLI method).<br>
Please refer to the following examples.

    * Custom resource method<br>
      Example: `Samples/Backend/backend-sample.yaml` 

      ```yaml
      apiVersion: v1
      kind: Secret
      metadata:
        name: backend-qts-secret # Required. Name your secret.
        namespace: trident
      type: Opaque
      stringData:
        username: user # Required. Your NAS username.
        password: 0000 # Required. Your NAS password.
        storageAddress: 0.0.0.0 # Required. Your NAS IP address.
      ---
      apiVersion: trident.qnap.io/v1
      kind: TridentBackendConfig
      metadata:
        name: backend-qts # Required. Name your backend in Kubernetes.
        namespace: trident
      spec:
        version: 1
        storageDriverName: qnap-nas #Required. Support 'qnap-nas'(latest) or 'qnap-iscsi'
        backendName: qts # Required. Name your backend in QNAP CSI.
        networkInterfaces: ["Adapter1"] # Optional. Your adapter name or leave it empty.
        credentials:
          name: backend-qts-secret # Required. Enter the secret name set in metadata.name.
        debugTraceFlags:
          method: true
        storage: # Required. Define one or more virtual pools.
          - serviceLevel: pool1 # Required. Name your virtual pool.
            labels: # Required. Define custom labels for your virtual pool.
              performance: performance1
            features: # Optional. Define features for your virtual pool.
              tiering: Enable
          - serviceLevel: pool2
            labels:
              performance: performance2
            features:
              tiering: Enable
              ssdCache: "true"
              raidLevel: "1"
      ```

    * CLI method<br>
      Example: `Samples/Backend/backend-sample.json`. 
      ```json  
      {
          "version": 1,
          "storageDriverName": "qnap-nas",
          "backendName": "qts",
          "storageAddress": "0.0.0.0",
          "username": "user",
          "password": "0000",
          "networkInterfaces": ["Adapter1"], 
          "debugTraceFlags": {"method": true},
          "storage": [
              {
                  "labels": {"performance": "performance1"},
                  "features":{
                      "tiering": "Enable"
                  },
                  "serviceLevel": "pool1"
              },
              {
                  "labels": {"performance": "performance2"},
                  "features":{
                      "tiering": "Enable",
                      "ssdCache": "true",
                      "raidLevel": "1"
                  },
                  "serviceLevel": "pool2"
              },
          ]
      }
      ```  

2. In the YAML or JSON file, configure the following based on your NAS settings and usage requirements.
    * Specify the correct NAS `user`, `password`, and `IP address`.
    * Name the secret and backend.
    * Optional: Set the network portal in the `networkInterfaces` field.

      > **Note:**<br>
      > Network settings support physical and virtual adapters. In the `networkInterfaces` field, assign an interface name (e.g., \["Adapter1"\]) or leave it empty to default to `storageAddress`. To view the available adapters on your NAS, open Network & Virtual Switch and go to "Interfaces".
      
    * Define the virtual pools based on your usage requirements.
      | Field                       | Description            | Example | 
      |------------------------------|------------------------|---------|
      | serviceLevel                 | Name the virtual pool.  | serviceLevel: pool1 |  
      | labels                       | Label the virtual pool with any key/value.  | usage: test |
      | features                     | Define the virtual pool with the specified features from the feature list below.       | features:<br>&nbsp;&nbsp;ssdCache: "true"   

      Feature list:
      | Feature   | Description |Value            | Note     |
      |-----------|----------|--------|----------|
      | raidLevel | Combines multiple disks to improve both redundancy and performance. |0, 1, 5, 6, etc.   |          |
      | ssdCache  | Boosts NAS performance by caching frequently accessed data on SSDs, reducing latency and speeding up access. |true, false     |          |
      | tiering   | Moves hot data to high-performance drives and cold data to cost-efficient drives, optimizing performance and total cost of operation.|enable, disable | Only QTS |

### Secert
If the Samba protocol is utilized, the `Secert` configuration is required.
Create or edit the YAML file `Samples/Secret/smb_user_secret.yaml`\
Example:
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: qts-csi-smb #Required. Name your secert for Samba.
    namespace: trident
type: Opaque
stringData:
    username: user1 #Required. The valid Samba username on the NAS.
    password: 0000 #Required. The valid Samba password on the NAS.
```
    
### StorageClass 
1. Create or edit the YAML file.<br>
    
    iSCSI: `Samples/StorageClass/sc_iscsi_sample.yaml`.<br>
    
    Example:
    ```yaml  
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: storageclass1 # Required. Name your storageclass.
    provisioner: csi.trident.qnap.io
    parameters:
      selector: "performance=performance1" # Required. Corresponds to the labels in the virtual pool.
      fsType: "ext4" # Optional. You can choose to enter ext4 (default), xfs, or ext3.
    allowVolumeExpansion: true
    ``` 
    Samba: `Samples/StorageClass/sc_smb_sample.yaml`.<br>
    
    Example:
    
    ```yaml  
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: storageclass1 # Required. Name your storageclass.
    provisioner: csi.trident.qnap.io
    parameters:
      selector: "performance=performance1" # Required. Corresponds to the labels in the virtual pool.
      trident.qnap.io/fileProtocol: "smb"        
      csi.storage.k8s.io/node-stage-secret-name: "qts-csi-smb" # Required. It must match the metadata.name in the Secret file.       
      csi.storage.k8s.io/node-stage-secret-namespace: "trident" # Required. It must match the metadata.namespace in the Secret file.  
    allowVolumeExpansion: true
    ```  
2. Configure the file based on your usage requirements.
    
    Bind the backend's virtual pool using the parameters.selector field, ensuring it matches the labels in the virtual pool.

### PVC
1. Create or edit the YAML file `Samples/Volumes/pvc-sample.yaml`<br>
    Example:
    ```yaml  
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: pvc1
      annotations: # Required. Define features for your volume.
        trident.qnap.io/Threshold: "90"
        trident.qnap.io/ThinAllocate: "true"
    spec:
      accessModes:
        - ReadWriteOnce # Required. iSCSI: ReadWriteOnce, Samba: ReadWriteMany
      resources:
        requests:
          storage: 10Gi # Required. Specify your resource size.
      storageClassName: storageclass1 # Required. Corresponds to the StorageClass name.
      ```

  2. Configure the file based on your usage requirements.

      * Bind the StorageClass using the `storageClassName` field.
      * Set your volume feature requirements in the `annotations` field using the prefix `trident.qnap.io/`. For example, `trident.qnap.io/threshold: "90"`. Refer to the following table for details.

        Volume features:
        | Feature            | Description |Value        | Note           |
        |--------------------|--------|------|----------------|
        | Threshold          | Monitors storage usage to trigger alerts when capacity limits are reached, preventing overuse. |0-100      |                |
        | ThinAllocate       | Dynamically allocates storage space to meet demands, optimizing usage and avoiding shortages. |true, false | 
        | SharedFolderRecycleBin       | Enable or disable the Recycle Bin based on your needs. |true, false |SMB protocol only |
        | Compression        | Compresses files to reduce size, saving space and allowing more storage on the NAS. |true, false | Only QuTS hero |
        | Deduplication      | Eliminates duplicate data to reduce storage needs and minimize network data transfers.|true, false | Only QuTS hero |
        | FastClone          | Creates file copies faster to save space by sharing data blocks between originals and copies.|true, false | Only QuTS hero |
        | importOriginalName | The original volume name. | string       | Refer to [Importing a PVC](#importing-a-pvc) in the "Operations" section.               |
        | importBackendName  | The imported backend name. |string       |       Refer to [Importing a PVC](#importing-a-pvc) in the "Operations" section.         |

<a name="Deployment"></a> 
## Deployment
When building the service, add the backend, the StorageClass, and then the PVC, in this order.

### 1. Add a Backend
Make sure you have a corresponding pool, and based on the type of your backend configuration file, choose one of the following methods to add the pool.

* Using the custom resource method to add a backend with the YAML file

  a. Add a backend based on the YAML file you configured earlier. 
     ```
     kubectl apply -f <your backend yaml file path>
     ```
     For example: `kubectl apply -f Samples/Backend/backend-sample.yaml`

  b. Check the result. 
     ```
     kubectl get tridentbackendconfig -n trident
     ```

* Using the CLI method to add a backend with the JSON file

  a. Ensure you have permission to execute `tridentctl`. 
     ``` 
     chmod u+x tridentctl 
     ```
  
  b. Add a backend based on the JSON file you configured earlier.  
     ``` 
     ./<your tridentctl path> create backend -f <backend.json> -n trident 
     ``` 
     For example: `./bin/linux-amd64/tridentctl create backend -f Samples/Backend/backend-sample.json -n trident` 
    
> [!NOTE]
> If it takes over 30 seconds and shows the error "Command terminated with exit code 1" due to timeout, check your network connection and try again.  

After adding the backend, check the result.  
``` 
kubectl get pods -n trident 
``` 

    
### 2. Add a StorageClass  
Execute the following command:
``` 
kubectl apply -f <your StorageClass yaml file path> 
``` 
For example: `kubectl apply -f Samples/StorageClass/sc-sample.yaml`  
    
> [!NOTE]
> If you want to use the SMB protocol, please execute the following command and apply the corresponding Secret before adding the StorageClass.
>``` 
>kubectl apply -f <your Secret yaml file path> 
>```
> For example: `kubectl apply -f Samples/Secret/smb_user_secret.yaml`  
 
### 3. Add a PVC
Execute the following command: 
``` 
kubectl apply -f <your pvc yaml file path> 
``` 
For example: `kubectl apply -f Samples/Volumes/pvc-sample.yaml`  

### 4. Verify the Connection (Optional)

After completing the deployment steps, verify if the connection is mapped in iSCSI & Fibre Channel on the NAS. You should locate the connection in the "iSCSI Target List" on the "iSCSI Storage" page, as shown in the red frame in Figure 2.

[![nas-iscsi.png](https://i.postimg.cc/qM02cB84/nas-iscsi.png)](https://postimg.cc/vDq1bsqN)
<p align="center">Figure 2. Checking the iSCSI target connection.</p>

<a name="Operations"></a>
## Operations
### Expanding a PVC  
``` 
kubectl edit pvc <your pvc name> 
``` 
For example: `kubectl edit pvc pvc-sample`  
   
After a few seconds, the capacity will increase on the NAS. 
After the pod restarts, the capacity information will be updated. 
 
### Cloning a PVC  

Create or edit the YAML file `Samples/Volumes/pvc-clone.yaml` based on your usage requirements.

Example:
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: pvc-from-clone
  spec:
    accessModes:
      - ReadWriteOnce # iSCSI: ReadWriteOnce, Samba: ReadWriteMany
    resources:
      requests:
        storage: 5Gi
    dataSource:
      kind: PersistentVolumeClaim
      name: pvc-sample
    storageClassName: storageclass1
  ```
  Name the cloned PVC using the `name` field, configure the `storage` field, and assign the source PVC in the `dataSource` field.
>[!Note]
>The cloned PVC's storage size must be greater than or equal to that of the source PVC.

  After configuring the YAML file, run the following command:
  ``` 
  kubectl apply -f <your pvc-clone yaml file path> 
  ``` 
  For example: `kubectl apply -f Samples/Volumes/pvc-clone.yaml`  

<a name="importing-a-pvc"></a> 
### Importing a PVC  

iSCSI:  
Create or edit the YAML file `Samples/Volumes/pvc-import.yaml` based on your usage requirements.
>[!NOTE]
>Only iSCSI LUNs are supported. 

Example:
  ```yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: pvc-import-iscsi
    namespace: default
    annotations:
      trident.qnap.io/importOriginalName: "test"
      trident.qnap.io/importBackendName: "hero"
  spec:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
    storageClassName: storageclass1
  ```
  After configuring the YAML file, run the following command:
  ``` 
  kubectl apply -f <your pvc-import yaml file path> 
  ``` 
  For example: `kubectl apply -f Samples/Volumes/pvc-import.yaml`

Samba:
Create or edit the YAML file `Samples/Volumes/pvc-import.yaml` based on your usage requirements.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
    name: pvc-import-file
annotations:
    trident.qnap.io/importOriginalName: "shared-name"
    trident.qnap.io/importBackendName: "qts"
spec:
    accessModes:
       - ReadWriteMany
    resources:
            requests:
                storage: 10Gi #The value entered should correspond to the actual or displayed size.
storageClassName: storageclass1
```
>[!Note]
>Due to filesystem limitations in QTS, the capacity shown in "Storage & Snapshot" may be slightly smaller than the specified size when creating a shared folder. For `resources.requests.storage`, enter either the specified size or the displayed size. 
For example: 
    - Specified size: 10GB
    - Displayed size: 9.34GB.
[![2025-01-02-120516.png](https://i.postimg.cc/LXcDRJhk/2025-01-02-120516.png)](https://postimg.cc/zbkKFXC3)
<br>In QuTS Hero, always use the specified size.

### Creating a VolumeSnapshot from a PVC  
    
Create or edit the YAML file `Samples/Snapshot/VolumeSnapshot.yaml` based on your usage requirements.

Example:
  ```yaml
  apiVersion: snapshot.storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    name: pvc-snapshot
  spec:
    volumeSnapshotClassName: trident-snapshotclass
    source:
      persistentVolumeClaimName: pvc-sample
  ```  
   After configuring the YAML file, run the following commands one at a time: 
  ``` 
  kubectl apply -f <VolumeSnapshotClass.yaml> 
  ``` 
  ``` 
  kubectl apply -f <VolumeSnapshot.yaml> 
  ``` 
   Check the result. 
  ``` 
  kubectl get volumesnapshot 
  ``` 
 
### Creating a PVC from a Snapshot  
Create or edit the YAML file `Samples/Volumes/pvc-from-snapshot.yaml` based on your usage requirements.

Example:    
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-from-snap
spec:
  accessModes:
    - ReadWriteOnce # iSCSI: ReadWriteOnce, Samba: ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  dataSource:
    name: pvc-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  storageClassName: storageclass1
```    
    
  After configuring the YAML file, run the following command:
  ``` 
  kubectl apply -f <pvc-from-snapshot.yaml> 
  ``` 
 
### Reboot 

  Execute the following commands in order:
  ```
  kubectl delete tridentorchestrator trident
  ```
  ```
  kubectl apply -f Deploy/Trident/tridentorchestrator.yaml
  ```
### Uninstallation
  * Uninstalling via Kustomize (normal method)  
  Execute the following commands in order: 
    ``` 
    kubectl delete deployment trident-operator -n trident 
    ``` 
    ``` 
    ./<your tridentctl path> uninstall -n trident 
    ``` 
    ``` 
    kubectl delete tridentorchestrator trident 
    ``` 
 
  * Uninstalling via Helm  
    ``` 
    helm delete qnap-trident -n trident 
    ``` 