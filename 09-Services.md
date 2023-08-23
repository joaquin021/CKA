# Services

Kubernetes Services provide a way to expose an application running as a set of Pods.  
The provide an abstract way for clients to access applications without needing to be aware of the application's Pods.  

**Service Routing:**  
Clients make request to a Service, which routes traffic to its Pods in a load-balances fashion.  

**Endpoints:**  
Endpoints are the backend entities to which Services router traffic. For a Service that routes traffic to a multiple Pods, each Pod will have an endpoint associated with the Service.   
One way to determine which Pod(s) a Service is routing traffic to is to look at that service's Endpoints.  

## Using K8s Services

### Service Types

Each Service has a type. The Service type determines how and where the Service wil expose your application. There are four service types:  
- ClusterIp
- NodePort
- LoadBalancer
- ExternalName (outside the scope of CKA)

#### ClusterIP Services

ClusterIP Services expose applications inside the cluster network. Use them when your clients will be other Pods within the cluster.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-clusterip
spec:
  type: ClusterIP
  selector:
    app: svc-example
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

#### NodePort Services

NodePort Services expose applications outside the cluster network. Use NodePort when applications or users will be accessing your application from outside the cluster.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nodeport
spec:
  type: NodePort
  selector:
    app: svc-example
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

#### LoadBalancer Services

LoadBalancer Services also expose applications outside the cluster network, but they use an external cloud load balancer to do so. This service type only works with cloud platforms that include load balancing functionality.  

## Discovering K8s Services with DNS

### Service DNS Names

The Kubernetes DNS assigns DNS names to Services, allowing application within the cluster to easily locate them.  
A service's fully qualified domain name has the following format:  
```
service-name.namespace-name.svc.cluster-domain.example
```

### Service DNS and Namespaces

A Service's fully qualified domain name can be used to reach the service from within any Namespace in the cluster.
```
my service-namespace.svc.cluster.local
```

However, Pods within the same Namespace can also simply use the service name.
```
my-service
```

## Managing Access from Outside with K8s Ingress

An Ingress is a Kubernetes object that manages external access to Services in the cluster.  
An Ingress is capable of providing more functionality than a simple NodePort Service, such as SSL termination, advanced load balancing, or name-based virtual hosting.  
https://kubernetes.io/docs/concepts/services-networking/ingress/

### Ingress Controllers

Ingress objects actually do nothing by themselves. In order for Ingresses to do anything, you must install one or more Ingress controllers.  
There are a variety of Ingress Controllers available - all of which implement different methods for providing external access to your Services.  
https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/

### Routing to a Service

Ingresses define a set of routing rules. A routing rule's properties determine to which requests it applies.  
Each rule has a set of paths, each with a backend. Requests matching a path will be routed to its associated backend.  

In the next example a request to http://<some-endopint>/somepath would be routed to port 80 on the my-service Service.  

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - http:
        paths:
          - path: /somepath
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  number: 80
```

### Routing to a Service with a Named Port

If a Service used a named port, an Ingress can also use the port's name to choose to which port it will route.  

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: ClusterIP
  selector:
    app: MyApp
  ports:
    - name: web
      protocol: TCP
      port: 80
      targetPort: 80
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
    - http:
        paths:
          - path: /somepath
            pathType: Prefix
            backend:
              service:
                name: my-service
                port:
                  name: web
```