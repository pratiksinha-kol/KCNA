## Kubernetes ConfigMaps


- Create a ConfigMap with two entries using the --from-literal parameter
```bash
kubectl create configmap NAME [--from-file=[key=]source] [--from-literal=key1=value1] [--dry-run=server|client|none]

kubectl create configmap colour-configmap --from-literal=COLOUR=red --from-literal=KEY=value
```

- To know more about the configap and delete it
```bash
kubectl describe configmap/colour-configmap 

OR

k describe cm colour-configmap 

kubectl delete configmap/colour-configmap
```

- Create file for configmap
```bash
cat <<EOF > configmap-colour.properties
COLOUR=green
KEY=value
EOF


cat configmap-colour.properties
```

- Create configmap via file
```
kubectl create configmap colour-configmap --from-env-file=configmap-colour.properties
```

- Let's capture the yaml for a pod that dumps environment variables, and then sleeps for infinity 
```bash
kubectl run --image=ubuntu --dry-run=client --restart=Never -o yaml ubuntu --command bash -- -c 'env; sleep infinity' | tee env-dump-pod.yaml
```

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
    - bash
    - -c
    - env; sleep infinity
    image: ubuntu
    name: ubuntu
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

- Apply the above configuration
```bash
kubectl apply -f env-dump-pod.yaml
```

- Update our pod yaml (env-dump-pod.yaml) so that it includes an envFrom, configMapRef
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
    - bash
    - -c
    - env; sleep infinity
    image: ubuntu
    name: ubuntu
    resources: {}
    envFrom:
    - configMapRef:
        name: colour-configmap  
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```
- If we attempt to apply this at present it will fail as we can only update certain fields
```
kubectl apply -f env-dump-pod.yaml
```

- To apply the above file successfully 
```bash
kubectl delete -f env-dump-pod.yaml --now; kubectl apply -f env-dump-pod.yaml

kubectl logs ubuntu
```

- ConfigMaps provide a centralised location for configuration data on our cluster, edit the current configmap and change the colour to red/green


```bash
kubectl edit configmap/colour-configmap

kubectl delete -f env-dump-pod.yaml --now; kubectl apply -f env-dump-pod.yaml

kubectl logs ubuntu
```

- Let's make our ConfigMap immutable. Update the configmap and add 'immutable: true' without any indentation in the file on a new line, also change the colour to purple

```bash
kubectl edit configmap/colour-configmap

kubectl delete -f env-dump-pod.yaml --now; kubectl apply -f env-dump-pod.yaml

kubectl logs ubuntu
```

```yaml
apiVersion: v1
data:
  COLOUR: purple
  KEY: value
immutable: true
kind: ConfigMap
metadata:
  creationTimestamp: "2024-03-29T12:01:40Z"
  name: colour-configmap
  namespace: default
  resourceVersion: "1932"
  uid: a2a9d6af-dbec-4178-b1c4-a0d2fad5ae57
```

- Cleanup
```bash
kubectl delete pod/ubuntu configmap/colour-configmap --now
```