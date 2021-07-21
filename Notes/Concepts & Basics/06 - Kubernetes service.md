Kubernetes Service objects come into play - they provide stable and reliable networking for a set of dynamics Pods.


## Theory
When talking about *Service*, we're talking about the Service object in Kubernetes that provides stable networking for Pods. Just like a Pod, ReplicaSet, or Deployment, a Kubernetes Service is a REST object in the API that you define in a manifest and POST to the API server.

You need to know that every Service gets its own stable IP address, its own stable DNS name , and its own stable port.

Third, you need to know that Service leverage *labels* to dynamically select the Pods in the cluster they will send traffic to.

![](Screen-shots/SERVICE%20PROVIDE%20IP,%20DNS,%20LOAD%20BALANCE.png)

With a Service in front of a set of Pods, the Pods can scale up and down, they can fail, and they can be updated, rolled back ... and while events like these occur, the Service in front of them observes the changes and updates its list of healthy Pods. But it never changes the stable IP, DNS, and port that it exposes.

Think of Services as having a static front-end and a dynamic back-end. The front-end, consisting of the IP, DNS name, and port, and never changes. The back-end, consisting of the Pods, can be constantly changing.

### Labels and loose coupling
Service are loosely coupled with Pods via *labels* and *label selectors*. This is the same technology that loosely couples Deployments to Pods and is key to the flexibility provided by Kubernetes.

![](Screen-shots/Service%20loosely%20couple%20with%20pods.png)
![](Screen-shots/not%20work%20couples.png)

>For a Service to match a set of Pods, and therefore send traffic to them, **the Pods must possess every label in the Services label selector**. However, the Pod can have additional labels that are not listed in the Service's label selector.

![](Screen-shots/example%203%20on%20services%20and%20labels.png)

The following excerpts, from a Service YAML and Deployment YAML, show how selectors and labels are implemented.
`svc.yml`
```yml
apiVersion: v1
kind: Service
metadata: 
  name: hello-svc
spec:
  ports:
  - port: 8080
  selector: 
    app: hello-world #label selector
    #Service is looking for Pods with the label `app=hello-world`
```

`Deployment.yml`
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world


  template: 
    metadata:
      labels:
        app: hello-world # Pod labels
        # The label matches the Service's label selector

    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
```

The Service has a label selector (`spec.selector`) with a single value `app=hello-world`. This is the label that the Service is looking for when it queries the cluster for matching Pods. The Deployment specifies a Pod template with the same `app=hello-world` label (`spec.template.metadata.labels`). It is these two attributes that loosely couple the Service to the Deployment's Pods.

### Services and Endpoint objects
As Pods come-and-go (scaling up and down, failures, rolling updates etc.), the Service dynamically updates its list of healthy matching Pods. It does this through a combination of the label selector and a construct called an Endpoints object.

>**Each Service that is created, automatically gets an associated Endpoints object. All this Endpoints object is a dynamic list of all of the healthy Pods on the cluster that match the Service's label selector**

It works like this...

Kubernetes is constantly evaluating the Service's label selector against the current list of healthy Pods on the cluster. Any new Pods that match the selector get added to the Endpoints object, and any Pods that disappear get removed. This means the Endpoints object is always up to date. Then, when a Service is sending traffic to Pods, it queries its Endpoints object for the latest list of healthy matching Pods.

When sending traffic to Pods, via a Service, and application will normally query the cluster's internal DNS for the IP address of a Service. It then sends the traffic to this stable IP address and the Service sends it on to a Pod. However, a Kubernetes-native application (that's a fancy way of saying an application that understands Kubernetes and can query the Kubernetes API) can query the Endpoints API directly, bypassing the DNS lookup and use of the Service's IP.

### Accessing Service from inside the cluster
Kubernetes supports several types of Service. The default type is CLUSTER IP

A ClusterIP Service ***has a stable IP address and port that is only accessible from inside the cluster***. It’s programmed into the network fabric and guaranteed to be stable for the life of the Service. Programmed into the network fabric is fancy way of saying the network just knows about it and you don’t need to bother with the details (stuff like low-level IPTABLES and IPVS rules etc).

The ClusterIp gets registered against the name of the Service on the cluster's internal DNS service. All Pods in the cluster are pre-programmed to know about the cluster's DNS service, meaning all Pods are able to resolve Service names.

Creating a new Service called “magic-sandbox” will trigger the following.
1. ***Kubernetes will register the name “magic-sandbox”, along with the ClusterIP and port***, with the cluster’s DNS service. The name, ClusterIP, and port are guaranteed to be long-lived and stable, and all Pods in the cluster send service discovery requests to the internal DNS and will therefore be able to resolve “magic-sandbox” to the ClusterIP.
2. ***IPTABLES or IPVS rules are distributed across the cluster*** that ensure traffic sent to the ClusterIP gets routed to Pods on the backend.

> As long as a Pod (application microservice) knows the name of a Service, it can resolve that to its ClusterIP address and connect to the desired Pods.

This only works for Pods and other objects on the cluster, as it requires access to the cluster's DNS service. It does not work outside of the cluster.

### Accessing Services from outside the cluster

Kubernetes has another type of Service called a NodePort Service. This build on top of ClusterIP and enables access from outside of the cluster.

You already know that the default Service type is ClusterIP, and it registers a DNS name, virtual IP, and port with the cluster’s DNS. A different type of Service, called a NodePort Service builds on this by adding another port that can be used to reach the Service from outside the cluster. This additional port is called the NodePort.

Example of a NodePort Service:
- name: magic-sandbox
- clusterIP: 172.12.5.17
- port: 8080
- nodePort: 30050

This magic-sandbox Service can be accessed from inside the cluster via magic-sandbox on port 8080 , or 172.12.5.17 on port 8080 . **It can also be accessed from outside of the cluster by sending a request to the IP address of any cluster node on port 30050.**

At the bottom of the stack are cluster nodes that host Pods. You add a Service and use labels to associate it with Pods. 

***The Service object has a reliable NodePort mapped to every node in the cluster –- the NodePort value is the same on every node. This means that traffic from outside of the cluster can hit any node in the cluster on the NodePort and get through to the application (Pods).***

![](Screen-shots/6.6.png)

Figure 6.6 shows a NodePort Service where 3 Pods are exposed externally on port 30050 on every node in the cluster. 
- In step 1, an external client hits Node2 on port 30050 . 
- In step 2 it is redirected to the Service object (this happens even though Node2 isn’t running a Pod from the Service). 
- Step 3 shows that the Service has an associated Endpoint object with an always-up-to-date list of Pods matching the label selector. 
- Step 4 shows the client being directed to pod1 on Node1.

The Service could just as easily have directed the client to pod2 or pod3. In fact, future requests may go to other Pods as the Service performs basic load-balancing.

There are other types of Services, such as LoadBalancer, and ExternalName. 

LoadBalancer Services integrate with load-balancers from your cloud provider such as AWS, Azure, DO, IBM Cloud, and GCP. They build on top of NodePort Services (which in turn build on top of ClusterIP Services) and allow clients on the internet to reach your Pods via one of your cloud’s load-balancers. They’re extremely easy to setup. However, they only work if you’re running your Kubernetes cluster on a supported cloud platform. E.g. you cannot leverage an ELB load-balancer on AWS if your Kubernetes cluster is running on Microsoft Azure.


ExternalName Services route traffic to systems outside of your Kubernetes cluster (all other Service types route
traffic to Pods in your cluster).

### Service discovery
Kubernetes implements Service discovery in a couple of ways.
- DNS (preferred)
- Environment variables (definitely not preferred)

DNS-based Service discovery requires the DNS cluster-add-on - this is just a fancy name for the native Kubernetes DNS service. 
Behind the scenes it implements:
- Control plane Pods running a DNS service
- A Service object called `kube-dns` that sists in front of the Pods
- Kubelets program every container with the knowledge of the DNS (via `/etc/resolv.conf`)

The DNS add-on constantly watches the API server for new Services and automatically registers them in DNS. This means every Service gets a DNS name that is resolvable across the entire cluster.

A major problem with environment variables is that they’re only inserted into Pods when the Pod is initially created. This means that Pods have no way of learning about new Services added to the cluster after the Pod itself is created. This is far from ideal, and a major reason DNS is the preferred method. Another limitation can be in clusters with a lot of Services.

### Summary of Service theory
Service are all about providing stable networking for Pods. They also provide load-balancing and ways to be accessed from outside of the cluster.

The front-end of a Service provides a stable IP, DNS name and port that is guaranteed not to change for the entire life of the Service. The back-end of a Service uses labels to load-balance traffic across a potentially dynamic set of application Pods.

## Hands-on with Services

### The imperative way
Warning! The imperative way is not the Kubernetes way. It introduces the risk that you make imperative changes and never update your declarative manifests, rendering the manifests incorrect and out-of-date. This introduces the risk that stale manifests are subsequently used to update the cluster at a later date, unintentionally overwriting important changes that were made imperatively

`deploy.yml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    matchLabels:
      app: hello-world

  template: 
    metadata:
      labels:
        app: hello-world # Pod labels
        # The label matches the Service's label selector

    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
```

```bash
$ kubectl apply -f deploy.yml 
deployment.apps/hello-deploy created
```

The command to imperatively create a Kubernetes Service is `kubectl expose`. Run the following command to create a new Service that will provide networking and load-balancing for the Pods deployed in the previous step.
```bash
$ kubectl expose deployment web-deploy \
--name=hello-scv \
--target-port=8080 \
--type=NodePort
service/hello-scv exposed
```

Inspecting the Service
```bash
$ kubectl describe service hello-scv
Name:                     hello-scv
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-world
Type:                     NodePort
IP Families:              <none>
IP:                       10.109.135.184
IPs:                      10.109.135.184
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30272/TCP
Endpoints:                172.17.0.13:8080,172.17.0.14:8080,172.17.0.15:8080 + 7 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

Some interesting values in the output include:
- `Selector` is the list of labels that Pods must have in order for the Service to send traffic to them
- `IP` is the permanent internal ClusterIP (VIP) of the Service
- `Port` is the port that the application is listening on
- `NodePort` is the cluster-wide port that can be used to access it from outside the cluster
- `Endpoints` is the dynamic list of healthy Pod IPs currently match the Service's label selector

Remove the service once done
```bash
$ kubectl delete services hello-scv
service "hello-scv" deleted
```

### The declarative way

#### A service manifest file
`service.yml`
```yaml
apiVersion: v1
kind: Service
metadata: 
  name: hello-svc
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    targetPort: 8080
    protocol: TCP
  selector: 
    app: hello-world #label selector
    #Service is looking for Pods with the label `app=hello-world`
```

Services are mature objects and are fully defined in the `v1` core API group (`.apiVersion`)

The `.metadata` section defines a name for the Service. You can also apply labels here. Any labels you add here are used to identify the Service and are not related to labels for selecting Pods.

The `.spec` section is where you actually define the Service. In this example, you’re telling Kubernetes to deploy a NodePort Service. The port value configures the Service to listen on port 8080 for internal requests, and the NodePort value tells it to listen on 30001 for external requests. The targetPort value is part of the Service’s backend configuration and tells Kubernetes to send traffic to the application Pods on port 8080. Then you’re explicitly telling it to use TCP (default).

Finally, `spec.selector` tells the Service to send traffic to all Pods in the cluster that have the `app=hello-world` label. This means it will provide stable networking and load-balancing across all Pods with that label

### Common Service Types
3 common *Service Types* are:
- `ClusterIP`: This is the default option and gives the Service a stable IP address internally within the cluster. 
- `NodePort`: This builds on top of `ClusterIP` and adds a cluster-wide TCP or UDP port. It makes the Service available outside of the cluster on a stable port.
- `LoadBalancer`: This builds on top of `NodePort` and integrates with cloud-based load-balancers.

There’s another Service type called ExternalName. This is used to direct traffic to services that exist outside of the Kubernetes cluster. 

The manifest needs POSTing to the API server. The simplest way to do this is with `kubectl apply`

```bash
$ kubectl apply -f service.yml 
service/hello-svc created
```

### Introspecting services
```bash
$ kubectl get service hello-svc
NAME        TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-svc   NodePort   10.101.82.118   <none>        8080:30001/TCP   38s
```

```bash
$ kubectl describe service hello-svc
Name:                     hello-svc
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-world
Type:                     NodePort
IP Families:              <none>
IP:                       10.101.82.118
IPs:                      10.101.82.118
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30001/TCP
Endpoints:                172.17.0.13:8080,172.17.0.14:8080,172.17.0.15:8080 + 7 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

### Endpoints objects
We said that every Service gets its own Endpoints object with the same name as the Service. This object holds a list of all the Pods the Service matches and is dynamically updated as matching Pods come and go. You can see Endpoints with the normal `kubectl` commands.

In the following command, you can use the Endpoint controller’s `ep` shortname instead of `endpoints`.

```bash
$ kubectl get endpoints hello-svc
NAME        ENDPOINTS                                                        AGE
hello-svc   172.17.0.13:8080,172.17.0.14:8080,172.17.0.15:8080 + 7 more...   5m44s
```

```bash
$ kubectl describe endpoints hello-svc
Name:         hello-svc
Namespace:    default 
Labels:       <none>
Annotations:  endpoints.kubernetes.io/last-change-trigger-time: 2021-07-20T10:33:01Z
Subsets:
  Addresses:          172.17.0.13,172.17.0.14,172.17.0.15,172.17.0.16,172.17.0.17,172
.17.0.18,172.17.0.19,172.17.0.20,172.17.0.21,172.17.0.22
  NotReadyAddresses:  <none>
  Ports:
    Name     Port  Protocol
    ----     ----  --------
    <unset>  8080  TCP

Events:  <none>
```

### Summary of deploying Services
As with all Kubernetes objects, the preferred way of deploying and managing Services is the declarative way. Labels allow them to send traffic to a dynamic set of Pods. This means you can deploy new Services that will work with Pods and Deployments that are already running on the cluster and already in-use. Each Service gets its own Endpoints object that maintains an up-to-date list of matching Pods.

## Real world example
< Read more in the book>

Clean up the lab
```bash
$ kubectl delete -f deploy.yml
deployment.apps "web-deploy" deleted

$ kubectl delete -f service.yml 
service "hello-svc" deleted
```

## Chapter Summary
In this chapter, you learned that Services bring stable and reliable networking to apps deployed on Kubernetes.

They also perform load-balancing and allow you to expose elements of your application to the outside world (outside of the Kubernetes cluster).

The front-end of a Service is fixed, providing stable networking for the Pods behind it.
The back-end of a Service is dynamic, allowing Pods to come and go without impacting the ability of the Service to provide load-balancing.

Services are first-class objects in the Kubernetes API and can be defined in the standard YAML manifest files. They use label selectors to dynamically match Pods, and the best way to work with them is declaratively.