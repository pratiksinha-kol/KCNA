## Argo CD 



- Welcome to the interactive lab on Cloud Native Application Delivery and GitOps. We will be exploring modern application deployment, focusing on cloud-native methodologies and the GitOps approach using Argo. Let's start by creating a namespace for ArgoCD
```bash
kubectl create namespace argocd
```

- Next, we'll install ArgoCD in our newly created namespace. We will use a YAML file from the ArgoCD repository to install it. This YAML file contains all the necessary Kubernetes resources for setting up ArgoCD [SOURCE](https://argo-cd.readthedocs.io/en/stable/getting_started/#1-install-argo-cd)
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- In our lab environment, we'll simplify the setup by switching ArgoCD's operation from HTTPS to HTTP. This step is for ease of use in the lab and is not recommended for production environments. We'll patch the ArgoCD config map and then restart the ArgoCD server deployment to apply these changes
```bash
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge -p '{"data":{"server.insecure":"true"}}'
```

```bash
kubectl -n argocd rollout restart deployment/argocd-server
```

- Now, we'll verify that the ArgoCD components are running correctly. The 'watch' command allows us to actively monitor the state of our resources. We'll use it to keep an eye on everything in the argocd namespace until they are all up and running
```bash
watch --differences kubectl get all -n argocd
```

- Our next step involves accessing the ArgoCD server. To log in, we need the ArgoCD admin password, which is stored in a Kubernetes secret. We'll retrieve this password by accessing the secret under .data.password and will decode with base64, copy this to your clipboard
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

- With the password at hand, let's find the ClusterIP of the ArgoCD server. This IP will be used to access the ArgoCD UI through a reverse proxy. We'll run a command to list the services in the argocd namespace and note the ClusterIP for the argocd-server service
```bash
kubectl -n argocd get svc
```
** **

It's time to access the ArgoCD UI. Use the retrieved ClusterIP and the default port of 80. Then use a username of admin and the clipboard password to log into the ArgoCD web interface. Once logged in, we'll create a new application named my-argo-app. Set the Project Name as default. Set the Sync Policy to Automatic and enable both Prune Resources and Self Heal options. Under Sync Options select Auto-Create Namespace. Use a Git repository for the application source and enter **https://github.com/spurin/argo-f-yourself**, specify the path as . Then under Destination choose the Cluster URL (select the default option). Then set the namespace to my-argo-ns - Lastly select the Directory Recurse option before clicking create. This process demonstrates how GitOps can be used for application deployment in a cloud native environment -

If you wish, at this point you could interact with the wordpress svc and access and setup wordpress through it's ui, click the svc, select the clusterIP and navigate to this with the reverse proxy. When you're finished, use the argocd-server ClusterIP to navigate back to Argo.


- Let's simulate a failure scenario to understand the self-healing capabilities of GitOps with ArgoCD. We'll delete the 'my-argo-ns' namespace, which contains our deployed application. Watch how ArgoCD detects this discrepancy between the desired state in Git and the actual state in the cluster and reconciles it by redeploying the application
```bash
kubectl delete namespace/my-argo-ns --now
```

- Cleanup: Finally, lets clean up our environment. We'll delete the namespaces argocd and my-argo-ns. This step ensures that all resources we created, including applications and ArgoCD components, are removed 
```bash
kubectl delete namespace/argocd namespace/my-argo-ns --now
```