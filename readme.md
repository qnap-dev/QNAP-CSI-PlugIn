# QNAP CSI Driver for Kubernetes  
This is the official [Container Storage Interface](https://github.com/container-storage-interface) driver for QNAP NAS devices.  
 
## Software Prerequisites  
### CSI Driver Version and Compatibility  
| **Driver Version** | **Supported Kubernetes Versions** | **Supported QNAP NAS**                        |  
|------------------- | --------------------------------- | --------------------------------------------- |  
| v1.3.0             | 1.24 to 1.30                      | NAS running QTS or QuTS Hero 5.1.0 or later   |  
 
### Supported Host Operating Systems  
- Debian 8 or later  
- Ubuntu 16.04 or later  
- CentOS 7.0 or later  
- RHEL 7.0 or later  
- CoreOS 1353.8.0 or later

### Supported platform
- AMD 64
- ARM 64 and ARM v7
 
## Supported Features  
- Add StorageClasses  
- Add, resize, clone, and import Persistent Volume Claims (PVCs)  
- Take snapshots, SSD Cache, RaidLevel, Qtier (QTS only)
- Threshold, ThinAllocate
- Hero only: Compression, Deduplication, Fast clone
 
## Deploying the Driver  
### Before You Start  
#### Install open-iscsi in Your Kubernetes Environment  
Run the following command in both the master and worker nodes. 
``` 
apt install open-iscsi 
``` 
 
#### Qualify Your Kubernetes Cluster  
_Note: Minikube is not supported._ 
1. Make sure `kubectl` is installed and working.  
   - Run the following commands one at a time. 
``` 
kubectl get pods 
``` 
``` 
kubectl version 
``` 
2. Verify that you are logged in as a Kubernetes cluster administrator.  
``` 
kubectl auth can-i '*' '*' --all-namespaces 
``` 
   - The result should be "yes". 
3. Verify that you can launch a pod that uses an image from Docker Hub and can reach your storage system over the pod network.  
``` 
kubectl run -i --tty ping --image=busybox --restart=Never --rm -- \ping <NAS management IP> 
``` 
   - For example: `kubectl run -i --tty ping --image=busybox --restart=Never --rm -- \ping 10.64.118.157`  
4. Verify that your NAS has created a storage pool and iSCSI service is enabled. 
   - To check storage pools on your NAS, open Storage & Snapshots and go to Storage > Storage/Snapshots. 
   - To check iSCSI service on your NAS, open iSCSI & Fibre Channel and verify that the toggle button is on. 
   
### Install the QNAP CSI Plugin 
1. Clone the git repository.  
``` 
git clone https://github.com/qnap-dev/QNAP-CSI-PlugIn.git 
``` 
2. Enter the directory. 
``` 
cd QNAP-CSI-PlugIn 
``` 
3. Select one of the following installation methods. 
 
#### Normal Installation Method 
Run the following commands one at a time in order. 
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
 
#### Installing via Kustomize  
Run the following commands one at a time in order. 
``` 
kubectl apply -k Deploy/crds 
``` 
``` 
kubectl apply -k Deploy/Trident 
``` 
 
#### Installing via Helm   
1. Install Helm (for Ubuntu). 
   - Run the following commands one at a time in order. 
```  
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -  
``` 
``` 
sudo apt-get install apt-transport-https --yes  
``` 
``` 
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list  
``` 
``` 
sudo apt-get update  
``` 
``` 
sudo apt-get install helm  
```  
2. Install the CSI plugin. 
``` 
helm install qnap-trident ./Helm/trident -n trident --create-namespace 
``` 

- Upgrade 
```
helm upgrade qnap-trident Helm/trident/ -n trident
```

### Install VolumeSnapshot (optional)  
_Note: You need `VolumeSnapshot` to take snapshots._ 
``` 
kubectl apply -k VolumeSnapshot 
``` 
 
### Check if Trident is Ready
``` 
kubectl get deployment -n trident 
``` 
   - The result should include `trident-controller` and `trident-operator`. 
``` 
kubectl get service -n trident 
``` 
   - The result should include `trident-csi`. 
 
## CSI Configuration  
### CR (TridentBackendConfig): backend-config-sample.yaml
Edit the file `Samples/backend/backend-sample-qts` or `backend-sample-hero.yaml` or create a new one as shown below. 
You must configure this file before you create a volume. Each column is required.  
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-qts
  namespace: trident
type: Opaque
stringData:
  username: david
  password: abcd1234
  storageAddress: 10.20.91.69
---
apiVersion: trident.qnap.io/v1
kind: TridentBackendConfig
metadata:
  name: backend-qts
  namespace: trident
spec:
  version: 1
  storageDriverName: qnap-iscsi
  backendName: qts
  networkInterfaces: ["K8s-ISCSI"] #optional
  credentials:
    name: backend-qts
  debugTraceFlags:
    method: true
  storage:
    - serviceLevel: Any
      labels:
        performance: any 
    - serviceLevel: SSD-Cache
      labels:
        performance: premium
      features:
        tiering: Enable
        ssdCache: "true"
    - serviceLevel: Tiering
      labels:
        performance: standard
      features:
        tiering: Enable
    - serviceLevel: Non-Tiering
      labels:
        performance: basic
      features:
        tiering: Disable
    - serviceLevel: RAID0
      labels:
        performance: raid0
      features:
        raidLevel: "0"
    - serviceLevel: RAID1
      labels:
        performance: raid1
      features:
        raidLevel: "1"
    - serviceLevel: RAID5
      labels:
        performance: raid5
      features:
        raidLevel: "5"
```
### CLI (tridentctl): Backend.json File  
Add a backend to your orchestrator. 
Edit the file `Samples/backend/backend-qts1.json` or create a new one as shown below. 
You must configure this file before you create a volume. Each column is required.  
```json  
{
    "version": 1,
    "storageDriverName": "qnap-iscsi",
    "backendName": "qts-david",
    "storageAddress": "10.20.91.69",
    "username": "david",
    "password": "abcd1234",
    "networkInterfaces": ["K8s-ISCSI"], 
    "debugTraceFlags": {"method":true},
    "storage": [
        {
            "labels": {"storage": "qts-david"},
            "serviceLevel": "Any"
        },
        {
            "labels": {"performance": "premium"},
            "features":{
                "tiering": "Enable",
                "ssdCache": "true"
            },
            "serviceLevel": "SSD-Cache"
        },
        {
            "labels": {"performance": "standard"},
            "features":{
                "tiering": "Enable"
            },
            "serviceLevel": "Tiering"
        },
        {
            "labels": {"performance": "basic"},
            "features":{
                "tiering": "Disable"
            },
            "serviceLevel": "Non-Tiering"
        }
    ]
}
```  
 
### StorageClass.yaml File  
Edit the file `Samples/StorageClass/sc.yaml` or create a new one as shown below.  
```yaml  
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium
provisioner: csi.trident.qnap.io
parameters:
  fsType: ext4
  selector: "performance=premium"
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: csi.trident.qnap.io
parameters:
  fsType: ext4
  selector: "performance=standard"
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: basic
provisioner: csi.trident.qnap.io
parameters:
  fsType: ext4
  selector: "performance=basic"
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: any
provisioner: csi.trident.qnap.io
parameters:
  fsType: ext4
  selector: "performance=any"
allowVolumeExpansion: true
```  
 
### PVC.yaml File 
Edit the file `Samples/Volumes/pvc-any.yaml` or create a new one as shown below. 
```yaml  
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-any-1
  annotations:
      # thin allocate & threshold is customized
    trident.qnap.io/threshold: "90"
    trident.qnap.io/ThinAllocate: "true"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: any
```  
 
## Installation
### Adding a Backend by CR(TridentBackendConfig)
1. Make sure you have a corresponding pool. 
2. Add a backend based on the yaml file you configured earlier. 
```
kubectl apply -f <backend yaml file path>
```
3. Check the result. 
   - Run the following commands one at a time. 
```
kubectl get tridentbackendconfigs.trident.qnap.io -n trident
```
### Adding a Backend by CLI(tridentctl)
1. Make sure you have a corresponding pool. 
2. Ensure you have the permission to execute `tridentctl`. 
``` 
chmod u+x tridentctl 
```  
3. Add a backend based on the JSON file you configured earlier.  
``` 
./bin/tridentctl create backend -f <backend.json> -n trident 
``` 
   - For example: `./bin/tridentctl create backend -f Samples/backend/backend-sample.json -n trident` 
   - This should take around 30 seconds or less. 
   - If it takes over 30 seconds and shows the error "Command terminated with exit code 1" due to timeout, check your network connection and try again.  
4. Check the result.  
   - Run the following commands one at a time. 
``` 
kubectl get pods -n trident 
``` 
 
### Adding a StorageClass  
``` 
kubectl apply -f <StorageClass.yaml> 
``` 
   - For example: `kubectl apply -f Samples/StorageClass/sc.yaml`  

 
### Persistent Volume Claims (PVCs) 
#### Adding a PVC  
``` 
kubectl apply -f <pvc.yaml> 
``` 
   - For example: `kubectl apply -f Samples/Volumes/pvc-any.yaml`  

#### Resizing a PVC  
``` 
kubectl edit pvc <pvc name> 
``` 
   - For example: `kubectl edit pvc pvc-any`  
   - After a few seconds, the capacity will increase on the NAS. 
   - After the pod restarts, information on the capacity will be updated. 
 
#### Cloning a PVC  
``` 
kubectl apply -f <pvc-clone.yaml> 
``` 

#### Importing a PVC  
``` 
kubectl apply -f <pvc-import.yaml> 
``` 
 
#### Creating a PVC from a Snapshot  
``` 
kubectl apply -f <vol-snapshot.yaml> 
```

#### Deploying a Pod  
1. Deploy a pod.  
``` 
kubectl apply -f <pod yaml file> 
``` 
   - For example: `kubectl apply -f Samples/pod.yaml`  
2. Check the result.  
``` 
kubectl get pods 
```  
3. Check the connection has been mapped on the NAS by opening iSCSI & Fibre Channel. 
4. Use the logs to check whether the pod is running normally. 
 
You can now mount a PVC to the pod. 
 
_Note: After deploying the pod, the only thing you can do with the pod is print out its timestamp.
 
### Snapshots  
#### Creating a VolumeSnapshot from a PVC  
1. Create a VolumeSnapshot from a PVC.  
   - Run the following commands one at a time. 
``` 
kubectl apply -f <VolumeSnapshotClass.yaml> 
``` 
``` 
kubectl apply -f <VolumeSnapshot.yaml> 
``` 
2. Check the result.  
   - Run the following commands one at a time. 
``` 
kubectl get volumesnapshot 
``` 
 
#### Creating a PVC From a Snapshot  
1. Create a PVC from a snapshot. 
   - Run the following commands one at a time. 
``` 
kubectl apply -f <pvc-from-snapshot.yaml> 
``` 
``` 
kubectl apply -f <pod2.yaml> 
```  
2. Verify the PVC has been successfully created from the snapshot. 
   - Create a new pod and mount the snapshot-pvc in the pod. 
   - Access the pod and check if the PVC contains the directory you created before. 
 
## Uninstalling Trident 
### Normal Uninstallation Method / Uninstalling via Kustomize  
Run the following commands one at a time in order. 
``` 
kubectl delete deployment trident-operator -n trident 
``` 
``` 
./bin/tridentctl uninstall -n trident 
``` 
``` 
kubectl delete tridentorchestrator trident 
``` 
 
### Uninstalling via Helm  
``` 
helm delete qnap-trident -n trident 
``` 

### Reboot 
```
kubectl delete tridentorchestrator trident
```
```
kubectl apply -f Deploy/Trident/tridentorchestrator.yaml
```
