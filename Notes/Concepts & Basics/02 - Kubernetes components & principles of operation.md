When you deploy Kubernetes, you get a **cluster**.

A Kubernetes cluster consists of a set of **worker** machines, called nodes, that run containerized applications. Every cluster has at least one worker node.

The worker node(s) host the Pods that are the components of the application workload. The control plane manages the worker nodes and the Pods in the cluster. In production environments, the control plane usually runs across multiple computers and a cluster usually runs multiple nodes, providing fault-tolerance and high availability

![](Screen-shots/kubernetes%20cluster%20diagram.png)



# Kubernetes from 40K feet
At the highest level, Kubernetes is two things
- A cluster for running applications
- An orchestrator of cloud-native microservices apps

### Kubernetes as a cluster
Kubernetes is like any other cluster - a bunch of nodes and a control plane. The control plane exposes an API, has a scheduler for assigning work to nodes, and state is recorded in a persistent store. Nodes are where application services run

Control plane as the brains of the cluster, and the nodes as the muscle.

### Kubernetes as an orchestrator
*Orchestrator* is just a fancy word for a system that takes care of deploying and managing applications

Organizes everything into a useful app and keeps things running smoothly. It even responds to events and other changes

### How it works
To make this happen, you start out with an app, package it up and give it to the cluster (Kubernetes). The cluster is made up of one or more masters and a bunch of nodes.

The masters, sometimes called heads or head nodes, are in-charge of the cluster. This means they make the scheduling decisions, perform monitoring, implement changes, respond to events, and more. For these reasons, we often refer to the masters as the control plane

The nodes are where application services run, and we sometimes call them the data plane. Each node has a reporting line back to the masters, and constantly watches for new work assignments

To run applications on a Kubernetes clsuter we follow this simple pattern

1. Write the application as small independent microservices in our favorite languages
2. Package each microservice in its own container
3. Wrap each container in its own Pod
4. Deploy Pods to the cluster via higher-level controllers such as; Deployments, DaemonSets, StatefulSets, CronJobs etc.
# Master and nodes


## Master (control plane)
A kubernetes master is a collection of system that make up the control plane of the cluster.

The simplest setups run all the master services on a single host. However, this is only suitable for labs and test environments. For production environments, multi-master high availability (HA) is a ***must have***.

### The API server
All communication, between all components, must go through the API server. All internal system components, as well as external user components, all communicates via the same API

It exposes a RESTful API that you POST YAML configuration files to over HTTPS. These YAML files, which we sometimes call manifests, contain the desired state of your application. This desired state includes things like; which container image to use, which ports to expose, and how many Pod replicas to run.

All requests to the API Server are subject to authentication and authorization checks, but once these are done, the config in the YAML file is validated, persisted to the cluster store, and deployed to the cluster.

### The cluster store
The cluster store is the only *stateful* part of the control plane, and it persistently stores the entire configuration and state of the cluster

The cluster store is currently based on `etcd`, a popular distributed database. As it's the single source of truth for the cluster, you should run between 3-5 `etcd` replicas for high-availability, adn you should provide adequate ways to recover when things go wrong

On the topic of availability, `etcd` prefers consistency over availability. This means that it will not tolerate a split-brain situation and will halt updates to the cluster in order to maintain consistency. However, if `etcd` becomes unavailable, applications running on the cluster should continue to work, you just won’t be able to update anything.

As with all distributed databases, consistency of writes to the database is vital. For example, multiple writes to the same value originating from different nodes needs to be handled. `etcd` uses the popular `RAFT` consensus algorithm to accomplish this.

### The controller manager
>The controller manager implements all of the background control loops that monitor the cluster and respond to events
It's a *controller of controllers*, meaning it spawns all of the independent control loops and monitors them 

Some of the control loops include; ***the node controller, the endpoints controller, and the replica set controller***. Each one runs as a background watch-loop that is constantly watching the API server for changes -- the aim of the game is to ensure the *current state* of the cluster matches the *desired state* 

The logic implemented by each control loop is effectively this:
1. Obtain desired state
2. Observe current state
3. Determine differences
4. Reconcile differences

This logic is at the heart of Kubernetes and declarative design patterns

Each control loop is also extremely specialized and only interested in its own little corner of the Kubernetes cluster. (They takes care of its own business and leaves everything else alone). This is the key to distributed design of Kubernetes and adheres to the Unix philosophy of building complex systems from small specialized parts.

### The scheduler
>At a high level, the scheduler watches the API server for new work tasks and assigns them to appropriate healthy nodes.

Behind the scenes, ***it implements complex logic that filters out nodes incapable of running the task, and then ranks the nodes that are capable***. The ranking system is complex, but the node with the highest-ranking score is selected to run the task.

When identifying nodes that are capable of running a task, the scheduler performs various predicate checks. These include; 
- is the node tainted, 
- are there any affinity or anti-affinity rules, 
- is the required network port available on the node, 
- does the node have sufficient free resources etc.

Any node incapable of running the task is ignored, and the remaining nodes are ranked according to things such as;
- does the node already have the required image
- how much free resource does the node have
- how many tasks is the node already running

Each criterion is worth points, and the node with the most points is selected to run the task

If the scheduler cannot find a suitable node, the task cannot be scheduled and is marked as pending.

The scheduler isn't responsible for running tasks, just picking the nodes a task will run on

### The cloud controller manager
If you're running your cluster on a supported public cloud platform, such as AWS, Azure, GCP, DO, IBM Cloud etc. your control plane will be running a *cloud controller manager*.

Its job is to manage integrations with underlying cloud technologies and services such as, instances, load-balancers, and storage. For example, if your application asks for an internet facing load-balancer, the cloud controller manager is involved in provisioning an appropriate load-balancer on your cloud platform.

![](Screen-shots/Kubernetes%20master.png)

## Nodes
*Nodes* are the workers of a Kubernetes cluster. At a high-level they do 3 things

1. Watch the API Server for new work assignments
2. Execute new work assignments
3. Report back to the control plane (via API server)

3 major components of a node
![](Screen-shots/Nodes.png)


### Kubelet
>The Kubelet is the star of the show on every node. It's the main Kubernetes agent, and it runs on every node in the cluster. In fact, it's common to use the term *node* and *kubelet* interchangeably

When you join a new node to a cluster, the process installs `kubelet` onto the node. The `kubelet` is then responsible for registering the node with the cluster. Registration effectively pools the node's CPU, memory, and storage into the wider cluster pool.

One of the main jobs of the `kubelet` is to watch the API server for new work assignments. Any time it sees one, it executes the task and maintains a reporting channel back to the control plane.

If a `kubelet` can’t run a particular task, it reports back to the master and lets the control plane decide what actions to take. 

<span style="color:red">For example, if a `Kubelet` cannot execute a task, *it is not responsible for finding another node to run it on*. *It simply reports back* to the control plane and the control plane decides what to do.</span>

### Container runtime
The `kubelet` needs a *container runtime* to perform container-related tasks -- things like pulling images and starting and stopping containers.

More recently, `Kubernetes` has moved to a *plugin model* called the ***Container Runtime Interface*** (CRI). At a high-level, the CRI masks the internal machinery of Kubernetes and exposes a clean documented interface for 3rd-party container runtimes to plug into.

There are lots of container runtimes available for Kubernetes. One popular example is `cri-containerd`. This is a community-based open-source project porting the CNCF `containerd` runtime to the CRI interface. It has a lot of support and is replacing Docker as the most popular container runtime used in Kubernetes.

### Kube-proxy
This runs on every node in the cluster and is responsible for local cluster networking. For example, it makes sure each node gets its own unique IP address, and implements local IPTABLES or IPVS rules to handle routing and load-balancing of traffic on the Pod network.


## Kubernetes DNS

As well as the various control plane and node components, every Kubernetes cluster has an internal DNS service that is vital to operations

The cluster's DNS service has a static IP address that is hard-coded into every Pod on the cluster, meaning all containers and Pods know how to find it. Every new service is automatically registered with the cluster's DNS so that all components in the cluster can find every Service by name.

Some other components that are registered with the cluster DNS are StatefulSets and the individual Pods that a StatefulSet manager.

Cluster DNS is based on CoreDNS

## Packaging apps for Kubernetes
For an application to run on a Kubernetes cluster it needs to tick a few boxes
1. Packaged as a container
2. Wrapped in a Pod
3. Deployed via a declarative manifest file

You write an application service in a language of your choice. You build it into a container image and store it in a registry. At this point, the application service is *containerized*

Next, you define a Kubernetes Pod to run the containerized application. At the kind of high level we're at, a Pod is just a wrapper that allws a container to run on a Kubernetes cluster. Once you've defined the Pod, you're ready to deploy it on the cluster

It is possible to run a standalone Pod on a Kubernetes cluster. But the preferred model is to deploy all Pods via higher-level controllers. The most common controller is the *Deployment*. It offers scalability, self-healing, and rolling updates. 

You define Deployments in `YAML` manifest files that specifies things like; which image to use and how many replicas to deploy.

![](Screen-shots/Packaging%20kuebrnetes.png)

Once everything is defined in the Deployment `YAML` file, you `POST` it to the API server as the *desired state* of the application and let kubernetes implement it

## The declarative model and desired state
The *declarative* model and the concept of desired state are at the very heart of Kubernetes

1. Declare the desired state of an application (microservice) in a manifest file

2. POST it to the API server

3. Kubernetes stores it in the cluster store as the application's desired state

4. Kubernetes implements the desired state on the cluster

5. Kubernetes implements watch loops to make sure the current state of the application doesn't vary from the desired state

Manifest files are written in simple YAML, and they tell Kubernetes how you want an application to look. This is called the desired state. It includes things such as; which image to use, how many replicas to run, which network ports to listen on, and how to perform updates    

Once you've created the manifest, you `POST` it to the API server. The most common way of doing this is with the `kubectl` command-line utility. This sends the manifest to the control plane as an HTTP POST, usually on port 443

Once the request is authenticated and authorized, Kubernetes inspects the manifest, identifies which controller to send it to (e.g. the *Deployments controller*), and records the configuration in the cluster store as part of the cluster's overall *desired state*.

Once this is done, the work gets scheduled on the cluster. This includes the hard work of pulling images, starting containers, building networks, and starting the application's processes

Finally, Kubernetes utilizes background reconciliation loops that constantly monitor the state of the cluster. If the *current state* of the cluster varies from the *desired state*, Kubernetes will perform what every tasks are necessary to reconcile the issues

### Declarative example
Assume you have an app with a desired state that includes 10 replicas of a web front-end Pod. If a node that was running two replicas fails, the current state will be reduced to 8 replicas, but the desired state will still be 10. This will be observered by a reconciliation loop and Kubernetes will schedule two new replicas to bring the total back up to 10

The same thing will happen if you intentionally scale the desired number of replicas up or down. You could even change the image you want to use. For example, if the app is currently using v2.00 of an image, and you update the desired state to use v2.01 , Kubernetes will notice the difference and go through the process of updating all
replicas so that they are using the new version specified in the new desired state.

To be clear. Instead of writing a long list of commands to go through the process of updating every replica to the new version, you simply tell Kubernetes you want the new version, and Kubernetes does the hard work for us.

Despite how simple this might seem, it’s extremely powerful and at the very heart of how Kubernetes operates. You give Kubernetes a declarative manifest that describes how you want an application to look. This forms the basis of the application’s desired state. The Kubernetes control plane records it, implements it, and runs background reconciliation loops that constantly check that what is running is what you’ve asked for. When current state matches desired state, the world is a happy place. When it doesn’t, Kubernetes gets busy fixing it.

## Pods
In the VMware world, the atomic unti of scheduling is the virtual machine (VM). In the Docker world, it's the container. Well ... in the Kubernetes world, it's the ***Pod***

It's true that Kubernetes runs containerized apps. However, you cannot run a container directly on a Kubernetes cluster - containers must always run inside of Pods

### Pods and containers
>The point is, a Kubernetes Pod is a construct for running one or more containers

The simplest model is to run a single container per Pod. However, there are advanced use-cases that run multiple containers inside a single Pod. These multi-container Pods are beyond the scope of what we're discussing here, but powerful examples include:

- Service meshes
- Web container supported by a *helper* container that pulls the latest content
- Containers with a tightly coupled log scraper

![](Screen-shots/Pod.png)

### Pod anatomy
At the highest-level, a *Pod* is ring-fenced environment to run containers. The Pod itself doesn't actually run anything, it's just a sandbox for hosting containers. Keeping it high level, you ring-fence an area of the host OS, build a network stack, create a bunch of kernel namespaces, and run one or more containers in it. That's a Pod.

If you're running multiple contaienrs in a Pod, they all share the same **Pod environment**. This include things like the IPC namespace, shared memory, volumes, network stack and more. This means that all containers in the same Pod will share the same IP address

![](Screen-shots/Pod%20anatomy.png)

If two containers in the same Pod need to talk to each other, they can use ports on the Pod's `localhost` interface as shown.

![](Screen-shots/multicontainer%20in%20a%20pod.png)

Multi-container Pods are ideal when you have requirements for tightly coupled containers that may need to share memory and storage. However, if you don’t need to tightly couple your containers, you should put them in their own Pods and loosely couple them over the network. This keeps things clean by having each Pod dedicated to a single task. It also creates a lot of network traffic that is un-encrypted. You should seriously consider using a service mesh to secure traffic between Pods and application services

### Pods as the unit of scaling
Pods are also the minimum unit of scheduling in Kubernetes. If you need to scale your app, you add or remove Pods. You do not scale by adding more containers to an existing Pod. Multi-container Pods are only for situations where two different, but complimentary, containers need to share resources

![](Screen-shots/Scaling%20pods.png)

### Pods - atomic operations
>The deployment of a Pod is an atomic operation. This means that a Pod is only considered ready for service when all of its containers are up and running. There is never a situation where a partially deployed Pod will service requests. The entire Pod either comes up and is put into service, or it doesn't and it fails.

A single Pod can only be scheduled to a single node. This is also true of multi-container Pods - all containers in the same Pod will run on the same node

### Pod lifecycle
>Pods are mortal. They're created, they live, and they die. If they die unexpectedly, you don't bring them back to life. Instead, Kubernetes starts a new one in its place. 

However, even though the new Pods looks, smells, and feels like the old one, it isn't. It's a shiny new Pods with a new shiny new ID and IP address

***This has implication on how you should design your applications. Don't design them so they are tightly coupled to a particular instance of a Pod***. Instead, design them so that when Pods fail, a totally new one (with a new ID and IP address) can pop up somewhere else in the cluster and seamlessly take its place

## Deployments
Most of the time you'll deploy Pods indirectly via a higher-level controller. Examples of higher-level controllers include; *Deployments*, *DaemonSets*, and *StatefulSets*

For example, a Deployment is a higher-level Kubernetes object that wraps around a particular Pod and adds features such as scaling, zero-downtime updates, and versioned rollbacks.

### Services and network stable networking
Pods are mortal and can die. However, if they're managed via Deployments or DeamonSets, they get replaced when they fail. But replacements come with totally different IP addresses, whereas scaling down takes existing Pods away. Events like these cause a lot of IP churn

**Pods are unreliable**, which poses a challenge ... Assume you've go a microservices app with a bunch of Pods performing video rendering. How will this work if other parts of the app that need to use the rendering service cannot rely on the rendering Pods being there when they need them?

This is where `Services` come in to play. **Services provide reliable networking for a set of Pods**

The Kubernetes Service provides a reliable name and IP, and is load-balancing requests 
![](Screen-shots/Services.png)

Services are fully-fledged objects in the Kubernetes API - jsut like Pods and Deployments. 
**They have a front-end that consists of a stable DNS name, IP address, and port.** 
**On the back-end, they load-balance across a dynamic set of Pods**. As Pods come and go, the Service observes this, automatically updates itself, and continues to provide that stable networking endpoint.

The same applies if you scale the number of Pods up or down. New Pods are seamlessly added to the Services and will receive traffic. Terminated Pods are seamlessly removed from the Service and will not receive traffic

>That's the job of a Service - it's a stable network abstraction point that provides TCP and UDP load-balancing across a dynamic set of Pods. Services bring stable IP addresses and DNS names to the unstable world of Pods.

As they operate at the TCP and UDP layer, Services do not possess application intelligence and cannot provide application-layer host and path routing. For that, you need an Ingress, which understands HTTP and provides host and path-based routing.

### Connecting Pods to Services
Services use *labels* and a *label selector* to know which set of Pods to load-balance traffic to. The Service has a *label selector* that is a list of all the labels Pods must possess in order for it to receive traffic from the Service

One final thing about Services. They only send traffic to **healthy Pods**. This means a Pod that that is failing health-checks will not receive traffic from the Service

![](Screen-shots/Services%20labels%20selector.png)

## Summary
The masters are where the control plane components run. Under-the-hood, there are several system-services, including the API Server that exposes a public REST interface to the cluster. Masters make all of the deployment and scheduling decisions, and multi-master HA is important for production-grade environments.

Nodes are where user applications run. Each node runs a service called the kubelet that registers the node with the cluster and communicates with the API Server. It watches the API for new work tasks and maintains a reporting channel. Nodes also have a container runtime and the kube-proxy service. The container runtime, such as Docker or containerd, is responsible for all container-related operations. The kube-proxy is responsible for networking on the node.

We also talked about some of the major Kubernetes API objects such as Pods, Deployments, and Services. The Pod is the basic building-block. Deployments add self-healing, scaling and updates. Services add stable networking and load-balancing.