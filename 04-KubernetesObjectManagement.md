# Kubernetes object management

## kubectl

### kubectl get

Use `kubectl get` to los objects in the kubetnetes cluster.  
Output formats: wide, json, yaml
```bash
kubectl get <object-type> <object-name> -n <namespace> -o <output-format> --sort-by <json-path-expression> --selector <selector-filter-by-label>
```

### kubectl describe

You can get detailed information about kubernetes objects using `kubectl describe`.
```bash
kubectl describe <object-type> <object-name>
```

### kubectl create

Use `kubectl create` to create objects.  
Supply a YAML file with -f to create an object from a YAML descriptor stored in the file.
```bash
kubectl create -f <file-name>
```

### kubectl apply 

`kubectl apply` is similar to `kubectl create`.  
However, if you use `kubectl apply` on an object that already exists, it will modify the existing object, if possible.
```bash
kubectl apply -f <file-name>
```

### kubectl delete

Use `kubectl delete` to delete objects from the cluster.
```bash
kubectl delete <object-type> <object-name>
```

### kubectl exec

`kubectl exec` can be used to run commands inside containers. Keep in mind that, in order for a command to succeed, the necessary software must exist within the containter to run it.  
If the pod has only one container, it's unnecessary specify the container name with `-c`.
```bash
kubectl exec <pod-name> -c <container-name> -- <command>
```

### `--record` flag

This flag saves the command as an annotation on the affected object. Kubernetes Documentation: [Flags](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#flags).  
Note: The --record flag is still valid, but it may be replaced by a different mechanism which annotates HTTP requests with kubectl command details. More details [here](https://github.com/kubernetes/kubernetes/pull/102873).

## Managing K8s Role-Based Access Control (RBAC)

RBAC in K8s allows you to control what users are allowed to do and access within your cluster.  
- Role: It defines permissions within a particular namespace.
- ClusterRole: It defines cluster-wide permissions not specific to a single namespace.
- RoleBinding: They are objects that link users to roles.
- ClusterRoleBinding: They are objects that link users to ClusterRoles.


### Create a role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "watch", "list"]
```

```bash
kubectl apply -f role.yml
```

#### Bind the role to `dev` user

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader
  namespace: default
subjects:
- kind: User
  name: dev
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rolebinding.yml
```

### Create service account

In K8s, a `service account` is an account used by container processes within Pods to authetincate with the K8s API.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-serviceaccount
  namespace: default
```

```bash
kubectl apply -f serviceaccount.yml
```

Another way to create it:
```bash
kubectl create sa my-serviceaccount -n default
```

#### Bind the role to service account

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: sa-pod-reader
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-serviceaccount
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f sa-pod-reader.yml
```

## Inspecting Pod Resource Usage

### Kubernetes Metrics Server

In order to view metrics about the resources pods and containers are using, we need an add-on to collect and provide that data. One such add-on is `Kubernetes Meterics Server`.

#### Install 
```bash
kubectl apply -f https://raw.githubusercontent.com/linuxacademy/content-cka-resources/master/metrics-server-components.yaml
```

### kubectl top

With `kubectl top`, you can view data about resource usage in your pods and nodes.

```bash
kubectl top pod --sot-by <expression> --selector <selector-filter-by-label>
```

```bash
kubectl top node
```