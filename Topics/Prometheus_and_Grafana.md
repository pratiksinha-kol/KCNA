## Prometheus and Grafana

- Let's start by setting up our environment using Helm to install Prometheus and Grafana, we'll install git
```bash
apt update && apt install -y git
```

- And we'll install helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

- Now, let's add the Prometheus community's Helm chart repository, which contains preconfigured charts for deploying Prometheus and Grafana in Kubernetes. This step simplifies the installation and management of these monitoring tools 
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

- To ensure we install a stable version of Prometheus and Grafana, let's first search for the available versions of the kube-prometheus-stack in the repository. We aim to install a specific version (e.g., 55.5.0) to achieve consistent results throughout this tutorial
```bash
helm search repo prometheus-community/kube-prometheus-stack -l
```

- With the desired version identified, it's time to install the kube-prometheus-stack using Helm. This step will deploy Prometheus, Grafana, and related components into our Kubernetes cluster. We'll name our installation 'my-observability' and specify the version for consistency
```bash
helm install my-observability prometheus-community/kube-prometheus-stack --version 55.5.0
```

After initiating the installation, take a brief pause to allow Kubernetes to set up the components and start collecting metrics. This waiting period is crucial for generating meaningful data for our later analysis. Consider taking a **10-minute break** at this point

** ** 


- With the installation complete and metrics being collected, let's explore what has been installed. Use kubectl to list all the resources in the cluster. Pay attention to resources prefixed with 'my-observability', which include Prometheus, Grafana, and other related components
```bash
kubectl get all -A
```

- To access Prometheus and start exploring its capabilities, we need to locate its service endpoint. Run the following command to list the services and find the ClusterIP of 'my-observability-kube-prom-prometheus'. We will use this IP and port to access Prometheus through our reverse proxy 
```bash
kubectl get svc
```

** ** 

Now, let's interact with Prometheus's web interface. Use the reverse proxy feature in the lab environment to access Prometheus by entering the previously found ClusterIP and port 9090. Once in the Prometheus UI, explore the Metrics Explorer and try executing queries using PromQL, like 'kube_pod_ips', to see the power of Prometheus in action, switch to the Graph view, click the stacked graph button (next to Show Exemplars) and enter the custom PromQL query of 'sum by (namespace) (kube_pod_ips)


- Next, let's generate some activity in our cluster to see how Prometheus tracks changes over time. Run a simple loop in the terminal to create multiple nginx pods, which will be monitored by Prometheus. This process will take around 5 minutes 
```bash
for i in {1..10}; do kubectl run nginx-${i} --image=nginx; sleep 30; done
```

Returning to Prometheus, adjust the time interval to the last 5 minutes and observe how the metrics update, reflecting the newly created pods. This monitoring is a key feature of Prometheus, offering valuable insights into cluster activities

- Now, let's turn our attention to Grafana for advanced visualization. First, find the service endpoint for Grafana using kubectl to get the services. Look for the ClusterIP of 'my-observability-grafana' and it's Port (80) and use this for access through the reverse proxy 
```bash
kubectl get svc
```

Access Grafana (see the note at the top if you're using the Docker Desktop Extension lab) via the reverse proxy using the noted ClusterIP and port 80. Log in to Grafana with the default credentials (username: admin, password: prom-operator) and explore the pre-configured dashboards linked to Prometheus. Import community dashboards like '15759' for nodes and '15757' for Kubernetes cluster views to enhance your observability experience


** ** 
- **Cleanup** - Before concluding this lab, it's important to clean up the environment. Use Helm to uninstall the 'my-observability' release and remove any remaining resources, including the nginx pods we created. This step ensures our lab environment is ready for future use
```bash
helm list

helm uninstall my-observability

kubectl get all -A

kubectl -n kube-system delete service/my-observability-kube-prom-kubelet --now

for i in {1..10}; do kubectl delete pod/nginx-${i} --now; done
```