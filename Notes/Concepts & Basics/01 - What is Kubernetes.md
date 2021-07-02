## Table of Contents
- [[#Container overview|Container overview]]
- [[#Why Kubernetes and what it can do|Why Kubernetes and what it can do]]
- [[#What kubernetes is not|What kubernetes is not]]


Kubernetes is a portable, extensible, open-source platform for managing containerized workloads and services, that facilitates both declarative configuration and automation. It has a large, rapidly growing ecosystem. Kubernetes services, support, and tools are widely available.

## Container overview

**Container deployment era**: COntainers are similar to VMs, but they have relaxed isolation properties to share the OS among the application. Therefore, containers are considered lightweight. Similar to a VM, a container has its own filesystem, share of CPU, memory, process space, and more. As they are decoupled from the underlying infrastructure, they are portable across clouds and OS distributions

## Why Kubernetes and what it can do
If a container goes down, another container needs to start. Wouldn't it be easier if this behavior was handled by a system?

Kubernetes provides you with a framework to run distributed systems resiliently. It takes care of scaling and failover for your application, provides deployment patterns, and more. For example, Kubernetes can easily manage a canary deployment for your system.

Kubernetes provides you with:
- **Service discovery and load balancing**. Kubernetes can expose a container using the DNS name or using their own IP address. If traffic to a container is high, Kubernetes is able to load balance and distribute the network traffic so that the deployment is stable

- **Storage orchestration**. Kubernetes allows you to automatically mount a storage system of your choice, such as local storages, public cloud providers, and more.

- **Automatic bin packing**. You provide Kubernetes with a cluster of nodes that it can use to run containerized tasks. You tell Kubernetes how much CPU and memory (RAM) each container needs. Kubernetes can fit containers onto your nodes to make the best use of your resources

- **Automated rollouts and rollbacks**. You can describe the desired state for your deployed containers using Kubernetes, and it can change the actual  state to the desired state at a controlled rate. For example, you can automated Kubernetes to create new containers for your deployement, remove existing containers and adopt all their resources to the new container

- **Self-healing**. Kubernetes restarts containers that fail, replaces containers, kills containers that don't respond to your user-defined health check, and doesn't advertise them to clients until they are ready to serve.

- **Secret and configuration management**. Kuberentes lets you store and manage sensitive information, such as passwords, OAuth tokens, and SSH keys. You can deploy and update secrets and application configuration without rebuilding your container images, and without exposing secrets in your stack configuration

## What kubernetes is not
Kuberentes is not a traditional, all-inclusive PaaS (platform as a service) system. Kubernetes provides the building blocks for building developer platforms, but preserves user choice and flexibility where it is important.

Kubernetes:
- ***Does not*** limit the types of applications supported. Kuberentes aims to support and extremely diverse variety of workloads, including stateless, stateful, and data processing workloads. If an application can run in a container, it should run greate on Kubernetes

- ***Does not*** deploy source code and does not build your application. CI/CD workflows are determined by organization cultures and preferences as well as technical requirements.

- ***Does not*** provide application-level services, such as middleware (for example, essage buses), data-processing frameworks (Spark,...), databases (Mysql, ...), caches, nor cluster storage systems (Ceph, ...) as built-in services. Such components can run on Kubernetes, and/or can be accessed by applications running on Kubernetes through portable mechanisms, such as the Open Service Broker 

- ***Does not*** dictate logging, monitoring, or alerting solutions. It provides some integrations as proof of concept, and mechanisms to collect and export metrics.

- ***Does not*** provide nor mandate a configuration language/system (for example, Jsonnet). It provides a declarative API that may be targeted by arbitrary forms of declarative specifications

- Additionally, ***Kubernetes is not a mere orchestration system***. In fact, it eliminates the need for orchestration. The technical definition of orchestration is execution of a defined workflow; first do A, then B, then C. In contrast, Kubernetes comprises a set of independent, composable control processes that continuously drive the current state towards the provided desired state. This result in a system that is easier to use and more powerful, robust, resilient, and extensible

## Kubernetes and Docker
Kubernetes and Docker are complementary technologies. For example, it's common to develop you applications with Docker and use Kubernetes to orchestrate them in production

At a high-level , you might have a Kubernetes cluster with 10 nodes to run your production applications. Behind the scenes, each node is running Docker as its *container runtime*. This means that Docker is the low-level technology that starts and stops the containerized applications. Kubernetes is the higher-level technology that looks after the bigger picture, such as; deciding which nodes to run containers on, deciding when to scale up or down, and executing updates

![](Screen-shots/Cluster.png)

Kubernetes has a couple of features that abstract the container runtime (make it interchangeable)

1. The container Runtime Interface (CRI) is an abstraction layer standardizes the way 3rd-party container runtimes interface with Kubernetes. It allows the container runtime code to exist outside of Kubernetes, but interface with it in a supported and standardized way

2. Runtime Classes allows for different *classes* of runtimes. For example, the `gVisor` or `Kata Containers` runtimes might provide better workload isolation than the `Docker` and `containerd` runtimes

At the time of writing, `containerd` is catching up to Docker as the most commonly used container runtime in Kubernetes. It is a stripped-down version of Docker with just the stuff that Kubernetes needs

## The operating system of the cloud
In many ways, kubernetes is like an operating system (OS) for the cloud
- You install a traditional OS (Linux or Windows) on a server, and the OS abstracts the physical server's resources and schedules processes

- You install Kubernetes on a cloud, and it *abstracts* the cloud's resources and schedules the various microservices of cloud-native applications

In the same way that Linux *abstract* the hardware differences of different server platforms, ***Kubernetes abstracts the differences between different private and public clouds***. Net result ...as long as you're running Kubernetes, it doesn't matter if the underlying systems are in premises in your own data center, edge clusters, or in the public cloud

With this in mind, Kubernetes enables a true *hybird cloud*, allowing you to seamlessly move and balance workloads across multiple different public and private cloud infrastructure. You can also migrate to and from different clouds, meaning you can choose a cloud today and not have to stick with that decision for the rest of your life

## Application Scheduling
At a high-level, a cloud or data center is a pool of compute, network and storage. Kubernetes abstract it. This means we don't have hard code which node or storage volume our applications run on, we don't even have to care which cloud they run on - we let Kubernetes take care of that.