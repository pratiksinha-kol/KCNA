## Helm and Helm Charts

- Helm is a package manager for Kubernetes, offering advanced deployment and configuration management capabilities. 

- Simplifies the management of the Kubernetes Application with the help of Helm Charts

- Helm Charts are packages of preconfigured Kubernetes resources. 

- Helm Chart facilitates easier deployment, management, update of Kubernetes Applications. 

- You can think Helm as apt/yum of Kubernetes application management. 

In this we will see how to _install_ Helm, _create_ a basic Helm chart, how to _package_ a helm chart, and how to _deploy_ our application from a packaged chart. 

[SOURCE](https://helm.sh/) <br>
[HELM ARCHITECTURE](https://helm.sh/docs/topics/architecture/)

** **

- Before we begin, ensure you have git installed, as it's required for Helm's plugin installation. We'll start by installing git, and at the same time we also going to install tree for a clearer view of directory structures used in Helm
```bash
apt update && apt install -y git tree
```

- Helm installation can be done in various ways. For convenience in this lab, we will use the get_helm.sh script. Normally, you should inspect scripts before executing them, but for this tutorial, we'll safely pipe the script to bash directly. Let's install Helm now
[REF](https://helm.sh/docs/intro/install/#from-script)
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

- Verify that Helm is installed and available in your path by checking its version. This command will show the installed Helm version details
```bash
helm version
```

- Let's create a Helm chart for the Flappy Bird style application. This process generates a basic structure for our application. Run the following command to create a new Helm chart named "flappy-app" 
```bash
helm create flappy-app
```

- Navigate into the newly created flappy-app directory to explore the Helm chart's structure. This will help us understand the components of a Helm chart 
```bash
cd flappy-app
```

- Let's examine the directory structure of our Helm chart using the tree command. This gives us an overview of the files and directories in our chart 
```bash
tree
```

- Now, customize the Chart.yaml file. This file contains key information about our Helm chart, such as the chart name and description. Edit the file to set the description to **Flappy Dock Game Helm chart** (without quotes) and leave the version as it's default
```bash
vim Chart.yaml
```

- Modify the values.yaml file to set the image repository to **spurin/flappy-dock** and use the **latest** tag (again without quotes for both). Also, disable the serviceAccount by setting its related boolean values to false
```bash
vim values.yaml
```

- Package the Helm chart for distribution. This step demonstrates Helm's versatility in chart management and version control
```bash
helm package .
```

- Deploy the application using the packaged Helm chart. This showcases Helm's ability to manage applications from packaged charts, which is useful for version control and distribution
```bash
helm install flappy-app ./flappy-app-0.1.0.tgz
```

- Helm would have provided us with some convenient commands, for ease we can run these as follows, this will also set up a port-forward to our application (you may need to retry if the app isn't running)
```bash
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=flappy-app,app.kubernetes.io/instance=flappy-app" -o jsonpath="{.items[0].metadata.name}"); export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}"); echo "Visit http://127.0.0.1:8080 to use your application"; kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```

Bring up a new tab to the Reverse Proxy and select the following - **Control Plane ➜ 127.0.0.1 ➜ 8080** and then press Go, have some fun with the game and when complete, move back to the previous tab and press ctrl-c

** **

- Explore the deployed Kubernetes resources to understand how Helm interacts with Kubernetes. Check the deployment, pods, and services created by our Helm chart 
```bash
kubectl get deployment; echo; kubectl get pods; echo; kubectl get svc
```

- Check whats showing from a helm viewpoint
```bash
helm list
```

- And then, let's uninstall the Helm chart to clean up the deployed resources 
```bash
helm uninstall flappy-app
```

- Cleanup
```bash
cd ..; rm -rf flappy-app
```

Congratulations on completing the Helm lab! You have learned how to install Helm, create and customize a Helm chart, package and deploy an application, and clean up resources. These skills are valuable for managing Kubernetes workloads efficiently.