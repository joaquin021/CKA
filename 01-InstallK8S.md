# Install cluster servers
Three servers are installed, one for the k8s control node and the other two for its workers.  
I have used virtual machines with ubuntu server 22.04 deployed in a proxmox.

# Configure servers

## Update servers

```bash
sudo apt-get update
sudo apt-get upgrade
```

## Config hostname

On the control plane node:  
```bash
sudo hostnamectl set-hostname k8s-control.cka
```
On the first worker node:  
```bash
sudo hostnamectl set-hostname k8s-worker1.cka
```
On the second worker node:
```bash
sudo hostnamectl set-hostname k8s-worker2.cka
```

On all nodes, set up the hosts file to enable all the nodes to reach each other using these hostnames:  
```bash
sudo nano /etc/hosts
```
On all nodes, add the following at the end of the file. You will need to supply the actual private IP address for each node:
```bash
192.168.1.100 k8s-control.cka
192.168.1.101 k8s-worker1.cka
192.168.1.102 k8s-worker2.cka
```

## Disable swap
Kubernetes requires swap to be disabled to works.  
Do it in all nodes.  
First check if swap is enabled:
```bash
sudo swapon --show
```
If swap is enabled you should see the path to the swap file and its size.
```bash
cloud_user@k8s-control.cka:~$ sudo swapon --show
NAME      TYPE SIZE USED PRIO
/swap.img file 1.9G 4.5M   -2
```

Run the following command to disable Swap:
```bash
sudo swapoff -a
```
Now remove the Swap file:
```bash
sudo rm /swap.img
```
Remove following line from `/etc/fstab` so that the Swap file is not re-created after a system reboot.
```bash
/swap.img       none    swap    sw      0       0
```

Reboot the system:
```bash
sudo reboot
```

Verify that swap is disabled.
```bash
sudo swapon --show
```

## Config kernel modules and forwarding ipv4
On all nodes, you will need to load some kernel modules and modify some system settings as part of this process:  
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
# Ensure you load modules
sudo modprobe overlay
sudo modprobe br_netfilter
```
Set up forwarding IPv4:
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot

sudo sysctl --system
```

# Install dependencies
Do it in all nodes.  

## Install containerd from Docker repositories

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
```

Add Docker gpg key:
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Set up the repository:
```bash
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Install containerd:
```bash
sudo apt-get update
sudo apt-get install containerd.io
```

Turn off automatic updates:
```bash
sudo apt-mark hold containerd.io
```

Set the cgroup driver for runc to systemd required for the kubelet:  
```bash
sudo mkdir /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/            SystemdCgroup = false/            SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

## Install kubernetes

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

Download the Google Cloud public signing key:
```bash
curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
```

Add the Kubernetes apt repository:
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Reboot the system:
```bash
sudo reboot
```

# Config cluster

## Initialize cluster

Initialize the Kubernetes cluster on the control plane node using kubeadm.
```bash
sudo kubeadm config images pull
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --upload-certs --control-plane-endpoint=k8s-control.cka
```

Set kubectl access:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Test access to the cluster:
```bash
joaquin@k8s-control:~$ kubectl cluster-info
Kubernetes control plane is running at https://k8s-control.cka:6443
CoreDNS is running at https://k8s-control.cka:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS     ROLES           AGE     VERSION
k8s-control.cka   NotReady   control-plane   3m59s   v1.27.4
```

### Deploy a pod network

You should now deploy a pod network to the cluster.  
I use the Calico Network Add-On.
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
```

Wait two minutes and check the status of the control plane node:
```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE     VERSION
k8s-control.cka   Ready    control-plane   6m51s   v1.27.4
```

```bash
joaquin@k8s-control:~$ kubectl get deployments -A
NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   calico-kube-controllers   1/1     1            1           56s
kube-system   coredns                   2/2     2            2           7m8s
```

```bash
joaquin@k8s-control:~$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-85578c44bf-vhkmd   1/1     Running   0          85s
kube-system   calico-node-fkg9d                          1/1     Running   0          85s
kube-system   coredns-5d78c9869d-5lc7x                   1/1     Running   0          7m22s
kube-system   coredns-5d78c9869d-sv7vr                   1/1     Running   0          7m22s
kube-system   etcd-k8s-control.cka                       1/1     Running   0          7m37s
kube-system   kube-apiserver-k8s-control.cka             1/1     Running   0          7m37s
kube-system   kube-controller-manager-k8s-control.cka    1/1     Running   0          7m38s
kube-system   kube-proxy-ghvfn                           1/1     Running   0          7m22s
kube-system   kube-scheduler-k8s-control.cka             1/1     Running   0          7m37s
```

## Join the Worker Nodes to the Cluster

In the control plane node, create the token and copy the kubeadm join command:
```bash
sudo kubeadm token create --print-join-command
```

Copy the full output from the previous command used in the control plane node.  
In both worker nodes, paste the full kubeadm join command to join the cluster. Use sudo to run it as root:  
```bash
sudo kubeadm join k8s-control.cka:6443 --token mk39f6.3dziwq2mb2um4nnq --discovery-token-ca-cert-hash sha256:083f118a945d6a4d341177a7e36243fd54447f37d333cac89a63f952f1c4509e 
```

In the control plane node, view the cluster status:
```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE   VERSION
k8s-control.cka   Ready    control-plane   10m   v1.27.4
k8s-worker1.cka   Ready    <none>          54s   v1.27.4
k8s-worker2.cka   Ready    <none>          38s   v1.27.4
```

# Deploy first application
In the control plane node execute:
```bash
kubectl create deployment nginx --image=nginx
kubectl get deployments
kubectl create service nodeport nginx --tcp=80:80
kubectl get svc
curl http://k8s-control.cka:30634
```
To verify in which worker the pod is running, we execute:
```bash
joaquin@k8s-control:~$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP             NODE              NOMINATED NODE   READINESS GATES
nginx-77b4fdf86c-t8l28   1/1     Running   0          95s   10.244.124.1   k8s-worker1.cka   <none>           <none>
```
