# Backup and restore etcd

Log in to the control plane server:
```bash
ssh joaquin@k8s-control.cka
```

Install command line etcdctl for etcd:
```bash
sudo snap install etcd
```

Copy the certificates and grant permissions.
*WARNING: Any malicious user with these certificates could access the database.*
```bash
mkdir $HOME/etcd-certs
sudo cp /etc/kubernetes/pki/etcd/ca.crt $HOME/etcd-certs/ca.crt
sudo cp /etc/kubernetes/pki/etcd/server.crt $HOME/etcd-certs/server.crt
sudo cp /etc/kubernetes/pki/etcd/server.key $HOME/etcd-certs/server.key
sudo chmod 644 $HOME/etcd-certs/ca.crt $HOME/etcd-certs/server.crt $HOME/etcd-certs/server.key
sudo chown -R joaquin:joaquin $HOME/etcd-certs/*
```


List keys for check connectivity: 

```bash
ETCDCTL_API=3 etcdctl get "" --prefix --keys-only --endpoints=https://k8s-control.cka:2379 --cacert=$HOME/etcd-certs/ca.crt --cert=$HOME/etcd-certs/server.crt --key=$HOME/etcd-certs/server.key
```

Back up etcd:
```bash
ETCDCTL_API=3 etcdctl snapshot save $HOME/etcd_backup.db \
  --endpoints=https://k8s-control.cka:2379 \
  --cacert=$HOME/etcd-certs/ca.crt \
  --cert=$HOME/etcd-certs/server.crt \
  --key=$HOME/etcd-certs/server.key
```

Restore etcd:
```bash
sudo systemctl stop etcd
sudo rm -rf /var/lib/etcd
sudo ETCDCTL_API=3 etcdctl snapshot restore $HOME/etcd_backup.db \
  --initial-cluster etcd-restore=https://k8s-control.cka:2379 \
  --initial-advertise-peer-urls https://k8s-control.cka:2379 \
  --name etcd-restore \
  --data-dir /var/lib/etcd
```

Set ownership:
```bash
sudo chown -R etcd:etcd /var/lib/etcd
```

Start etcd:
```bash
sudo systemctl start etcd
```