## Service Meshes 

For the KCNA exam pay particular attention to the following -

1. The purpose of Service Meshes

2. The main components, Proxy and Data Plane

3. Common open source offerings for Service Meshes

4. The role of the SMI

##

Consider Service Meshes as a helper to your microservices. 
- It communicates efficiently, reliably and securely
- Specially helpful as the scale and complexity of your application grows. 
- It usually consistis of Data Plane and Control Plane.

### Data Plane
Data Plane typically implements sidecar pattern where sidecar proxy are attahced to the container. (Istio and Linkerd uses this approach).
Some meshes uses Host/Node proxy architecture. Example like Traefik install proxies are installed at Node level. However, it doesn't offer features/flexibility as sidecar pattern.  

### Control Plane
It sits above these and acts as management hub. Serves as a management hub, configuring and directing proxies. 

### Benefits of Service Meshes
- Mutual TLS between client and server  for enhanced security (two way verification)
- Access Control Policies 
- Improved Observability with Tracing
- Monitoring Tool
- Increased reliability through Rate Limiting and Circuit Braking

### _SMI (Service Mesh Interface): A standard interface for service meshes in Kubernetes. It provides a common, interoperable interface for various Service Mesh solutions._ 


[SMI](https://smi-spec.io/) <br>
[ISTIO](https://istio.io/)