# Pods and Containers

## Managing application configuration

### ConfigMaps

You can store configuration data in kubernetes using `ConfigMaps`.  
ConfigMaps store data in the form of a key-value map. ConfigMap data can be passed to your container applications.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-configmap
data:
  key1: Hello, world!
  key2:
    subkey:
  	  morekeys: data
  	  evenmover: some more data
  key3: |
    Test
    multiple lines
    more lines
```

### Secrets

`Secrets` are similar to ConfigMaps but are designed to store sensitive data, such as passwords or API keys, more securely. They are created and used similarly to ConfigMaps.

```bash
echo -n 'user' | base64
echo -n 'mypassword' | base64
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
type: Opaque
data:
  username: dXNlcg==
  password: bXlwYXNzd29yZA==
```

### Pass ConfigMap and Secret data to your containers

#### As environment variables

You can pass ConfigMap and Secret data to your containers as environment variables. These variables will be visible to your container process at runtime.

```yaml
spec:
  containers:
  - ...
    env:
    - name: ENVVAR
      valueFrom:
      	configMapKeyRef:
      	  name: my-configmap
      	  key: mykey
      	secretKeyRef:
          name: my-secret
          key: secretkey1
```

#### As mounted volumes

Configuration data from ConfigMaps and Secrets can also be passed to containers in the form of mounted volumes. This will cause the configuration data to appear in files available to the container file system.

Each top-level key in the configuration data will appear as a file containing all keys bellow that to-level key.

```yaml
spec:
  containers:
  - ...
    volumeMounts:
    - name: configmap-volume
      mountPath: /etc/config/configmap
    - name: secret-volume
      mountPath: /etc/config/secret
  volumes:
  - name: configmap-volume
    configMap:
      name: my-configmap
  - name: secret-vol
    secret:
      secretName: my-secret
```

```bash
kubectl exec volume-pod -- ls /etc/config/configmap
kubectl exec volume-pod -- cat /etc/config/configmap/key1
kubectl exec volume-pod -- cat /etc/config/configmap/key2
kubectl exec volume-pod -- ls /etc/config/secret
kubectl exec volume-pod -- cat /etc/config/secret/secretkey1
kubectl exec volume-pod -- cat /etc/config/secret/secretkey2
```

## Managing container resources

### Resource Requests

`Resource requests` allow you to define an amount of resources (such as CPU or memory) you expect a container to use. The kubernetes scheduler will use resource requests to avoid scheduling pods on nodes that not have enough available resources.  

Containers are allowed to use more or less than the requested resources. Resource requests only affect scheduling.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: big-request-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
    resources:
      requests:
        cpu: "10000m"
        memory: "128Mi"
```

### Resource Limits

`Resource limits` provide a way for you to limit the amount of resources your containers can use. The container runtime is responsible for enforcing these limits, and different container runtimes do this differently.   

Some runtimes will enforce these limits by terminating container processes that attempt to use more than the allowed amount of resources.  
For example, Docker throttle processes based on the
CPU limits. So if the process tries to use more CPU than defined CPU, it's going to get throttled, but in the case of memory, Docker will actually kill the process if it goes above
the defined memory.

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
    resources:
      requests:
        cpu: "500m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

## Monitoring container health with probes

### Container Health

K8s provides a number of features that allow you to build robust solutions, such as the ability to automatically restart unhealthy containers. To make the most of these features, K8s, needs to be able to accurately determine the status of your applications. This means actively monitoring `container health`.

[Container probes](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)

### Liveness Probes

`Liveness probes` allow you to automatically determine whether a container application is in a healthy state.  
By default, K8s will only consider a container to be "down" if the container process stops.  
Liveness probes allow you to customize this detection mechanism and make it more sophisticated.

Example liveness http:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-pod-http
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Startup Probes

`Startup probes` are very similar to liveness probes. However, while liveness probes run constantly on a schedule, startup probes run at container startup and stop running one they succeed.  
They are used to determine when the application has successfully started up. Startup probes are especially useful for legacy applications that can have long startup times.  

Example startup http:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
```

### Readiness Probes

`Readiness probes` are used to determine when a container is ready to accept requests. When you have a service backend by multiple container endpoints, user traffic will not be sent to a particular pod until its containers have all passed the readiness checks defined by their readiness probes.  
Use readiness probes to prevent user traffic from being sent to pods that are still in the process of starting up.  

Example readiness http:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

## Restart Policies

K8s can automatically restart containers when they fail. Restart policies allow you to customize this behavior by defining when you want a pod's containers to be automatically restarted.  
Restart policies are an important component of self-healing applications, which are automatically repaired when a problem arises.  
There are three possible values for a pod's restart policy in k8s: `Always`, `OnFailure` and `Never`.  

### Always

`Always` is the default restart policy in K8s.  
With this policy, containers will always be restarted if they stop, even if they completed successfully. Use this policy for applications that should always be running.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: always-pod
spec:
  restartPolicy: Always
  containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'sleep 10']
```

### OnFailure

The `OnFailure` restart policy will restart containers only if the containers process exists with an error code or the container is determined to be unhealthy by a liveness probe. Use this policy for applications that need tro run successfully and then stop.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: onfailure-pod
spec:
  restartPolicy: OnFailure
  containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'sleep 10; this is a bad command that will fail']
```

### Never

The `Never` restart policy will cause the pod's containers to never ve restarted, even if the container exits or a liveness probe fails. Use this for applications that should run once and never be automatically restarted.  

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: never-pod
spec:
  restartPolicy: Never
  containers:
    - name: busybox
      image: busybox
      command: ['sh', '-c', 'sleep 10; this is a bad command that will fail']
```

## Creating Multi-Container Pods

A Kubernetes Pod can have one or more containers. A pod with more than one container is a `multi-container Pod`.  
In a multi-container Pod, the containers share resources such as network and storage. The can interact with one another, working together to provide functionality.  
It is a best practice keep container in separate Pods unless they need to share resources.  

Containers sharing the same Pod can interact with one another using shared resources.  
- Network: Containers share the same networking namespace and can communicate with one another on any port, even if that port is not exposed to the cluster.
- Storage: Containers can use volumes to share data in a Pod.

### Example use case

You have an application that is hard-coded to write log output to a file on disk.  
You add a secondary container to the Pod (sometimes called a sidecar) that reads the log file from a shared volume and prints it to the console so the log output will appear in the container log.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-pod
spec:
  containers:
    - name: busybox1
      image: busybox
      command: ['sh', '-c', 'while true; do echo logs data > /output/output.log; sleep 5; done']
      volumeMounts:
      - name: sharedvol
        mountPath: /output
    - name: sidecar
      image: busybox
      command: ['sh', '-c', 'tail -f /input/output.log']
      volumeMounts:
      - name: sharedvol
        mountPath: /input
  volumes:
  - name: sharedvol
    emptyDir: {}
```

## Init containers

Init containers are containers that run once during the startup of a pod. A pod can have any number of init containers, and they will each run one (in order) to completion.  
You can use init containers to perform a variety of startup tasks. The can contain and use software and setup scripts that are not needed by your main containers.  
They are often useful in keeping your main containers lighter and more secure by offloading startup tasks to a separate container.  

Some sample use cases for init containers:
- Cause a pod to wait for another K8s resource to be created before finishing startup.
- Perform sensitive startup steps securely outside of app containers.
- Populate data into a shared volume at startup.
- Communicate with another service at startup. For example, register your pod in another external services.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.19.1
  initContainers:
  - name: delay
    image: busybox
    command: ['sleep', '30']
```

