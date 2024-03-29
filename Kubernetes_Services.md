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
