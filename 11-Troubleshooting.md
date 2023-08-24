# Troubleshooting

## Troubleshooting your K8s Cluster

### Kube API Server

If the K8s API server is down, you will not be able to use kubectl to interact with the cluster. You may get a message that looks something like:  
```
The connection to the server localhost:6443 was refused - did you specify the right host or port?
```

Assuming your kubeconfig is set up correctly, this may mean the API server is down.  

Possible fixes:  
Make sure the docker and kubelet services are up and running on your control plane node(s).

### Checking Node Status

Check the status of your nodes to see if any of them are experiencing issues.  
You use the next command to see the overall status of each node.  
```bash
kubectl get nodes
```

And you use the next command to get more information on any nodes that are not in READY state.  
```bash
describe node <node-name>
```

If a node is having problems, it may be because a service is down on that node.  
Each node runs the kubelet and container runtime services.  

```bash
systemctl status kubelet
systemctl start kubelet
systemctl enable kubelet
```

## Checking System Pods

In a kubeadm cluster, several K8s components run as pods in the kube-system namespace.  
Check the status of these components.

```bash
kubectl get pods -n kube-system
kubectl describe <pod-name> -n kube-system
```

## Checking Cluster and Node logs

### Service Logs

You can check the logs for K8s-related services on each node using journalctl.  
```bash
sudo journalctl -u kubelet
sudo journalctl -u docker
```

### Cluster Component Logs

The kubernetes cluster components have log output redirected to /var/log. For example:  
```bash
/var/log/kube-apiserver.log
/var/log/kube-scheduler.log
/var/log/kube-controller-manager.log
```

These log files may not appear for kubeadm clusters, since some components run inside containers. In that case, you can access then with kubectl logs.  
```bash
kubectl logs -n kube-system kube-apiserver-k8s-control.cka
kubectl logs -n kube-system kube-scheduler-k8s-control.cka
kubectl logs -n kube-system kube-controller-manager-k8s-control.cka
```

## Troubleshooting your applications

### Checking Pod Status

You can see a Pod's status with `kubectl get pods`.  
Use `kubectl describe pod` to get more information about what may be going on with an unhealthy Pod.  

### Running Commands Inside Containers

If you need to troubleshoot what is going on inside a container, you can execute commands within the container with `kubectl exec`.  
```bash
kubectl exec <pod-name> -c <container-name> -- <command>
```

Note that you cannot use `kubectl exec` to run any software that is not present within the container.  

Interactive prompt:  
```bash
kubectl exec <pod-name> -c <container-name> --stdin --tty -- <command>
```
## Checking Container Logs

K8s containers maintain logs, which you can use to gain insight into what is going on within the container.  
A container's log contains everything written to the standard output (stdout) an error (stderr) streams by the container process.  

Use the `kubectl logs` command to view a container's logs.  
```bash
kubectl logs <pod-name> -c <container-name>
```

Kubernetes typically stores container logs in the path: 
```bash
/var/log/pods
/var/log/containers
```
These logs are the same ones you can view using the `kubectl logs` command, but in case you ever encounter any issues with the command, it's useful to know their location.  

## Troubleshooting K8s Networking Issues

### kube-proxy and DNS

In addition to checking on your K8s networking plugin, it may be a good idea to look at kube-proxy and the K8s DNS if you are experiencing issues within the K8s cluster network.  
In a kubeadm cluster the K8s DNS and kube-proxy run as Pods in the kube-system namespace.  

### netshoot

You can run a container in the cluster that you can use to run commands to test and gather information about network functionality.  
The `nicolaka/netshoot` image is a great tool for this. This image contains a variety of networking exploration and troubleshooting tools.  
Create a container running this image, and then use `kubectl exec` to explore away.  

