## PODS


Create a combination of two pods definition file into a single one. (`nginx.yaml` and `ubuntu.yaml` are two files that creates a single yaml definition file `combined.yaml`) 

```
{ cat nginx.yaml; echo "---"; cat ubuntu.yaml; } | tee combined.yaml
```

Create pods
```
k apply -f combined.yaml
```

Delete pods
```
k delete -f combined.yaml --now
```


### CURL

```
k run --rm -it curl --image=curlimages/curl --restart=Never -- http://10.10.0.12
```

### ANOTHER COMBINED SAMPLE

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

You can get IP and curl it: 

```sh
k get pods mypod -o wide
k run --rm -it curl --image=curlimages/curl --restart=Never -- http://10.10.0.12
```

To simulate crash as mentioned above in pod definition file
```sh
 k exec mypod -c sidecar -it -- touch /tmp/crash
```


To get logs from sidecar (command below uses the name mentioned above in pod definition file):

```sh
k logs mypod -c sidecar
``` 

You can also see previous logs by: 

```sh
k logs pod/mypod -p -c sidecar
```


## DEPLOYMENT

- Generate `deployment` definition yaml file
```sh
k create deployment nginx --image=nginx --dry-run=client -o yaml | tee nginx.yaml
```

- Generate `deployement` definition yaml file and also apply it
```sh
k create deployment nginx --image=nginx --dry-run=client -o yaml | tee nginx.yaml | k apply -f -

k get deployment
k get rs
k get po
```

- Check rollout history
```sh
k rollout history deployment nginx
OR
kubectl rollout history deployment/nginx
```

- To describe the rollout, you need to annotate it [LINK](https://kubernetes.io/docs/reference/labels-annotations-taints/)
```sh
kubectl annotate deployment/nginx kubernetes.io/change-cause="initial deployment"
```

- Now check the rollout history, you will notice the annotation
```sh
kubectl rollout history deployment/nginx
```
- Scale the deployment
```sh
k scale deployment nginx --replicas=10

k scale deployment nginx --replicas=10; k get pods -o wide --watch
```

- You can also scale by editing the deployment file (under `replicas`)
```sh
k edit deployments.apps nginx 
k edit deployment/nginx 
```

OR by editing the yaml file we had generated in the first step 

```
vi nginx.yaml
```

- It doesn't affect the rollout history
```sh
k rollout history deployment nginx

OR

k rollout history deployment/nginx
```

- To check rolling stratergy
```sh
k describe deployments.apps nginx 

OR 

kubectl get deployment/nginx -o yaml
```

- Now, if you modify the image in the yaml definition file, and check the rollut status, you can see the difference. 
```sh
kubectl apply -f nginx-deployment.yaml && kubectl rollout status deployment/nginx
```

- You can annotate the changes and see the hsitory.
```sh
kubectl annotate deployment/nginx kubernetes.io/change-cause="Change image to nginx:stable"

k rollout history deployment nginx 
```

- We can also change a deployment image, using the CLI. Set the image to alpine and watch the rollout status. (Previous two step in another way)
```sh
kubectl set image deployment/nginx nginx=nginx:alpine && kubectl rollout status deployment/nginx

kubectl annotate deployment/nginx kubernetes.io/change-cause="change image to nginx:alpine"

kubectl rollout history deployment/nginx
```

- Now set the image to perl and see:  
```sh
kubectl set image deployment/nginx nginx=nginx:perl && watch kubectl get pods -o wide

kubectl annotate deployment/nginx kubernetes.io/change-cause="change image to nginx:perl"

kubectl rollout history deployment/nginx

kubectl describe deployment/nginx
```

- Now to test the rollback feature by using a wrong image:  
```sh
kubectl set image deployment/nginx nginx=nginx:bananas && watch kubectl get pods -o wide
kubectl annotate deployment/nginx kubernetes.io/change-cause="change image to nginx:bananas - failed"
kubectl rollout history deployment/nginx

kubectl rollout undo deployment/nginx && kubectl rollout status deployment/nginx
```

- Undo the rollout
```sh
kubectl rollout undo deployment/nginx && kubectl rollout status deployment/nginx

OR

k rollout undo deployment/nginx 

```

- We can also implicity rollback to a specific revision. If we rollback to the initial deployment which is nginx alone, we’ll see that the first replicaSet, will become the replicaSet in use

```sh
kubectl rollout undo deployment/nginx --to-revision=1 && kubectl rollout status deployment/nginx

kubectl get replicaset -o wide
```

## KUBERNETES SERVICES


### ClusterIP Service
- Creating an nginx deployment using the spurin/nginx-debug image, we'll use 3 replicas. We will expose a port as well

```sh
kubectl create deployment nginx --image=spurin/nginx-debug --port=80 --replicas=3 -o yaml --dry-run=client

kubectl create deployment nginx --image=spurin/nginx-debug --port=80 --replicas=3 (DEPLOYENT)
```

- We're going to expose this as a ClusterIP service, firstly, let's review the yaml declaration. After review, deploy it. Check the `selector` under `spec` along with the `port` information, it references the label we have declared previously automatically. 
```sh
kubectl expose deployment/nginx --dry-run=client -o yaml
kubectl expose deployment/nginx

kubectl get service
```

- It also create endpoints. Pods endpoint are available. 
```sh
kubectl get endpoints

kubectl get pods -o wide

kubectl describe service/nginx
```

- To use the service (ClusterIP is only available locally)
```sh
CLUSTER_IP=$(kubectl get services | grep ClusterIP | grep nginx | awk {'print $3'}); echo $CLUSTER_IP
curl $CLUSTER_IP
```

The same can be accessed from another pod which we will create and curl the port from it. The last command is possible because Pods automatically inherit a DNS search path relating to the service, if we check /etc/resolv.conf, we can see this search path. 
```sh
kubectl run --rm -it curl --image=curlimages/curl:8.4.0 --restart=Never -- sh

curl nginx.default.svc.cluster.local

curl nginx
```
- Delete the service nginx
```sh
k delete svc nginx
```

### NodePort Service

- We are going to re-create our service but as a NodePort
```sh
kubectl expose deployment/nginx --type=NodePort
```
- If we check our services, we will see this listed as a NodePort with two ports, the first being for the application and the second being the NodePort Port
```sh
kubectl get service
```

- View the node information
```sh
kubectl get nodes -o wide
```

- We'll capture the required information, firstly the IP of our control-plane node and designated Nodeport
```sh
CONTROL_PLANE_IP=$(kubectl get nodes -o wide | grep control-plane | awk {'print $6'}); echo $CONTROL_PLANE_IP

NODEPORT_PORT=$(kubectl get services | grep NodePort | grep nginx | awk -F'[:/]' '{print $2}'); echo $NODEPORT_PORT
```
- Using both of these, we can query the NodePort service from one of the Nodes
```sh
curl ${CONTROL_PLANE_IP}:${NODEPORT_PORT}
```

- Delete the service nginx
```sh
k delete svc nginx
```

## LoadBalancer Service

- Create a LoadBalancer service 
```sh
kubectl expose deployment/nginx --type=LoadBalancer --port 8080 --target-port 80

kubectl get service
```

- watch the curl connections to the LoadBalancer
```sh
LOADBALANCER_IP=$(kubectl get service | grep LoadBalancer | grep nginx | awk '{split($0,a," "); split(a[4],b,","); print b[1]}'); echo $LOADBALANCER_IP

LOADBALANCER_PORT=$(kubectl get service | grep LoadBalancer | grep nginx | awk -F'[:/]' '{print $2}'); echo $LOADBALANCER_PORT

watch --differences "curl ${LOADBALANCER_IP}:${LOADBALANCER_PORT} 2>/dev/null"


watch --differences "curl 172.20.0.2:30115 2>/dev/null"
```

- Let's scale down our deployment to 1 replica and watch connections. Notice that `Same` pod is being used
```sh
kubectl scale deployment/nginx --replicas=1; watch --differences "curl ${LOADBALANCER_IP}:${LOADBALANCER_PORT} 2>/dev/null"
```

- Let's scale up our deployment to 3 replicas and watch connections
```sh
kubectl scale deployment/nginx --replicas=3; watch --differences "curl ${LOADBALANCER_IP}:${LOADBALANCER_PORT} 2>/dev/null"
```

- Delete deployment and service
```sh
kubectl delete deployment/nginx service/nginx
```

## ExternalName Service

- Firstly create a deployment called nginx-red and another with name nginx-blue

```sh
kubectl create deployment nginx-red --image=spurin/nginx-red --port=80
kubectl create deployment nginx-blue --image=spurin/nginx-blue --port=80
```

- Expose nginx-red and nginx-blue deloyment
```sh
kubectl expose deployment/nginx-red
kubectl expose deployment/nginx-blue

k get svc -o wide
```

- Lets create an ExternalName service of my-service that points to nginx-red
```sh
kubectl create service externalname my-service --external-name nginx-red.default.svc.cluster.local

kubectl get service
```

- Let's take a look at this from the viewpoint of a pod, we'll execute a curl pod
```sh
kubectl run --rm -it curl --image=curlimages/curl:8.4.0 --restart=Never -- sh

curl nginx-red

curl nginx-blue

curl my-service
```

- If we take a look with nslookup, we can see that my-service is a CNAME for nginx-red.default.svc.cluster.local
```sh
nslookup my-service
```

- Open another tab, and edit the service and change it to blue (under `externalName`)
```sh
kubectl edit service/my-service
```

- Go to the previus tab where you ran the nslook and rerun it. It will point to blue `canonical name = nginx-blue.default.svc.cluster.local`
```sh
nslookup my-service
```

- Clean up our deployments and services
```sh
kubectl delete deployment/nginx-blue deployment/nginx-red service/nginx-blue service/nginx-red service/my-service
```

## Headless Service

- We will now take a look at Headless services which are a variation of ClusterIP, let's create a deployment using spurin/nginx with 3 replicas and a specified port of 80
```sh
kubectl create deployment nginx --image=spurin/nginx-debug --replicas=3 --port=80
```

- To create a headless service, we need to modify a ClusterIP Yaml Declaration, capture the yaml from a basic ClusterIP Deployment and save to headless.yaml
```sh
kubectl expose deployment/nginx --dry-run=client -o yaml --type=ClusterIP | tee headless.yaml
```

- Edit the generated file set the `ClusterIP` as `None`
```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  clusterIP: None
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
status:
  loadBalancer: {}
```

- Apply this file and see the service. Notice the output that `Cluster-IP` is `None`
```sh
kubectl apply -f headless.yaml 

kubectl get service
```

- But lets curl in it to see it: If we watch the output of nslookup nginx, we will see this dynamically changing
```sh
kubectl run --rm -it curl --image=curlimages/curl:8.4.0 --restart=Never -- sh

watch nslookup nginx
exit
```

- Cleanup
```sh
rm headless.yaml; kubectl delete deployment/nginx service/nginx
```

- 
```sh

```
