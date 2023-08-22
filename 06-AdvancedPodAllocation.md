# Advanced Pod Allocation

## Exploring K8s Scheduling

Scheduling is the process of assigning Pods to Nodes so kubelets can run them.  
The scheduler is a particular control plane component that handles scheduling.   

### Scheduling Process in K8s

The Kubernetes scheduler selects a suitable Node for each Pod. It takes into account things like:
- Resource requests vs available node resources.
- Various configurations that affect scheduling using node labels.

### nodeSelector

You can configure a `nodeSelector` for your pods to limit which Node(s) the Pod can be scheduled on.  
Node selectors use node labels to filter suitable nodes.  

```bash
kubectl label nodes <node name> mylabel=myvalue
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodeselector-pod
spec:
  nodeSelector:
    mylabel: myvalue
  containers:
  - name: nginx
    image: nginx:1.19.1
```

### nodeName 

You can bypass scheduling and assign a Pod to a specific Node by name using `nodeName`.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nodename-pod
spec:
  nodeName: k8s-worker1
  containers:
  - name: nginx
    image: nginx:1.19.1
```

## Using DaemonSets

A DaemonSet automatically runs a copy of a Pod on each node.  
DaemonSets will run a copy of the Pod on new nodes as they are added to the cluster.  

### DaemonSets and Scheduling 

DaemonSets respect normal scheduling rules around node labels, taints and tolerations. If a pod would not normally be scheduled on a node, a DaemonSet will not create a copy of the Pod on that Node.  

### Example

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: my-daemonset
spec:
  selector:
    matchLabels:
      app: my-daemonset
  template:
    metadata:
      labels:
        app: my-daemonset
    spec:
      containers:
      - name: nginx
      image: nginx:1.19.1
```

## Using Static Pods

A static pod is a pod that is managed directly by the kubelet on a node, not by the K8s API server. They can run even if there isn't K8s API server present.  
Kubelet automatically creates static Pods from YAML manifest files located in the manifest path on the node.  

### Mirror Pods

Kubelet will create a mirror Pod for each static Pod. Mirror Pods allow you to see the status of the static Pod via the K8s API, but you cannot change or manage them via the API.  
So that mirror pod is essentially just a ghost representation of the static pod in the Kubernetes API that allows you to view it, but not make any changes to it.  

### Example

Create a static pod manifest:
```bash
sudo nano /etc/kubernetes/manifests/my-static-pod.yml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
```

Restart kubelet to start the static pod:
```bash
sudo systemctl restart kubelet
```