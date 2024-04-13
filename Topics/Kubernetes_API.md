## KUBERNETES API 

[Kubernetes API Part 2 Video](https://www.udemy.com/course/dive-into-cloud-native-containers-kubernetes-and-the-kcna/learn/lecture/42081048#overview)

- We can see the API requests that are being made by adding verbosity flags 

```bash
kubectl get nodes

kubectl get nodes --v=6
```
- If we try the equivalent call ourselves it **won't work** as it is missing authentication information supplied by our .kube/config file
```bash
curl --insecure https://127.0.0.1:6443/api/v1/nodes?limit=500
```
- Let's run a kubectl proxy to interact with the API in the background, we'll store the pid information to a file to assist with cleaning this up later, press return after executing
```bash
kubectl proxy & echo $! > /var/run/kubectl-proxy.pid
```
- We adjust our previous request and this should now work via the proxy. Removing HTTPS and changing port
```bash
curl -s http://127.0.0.1:8001/api/v1/nodes?limit=500 | more
OR
curl --insecure http://127.0.0.1:8001/api/v1/nodes?limit=500
```
- With our proxy running, we can use this to fetch an OpenAPI specification in v2 -
```bash
curl localhost:8001/openapi/v2
```

- And in v3, if you desired you could redirect these to a file for use elsewhere
```bash
curl localhost:8001/openapi/v3
```

- Let's create a pod via the API, the process for generating
```yaml
curl --location 'http://localhost:8001/api/v1/namespaces/default/pods?pretty=true' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data '{
    "kind": "Pod",
    "apiVersion": "v1",
    "metadata": {
        "name": "nginx",
        "creationTimestamp": null,
        "labels": {
            "run": "nginx"
        }
    },
    "spec": {
        "containers": [
            {
                "name": "nginx",
                "image": "nginx",
                "resources": {}
            }
        ],
        "restartPolicy": "Always",
        "dnsPolicy": "ClusterFirst"
    },
    "status": {}
}'
```

We will also delete a pod, via the api, again the process for generating this is outlined in the video

```yaml
curl --location --request DELETE 'http://localhost:8001/api/v1/namespaces/default/pods/nginx' \
--header 'Content-Type: application/json' \
--header 'Accept: application/json' \
--data '{
    "gracePeriodSeconds": 1,
    "propagationPolicy": "Background"
}'
```

- 
```bash

```