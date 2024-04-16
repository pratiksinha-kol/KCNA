## KUBERNETES StatefulSets

**[SOURCE](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)**

Kubernetes StatefulSets are workload API objects used to manage stateful applications. 
StatefulSets are used for stateful applications, and they maintain a sticky identity for each of their pods. DaemonSet are used to keep a copy of a pod on all the nodes inside the cluster, making them a great choice for node-level services. 
In StatefulSets, each of the pod creates its own claim with the StorageClass provider. Hence, for three pods, each of them will have three PVC and PV. StatefulSets are valuable for application that requires one or more of the following: 
- Stable / Unique network identifiers 
- Stable Persistent Storage 
- Ordered graceful deployment and Scaling
- Ordered Automated rolling update

** ** 
<br>

- There isn't a quick way of creating a StatefulSet from the CLI but a Deployment gets us very close, we'll mock up a Deployment and tee to a file
```bash
kubectl create deployment nginx --image=nginx --replicas=3 --dry-run=client -o yaml | tee statefulset.yaml
```

- And we'll modify this file (`statefulset.yaml`), to change it to a StatefulSet, we're also going to include a reference to `serviceName` to give it a stable network connection. Additionally, we changed the `kind` to `StatefulSet`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  serviceName: nginx      
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

- Apply this file and check statefulset
```bash
kubectl apply -f statefulset.yaml

k get sts OR kubectl get statefulset
```

- If we check our pods, these will have been named accordingly
```bash
kubectl get pods -o wide
```

- If a pod in a statefulset was removed, causing a restart, whilst the ip may change, the `name` will stay **consistent**. Delete pod/nginx-2 and then verify
```bash
kubectl delete pod/nginx-2 --now

kubectl get pods -o wide
```

- Let's make use of the statefulsets service functionality, we'll create a `headless` service called nginx
```bash
kubectl create service clusterip --clusterip=None nginx
```
**To query the DNS, use `<hostname>.<servicename>.<namespace>.svc.cluster.local`**

- Let see our endpoint slices
```bash
kubectl get endpoints
```

- The yaml output is great for viewing endpoints 
```bash
kubectl get endpoints/nginx -o yaml
```

The stable network ID is in the form of: `<hostname>.<servicename>.<namespace>.svc.cluster.local`. So, in our case it will be as follows: `nginx-0.nginx.default.svc.cluster.local`, `nginx-1.nginx.default.svc.cluster.local`, `nginx-2.nginx.default.svc.cluster.local`  

- Let's check this from the viewpoint of a pod, we'll run a curlimages/curl container
```bash
kubectl run --rm -i --tty curl --image=curlimages/curl:8.4.0 --restart=Never -- sh
```

- And let's curl our statefulset service name from inside the container: 
```bash
curl nginx-0.nginx.default.svc.cluster.local OR 
curl nginx-1.nginx.default.svc.cluster.local OR 
curl nginx-2.nginx.default.svc.cluster.local

exit
```

- Take a look at the statefulset yaml, you'll notice that the updateStrategy/rollingUpdate has a `partition value of 0`
```bash
kubectl get statefulset/nginx -o yaml | more
```

- We will change statefulset the partition value to 2
```bash
k edit statefulset nginx 

OR

kubectl patch statefulset/nginx -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
```

- We will set the image to nginx:alpine and watch the rollout
```bash
kubectl set image statefulset/nginx nginx=nginx:alpine && kubectl rollout status statefulset/nginx
```

- Because of the partition value, the rollout only took place on nginx-2, we can verify this with the following commands. This helps to test your application with canary deployment with new releases. 
```bash
kubectl describe pod/nginx-2 | more
```
Partition Counter starts at 0 when naming pods, i.e. 1st Pod[0]: nginx-0, 2nd Pod[1]: nginx-1, 3rd Pod[3]: nginx-2. When partition is set 2, it starts from 3rd Pod which is `nginx-2`.

** **

### StatefulSet Persistent Storage

This means that each Pods can have their own persistent volume. <br>
[REF](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#components)

- Update our statefulset to include a dynamic volume, i.e. `volumeMounts` and `volumeClaimTemplates`
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  serviceName: nginx      
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        volumeMounts:
        - name: nginx
          mountPath: /data 
  volumeClaimTemplates:
  - metadata:
      name: nginx
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 1Gi   
```

- We will delete our OLD statefulset and apply this new file 
```bash
kubectl delete -f statefulset.yaml && kubectl apply -f statefulset.yaml
```

- Watch our pods start, press ctrl-c to exit
```bash
watch --differences kubectl get pods -o wide
```

- We can verify that we have an individual pvc and pv for each pod
```bash
kubectl get pvc

kubectl get pv
```

- We'll execute a shell into nginx-0 and create some data
```bash
kubectl exec -it nginx-0 -- bash

cd /data; touch this_is_nginx-0; ls -l

exit
```
- Let's delete the pod and watch it be recreated
```bash
kubectl delete pod/nginx-0 --now; watch --differences kubectl get pods -o wide
```

- We execute a shell again and see if our data persists which we had created earlier. 
```bash
kubectl exec -it nginx-0 -- bash

ls /data

exit
```


- And we can take this further, we can delete the entire statefulset 
```bash
kubectl delete -f statefulset.yaml
```


- Our pvc and pv will still exist
```bash
kubectl get pvc

kubectl get pv
```


- If we recreate the statefulset it will use the same PV and PVC. Lets test this out by apply the file again
```bash
kubectl apply -f statefulset.yaml
```


- check our pods
```bash
kubectl get pods -o wide
```


- Again we will exec into our pods and check for the data folder to see the file we had created. Remember it is persistent storage, so data will persist: 
```bash
kubectl exec -it nginx-0 -- bash

ls /data

exit
```

- Cleanup
```bash
kubectl delete statefulset/nginx --now
for i in 0 1 2; do kubectl delete pvc/nginx-nginx-$i --now; done; kubectl delete service/nginx; rm statefulset.yaml
```