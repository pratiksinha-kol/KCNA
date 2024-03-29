## KUBERNETES LABELS

- Show the yaml output for an nginx pod, note the default label of `'run:nginx'` & create it
```bash
kubectl run nginx --image nginx --port 80 -o yaml --dry-run=client

kubectl run nginx --image nginx --port 80
```

- Review the yaml for exposing this pod, note that the selector references 'run:nginx'. You will notice that the `label` is used as `selector` & finally expose it
```bash
kubectl expose pod/nginx --dry-run=client -o yaml

kubectl expose pod/nginx
```

- As we are using label `run:nginx`, we can fetch all the information about it. 
```bash
kubectl get all --selector run=nginx
```

- Let create three pods 
```bash
YAML=$(kubectl run --image=ubuntu ubuntu --dry-run=client -o yaml --command sleep infinity)

echo -e "${YAML}\n---\n${YAML}\n---\n${YAML}"

vi coloured_pods.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
    colour: red  
  name: ubuntu-red
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
    colour: green  
  name: ubuntu-green
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
---
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
    colour: yellow  
  name: ubuntu-yellow
spec:
  containers:
  - command:
    - sleep
    - infinity
    image: ubuntu
    name: ubuntu
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Create Pods using the above pods definiton file
```bash
kubectl apply -f coloured_pods.yaml
```

- We can specifically select resources based on that selector
```bash
kubectl get all --selector colour=green

kubectl get all -l colour=green

kubectl get all -l colour=red
kubectl get all -l colour=yellow
```