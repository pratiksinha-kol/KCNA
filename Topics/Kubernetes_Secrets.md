## KUBERNETES SECRETS

It is important to understand that secrets are not encrypted, they are encoded with base64. To know more about it, use `k create secret --help`. 
It has 3 Available Commands:
| Command | Description |
| ----    |  -----      | 
  docker-registry |  Create a secret for use with a Docker registry
  generic         |  Create a secret from a local file, directory, or literal value
  tls             |  Create a TLS secret 

- Create a secret using the --from-literal approach which is identical in usage to that of ConfigMaps, send the output to yaml as a dry-run and note the encoded values for COLOUR and KEY
```bash
kubectl create secret generic colour-secret --from-literal=COLOUR=red --from-literal=KEY=value --dry-run=client -o yaml
```

```yaml
apiVersion: v1
data:
  COLOUR: cmVk
  KEY: dmFsdWU=
kind: Secret
metadata:
  creationTimestamp: null
  name: colour-secret
```

- As discussed, this is a base64 encoded value. Confirm it by running these commands 
```bash
echo -n red | base64
echo -n value | base64
```

- Create secret
```bash
kubectl create secret generic colour-secret --from-literal=COLOUR=red --from-literal=KEY=value
```

- Get secrets and show it in yaml format
```bash
k get secrets

k get secrets -o yaml
```

```yaml
apiVersion: v1
items:
- apiVersion: v1
  data:
    COLOUR: cmVk
    KEY: dmFsdWU=
  kind: Secret
  metadata:
    creationTimestamp: "2024-03-29T13:27:30Z"
    name: colour-secret
    namespace: default
    resourceVersion: "3867"
    uid: de03c320-93f1-4603-a3ee-2f50c7673cbd
  type: Opaque
kind: List
metadata:
  resourceVersion: ""
```

- Lets create ubuntu pod that sleeps for infinity and dumps environment variables, this time we will use envFrom with secretRef.
```bash
kubectl run --image=ubuntu --dry-run=client --restart=Never -o yaml ubuntu --command bash -- -c 'env; sleep infinity' | tee env-dump-pod.yaml

vi env-dump-pod.yaml 
```
*we will use envFrom with secretRef*
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
    - secretRef:
        name: colour-secret
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}
```

- Apply the secret file to create a new pod
```bash
kubectl apply -f env-dump-pod.yaml
```

- check the logs to see the secrets in simple text
```bash
k logs ubuntu 
```

- Cleanup
```bash
kubectl delete pod/ubuntu secret/colour-secret --now

rm env-dump-pod.yaml
```