## Pod theory
In the Kubernetes world, the atomic unit is the *Pod*. This means deploying applications on Kubernetes means stamping them out in Pods

As Pods are the fundamental unit of deployment in Kubernetes, it’s vital you understand how they work.

### Pod vs containers
We said that Pod hosts one or more containers. From a footprint perspective, this puts Pods somewhere in between containers and VMs - they're a tiny bit bigger than a container, but a lot smaller than a VM.

A Pod is a *shared execution environment* for one or more containers.
The simplest model is the one-container-per-Pod model. However, multi-container Pods are gaining in popularity and are important for advanced configurations

An application-centric use-case for multi-container Pods is co-scheduling tightly-coupled workloads. For example, two containers that share memory won’t work if they are scheduled on different nodes in the cluster.
By putting both containers inside the same Pod, you ensure that they are scheduled to the same node and share the same execution environment

An infrastructure-centric use-case for multi-container Pods is a service mesh. In the service mesh model, a proxy container is inserted into every application Pod.
This proxy container handles all network traffic entering and leaving the Pod, meaning it is ideally placed to implement features such as traffic encryption, network telemetry, intelligent routing, and more.

### Multi-container Pods: the typical example
A common example for comparing single-container and multi-container Pods is a web server that utilizes a file synchronizer

In this example there are two clear concerns:
1. Serving the web page
2. Making sure the content is up to date

the question is whether to address the two concerns in a single container or two separate containers.

In this context, a concern is a requirement or a task. Generally speaking, microservices design patterns dictate that we separate concerns. This means we only every deal with one concern per container.

Assuming the previous example, this will require 2 containers: one for the web service, another for the file-sync service.

Advantages, including:
- Different teams can be responsible for each of the 2 containers
- Each container can be scaled independently
- Each container can be developed and iterated independently
- Each container can have its own release cadence
- If one fails, the other keeps running

It's often a requirement to co-schedule those separate containers in a single Pod. This makes sure both containers are scheduled to the same node and share the same execution environment (the Pod's environment)

Common user-cases for multi-container Pods include; two containers that need to share memory or share a volume

![](Screen-shots/multi-container%20pods.png)

### How do we deploys pods
Pods are just a vehicle for executing an application. Therefore, any time we talk about running or deploying Pods, we're talking about running and deploying applications

To deploy a Pod to a Kubernetes cluster, you define it in a *manifest file* and `POST` that manifest file to the API server.

The control plane verifies the configuration of the `YAML` file, writes it to the cluster store as a record of intent, and the scheduler deploys it to a healthy node with enough available resources. This process is identical for single-container Pods and multi-container Pods


![](Screen-shots/Deploying%20pod.png)

### The anatomy of a Pod
At the highest level, a Pod is a shared execution environment for one or more containers. *Shared execution environment* means that the Pod has a set of resources that are shared by every container that is part of the Pod. These resources include; IP addresses, ports, hostname, sockets, memory, volumes, and more...

>If you're using Docker as the container runtime, **a Pod is actually a** special type of container called a **pause container**. That's right, a Pod is just a fancy name for a special container. ***This means containers running inside of Pods are really containers running inside of containers.***

The Pod (pause container) is just a collection of system resources that containers running inside of it will inherit and share. These system resources are kernel namespaces and include:
- **Network namespace**: IP address, port range, routing table ...
- **UTS namespace**: Hostname
- **IPC namespace**: Unix domain sockets

As we just mentioned, this means that all containers in a Pod share a hostname, IP address, memory address space, and volumes

### Pods and shared networking

Each Pod creates its own network namespace. This includes a single IP address, a single range of TCP and UDP ports, and a single routing table. If a Pod has a single container, that container has full access to the IP, port range and routing table. If it's a multi-container Pod, all containers in the Pod will share the IP, port range and routing table

![](Screen-shots/Networking%20in%20Pods.png)

All containers in a Pod have access to the same volumes, the same memory, the same IPC sockets, and more. Technically speaking, the Pod holds all the namespaces, any containers that are part of the Pod inherit them and share them.

This networking model makes inter-Pod communication really simple. Every Pod in the cluster has its own IP addresses that's fully routable on the Pod network

Intra-pod communication - where two containers in the same Pod need to communicate - can happen via the Pod's `localhost` interface.

If you need to make multiple containers int he same Pod available to the outside world, you can expose them on individual ports. Each container needs its own port, and two containers in the same Pod cannot use the same port.

### Pods and cgroups
At a high level, Control Groups (cgroups) are a linux kernel technology that prevents individual containers from consuming all of the available CPU, RAM, and IOPS on a node --> A POLICE resource usage

individual containers have their own cgroup limits

This means it's possible for two containers in the same Pod to have their own set of cgroup limits

### Atomic deployment of Pods
Deploying a Pod is an *atomic operation*. This means it's an all-or-nothing operation - there's no such thing as a partially deployed Pod that can service requests. It also means that all containers in a Pod will be scheduled on the same node

### Pod lifecycle

The lifecycle of a typical Pod goes something like this. 
- You define it in a YAML manifest file and POST the manifest to the API server. Once there, the contents of the manifest are persisted to the cluster store as a record of intent (desired state), and the Pod is scheduled to a healthy node with enough resources. 
- Once it’s scheduled to a node, it enters the pending state while the container runtime on the node downloads images and starts any containers. The Pod remains in the pending state until all of its resources are up and ready.
- Once everything’s up and ready, the Pod enters the running state.
- Once it has completed all of its tasks, it gets terminated and enters the succeeded state.

When a Pod can't start, it can remain in the *pending* state or go to the failed state.

![](Screen-shots/pod%20lifecycle.png)

Pods that are deployed via pod manifest files are *singletons* - they are not managed by a controller that might add features such as auto-scaling and self-healing capabilities. For this reason, we almost always deploy Pods via higher-level controllers such as Deployments and DaemonSets, as these can reschedule Pods when they fail.

This is one of the main reasons you should design your applications so that they don't store state in Pods. It's also why we shouldn't rely on individual pod IPs. Singletion Pods are not reliable



### Pod Theory SUMMMARY
1. Pods are the atomic unit of scheduling in Kubernetes
2. You can have more than one container in a Pod. Single-container Pods are the simplest, but multi-container Pods are ideal for containers that need to be tightly coupled. They’re also great for logging and service meshes
3. Pods get scheduled on nodes – you can’t schedule a single Pod instance to span multiple nodes
4. Pods are defined declaratively in a manifest file that is POSTed to the API Server and assigned to nodes by the scheduler
5. You almost always deploy Pods via higher-level controllers

## Pod hands-on
### Pod manifest file
```yml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1

spec:
  containers:
    - name: hello-ctr
      image: nigelpoulton/k8sbook:latest
      ports:
        - containerPort: 8080
```

We can see four top-level resources:
- `apiVersion`
- `kind`
- `metadata`
- `spec`

The `apiVersion` field tells you two things - the API group and the API version. The usual format for `apiVersion` is `<api-group>/<version>`. However, Pods are in the core API group which is special, as it omits the API group name, so we describe them in YAML files as just `v1`.

The `kind` field tells Kubernetes the type of object is being deployed

The `metadata` section is where you attach a name and labels. These help you identify the object in the cluster, as well as create loose coupling between different objects. 
You can also define the Name space that an object should be deployed to. Keeping things brief, Namespace are a way to logically divide a cluster into multiple virtual clusters for management purposes. In the real world, it's highly recommended to use namespaces, however, you should no think of them as strong security boundaries.

As the `metadata` section does not specify a Namespace, the Pod will be deployed to the default Namespace. It’s not good practice to use the default namespace in the real world, but it’s fine for these examples.

The `spec` section is where you define the containers that will run in the pod. This example is deploying a Pod with a single container based on the `nigelpoulton/k8sbook:latest` image.

If this was a multi-container Pod, you'd define additional containers in the `spec` section.

### Deploying Pods from a manifest file
```bash
kubectl apply -f pod.yml

kubectl get pods
```

### Introspecting running Pods
As good as the `kubectl get pods` command is, it's a bit light on detail. Not to worry though, there's plenty of options for deeper introspection.

The `-o wide` flag gives a couple more columns but is still a single line of output.
The `-o yaml` flag takes things to the next level. This returns a full copy of the Pod manifest from the cluster store.
The output is broadly divided into 2 parts:
- Desired state (`spec`)
- current observed state (`status`)

```bash
$ kubectl get pods -o yaml                                                                            
apiVersion: v1                                                                                           
items:                                                                                                   
- apiVersion: v1                                                                                         
  kind: Pod                                                                                              
  metadata:                                                                                              
    annotations:                                                                                         
      kubectl.kubernetes.io/last-applied-configuration: |                                                
        {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"labels":{"version":"v1","zone":"prod"},"name":"hello-pod","namespace":"default"},"spec":{"containers":[{"image":"nigelpoulton/k8sbook:latest"
,"name":"hello-ctr","ports":[{"containerPort":8080}]}]}}                                                 
    creationTimestamp: "2021-07-19T04:12:38Z"                                                                                                                                                                          labels:                                         
      version: v1                                                                                        
      zone: prod                                                                                         
    name: hello-pod                                                                                      
    namespace: default                                                                                   
    resourceVersion: "1419"                                                                              
    uid: 7fc2aba3-6437-42e9-9ce4-814fd811c644                                                            
  spec:                                                                                                  
    containers:                                                                                          
    - image: nigelpoulton/k8sbook:latest                                                                 
      imagePullPolicy: Always                                                                            
      name: hello-ctr                                                                                    
      ports:                                                                                             
      - containerPort: 8080                                                                              
        protocol: TCP                                                                                    
      resources: {}                                                                                      
      terminationMessagePath: /dev/termination-log                                                       
      terminationMessagePolicy: File                                                                     
      volumeMounts:                                                                                      
      - mountPath: /var/run/secrets/kubernetes.io/serviceaccount                                         
        name: default-token-6w5wv                                                                        
        readOnly: true                 
	<snip>
```

#### kubectl describe
```bash
$ kubectl describe pods hello-pod 
Name:         hello-pod
Namespace:    default
Priority:     0
Node:         minikube/192.168.49.2
Start Time:   Mon, 19 Jul 2021 11:12:38 +0700
Labels:       version=v1
              zone=prod
Annotations:  <none>
Status:       Running
IP:           172.17.0.3
IPs:
  IP:  172.17.0.3
Containers:
  hello-ctr:
    Container ID:   docker://1392e48cf10ceb28d81d79168b777bb5b4f37ff1570b2314bf2e02a560498679
    Image:          nigelpoulton/k8sbook:latest
    Image ID:       docker-pullable://nigelpoulton/k8sbook@sha256:a983a96a85151320cd6ad0cd9fda3b725a743ed642e58b0597285c6bcb46c90f
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 19 Jul 2021 11:13:00 +0700
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-6w5wv (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-6w5wv:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-6w5wv
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/hello-pod to minikube
  Normal  Pulling    10m   kubelet            Pulling image "nigelpoulton/k8sbook:latest"
  Normal  Pulled     10m   kubelet            Successfully pulled image "nigelpoulton/k8sbook:latest" in 21.433527195s
  Normal  Created    10m   kubelet            Created container hello-ctr
  Normal  Started    10m   kubelet            Started container hello-ctr
```

#### kubectl exec: running commands in pods
```bash
$ kubectl exec hello-pod -- ps aux 
PID   USER     TIME  COMMAND
    1 root      0:00 node ./app.js
   18 root      0:00 ps aux
```

You can also log-in containers running in Pods using `kubectl exec`. When you do this, your terminal prompt will change to indicate your session is now running inside of a container in the Pod, and you'll be able to execute commands from there.

```bash
$ kubectl exec -it hello-pod -- sh  
/src # whoami
root
/src # hostname
hello-pod
```

#### kubectl logs
One other useful command for introspecting Pods is the `kubectl logs` command. Like other Pod-related commands, if you don't use `--container` to specify a container by name, it will execute against the first container in the Pod. The format of the command is `kubectl logs <pod>`.

### Delete pod
```bash
$ kubectl delete -f pod.yml
pod "hello-pod" deleted
```

## Chapter summary
In this chapter, you learned that the atomic unit of deployment in the Kubernetes world is the Pod. Each Pod consists of one or more containers and gets deployed to a single node in the cluster. The deployment operation is an all-or-nothing atomic operation.

Pods are defined and deployed declaratively using a YAML manifest file, and it’s normal to deploy them via higher-level controllers such as Deployments. You use the kubectl command to POST the manifest to the API Server, it gets stored in the cluster store and converted into a PodSpec that is scheduled to a healthy cluster node with enough available resources.

The process on the worker node that accepts the PodSpec is the kubelet. This is the main Kubernetes agent running on every node in the cluster. It takes the PodSpec and is responsible for pulling all images and starting all containers in the Pod.

If you deploy a singleton Pod (a Pod that is not deployed via a controller) to your cluster and the node it is running on fails, the singleton Pod is not rescheduled on another node. Because of this, you almost always deploy Pods via higher-level controllers such as Deployments and DaemonSets. These add capabilities such as self-healing and rollbacks which are at the heart of what makes Kubernetes so powerful.

