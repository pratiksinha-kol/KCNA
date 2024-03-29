## KUBERNETES DEPLOYMENT

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