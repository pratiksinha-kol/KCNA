## Kubernetes NetworkPolicies

NetworkPolicies are an application-centric construct which allow you to specify how a pod is allowed to communicate with various network "entities" (we use the word "entity" here to avoid overloading the more common terms such as "endpoints" and "services", which have specific Kubernetes connotations) over the network. NetworkPolicies apply to a connection with a pod on one or both ends, and are not relevant to other connections.

Network policies do not conflict; they are additive. If any policy or policies apply to a given pod for a given direction, the connections allowed in that direction from that pod is the union of what the applicable policies allow. Thus, order of evaluation does not affect the policy result.

We'll explore how Network Policies can help isolate pods and control their communication within a Kubernetes cluster. This is particularly useful in multi-tenancy environments or when you need to restrict access to certain services within your cluster

**[SOURCE](https://kubernetes.io/docs/concepts/services-networking/network-policies/)**

- Let's create an nginx pod
```bash
kubectl run nginx --image=nginx
```

- Next, expose the nginx pod as a service with a ClusterIP. This step makes the nginx pod accessible within the Kubernetes cluster. Use the following command to expose the nginx pod
```bash
kubectl expose pod/nginx --port=80
```

- Now, let's test the accessibility of the nginx service. We'll run a temporary curl pod and use it to send a request to the nginx service. This demonstrates that, by default, any pod in the cluster can access the nginx service. Execute the following command to start an interactive curl session
```bash
kubectl run --rm -i --tty curl --image=curlimages/curl:8.4.0 --restart=Never -- sh
```

- Once in the curl pod, use curl to request the nginx page. Notice that there are no restrictions, and the nginx service responds
```bash
curl nginx.default.svc.cluster.local

exit
```


- Now, we will implement a Network Policy to restrict access to the nginx pod. Network Policies in Kubernetes allow you to control the flow of traffic. We will create a policy that allows access to the nginx pod only from pods with specific labels. Start by creating a network policy configuration file named `networkpolicy.yaml` <br>[REF](https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      run: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
      - podSelector:
          matchLabels:
            run: curl
```

- Apply the network policy to your cluster. This will enforce the rules defined in the policy, restricting the access to the nginx pod as specified. Run the following command to apply the network policy
```bash
kubectl apply -f networkpolicy.yaml     
```

- Test the network policy. Start another curl pod which we'll be using to accessing the nginx service again. Since this pod has the label `run:curl`, it should be allowed by the network policy. Run the following command to start an interactive session in a new curl pod
```bash
kubectl run --rm -i --tty curl --image=curlimages/curl --restart=Never -- sh
```

- In the curl pod, use curl to test access. The request should succeed, demonstrating the network policy's effect 
```bash
curl nginx.default.svc.cluster.local

exit
```


- To further validate the network policy, let's test with a pod that doesn't match the network policy criteria. Create a new curl pod with a different name, which results in a different label. This pod should not be able to access the nginx service. Run the following command to start an interactive session in the new pod
```bash
kubectl run --rm -it horse --image=curlimages/curl --restart=Never -- sh

OR

kubectl run --rm -i --tty curl2 --image=curlimages/curl --restart=Never -- sh
```

- Inside the `curl2/horse` pod, again, attempt to access the nginx service using curl. The request should be blocked by the network policy, illustrating how Network Policies can be used to control access in a Kubernetes cluster 
```bash
curl nginx.default.svc.cluster.local

exit
```

- Cleanup
```bash
kubectl delete pod nginx; kubectl delete service nginx; kubectl delete networkpolicy allow-nginx-access; rm -rf networkpolicy.yaml
```
