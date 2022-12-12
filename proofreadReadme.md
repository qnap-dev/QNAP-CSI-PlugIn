# QNAP CSI Driver for Kubernetes  
This is the official [Container Storage Interface](https://github.com/container-storage-interface) driver for QNAP NAS devices.  
 
## Software Prerequisites  
### CSI Driver Version and Compatibility  
| **Driver Version** | **Supported Kubernetes Versions** | **Supported QNAP NAS**                |  
|------------------- | --------------------------------- | ------------------------------------- |  
| v1.0.0-beta        | 1.14 to 1.23                      | NAS running QTS 5.0.0 or later        |  
 
### Supported Host Operating Systems  
- Debian 8 or later  
- Ubuntu 16.04 or later  
- CentOS 7.0 or later  
- RHEL 7.0 or later  
- CoreOS 1353.8.0 or later  
 
## Supported Features  
- Add StorageClasses  
- Add, resize, clone, and import Persistent Volume Claims (PVCs)  
- Take snapshots  
 
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
kubectl run -i --ttyping --image=busybox --restart=Never --rm --\ping <NAS management IP> 
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
kubectl apply -f Deploy/Trident/crds/trident_CRD.yaml 
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
helm install qnap-trident ./qnap-trident -n trident --create-namespace 
``` 
 
### Install VolumeSnapshot (optional)  
_Note: You need `VolumeSnapshot` to take snapshots._ 
``` 
kubectl apply -k VolumeSnapshot 
``` 
 
### Check if Trident is Ready to Deploy 
``` 
kubectl get deployment -n trident 
``` 
   - The result should include `trident-csi` and `trident-operator`. 
 
## CSI Configuration  
### Backend.json File  
Add a backend JSON file to your orchestrator.  
Edit the file `Samples/backend-qts1.json` or create a new one as shown below. 
You must configure this file before you create a volume. Each column is required.  
```json  
{  
    "version": 1,  
    "operatorVersion": "v1.0.0-beta",  
    "storageVersion": "v1.0.0-beta",  
    "storageDriverName": "qnap-iscsi",  
    "backendName": "QTS1",  
    "storageAddress": "<QTS IP Address>",  
    "username": "<QTS Username>",  
    "password": "<QTS Password>",  
    "debugTraceFlags": {"api":false, "method":true},  
    "storage": [  
        {  
            "labels": {"performance": "premium"},  
            "features":{  
                "tiering": "Enable",  
                "tierType": "SSD",  
                "ssdCache": "true"  
            },  
            "serviceLevel": "premium"  
        },  
        {  
            "labels": {"performance": "standard"},  
            "features":{  
                "tiering": "Enable",  
                "tierType": "SSD"  
            },  
            "serviceLevel": "standard"  
        },  
        {  
            "labels": {"performance": "basic"},  
            "features":{  
                "tiering": "Disable",  
                "tierType": "SATA"  
            },  
            "serviceLevel": "basic"  
        }  
    ]  
}  
```  
 
### StorageClass.yaml File  
Edit the file `Samples/storage-class-qnap-qos.yaml` or create a new one as shown below.  
```yaml  
apiVersion: storage.k8s.io/v1 
kind: StorageClass 
metadata: 
  name: premium 
provisioner: csi.trident.qnap.io #k8s CSI provisioner 
parameters: 
  selector: "performance=premium" 
allowVolumeExpansion: true 
```  
 
### PVC.yaml File 
Edit the file `Samples/pvc-basic.yaml` or create a new one as shown below. 
```yaml  
kind: PersistentVolumeClaim  
apiVersion: v1  
metadata:  
  name: pvc-basic  
  annotations:  
    trident.qnap.io/ThinAllocate: "false"  
spec:  
  accessModes:  
    - ReadWriteOnce  
  resources:  
    requests:  
      storage: 5Gi  
  storageClassName: basic  
```  
 
## Installation  
### Adding a Backend  
1. Make sure you have a corresponding pool. 
2. Ensure you have the permission to execute `tridentctl`. 
``` 
chmod u+x tridentctl 
```  
3. Add the backend JSON file you configured earlier.  
``` 
./tridentctl create backend -f <backend.json> -n trident 
``` 
   - For example: `./tridentctl create backend -f Samples/backend-qts1.json -n trident` 
   - This should take around 30 seconds or less. 
   - If it takes over 30 seconds and shows the error "Command terminated with exit code 1" due to timeout, check your network connection and try again.  
4. Check the result.  
   - Run the following commands one at a time. 
``` 
kubectl get pods -n trident 
``` 
``` 
kubectl get qpools -n trident 
``` 
 
### Adding a StorageClass  
1. Apply the StorageClass YAML file you configured earlier.  
``` 
kubectl apply -f <StorageClass.yaml> 
``` 
   - For example: `kubectl apply -f Samples/storage-class-qnap-qos.yaml`  
2. Check the result.  
``` 
kubectl get sc -n trident 
```  
 
### Persistent Volume Claims (PVCs) 
#### Adding a PVC  
1. Add a PVC.  
``` 
kubectl apply -f <pvc.yaml> 
``` 
   - For example: `kubectl apply -f Samples/pvc-basic.yaml`  
   - This example creates a thick LUN. If you want to create a thin LUN, refer to the `Samples/pvc-standard.yaml` file. 
2. Check that the PVC is on the PVC list and its status is "Bound". 
``` 
kubectl get pvc 
``` 
3. Check that the NAS has created the LUN. 
``` 
kubectl get qnapvolume -n trident 
```  
 
#### Resizing a PVC  
1. Resize a PVC. 
``` 
kubectl edit pvc <pvc name> 
``` 
   - For example: `kubectl edit pvc pvc-basic`  
   - After a few seconds, the capacity will increase on the NAS. 
2. Check the result.  
``` 
kubectl get pvc 
``` 
   - After the pod restarts, information on the capacity will be updated. 
 
#### Cloning a PVC  
1. Clone a PVC.  
``` 
kubectl apply -f <pvc-clone.yaml> 
``` 
2. Check the result.  
``` 
kubectl get pvc 
``` 
 
#### Importing a PVC  
1. Import a PVC.  
``` 
./tridentctl import volume <Backend Name> <LUN Name> -f <pvc-import.yaml> -n trident 
``` 
2. Check the result.  
``` 
kubectl get pvc -n trident (or -A) 
``` 
 
#### Creating a PVC from a Snapshot  
``` 
kubectl apply -f <vol-snapshot.yaml> 
``` 
_Note: Once a PVC is successfully created, the corresponding `qnapvolume` will also be created, which has detailed information on the volume. An admin can check whether the volume has been created via the Kubernetes user interface._ 
 
### Deploying a Pod  
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
 
_Note: After deploying the pod, the only thing you can do with the pod is print out its timestamp._ 
 
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
``` 
kubectl get qsnapshot -n trident 
``` 
 
_Note: 
Once `volumesnapshot` is created successfully, the corresponding `qsnapshot` will also be created, which records the snapshot information in the storage. An admin can check whether the snapshot has been created via the Kubernetes user interface._ 
 
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
./tridentctl uninstall -n trident 
``` 
``` 
kubectl delete tridentorchestrator trident 
``` 
 
### Uninstalling via Helm  
``` 
helm delete qnap-trident -n trident 
``` 
