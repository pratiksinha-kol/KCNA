## Kubernetes Security

[SOURCE](https://kubernetes.io/docs/concepts/security/)

For the KCNA exam, the following considerations -

1. Awareness of Security Tools and their function including Falco and Open Policy Agent

2. Knowledge that Kubescape can be used for hardening to NSA and CISA standards

3. OpenID Connect and it's purpose, be aware that the exam may shorten this to OIDC

4. Legacy: Although Pod Security Policies are deprecated, this was previously a common question in the exam and you should be aware that Pod Security Policies manage Clusters and Namespaces at runtime

5. The 4C's of Cloud Native Security

** ** 

For awareness and also for the purposes of the KCNA exam, you should be aware of the 4C’s of Cloud Native Security, namely Cloud, Cluster, Container, Code.

Each endpoint acts as a layer from the outside-in and is a security best practice in Cloud Native Computing.

Each subsequent layer acts as a reinforcement for the layer within. For example, Code would benefit from the security implementations of Container, Cluster and Cloud respectively.

See the following image with a friendly poem to help you remember this!
![The 4C's of Cloud Native Security](../Kubernetes%20Security%20-%20The%204C's%20of%20Cloud%20Native%20Security.png)

** **

### Kubernetes Security Context
[REF](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)

A security context defines privilege and access control settings for a Pod or Container. 

- Let's start with Security Contexts which define privilege and access control settings for a Pod or Container. We will use a custom image that includes a rogue binary for demonstration purposes. This binary allows escalation from a normal user to root in a Kubernetes pod
```bash
kubectl run ubuntu --image=spurin/rootshell:latest -o yaml --dry-run=client -- sleep infinity | tee ubuntu_secure.yaml
```


- Let's apply this to create the pod in our Kubernetes cluster
```bash
kubectl apply -f ubuntu_secure.yaml
```

- With the pod running, let's execute into it and observe the current user privileges. You'll notice that we have root access
```bash
kubectl exec -it ubuntu -- bash

exit
```

- Modify our YAML file (`ubuntu_secure.yaml`) to include a Security Context that specifies running as a non-root user with a specific UID and GID
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - args:
    - sleep
    - infinity
    image: spurin/rootshell:latest
    name: ubuntu
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Let's `replace` the existing pod with our updated configuration. This replacement might take a little longer
```bash
kubectl replace --force -f ubuntu_secure.yaml
```

- Let's exec into the pod again. This time, you'll observe that we're logged in as a non-root user
```bash
kubectl exec -it ubuntu -- bash
```

- If we run id, this will confirm we are a non root user
```bash
id
```

- However, if we run the /rootshell binary this will allow privilege escalation
```bash
/rootshell
```

- We can even run commands that require root access like apt update
```bash
apt update

exit
```

- To further secure our pod, we will add an additional Security Context i.e. `allowPrivilegeEscalation: false` to the container specification, disallowing privilege escalation (`ubuntu_secure.yaml`)
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ubuntu
  name: ubuntu
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
  containers:
  - args:
    - sleep
    - infinity
    image: spurin/rootshell:latest
    name: ubuntu
    resources: {}
    securityContext:
      allowPrivilegeEscalation: false
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

- Again, let's replace the pod with our new security settings
```bash
kubectl replace --force -f ubuntu_secure.yaml
```

- Now, exec into the pod one more time
```bash
kubectl exec -it ubuntu -- bash
```

- And try to escalate privileges, you can try this multiple times, the command will succeed but without root priviledge. It is because we have added the security context inside the container
```bash
/rootshell

exit
```

- Cleanup
```bash
kubectl delete -f ubuntu_secure.yaml --now ; rm -rf ubuntu_secure.yaml
```

### Kubernetes PodSecurityPolicies

Cluster Level Restriction for Pods [Deprecated in v1.21 and removed from v1.25]

[REF](https://kubernetes.io/docs/concepts/security/pod-security-policy/)

- Example
```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: non-root-no-escalation-psp
spec:
  priviledge: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities: 
    - ALL
  readOnlyRootFilesystem: false
  allowedCapabilities:
  - CHOWN
  - DAC_OVERRIDE
  - SETGID
  - SETUID
  - NET_BIND_SERVICE
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
```

### Kubernetes Application Controllers

Since, v1.25 Application Controllers is the new standard. 
- It is integral to Kubernetes Security 
- They act as gatekeepers controlling what pod can and cannot do. 
- It helps in encforicng critical security policies like preventing containers running as `ROOT` users.
- It reduces attack vectors 
- It supports modular approach (As it is not built-in into Kubernetes)
- Third-party tools such as Kyverno, Open Policy Agent Gatekeeper, Falco, Kubescape, OpenID Connect or OIDC (authentication protocol). 

**Falco** is an open-source runtime security project that integrates with Kubernetes for identifying abnormal behavior and potential security threats
