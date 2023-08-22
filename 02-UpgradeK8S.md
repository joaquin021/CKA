# Upgrade Kubernetes cluster

## Upgrade control plane node

Log in to the control plane server:
```bash
ssh joaquin@k8s-control.cka
```

Upgrade kubeadm:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.28.0-00
kubeadm version
```

Drain the control plane node:
```bash
kubectl drain k8s-control.cka --ignore-daemonsets
```

Plan the upgrade:
```bash
sudo kubeadm upgrade plan v1.28.0
```

Upgrade the control plane:
```bash
sudo kubeadm upgrade apply v1.28.0
```

Upgrade kubelet and kubectl:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.28.0-00 kubectl=1.28.0-00
```

Restart kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Uncordon the control plane node:
```bash
kubectl uncordon k8s-control.cka
```

View the cluster status:
```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE   VERSION
k8s-control.cka   Ready    control-plane   24d   v1.28.0
k8s-worker1.cka   Ready    <none>          24d   v1.27.4
k8s-worker2.cka   Ready    <none>          24d   v1.27.4
```


# Upgrade worker node 1
 
Log in to the worker1 server
```bash
ssh joaquin@k8s-worker1.cka
```

Run the following on the control plane node to drain worker node:
```bash
kubectl drain k8s-worker1.cka --ignore-daemonsets --force
```

Upgrade kubeadm on worker node:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.28.0-00
kubeadm version
```

Upgrade the kubelet configuration on the worker node:
```bash
sudo kubeadm upgrade node
```

Upgrade kubelet and kubectl on worker node:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.28.0-00 kubectl=1.28.0-00
```

Restart kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

From the control plane node, uncordon worker node 1:
```bash
kubectl uncordon k8s-worker1.cka
```

Check the cluster status:
```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE   VERSION
k8s-control.cka   Ready    control-plane   24d   v1.28.0
k8s-worker1.cka   Ready    <none>          24d   v1.28.0
k8s-worker2.cka   Ready    <none>          24d   v1.27.4
```

# Upgrade worker node 2
 
Log in to the worker2 server:
```bash
ssh joaquin@k8s-worker2.cka
```

Run the following on the control plane node to drain worker node:
```bash
kubectl drain k8s-worker2.cka --ignore-daemonsets --force
```

Upgrade kubeadm on worker node:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.28.0-00
kubeadm version
```

Upgrade the kubelet configuration on the worker node:
```bash
sudo kubeadm upgrade node
```

Upgrade kubelet and kubectl on worker node:
```bash
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.28.0-00 kubectl=1.28.0-00
```

Restart kubelet:
```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
From the control plane node, uncordon worker node 2:
```bash
kubectl uncordon k8s-worker2.cka
```

Check the cluster status:
```bash
joaquin@k8s-control:~$ kubectl get nodes
NAME              STATUS   ROLES           AGE   VERSION
k8s-control.cka   Ready    control-plane   24d   v1.28.0
k8s-worker1.cka   Ready    <none>          24d   v1.28.0
k8s-worker2.cka   Ready    <none>          24d   v1.28.0
```