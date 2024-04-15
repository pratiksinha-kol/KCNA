## KUBERNETES SCHEDULING AND NODENAME

Kube-Scheduler has three main set operations:
1. Filtering: If a pod requires more memory than a certain node has available, that node will be filtered out. 
2. Scoring: The scheduler scores the individual nodes to find the most appropriate node. 
3. Binding: A pod is bound to a node using the Binding object. It instructs the kubelet on that node to start the pod. The name of the node is stored in the `node name` field of the pod.  
Since, kube-scheduler is a modular system, it is possible to create and run your own Schedulers. 

- Let's begin by templating an nginx pod that we'll later target with a custom scheduler 
```bash
kubectl run nginx --image=nginx -o yaml --dry-run=client | tee nginx_scheduler.yaml
```

- We are interested in a pod.spec option known as schedulerName, use kubectl explain to see more information on this
```bash
kubectl explain pod.spec | more
```
_`schedulerName <string>` : If specified, the pod will be dispatched by specified scheduler. If not
    specified, the pod will be dispatched by default scheduler._

- We will edit the created file and add `schedulerName` in the pod definition file. FYI we will create `my-scheduler`. 
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Apply this file and you will notice that pod will be stuck in pending as there is no scheduler called my-scheduler running 
```bash
kubectl apply -f nginx_scheduler.yaml

kubectl get pods -o wide
```

- We're going to make use of a convenient code example that mimicks the functionality of a scheduler as a bash script, firstly install git and jq if required 
```bash
apt update && apt install -y git jq
```

- Clone this project
```bash
git clone https://github.com/spurin/simple-kubernetes-scheduler-example.git

cd simple-kubernetes-scheduler-example; more my-scheduler.sh
```

- And if you're interested in seeing how the json traversal in the script works, take a look at the json as follows
```bash
kubectl get nodes -o json
```

- If we run this scheduler, it should schedule our pending pod, press ctrl-c when done
```bash
./my-scheduler.sh
```

- Now if we check our pods, this will now be scheduled 
```bash
kubectl get pods -o wide
```

- CleanUp
```bash
kubectl delete pod/nginx --now
```
** **

## nodeName

`nodeName <string>`: NodeName is a request to schedule this pod onto a specific node. If it is
    non-empty, the scheduler simply schedules this pod onto that node, assuming
    that it fits resource requirements.

```bash
kubectl explain pod.spec | more
```

An alternative way of scheduling is by directly specifying the `nodeName` in which to schedule the pod, firstly go back to the previous directory. `cd ..`

- Update the yaml so that it directly specifies a `nodeName` of worker-2
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  nodeName: worker-2
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Apply this file and see it running as pods is scheduled to `worker-2`
```bash
kubectl apply -f nginx_scheduler.yaml

kubectl get pods -o wide
```

- CleanUp
```bash
kubectl delete pod/nginx --now
rm -rf nginx_scheduler.yaml simple-kubernetes-scheduler-example/
```

** ** 

## nodeSelector

An alternative approach is the use of nodeSelector which makes use of known labels, to select a node. It is also possible to use a more targeted approach than NodeName through the use of NodeSelectors, the approach is similar but, instead of a direct node we will make use of Kubernetes labels to identify our desired target. 

`nodeSelector  <map[string]string>`: NodeSelector is a selector which must be true for the pod to fit on a node.
    Selector which must match a node's labels for the pod to be scheduled on
    that node. More info:
    https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

- If you run a describe on the worker-1 node you'll see the `labels` section at the top. We will target the `label: kubernetes.io/hostname=worker-1`
```bash
kubectl describe node/worker-1 | more
```

- create the file if deleted....
```bash
kubectl run nginx --image=nginx -o yaml --dry-run=client | tee nginx_scheduler.yaml
```


-Edit the file to add `nodeSelector`
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-1
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```
- Apply this update and check, this will have been scheduled to `worker-1`
```bash
kubectl apply -f nginx_scheduler.yaml

kubectl get pods -o wide
```

- CleanUp
```bash
kubectl delete pod/nginx --now; rm -rf simple-kubernetes-scheduler-example; rm -rf nginx_scheduler.yaml
```