## Quick background
Applications run inside of containers and containers run inside of Pods. Every Kubernetes Pod gets its own unique IP address, and all Pods connect to the same flat network called the Pod network. However, Pods are ephemeral. This means they come and go and should not be considered reliable. For example, scaling operations, rolling updates, rollbacks and failures all cause Pods to be added or removed from the network.

To address the unreliable nature of Pods, Kubernetes provides a Service object that sits in front of a set of Pods and provides a reliable name, IP address, and port. Clients connect to the Service object, which in turn load-balances requests to the target Pods

Modern cloud-native applications a re comprised of lots of small independent microservices that work together to create a useful application. For these microservices to work together, they need to be able to discover and connect to each other. This is where service discovery comes into play.

There are 2 major components to service discovery:
- Service registration
- Service discovery

## Service registration
Service registration is the process of a microservice registering its connection details in a service registry so that other microservices can discover it and connect to it.
![](Screen-shots/Service%20registry.png)

A few important things to note about this in Kubernetes
1. Kubernetes uses an internal DNS service as its service registry
2. Services register with DNS (not with individual Pods)
3. The *name, IP address, and network port* of every Service is registered

For this to work, Kubernetes provides a well-known internal DNS service that we usually call the "cluster DNS". The term well known means that it operates at an address known to every Pod and container in the cluster. It's implemented in the `kube-system` Namespace as a set of Pods managed by a Deployment called `coredns`. These Pods are fronted by a Service called `kube-dns`. Behind the scenes, it's based on a DNS technology called CoreDNS and runs as a *Kubernetes-native application*.

```bash
$ kubectl get services -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   18d
```

```bash
$ kubectl get deploy -n kube-system -l k8s-app=kube-dns
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   1/1     1            1           18d
```

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns
NAME                      READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-28sgz   1/1     Running   9          18d
```

Every Kubernetes Service is automatically registered with the cluster DNS when it's created. The registration process looks like this (exact flow might slightly differ):

1. You POST a new Service manifest to the API Server
2. The request is authenticated, authorized, and subjected to admission policies
3. The Service is allocated a virtual IP address called a ClusterIP
4. An Enpoints object is created to hold a list of Pods the Service will load-balance traffic to
5. The Pod network is configured to handle traffic sent to the ClusterIP (more on this later)
6. The Service's name and IP are registered with the cluster DNS


We mentioned earlier that the cluster DNS is a Kubernetes-native application. This means it knows it’s running on Kubernetes and implements a controller that watches the API Server for new Service objects. Any time it observes a new Service object, it creates the DNS records that allow the Service name to be resolved to its ClusterIP. This means that applications and Services do not need to perform service registration – the cluster DNS is constantly looking for new Services and automatically registers their details.

The name registered for the Service is the value stored in its `metadata.name` property. The ClusterIP is dynamically assigned by Kubernetes
```yml
apiVersion: v1
kind: Service
metadata:
name: ent <<---- Name registered with cluster DNS
spec:
selector:
app: web
ports:
```

### The service back-end
Now that the front-end of the Service is registered, the back-end needs building. This involves creating and maintaining a list of Pod IPs that the Service will load-balance traffic to.

Every Service has a `label selector` that determines which Pods the Service will-load-balance traffic to.

![](Screen-shots/Service%20backend.png)

>Kubernetes automatically creates an Endpoints object (or Enpoint slices) for every Service. These hold the lost of Pods that match the label selector and will receive traffic from the Service.
They're also critical to how traffic is routed from the Service's ClusterIP to Pod IPs.


```bash
$ kubectl get endpoints kubernetes
NAME         ENDPOINTS           AGE
kubernetes   192.168.49.2:8443   18d
```

The figure below shows a Service called `ent` that will load-balance to two Pods. It also shows the Endpoints object with the IPs of the 2 pods that match the Service's label selector.
![](Screen-shots/load%20balance%20service%20endpoint.png)


The `kubelet` process on every node is watching the API Server for new Endpoints objects. When it sees them, it creates local networking rules that redirect ClusterIP traffic to Pod IPs. In modern Linux-based Kubernetes cluster the technology used to create these rules is the Linux IP Virtual Server (IPVS). Older versions of Kubernetes used iptables.

### Summarising service registration
![](Screen-shots/Summarize%20service%20registration.png)

You POST a new Service configuration to the API Server and the request is authenticated and authorized. The Service is allocated a ClusterIP and its configuration is persisted to the cluster store. An associated Endpoints object is created to hold the list of Pod IPs that match the label selector. The cluster DNS is running as a Kubernetes-native application and watching the API Server for new Service objects. It sees the new Service and registers the appropriate DNS A and SRV records. Every node is running a kube-proxy that sees the new Service and Endpoints objects and creates IPVS rules on every node so that traffic to the Service’s ClusterIP is redirected to one of the Pods that match its label selector.

## Service discovery
For Service discovery to work, every microservice needs to know 2 things
1. The **name** of the remote microservice they want to connect to
2. How to convert the **name** to an IP address

The applicaiton developer is responsible for point 1 - coding the microservice with the names of microservices they connect to. Kubenretes takes care of point 2.

### Converting names to IP addresses using the cluster DNS
Kuberentes automatically configures every container so that it can find and use the cluster DNS to convert Service names to IPs. It does this by populating every container's `/etc/resolv.conf` file with the IP address of cluster DNS Service as well as any search domains that should be appended to unqualified names.

**Note**: An “unqualified name” is a short name such as `ent`. Appending a search domain converts an unqualified name into a fully qualified domain name (FQDN) such as `ent.default.svc.cluster.local.`

The `/etc/resolve.conf` file of a container
```bash
$ kubectl exec -it hello-deploy-666cd5df7f-8vrbj sh
cat /etc/resolv.conf 
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The nameserver in `/etc/resolv.conf` matches the IP address of the cluster DNS (the kube-dns Service)
```bash
$ kubectl get svc -n kube-system -l k8s-app=kube-dns
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   18d
```

### Some network magic
Once a Pod has the ClusterIP of a Service, it sends traffic to that IP address. However, the address is on a special network called the service network and there are no routes to it! This means the apps container doesn’t know where to send the traffic, so it sends it to its *default gateway*.

>Note: A default gateway is where a device sends traffic that it doesn’t have a specific route for. The default gateway will normally forward traffic to another device with a larger routing table that might have a route for the traffic.

The containers default gateway sends the traffic to the Node it is running on.

***The Node doesn't have a route to the service network either***, so it sends the traffic to its own default gateway. Doing this causes the traffic to be processed by the ***Nodes kernel, which is where the magic happens!***


Every Kubernetes Nodes runs a system service called `kube-proxy`. At a high-level, `kube-proxy` is responsible for capturing traffic destined for ClusterIPs and redirecting it to the IP addresses of Pods that match the Service's label selector. Let's look a bit closer...

`kube-proxy` is a Pod-based Kubernetes-native app that implements a controller that watches the API server for new Service and Endpoints object. When it sees them, it creates local IPVS rules that tell the Node to intercept traffic destined for the Service's ClusterIP and forward it to individual Pod IPs.

This means that every time a Nodes kernel processes traffic headed for an address on the *service network*, a trap occurs and the traffic is redirected to the IP of a healthy Pod matching the Service's label selector.

Kubernetes originally used iptables to do this trapping and load-balancing. However, it was replaced by IPVS in Kubernetes 1.11. The is because IPVS is a high-performance kernel-based L4 load-balancer that scales better than iptables and implements better load-balancing algorithms.

### Summarising service discovery
![](Screen-shots/sumarize%20discovery%20service.png)

## Service discovery and Namespaces
Two things are important if you want to understand how service discovery works within and across Namespaces:
1. Every cluster has an address space
2. Namespace partition the cluster address space

Every cluster has an address space based on a DNS domain that we usually call the cluster domain. By default, it’s called `cluster.local`, and Service objects are placed within that address space. For example, a Service called `ent` will have a fully qualified domain name (FQDN) of `ent.default.svc.cluster.local`

The format of the FQDN id `<object-name>.<namespace>.svc.cluster.local`

Namespaces allow you to partition the address space below the cluster domain. For example, creating a couple of Namespaces called `prod` and `dev` will give you two address spaces that you can place Services and other objects in:
- dev: `<object-name>.dev.svc.cluster.local`
- prod:`<object-name>.prod.svc.cluster.local`

Object names must be unique within Namespaces but not across Namespaces. This means that you cannot have two Service objects called `ent` in the same Namespace, but you can if they are in different Namespaces. This is useful for parallel development and production configurations.

![](Screen-shots/namespaces.png)

Pods in the `prod` Namespace can connect to Services in the local Namespace using short names such as `ent` and `voy`. To connect to objects in a remote Namespace requires FQDNs such as `ent.dev.svc.cluster.local` and `voy.dev.svc.cluster.local`.

Namesapces partition the cluster address space. They are also good for implementing access control and resource quotas. However, they are not workload isolation boundaries and should not be used to isolate hostile workloads.

## Service discovery example	
![](Screen-shots/service%20discovery%20example.png)


`sd-example.yml`
```yml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: enterprise
  namespace: dev
  labels:
    app: enterprise
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-dev
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: enterprise
  namespace: prod
  labels:
    app: enterprise
spec:
  selector:
    matchLabels:
      app: enterprise
  replicas: 2
  template:
    metadata:
      labels:
        app: enterprise
    spec:
      terminationGracePeriodSeconds: 1
      containers:
      - image: nigelpoulton/k8sbook:text-prod
        name: enterprise-ctr
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: manchester
  namespace: dev 
  labels:
    event: manchester
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080 
  selector:
    event: manchester
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: dev
spec:
  ports:
  - port: 8080
  selector:
    app: enterprise
---
apiVersion: v1
kind: Service
metadata:
  name: ent
  namespace: prod
spec:
  ports:
  - port: 8080
  selector:
    app: enterprise
---
apiVersion: v1
kind: Pod
metadata:
  name: jump
  namespace: dev
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - image: ubuntu
    name: jump
    tty: true
    stdin: true
```

```bash
$ kubectl apply -f sd-example.yml 
namespace/dev created
namespace/prod created
deployment.apps/enterprise created
deployment.apps/enterprise created
service/manchester created
service/ent created
service/ent created
pod/jump created
```

Check the configuration for each namespace
```bash
$ kubectl get all --namespace dev
NAME                              READY   STATUS    RESTARTS   AGE
pod/enterprise-584b544bb6-dhrgf   1/1     Running   0          4m17s
pod/enterprise-584b544bb6-xbgjf   1/1     Running   0          4m17s
pod/jump                          1/1     Running   0          4m17s

NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/ent          ClusterIP      10.108.238.42   <none>        8080/TCP       4m17s
service/manchester   LoadBalancer   10.106.50.242   <pending>     80:31260/TCP   4m17s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           4m17s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/enterprise-584b544bb6   2         2         2       4m17s
```

```bash
$ kubectl get all --namespace prod
NAME                             READY   STATUS    RESTARTS   AGE
pod/enterprise-8b6fdc8c4-ml587   1/1     Running   0          5m7s
pod/enterprise-8b6fdc8c4-rhj5m   1/1     Running   0          5m7s

NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service/ent   ClusterIP   10.100.168.186   <none>        8080/TCP   5m7s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/enterprise   2/2     2            2           5m7s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/enterprise-8b6fdc8c4   2         2         2       5m7s
```

The next steps will:
1. Log on to the main container of jump Pod in the `dev` Namespace
2. Check the container's `/etc/resolv.conf` file
3. Connect to the `ent` app in the `dev` Namespace using the Service's shortname
4. Connect to the `ent` app in the `prod` Namespace using the Service's FQDN

Logon to the `jump` pod
```bash
$ kubectl exec -it jump --namespace dev -- bash
root@jump:/ cat /etc/resolv.conf 
nameserver 10.96.0.10
search dev.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Install `curl` and use it to connect to the version of the app running in the `dev` by using the `ent` short name

```bash
root@jump:/ curl ent:8080
Hello form the DEV Namespace!
Hostname: enterprise-584b544bb6-dhrgf
```

When the `curl` command was issued, the container automatically appended `dev.svc.cluster.local` to the `ent` name and sent the query to the IP address of the cluster DNS specified in `/etc/resolv.conf`. DNS returned the ClusterIP for the `ent` Service running in the local `dev` Namespace and the app sent the traffic to that address. En-route to the Nodes default gateway the traffic triggered a trap in the Node's kernel and was redirected to one of the Pods hosting the simple web application.

Run the `curl` command again but this time append the domain name of the `prod` Namespace. This will cause the cluster DNS to return the ClusterIP for the instance in the prod Namespace and traffic will eventually reach a Pod running in `prod`.
```bash
root@jump:/ curl ent.prod.svc.cluster.local:8080
Hello form the PROD Namespace!
Hostname: enterprise-8b6fdc8c4-rhj5m
```

This time the response comes from a Pod in the `prod` Namespace.

>The test proves that short names are resolved to the local Namespace (the same Namespace the app is running in) and connecting across Namespaces requires FQDNs.

## Trouble shooting service discovery
Kubernetes uses the cluster DNS as its service registry. It runs as a set of Pods in the `kube-system` Namespace with a Service object providing a stable network endpoint. The important components are:
- Pods: Managed by the `coredns` Deployment
- Service: A ClusterIP Service called `kube-dns` listening on port 53 TCP/UDP
- Endpoint: Also called `kube-dns`

All objects relating to the cluster DNS are tagged with the `k8s-app=kube-dns` label. This is helpful when filtering `kubectl` output.

Make sure that the `coredns` Deployment and its managed Pods are up and running

```bash
$ kubectl get deploy -n kube-system -l k8s-app=kube-dns
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   1/1     1            1           19d
```

```bash
$ kubectl get pods -n kube-system -l k8s-app=kube-dns  
NAME                      READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-28sgz   1/1     Running   10         19d
```

Check the logs from the each of the `coredns` Pods.

```bash
$ kubectl logs coredns-74ff55c5b-28sgz -n kube-system
.:53
[INFO] plugin/reload: Running configuration MD5 = cec3c60eb1cc4909fd4579a8d79ea031
CoreDNS-1.7.0
linux/amd64, go1.14.4, f59c03d
```

Assuming the Pods and Deployment are working, you should also check the Service and associated Endpoints object. The output should show that the service is up, has an IP address in the ClusterIP field, and is listening on port 53 TCP/UDP.

The ClusterIP address for the `kube-dns` Service should match the IP address in the `/etc/resolv.conf` files of all containers running on the cluster. If the IP addresses are different, containers will send DNS requests to the wrong IP address.

```bash
$ kubectl get svc kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   19d
```

The associated `kube-dns` Endpoints object should also be up and have the IP addresses of the `coredns` Pods listening on port 53 TCP and UDP.

```bash
$ kubectl get ep -n kube-system -l k8s-app=kube-dns
NAME       ENDPOINTS                                        AGE
kube-dns   172.17.0.11:53,172.17.0.11:53,172.17.0.11:9153   19d
```

Once you’ve verified the fundamental DNS components are up and working, you can proceed to perform more detailed and in-depth troubleshooting. Here are a few basic tips.

Start a troubleshooting Pod that has your favourite networking tools installed (ping, traceroute, curl, dig, nslookup
etc.). The standard `gcr.io/kubernetes-e2e-test-images/dnsutils:1.3 image` is a popular choice if you don’t have your own custom image with your tools installed. Unfortunately, there is no latest image in the repo. This means you have to specify a version. At the time of writing, 1.3 was the latest version.

The following command will start a new standalone Pod called `netutils`, based on the `dnsutils` image just mentioned. It will also log your terminal on to it

```bash
kubectl run -it dnsutils \
--image gcr.io/kubernetes-e2e-test-images/dnsutils:1.3
```

A common way to test DNS resolution is to use `nslookup` to resolve the `kubernetes.default` Service that sits in front of the API Server. The query should return an IP address and the name `kubernetes.default.svc.cluster.local.`

```bash
nslookup kubernetes
Server: 192.168.200.10
Address: 192.168.200.10#53
Name: kubernetes.default.svc.cluster.local
Address: 192.168.200.1
```

The first two lines should return the IP address of your cluster DNS. The last two lines should show the FQDN of the kubernetes Service and its ClusterIP. You can verify the ClusterIP of the kubernetes Service by running a `kubectl get svc kubernetes` command

Errors such as “nslookup: can’t resolve kubernetes” are possible indicators that DNS is not working. A possible solution is to restart the coredns Pods. These are managed by a Deployment object and will be automatically recreated.


The following command deletes the DNS Pods and must be ran from a terminal with kubectl installed. If you’re still logged on to the netutils Pod you’ll need to type exit to log off.

```bash
$ kubectl delete pod -n kube-system -l k8s-app=kube-dns
pod "coredns-5644d7b6d9-2pdmd" deleted
pod "coredns-5644d7b6d9-wsjzp" deleted
```

Verify that the Pods have restarted and test DNS again.

## Summary
In this chapter, you learned that Kubernetes uses the internal cluster DNS for service registration and service
discovery.

All new Service objects are automatically registered with the cluster DNS and all containers are configured to know where to find the cluster DNS. This means that all containers will talk to the cluster DNS when they need to resolve a name to an IP address.

The cluster DNS resolves Service names to ClusterIPs. These IP addresses are on a special network called the service network and there are no routes to this network. Fortunately, every cluster Node is configured to trap on packets destined for the service network and redirect them to Pod IPs on the Pod network.