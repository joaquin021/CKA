# Deployments

A deployment is simply a K8s object that defines a desired state for a ReplicaSet (a set of replica pods). The Deployment Controller seeks to maintain the desired state by creating, deleting, and replacing Pods with new configuration.  

The most important fields a deployment's desired statue includes:
- `replicas`: The number of replica Pods the Deployment will seek to maintain.
- `selector`: A label selector used to identify the replica Pods managed by the deployment.
- `template`: A template Pod definition used to create replica Pods.

## Use cases

- Easily scale an application up or down by changing the number of replicas.
- Perform rolling updates to deploy a new software version.
- Roll back to a previous software version.

## Example deployment yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-deployment
  template:
    metadata:
      labels:
        app: my-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.19.1
        ports:
        - containerPort: 80
```

## Scaling Applications with Deployments

Scaling refers to dedicating mor (or fewer) resources to an application in order to meet changing needs.  
K8s Deployments are very useful in horizontal scaling, which involves changing the number of containers running an application.  

The Deployment's replicas setting determines how many replicas are desired in its desired state. If the replicas number is changed, replica Pods will be created or deleted to satisfy the new number.  


### How to Scale a Deployment

#### Edit YAML

You can scale a deployment simply by changing the number of replicas in the YAML descriptor with `kubectl apply` or `kubectl edit`.

```yaml
...
spec:
  replicas: 5
...
```

```bash
kubectl apply -f my-deployment.yml
```

#### kubectl scale

You can use the special `kubectl scale` command.

```bash
kubectl scale deployment.v1.apps/my-deployment --replicas=3
```

## Managing Rolling Updates with Deployments

### Rolling Update

Rolling updates allow you to make changes to a Deployment's Pods at a controlled rate, gradually replacing old Pods with new Pods. This allows you to update your Pods without incurring downtime.  

### Rollback

If an update to a deployment causes a problem, you can roll back the deployment to a previous working state.  

### Example

Edit the deployment spec, changing the image version to 1.19.2

```bash
kubectl edit deployment my-deployment
```

```yaml
...
spec:
  containers:
    - name: nginx
      image: nginx:1.19.2
...
```

Check the rollout status, deployment status, and pods.

```bash
kubectl rollout status deployment.v1.apps/my-deployment
kubectl get deployment my-deployment
kubectl get pods
```

Perform another rollout, this time using the kubectl set image method. Intentionally use a bad image version.

```bash
kubectl set image deployment/my-deployment nginx=nginx:broken --record
```

Check the rollout status again. You will see the rollout unable to succeed due to a failed image pull.

```bash
kubectl rollout status deployment.v1.apps/my-deployment
kubectl get pods
```

Check the rollout history.

```bash
kubectl rollout history deployment.v1.apps/my-deployment
```

Roll back to an earlier working version with one of the following methods

```bash
kubectl rollout undo deployment.v1.apps/my-deployment
```
or
```bash
kubectl rollout undo deployment.v1.apps/my-deployment --to-revision=<last working revision>
```