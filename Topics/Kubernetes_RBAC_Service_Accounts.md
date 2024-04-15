## KUBERNETES RBAC SERVICE ACCOUNTS

It is also worth being aware of ServiceAccounts and their relation to Pods and how these operate behind the scenes.

Each Pod in Kubernetes is automatically associated with a Service Account. This association is key to determining the permissions and resources that the processes inside the Pod will have access to.

If you don’t specify a Service Account for a Pod, it defaults to using the 'default' Service Account in the same namespace and the Pod will have limited access to the Kubernetes API from within the Pod. This adheres to the Kubernetes best practice principle of least privilege.

In everyday operations, you would not need to make use of a custom Service Account unless you wished for the Pod in Kubernetes to be able to perform specific operations against the Kubernetes cluster from within the Pod. For example, a Pod being permitted to create another Pod/Deployment/Secret etc.

- We'll start by creating a simple pod with curlimages/curl 
```bash
kubectl run curl --image=curlimages/curl:8.4.0 -- sleep infinity
```

- Inspect the pod, you will see Service Account at the top and it will be assigned the value of default
```bash
kubectl describe pod/curl | more
```

- Kubectl exec into the curlimages/curl container with a sh shell
```bash
kubectl exec -it curl -- sh
```

- We can capture the default ServiceAccount token that has been assigned to the pod by accessing the token file that is available through `/var/run/secrets/kubernetes.io`, all pods receive API information in this way, via the pods filesystem 
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

echo $TOKEN
```

- Using this token, we can query the API using curl, for example we could query the `/api/v1/pods` endpoint. We will use the convenient DNS name of `https://kubernetes.default.svc` which is setup by default for convenience to allow pods to communicate with the API. If you attempt this at this stage, access will be restricted as there are no access permissions with the default account
```bash
curl --header "Authorization: Bearer $TOKEN" --insecure https://kubernetes.default.svc/api/v1/pods
```

- Exit the pod, and clean-up
```bash
exit

kubectl delete pod/curl --now
```

- Let’s create a pod-serviceaccount 
```bash
kubectl create serviceaccount pod-serviceaccount
```

- Create a ClusterRole or Role as per our video with the desired resources and verbs. We’ll re-use the example we used for deadpool
```bash
kubectl create clusterrole cluster-pod-manager --verb=list,get,create,delete --resource='pods'
```

- Create a ClusterRoleBinding/RoleBinding, again we’ll re-use the deadpool example but instead of specifying a user, we’ll specify a serviceaccount, the format for this is namespace:serviceaccount
```bash
kubectl create clusterrolebinding cluster-pod-manager --clusterrole=cluster-pod-manager --serviceaccount=default:pod-serviceaccount
```

- This time we’ll re-create the Pod but we’ll specify our serviceaccount. Sadly it isn’t possible to specify a serviceaccount as part of the CLI but, we can inject this into the yaml on the fly as follows
```bash
kubectl run curl --image=curlimages/curl:8.4.0 --overrides='{ "spec": { "serviceAccount": "pod-serviceaccount" } }' -- sleep infinity
```

- If we describe the pod, this time we will see pod-serviceaccount as the service account
```bash
kubectl describe pod/curl | more
```

- Kubectl exec into the curlimages/curl container with an sh shell
```bash
kubectl exec -it curl -- sh
```

- Again, capture the default ServiceAccount token that has been assigned to the pod by accessing the token file that is available through /var/run/secrets/kubernetes.io
```bash
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

echo $TOKEN
```

- Cleanup
```bash
exit

kubectl delete pod/curl serviceaccount/pod-serviceaccount clusterrole/cluster-pod-manager clusterrolebinding/cluster-pod-manager --now

```

- 
```bash

```

- 
```bash

```

- 
```bash

```

- 
```bash

```

- 
```bash

```