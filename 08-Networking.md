# Networking

## Kubernetes Network Model

 The Kubernetes network model is a set of standards that define how networking between Pods behaves.  
 There are a variety of different implementations of this model, including the Calico network plugin.  
 
## Network Model Architecture

The Kubernetes network model defines how Pods communicate with each other, regardless of which Node they are running.  
Each Pod has its own unique IP address within the cluster.  
Any Pod can reach any other Pod using that Pod's IP address. This creates a virtual network that allows Pods to easily communicate with each other, regardless of which node they are on.  

## CNI Plugins Overview

CNI plugins are type of Kubernetes network plugin. These plugins provide network connectivity between Pods according to the standard set by the Kubernetes network model.  
There are many CNI network plugins available.  

Which network plugin is best for you will depend on your specific situation.  
Check the Kubernetes documentation for a list of available plugins. You may need to research some of these plugins for yourself, depending on your production use case.  

### Installing Network Plugins

Each plugin has its own unique installation process.  
We installed the Calico network plugin when set up the cluster.  

Kubernetes nodes will remain NotReady until a network plugin is installed. You will be unable to run Pods while this is the case.  

## Understanding K8s DNS

The K8s virtual network uses a DNS to allow Pods to locate other Pods and Services using domain names instead of IP addresses.  
This NDS runs as a Service within the cluster. You can usually find it in the kube-system namespace.  
Kubeadm clusters use CoreDNS.  


### Pod Domain Names

All Pods in our kubeadm cluster are automatically given a domain name of the followin form:

```
pod-ip-address.namespace-name.pod.cluster.local
```
Example:
```
192-168-10-100.default.pod.cluster.local
```

These DNS names are not useful when you are communicating between pods. They include the pod IP address.  
This will be more useful when we use services.  

## Using NetworkPolicies

A K8s Network Policy is an object that allows you to control the flow of network communication to and from Pods.  
This allows you to build a more secure cluster network by keeping Pods isolated from traffic they do not need.  


### Pod Selector

`podSelector` determines to which Pods in the namespace the NetworkPolicy applies.  
The podSelector can select Pods using Pod labels.  

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db 
```

By default, Pods are considered non-isolated and completely open to all communication.  
If any NetworkPolicy selects a Pod, the Pod is considered isolated and will only be open to traffic allowed by NetworkPolicies.  

### Ingress and Egress

INGRESS: Incoming network traffic coming into the pod from another source.  
EGRESS: Outgoing network traffic leaving the Pod for another destination.  

#### from and to Selectors

from selector: Selects ingress traffic that will be allowed.  
to selector: Selects egress traffic that will be allowed.  

```yaml
spec:
  ingress:
    - from:
      ...
  egress:
    - to:
    ...
```

Types of from and to Selectors:

`podSelector`: Select Pods to allow traffic from/to.
```yaml
spec:
  ingres:
    - from:
        - podSelector:
            matchLabels:
              app: db
```

`namespaceSelector`: Select namespaces ato allow traffic from/to.
```yaml
spec:
  ingres:
    - from:
        - namespaceSelector:
            matchLabels:
              app: db
```

`ipBlock`: Select an ip range to allow traffic from/to.
```yaml
spec:
  ingres:
    - from:
        - ipBlock:
            cidr: 10.15.0.0/16
```

`port`: Specifies one or more ports that will allow traffic.  
```yaml
spec:
  ingres:
    - from:
      ports:
        - protocol: TCP
          port: 80
```

Traffic is only allowed if it matches both an allowed port and one of the from/to rules.  

### Example policy
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-networkpolicy
  namespace: np-test
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              team: np-test
      ports:
        - port: 80
          protocol: TCP
```