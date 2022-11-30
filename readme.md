# QANP CSI Driver for Kubernetes
This is official [Container Storage Interface](https://github.com/container-storage-interface) driver for QNAP NAS.

## Software Prerequisites
### CSI Driver version and Compatibility
| Driver Version  | Supported k8s version   | Supported QNAP NAS version |
| ----------------| ------------------------|----------------------------|
| v0.0.0-beta     |     1.14 to 1.23        | QuTS 5.0.0 or later        |

### Supported host operating systems
- Debian 8 or later
- Ubuntu 16.04 or later
- CentOS 7.0 or later
- RHEL 7.0 or later
- CoreOS 1353.8.0 or later

## Supported feature
- Add StorageClass
- Provision Volume (include clone, import)
- Take snapshot

# Deploy
## Before starting
### Additional install item in k8s environment
Master and worker node should install: 
Run `apt install open-iscsi`

### Qualify your Kubernetes cluster
1. **Minikube is not supported**
2. Make sure “kubectl” works well
   - Run `kubectl get pods`
   - Run `kubectl version`
3. Make sure you are Kubernetes cluster administrator
   - Run `kubectl auth can-i '*' '*' --all-namespaces`
   - The result should be yes
4. Can you launch a pod that uses an image from Docker Hub and can reach your storage system over the pod network?
   - Run `kubectl run -i --ttyping --image=busybox --restart=Never --rm --\ping <management IP>`
   - Example `kubectl run -i --tty ping --image=busybox --restart=Never --rm -- \ping 10.64.118.157`
5. Check your NAS has been create storage pool and open the iscsi service.

## Start install QNAP CSI Plugin
1. Clone the git repository. `gh repo clone qnap-dev/QNAP-CSI-PlugIn`
2. Enter the directory. `cd QNAP-CSI-PlugIn`
### Normal install
1. Run `kubectl apply -f Deploy/Trident/namespace.yaml`
2. Run `kubectl apply -f Deploy/Trident/trident_CRD.yaml`
3. Run `kubectl apply -f Deploy/Trident/bundle.yaml`
4. Run `kubectl apply -f Deploy/Trident/tridentorchestrator.yaml`

### Install by Helm 
1. Install Helm (for Ubuntu)
   - curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
   - sudo apt-get install apt-transport-https --yes
   - echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
   - sudo apt-get update
   - sudo apt-get install helm
2. Install CSI Plugin
Run `helm install qnap-trident ./qnap-trident -n trident --create-namespace`

## VolumeSnapshot Install (optional)
It's necessary for taking snapshot.
Run `kubectl apply -k VolumeSnapshot`

# CSI Configuration
## Backend.json
Add backend into orchestrator; it is essential before creating volume. Each column is required.
Edit the backend.json file `Samples/backend-qts1.json` or create a new one like the example below:
```
{
    "version": 1,
    "operatorVersion": "v1.0.0-beta",
    "storageVersion": "v1.0.0-beta",
    "storageDriverName": "qnap-iscsi",
    "backendName": "<BackendName>",
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

## StorageClass.yaml
Edit the backend.json file `Samples/storage-class-qnap-qos.yaml` or create a new one like the example below:
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium
provisioner: csi.trident.qnap.io #k8s CSI provisioner
parameters:
  selector: "performance=premium"
allowVolumeExpansion: true
```

## PVC.yaml
Edit the backend.json file `Samples/pvc-basic.yaml` or create a new one like the example below:
```
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

# Installation
## Add Backend
1. Make sure there have corresponded pool.
2. Run `chmod u+x tridentctl`
3. Add backend, run `./tridentctl create backend -f <backend.json> -n trident`
   - Example: `./tridentctl create backend -f Samples/backend-qts1.json -n trident`
   Running around 30 seconds
4. Check result:
   - Run `kubectl get pods -n trident`
   - Run `kubectl get qpools -n trident`

## Add StorageClass
1. Run kubectl apply -f <StorageClass.yaml>
   - Example: `kubectl apply -f Samples/storage-class-qnap-qos.yaml`
2. Check Result:
   - Run `kubectl get sc -n trident`

## Provision Volume Claim
### Add PVC
1. Run `kubectl apply -f <pvc.yaml>`
   - Example: `kubectl apply -f Samples/pvc-basic.yaml`
2. Check Result:
   - Run `kubectl get pvc`
   - Run `kubectl get qnapvolume -n trident`
   
### Clone PVC
1. Run `kubectl apply -f <pvc-clone.yaml>`
2. Check Result: `kubectl get pvc`

### Import PVC
1. Run `./tridentctl import volume <Backend Name> <LUN Name> -f <pvc-import.yaml> -n trident`
2. Check Result: `kubectl get pvc -n trident (or -A)`

### Create PVC from snapshot
1. Run `kubectl apply -f <vol-snapshot.yaml>`
   
**Notes:**
**Once pvc is created successfully, the corresponding qnapvolume will also be created which has the detail information of volume. The admin can also check whether the volume is created through UI.**

## Deploy Pod
1. Run `kubectl apply -f <pod.yaml>`
   - Example: `kubectl apply -f Samples/pod.yaml`
2. Check Result:
   - Run `kubectl get pods`
   
**Notes:**
**The image (davidcheng0922/docker-demo) just do time click and print it out; the above pod mounts the pvc we created. Use logs to check whether the pod works well.**

## Snapshot
### Create VolumeSnapshot from PVC
1. Run `kubectl apply -f <VolumeSnapshotClass.yaml>`
2. Run `kubectl apply -f <VolumeSnapshot.yaml> `
3. Check Result:
   - Run `kubectl get volumesnapshot`
   - Run `kubectl get qsnapshot -n trident`
   
**Notes:**
**Once volumesnapshot is created successfully, the corresponding qsnapshot is also created which record the snapshot information in the storage. The admin can also check whether the volume is created through UI.**

### Create PVC From Snapshot
The <pvc-from-snapshot.yaml> will contain the snapshot name we created above and created the volume depending on the snapshot. After volume is bound, we have to verify if the snapshot success; we create a new pod and mount the snapshot-pvc in pod. Access to the pod and check if the volume contains the directory we created before.  
1. Run `kubectl apply -f <pvc-from-snapshot.yaml>`
2. Run `kubectl apply -f <pod2.yaml>`

# Remove Trident
## Normal install
1. Run `kubectl delete deployment trident-operator -n trident`
2. Run `./tridentctl uninstall -n trident`
3. Run `kubectl delete tridentorchestrator trident`

## Install by Helm
Run `helm delete qnap-trident -n trident`
