## KUBERNETES RBAC


### USERS (CN): They can be any individual or a process that intercats with Kubernetes Cluster. It can be admin, developer or even an automated system interacting with the cluster. users are not managed by kubernetes and is assumed that they are managed externally. users are represted as a string like "pratik" or "pratik@example.com"

### GROUPS (O): Groups are also managed outside Kubernetes. A group represents multiple users and its a way to attach a set of users to a vertain set of permissions. _When permissions are give to a group, all users that are part of the group receives those permissions._
 
### SERVICE ACCOUNT: A service account is a type of non-human account that, in Kubernetes, provides a distinct identity in a Kubernetes cluster. Application Pods, system components, and entities inside and outside the cluster can use a specific ServiceAccount's credentials to identify as that ServiceAccount. 

They are used by applications running inside a cluster. Unlike Groups or users, they are managed by Kubernetes and are Kubernetes Objects. This identity is useful in various situations, including authenticating to the API server or implementing identity-based security policies. They are tied to a specific namespace and their permission is scoped to this namespace. Service Accounts are used to give application running on a pod the necessary permissions to interact with Kubernetes API. 


[User accounts versus service accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#:~:text=User%20accounts%20are%20for%20humans,all%20namespaces%20of%20a%20cluster.)

** **

- Let's start by taking a look at our current kubeconfig
```bash
kubectl config view
```

- To see the information that is currently hidden, we can add the parameter of --raw 
```bash
kubectl config view --raw
OR
cat .kube/config
```

- To know more about the `certificate-authority-data` from above command:
```bash
echo `certificate-authority-data-VALUE` | base64 -d
```
For more info on certs: 

```
echo `certificate-authority-data-VALUE` | base64 -d | openssl x509 -text --noout
```

** **

### Cluster role bindings link accounts to cluster roles and grant access across all resources. To grant permissions across a whole cluster, you can use a ClusterRoleBinding.

### ClusterRole: A ClusterRole is a non-namespaced resources, meaning it applies to entire cluster and it allows us to set permission on resources across all namespaces. FYI: ClusterRole is non NAMESPACED. 

### ClusterRoleBindings: It binds a ClusterRole with Service Accouts, Users and Groups. 


```bash
k get clusterrolebinding -o wide
```
Example from above command: _The **ClusterRoleBinding** of cluster-admin binds the **ClusterRole** of cluster-admin and the **Group** system:masters TOGETHER_

```
kubectl describe ClusterRole/cluster-admin
OR
k describe clusterrole cluster-admin 
```

As mentioned above, the certifcates gives you comprehensive details about the issued certificate. Keep a closer look at the `Subject` under `Validity`. 
```echo client-certificate-data | base64 -d | openssl x509 -text --noout```   


Refer to [Users](#users-they-can-be-any-individual-or-a-process-that-intercats-with-kubernetes-cluster-it-can-be-admin-developer-or-even-an-automated-system-interacting-with-the-cluster-users-are-not-managed-by-kubernetes-and-is-assumed-that-they-are-managed-externally-users-are-represted-as-a-string-like-pratik-or-pratikexamplecom), [Groups](#groups-groups-are-also-managed-outside-kubernetes-a-group-represents-multiple-users-and-its-a-way-to-attach-a-set-of-users-to-a-vertain-set-of-permissions-when-permissions-are-give-to-a-group-all-users-that-are-part-of-the-group-receives-those-permissions) and [Service Accounts](#service-account-a-service-account-is-a-type-of-non-human-account-that-in-kubernetes-provides-a-distinct-identity-in-a-kubernetes-cluster-application-pods-system-components-and-entities-inside-and-outside-the-cluster-can-use-a-specific-serviceaccounts-credentials-to-identify-as-that-serviceaccount).


** ** 

## Lets create ClusterRole and ClusterRoleBindings which we saw previously 

Let's take a look at this from the view of Cluster Role Bindings in RBAC, review the Users, Groups and ServiceAccounts, note that group system:masters has a binding with cluster-admin and a role of ClusterRole/cluster-admin (you can click the Tutorial button to toggle the sidebar if you need more viewing space)

- Create ClusterRole (`cluster-superheroes` or `cluster-pratik`)
```bash
kubectl create clusterrole cluster-superhero --verb='*' --resource='*'
OR
kubectl create clusterrole cluster-pratik --verb='get,list,create' --resource='*'
```

```bash
k describe clusterrole cluster-superhero
k describe clusterrole cluster-pratik
```

- We'll bind this using a clusterrolebinding and we'll instruct it to bind to the group of `cluster-superheroes` or `cluster-pratik`
```bash
kubectl create clusterrolebinding cluster-superhero --clusterrole=cluster-superhero --group=cluster-superheroes
OR
kubectl create clusterrolebinding cluster-pratik --clusterrole=cluster-pratik --group=cluster-pratik 
```

Verfiy
```bash
kubectl get clusterrolebindings -o wide | egrep 'Name|^cluster-'
```

- We can use kubectl auth can-i to verify permissions, as an example, as our super user account check that we can do access any resource with any verb
```bash
kubectl auth can-i '*' '*'
```
We can expand this further, for example, we could check this against our group with an assigned user of batman

```
kubectl auth can-i '*' '*' --as-group="cluster-superheroes" --as="batman"
OR
kubectl auth can-i '*' '*' --as-group="cluster-pratik" --as="pratik"
OR
kubectl auth can-i 'list' '*' --as-group="cluster-pratik" --as="batman"
```
** **
## Configuring an RBAC User/Group Manually 

We'll see how we can configure a user/group for use with RBAC from scratch, first we'll start by generating an rsa private key for our user batman

[SOURCE](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)

- First we'll start by generating an rsa private key for our user `batman` or `andy`
```bash
openssl genrsa -out batman.key 4096
openssl genrsa -out andy.key 4096
```

- We'll now create a certificate signing request, we'll reference our private key, an output file and we'll specify the subject information with CN and O for our user and group
```bash
openssl req -new -key batman.key -out batman.csr -subj "/CN=batman/O=cluster-superheroes" -sha256
OR
openssl req -new -key andy.key -out andy.csr -subj "/CN=andy/O=cluster-pratik" -sha256
```
```bash
ls
cat andy.key OR cat andy.csr
```

- We'll now structure a CSR signing request, first we'll create variables with our data 
```bash
CSR_DATA=$(base64 batman.csr | tr -d '\n') OR CSR_DATA=$(base64 pratik.csr | tr -d '\n')
AND
CSR_USER=batman OR CSR_USER=pratik
```
Now verify if these variables are set: 

```
echo $CSR_DATA
echo $CSR_USER
```

- Let's template a Kubernetes Certificate Signing Request as yaml, we'll use placeholders for our variables and will redirect to a file
```yaml
cat <<EOF > batman-csr-request.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_USER}
spec:
  request: ${CSR_DATA}
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```
Verify the file
```
cat batman-csr-request.yaml
```

- Apply this request
```bash
kubectl apply -f batman-csr-request.yaml
```

- And if we check in Kubernetes we will see this certificate signing request as pending
```bash
kubectl get certificatesigningrequest
OR
kubectl get csr
```

- As our current super user we'll approve this request
```bash
kubectl certificate approve batman
```

- Check if the certificate signing request has been approved
```bash
kubectl get csr
```

- If we query the signed certificate using json as output, we will be able to see our certificate in base64
```bash
kubectl get csr batman -o json
```

- To directly capture this entry, we'll use jsonpath and will tell it to walk the path of status, then certificate
```bash
kubectl get csr batman -o jsonpath='{.status.certificate}'
```

- At the moment this is base64 so let's decode this
```bash
kubectl get csr batman -o jsonpath='{.status.certificate}' | base64 -d
```

- Redirect it to a file
```bash
kubectl get csr batman -o jsonpath='{.status.certificate}' | base64 -d > batman.crt
```

- We can use openssl to decode the approved/signed certificate, showing that the subject line contains CN and O 
```bash
openssl x509 -in batman.crt -text -noout
```
** ** 

## Adding Information to a Kubeconfig file

- Let's use our existing kubeconfig file as a template and copy it to a new `file` 
```bash
cp /root/.kube/config batman-clustersuperheroes.config
```

- If we take a look, this currently has a lot of information that we don't want our batman user to have 
```bash
cat batman-clustersuperheroes.config
```

- Let's clean this file, we'll use the `KUBECONFIG` variable and we'll unset users.default
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config unset users.default
```
- We'll delete the current context 
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config delete-context default
```
- And we'll unset the current context 
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config unset current-context
```

- Our base file is now simplified
```bash
cat batman-clustersuperheroes.config
```

- Let's embed the new information, first set-credentials as batman, we'll pass the certificate data and the private key, we'll also embed this information
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config set-credentials batman --client-certificate=batman.crt --client-key=batman.key --embed-certs=true
```

- Set the default context to use the default cluster with a user of batman 
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config set-context default --cluster=default --user=batman
```

- And use the context of default
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl config use-context default
```

- Now our configuration files looks like the following
```bash
cat batman-clustersuperheroes.config
```

- And we can test that this is working, we'll be able to get nodes, create a pod and delete a pod, essentially our user is a super user
```bash
KUBECONFIG=batman-clustersuperheroes.config kubectl get nodes
KUBECONFIG=./batman-clustersuperheroes.config kubectl run nginx --image=nginx
KUBECONFIG=./batman-clustersuperheroes.config kubectl delete pod/nginx --now
```
** ** 

## Automating an RBAC Kubeconfig file
We're going to setup a convenient tool for automating kubeconfig configurations, first install prerequisites

- We'll clone the project and change directory
```bash
git clone https://github.com/spurin/kubeconfig-creator.git

cd kubeconfig-creator
```

- And we'll create another user similar to batman, this time we'll create superman
```bash
./kubeconfig_creator.sh -u superman -g cluster-superheroes
```

- And because superman is also part of the cluster-superheroes group, we can execute commands as if we are a super user
```bash
KUBECONFIG=./superman-clustersuperheroes.config kubectl get nodes
```

- We'll create another user, this time we'll create wonder-woman
```bash
./kubeconfig_creator.sh -u wonder-woman -g cluster-superheroes
KUBECONFIG=./wonderwoman-clustersuperheroes.config kubectl get nodes
```

- And again, she will have full access 
```bash
KUBECONFIG=./wonderwoman-clustersuperheroes.config kubectl get nodes
```

** ** 

## Creating a watch only RBAC group


- We'll create a ClusterRole that can only read resources with the verbs list,get,watch
```bash
kubectl create clusterrole cluster-watcher --verb=list,get,watch --resource='*'
```

- We'll create our binding and our cluster-viewers group at the same time 
```bash
kubectl create clusterrolebinding cluster-watcher --clusterrole=cluster-watcher --group=cluster-watchers
```

- If we check the auth against a user called uatu in this group, we no longer have full access like before
```bash
kubectl auth can-i '*' '*' --as-group="cluster-watchers" --as="uatu"
```

- However if we check, we can observe
```bash
kubectl auth can-i 'list' '*' --as-group="cluster-watchers" --as="uatu"
```

- Let's create a KUBECONFIG file to test this for the uatu user
```bash
./kubeconfig_creator.sh -u uatu -g cluster-watchers
```

- If we run a get nodes, this will work 
```bash
KUBECONFIG=./uatu-clusterwatchers.config kubectl get nodes
```

- We can also query pods
```bash
KUBECONFIG=./uatu-clusterwatchers.config kubectl get pods
```

- However, if we try to create something as this user it will fail 
```bash
However, if we try to create something as this user it will fail 
```

** ** 

## Creating an RBAC managed user

All of our examples so far have been against wildcard ClusterRoles and Verbs, and against groups. 

- Let's try this out for a standalone user, we'll create a ClusterRole as a pod manager with the verbs list,get,create,delete and the resource as pods 
```bash
kubectl create clusterrole cluster-pod-manager --verb=list,get,create,delete --resource='pods'
```

- We'll create a ClusterRoleBinding which we'll use to bind to our ClusterRole and a user called deadpool 
```bash
kubectl create clusterrolebinding cluster-pod-manager --clusterrole=cluster-pod-manager --user=deadpool
```

- If we check with auth can -i list * we wont be able to do so as deadpool 
```bash
kubectl auth can-i 'list' '*' --as="deadpool"
```

- However, if we check against pods, this will be permitted
```bash
kubectl auth can-i 'list' 'pods' --as="deadpool"
```

- If we check the ClusterRoleBindings output, we'll see deadpool as a user at the bottom
```bash
kubectl get clusterrolebindings -o wide
```

- If we check the ClusterRoleBindings output, we'll see deadpool as a user at the bottom
```bash
kubectl get clusterrolebindings -o wide
```

- Let's create a KUBECONFIG file again, this time we wont use the group, just the user and observe the output, there will be no O entry as the subject line
```bash
./kubeconfig_creator.sh -u deadpool
```

- If we try to access pods as this user, it will work
```bash
KUBECONFIG=./deadpool.config kubectl get pods
```

- But if we try and access secrets, it will fail 
```bash
KUBECONFIG=./deadpool.config kubectl get secrets
```

** ** 

## Using Roles and RoleBindings

### ROLE: A Role can only be used to grant access to resources within a single namespace. (SCOPED IN A NAMESPACE)

### ROLEBINDING: Binds specific users/groups/service accounts to a Role within a namespace (SCOPED IN A NAMESPACE) 

### CLUSTERROLE: A ClusterRole can be used to grant the same permissions as a Role, but because they are cluster-scoped (SCOPED IN A CLUSTER-WIDE)

### CLUSTERROLEBINDING: Binds specific users/groups/service accounts to a ClusterRole, giving them permissions across the cluster. (SCOPED IN A CLUSTER-WIDE)

_We have been working with ClusterRole and ClusterRoleBinding_ 

- Let's switch to Roles and RoleBindings which are a namespaced resource, first we'll create a namespace called gryffindor 
```bash
kubectl create namespace gryffindor
```

- We'll create a role, in this new namespace with full access
```bash
kubectl -n gryffindor create role gryffindor-admin --verb='*' --resource='*'
```

- And we'll create a RoleBinding and specify our group
```bash
kubectl -n gryffindor create rolebinding gryffindor-admin --role=gryffindor-admin --group=gryffindor-admins
```

- If we check our auth against the group, we'll be rejected if we don't specify a namespace, however, if we do it will work as expected 
```bash
kubectl auth can-i '*' '*' --as-group="gryffindor-admins" --as="harry"

kubectl -n gryffindor auth can-i '*' '*' --as-group="gryffindor-admins" --as="harry"
```

- Let's create a kubeconfig file to check this as well, we'll use our convenient tool to do so and we'll set the namespace as we execute
```bash
./kubeconfig_creator.sh -u harry -g gryffindor-admins -n gryffindor
```

- If we check our configuration file, it has a namespace set
```bash
cat harry-gryffindoradmins.config
```

- If we try and access anything outside of a namespace it will fail
```bash
KUBECONFIG=./harry-gryffindoradmins.config kubectl get nodes
```

- If we try and access pods in the default namespace, this will also fail
```bash
KUBECONFIG=./harry-gryffindoradmins.config kubectl -n default get pods
```

- However, if we access resources in the gryffindor namespace, it will succeed
```bash
KUBECONFIG=./harry-gryffindoradmins.config kubectl get pods
```

- Cleanup 
```bash
cd ..

rm -rf batman* kubeconfig-creator

kubectl delete namespace/gryffindor

kubectl delete clusterrole/cluster-superhero clusterrole/cluster-watcher clusterrole/cluster-pod-manager

kubectl delete clusterrolebinding/cluster-superhero clusterrolebinding/cluster-watcher clusterrolebinding/cluster-pod-manager
```

** ** 

#### If you wish to see every resource and verb that is available we can do so with kube api-resources
```bash
kubectl api-resources --sort-by name -o wide | more
```