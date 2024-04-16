## KUBERNETES STORAGE

There are two types of storage in K8s, Ephemeral and Persistent. 

When a pod ceases to exist, Kubernetes destroys ephemeral volumes; however, Kubernetes does not destroy persistent volumes. For any kind of volume in a given pod, data is preserved across container restarts. At its core, a volume is a directory, possibly with some data in it, which is accessible to the containers in a pod. How that directory comes to be, the medium that backs it, and the contents of it are determined by the particular volume type used.

**PersistentVolume (PV):** It represents a piece of storage in the cluster that has been provisioned by the administrator or dynamicaaly using StorageClass. Its resources in the cluster that persists beyond pods lifecycle. 

**PersistentVolumeclaim (PVC):** A request for storage by a user. It specifies the amount and characteristics of storage needed by a a pod. Once bounded by a PV, the PVC can be used by a a pod to access the storage.  

**StorageClass:** Defines the type and characteristics of storage provided by the cluster. It allows administrator to describe the `classes` of storage they offer. 

[SOURCE](https://kubernetes.io/docs/concepts/storage/)

### Ephemeral Volumes
- We'll begin by capturing ubuntu pod yaml with sleep infinity as a command
```bash
kubectl run --image=ubuntu ubuntu -o yaml --dry-run=client --command sleep infinity | tee ubuntu_emptydir.yaml
```
_FYI: For a Pod that defines an `emptyDir` volume, the volume is created when the Pod is assigned to a node. As the name says, the `emptyDir` volume is initially empty. All containers in the Pod can read and write the same files in the `emptyDir` volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the `emptyDir` is deleted permanently._

_The `emptyDir.medium` field controls where `emptyDir` volumes are stored. By default `emptyDir` volumes are stored on whatever medium that backs the node such as disk, SSD, or network storage, depending on your environment. If you set the `emptyDir.medium` field to `"Memory"`, Kubernetes mounts a `tmpfs` (RAM-backed filesystem) for you instead. While `tmpfs` is very fast be aware that, unlike disks, files you write count against the memory limit of the container that wrote them._

- Update this specification so that it includes a `volumeMounts`, `volumes` and an `emptyDir` volume. Check [emptyDir ref](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
    volumeMounts:
    - mountPath: /cache
      name: cache-volume  
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
  - name: cache-volume
    emptyDir: {
      medium: Memory,
    }  
status: {}
```
- Apply the file and check our pod 
```bash
kubectl apply -f ubuntu_emptydir.yaml

kubectl get pods -o wide
```

- We'll hop into this pod with an interactive shell and if we go into cache and do a df -h, we can see that the filesytem being used is `tmpfs`
```bash
kubectl exec -it ubuntu -- bash

cd cache; df -h .
```

- With this being a memory based filesystem, it will be highly performant. Let's test this using dd
```bash
dd if=/dev/zero of=output oflag=sync bs=1024k count=1000

exit
```

- When we get rid of this container it will also remove the emptyDir volume
```bash
kubectl delete pod/ubuntu --now
```
** ** 

### Persistent Storage

A _PersistentVolume_ (PV) is a piece of storage in the cluster that has been provisioned by an administrator or dynamically provisioned using Storage Classes. It is a resource in the cluster just like a node is a cluster resource. PVs are volume plugins like Volumes, but have a lifecycle independent of any individual Pod that uses the PV. This API object captures the details of the implementation of the storage, be that NFS, iSCSI, or a cloud-provider-specific storage system.

A _PersistentVolumeClaim_ (PVC) is a request for storage by a user. It is similar to a Pod. Pods consume node resources and PVCs consume PV resources. Pods can request specific levels of resources (CPU and Memory). Claims can request specific size and access modes (e.g., they can be mounted ReadWriteOnce, ReadOnlyMany, ReadWriteMany, or ReadWriteOncePod, see AccessModes).



[SOURCE](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

- Let's take a look at the storageclasses available in our cluster, we will have a single storageclass and it will be marked as default
```bash
kubectl get storageclass

k get sc
```

- Familiarise yourself with the storageclass specication, in particular the reclaimPolicy section
```bash
kubectl explain storageclass | more
```

- We'll create a persistent volume using the k3s storageClass of local-path `manual_pv.yaml`. [REFERENCE](https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/#create-a-persistentvolume)
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: manual-pv001
spec:
  storageClassName: local-path
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/var/lib/rancher/k3s/storage/manual-pv001"
    type: DirectoryOrCreate
```

- Apply this file and Check the available persistent volumes and note that the reclaim policy is set to `Retain`
```bash
kubectl apply -f manual_pv.yaml

kubectl get pv
```

- Let's now create a manual persistent volume claim (PVC) against this persistent volume `manual_pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: manual-claim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
  storageClassName: local-path
  volumeName: manual-pv001
```

- Apply this file and check the persistent volume claim
```bash
kubectl apply -f manual_pvc.yaml

kubectl get persistentvolumeclaim OR k get pvc
```
** ** 

### Dynamic PVC

For this we don't have to create a persistent volume as it is done dynamically. 

- Let's create a dynamic version of our Persistent Volume Claim file (`dynamic_pvc.yaml`), this will be similar to our manual approach but without a `volumeName` entry as we did previously
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
  storageClassName: local-path
```

- Apply this file and check our PVC
```bash
kubectl apply -f dynamic_pvc.yaml

k get pvc
```

- If we run a describe on the pvc, we will see that it is awaiting a consumer before it is bound (Status `pending`)
```bash
kubectl describe pvc/dynamic-claim

k get pv
```

- Let's mock up a ubuntu pod with sleep infinity as a starting base
```bash
kubectl run --image=ubuntu ubuntu -o yaml --dry-run=client --command sleep infinity | tee ubuntu_with_volumes.yaml
```


- And we'll modify this file `ubuntu_with_volumes.yaml` to include volumeMounts and volumes for both our manual and dynamic pvc's [REFERENCE](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#claims-as-volumes)
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
    volumeMounts:
    - mountPath: /manual
      name: manual-volume
    - mountPath: /dynamic
      name: dynamic-volume  
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
    - name: manual-volume
      persistentVolumeClaim:
        claimName: manual-claim
    - name: dynamic-volume
      persistentVolumeClaim:
        claimName: dynamic-claim  
status: {}
```

- As the k3s storageclass provisioner is very basic, we should use a nodeSelector to guide this pod to a specific node, check the labels for node/worker-1 (i.e kubernetes.io/hostname=worker-1)
```bash
kubectl describe node/worker-1 | more
```
Update the ubuntu_with_volumes.yaml` file: 
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-1
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
    volumeMounts:
    - mountPath: /manual
      name: manual-volume
    - mountPath: /dynamic
      name: dynamic-volume  
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:
    - name: manual-volume
      persistentVolumeClaim:
        claimName: manual-claim
    - name: dynamic-volume
      persistentVolumeClaim:
        claimName: dynamic-claim  
status: {}
```

- We can apply this file and check the pod
```bash
kubectl apply -f ubuntu_with_volumes.yaml

kubectl get pod
```

- Now check PV and PVC, both of them are bound 
```bash
k get pv

k get pvc
```
_Manual PV `reclaim policy` is set as `retain` whereas for dynamic volume, it is set as `delete`._

- And let's write data to our volumes, access the pod with an interactive shell
```bash
kubectl exec -it ubuntu -- bash
```

- And if we check both of these, they are mounted volumes
```bash
cd /manual; df -h .

cd /dynamic; df -h .
```

- Let's write a text file to each of these and we'll exit
```bash
echo hello > /manual/test.txt

echo hello > /dynamic/test.txt

exit
```

- As these are persistent volumes, if we delete the pod and recreate it
```bash
kubectl delete pod/ubuntu --now && kubectl apply -f ubuntu_with_volumes.yaml
```

- And again access this pod
```bash
kubectl exec -it ubuntu -- bash
```

- We can see that our files will still exist as our volumes are persistent 
```bash
cat /manual/test.txt; cat /dynamic/test.txt

exit
```

- Let's see what happens when we delete the pod and pvc's
```bash
k get pv

k get pvc 

kubectl delete pod/ubuntu pvc/dynamic-claim pvc/manual-claim --now
```

**As the retain policy for the dynamic persistent volume was `Delete`, it will be deleted whereas for manual persistent volume, it is retained.** 

- Cleanup
```bash
kubectl delete pv/manual-pv001 --now
rm -rf dynamic_pvc.yaml manual_pv* ubuntu_emptydir.yaml ubuntu_with_volumes.yaml
```

** **
<br>
Although Rook is a Cloud Native solution on the CNCF Landscape, for the KCNA examination you should also be aware of Ceph, an offering from RedHat that provides object, block and file storage in one solution.

Given the commercial nature of Ceph's relationship with RedHat, it is sometimes used as a storage solution in the OpenShift distribution of Kubernetes. From a KCNA viewpoint you should be aware of the project and it being a unified object/block and file solution.

[REF 1](https://www.redhat.com/en/technologies/storage/ceph) <br>
[REF 2](https://docs.openshift.com/container-platform/3.11/install_config/storage_examples/ceph_example.html)
