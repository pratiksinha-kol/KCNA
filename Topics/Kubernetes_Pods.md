## KUBERNETES PODS


- Create a combination of two pods definition file into a single one. (`nginx.yaml` and `ubuntu.yaml` are two files that creates a single yaml definition file `combined.yaml`) 

```
{ cat nginx.yaml; echo "---"; cat ubuntu.yaml; } | tee combined.yaml
```

- Create pods
```
k apply -f combined.yaml
```

- Delete pods
```
k delete -f combined.yaml --now
```


#### CURL Docker Image 

```
k run --rm -it curl --image=curlimages/curl --restart=Never -- http://10.10.0.12
```

#### ANOTHER COMBINED SAMPLE

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: mypod
  name: mypod
spec:
  containers:
  - name: webserver
    image: nginx
    resources: {}
  - name: sidecar
    image: ubuntu
    args:
    - /bin/sh
    - -c
    - while true; do echo "$(date +'%T') - Hello From sidecar"; sleep 5; if [ -f /tmp/crash ]; then exit 1; fi; done
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}    
```

- You can get IP and curl it: 

```sh
k get pods mypod -o wide
k run --rm -it curl --image=curlimages/curl --restart=Never -- http://10.10.0.12
```

- To simulate crash as mentioned above in pod definition file
```sh
 k exec mypod -c sidecar -it -- touch /tmp/crash
```


- To get logs from sidecar (command below uses the name mentioned above in pod definition file):

```sh
k logs mypod -c sidecar
``` 

- You can also see previous logs by: 

```sh
k logs pod/mypod -p -c sidecar
```
