## Deployment theory
At a high level, you start with application code. That gets packaged as a container and wrapped in a Pod so it can run on Kubernetes. However, Pods don't self-heal they don't scale, and they don't allow for easy updates or rollbacks. Deployments do all of these. As a result, you'll almost always deploy Pods via a Deployment controller

![](Screen-shots/Deployment.png)

THe next thing to know is that Deployments are fully-fledged objects in the Kubernetes API. This means you define them in the manifest files that you POST to the API server.

The last thing to note, is that behind the scenes, Deployments leverage another object called a ReplicaSet. While it's best practice that you don't interact directly with ReplicaSets, it's important to understand the role they play.

Keeping it high-level, Deployments use ReplicaSets to provide self-healing and scaling.

In summary, think of Deployments as managing ReplicaSets, and ReplicaSets as managing Pods. Put them all together, and you've got a great way to deploy and manage applications on Kubernetes.

![](Screen-shots/Deployments%20with%20replicas.png)

### Self-healing and scalability
Deployments augment Pods by adding things like self-healing and scalability. This means:
- If a Pod managed by a Deployment fails, it will be replaced - *self-healing*
- If a Pod managed by a Deployment sees increased load, you can easily add more of the same Pod to deal with the load - *scaling*

Remember though, behind-the-scenes, Deployments use an object called a ReplicaSet to accomplish self-healing and scalability. 
However, ReplicaSets operate in the background and you should always carry out operations against the Deployment. For this reason, we’ll focus on Deployments

#### It's all about the *state*
3 concepts that are fundamental to everything about Kubernetes:
- Desired state
- Current state (sometimes called actual state or observed state)
- Declarative model

Desired state is what you **want**. Current state is what you **have**. If the 2 match, everyone is happy.

The declarative model is a way of telling Kubernetes what your desired state is, without having to get into the detail of how to implement it. You leave the how up to Kubernetes.

#### the declarative model
There are two competing models. The declarative model and the imperative model.

- The declarative model is all about describing the end-goal. Telling Kubernetes what you want.

- The imperative model is all about long lists of commands to reach the end-goal. Telling Kubernetes how to do something.

#### Reconciliation loops
Fundamental to desired state is the concept of background reconciliation loops (control loops)

>Kubernetes is constantly making sure that current state matches desired state.

### Rolling updates with deployments
As well as self-healing and scaling. Deployments give us zero-downtime rolling-updates.

As previously mentioned, Deployments use replicasets for some of the background legwork. In fact, every time you create a Deployment, you automatically get a ReplicaSet that manages the Deployment's Pods

Note: best practices states that you should not manage ReplicaSets directly. You should perform all actions against the Deployment object and leave the Deployment to manege ReplicaSets.

It works like this. **You design applications with each discrete service as a Pod.** For convenience - self-healing, scaling, rolling updates and more - you wrap Pods in Deployments. This means creating a YAML configuration file describing all of the following
- How many Pod replicas
- What image to use for the Pod's container(s)
- What network ports to use
- Details about how to perform rolling updates

You POST the YAML file to the API server and Kubernetes does the rest.

Once everything is up and running, Kubernetes sets up watch loops to make sure observed state machines desired state.

Now, assume you’ve experienced a bug, and you need to deploy an updated image that implements a fix. To do this, you update the same Deployment YAML file with the new image version and re-POST it to the API server.

This registers a new desired state on the cluster, requesting the same number of Pods, but all running the new version of the image. To make this happen, **Kubernetes creates a new ReplicaSet for the Pods with the new image**. 
You now have two ReplicaSets – the original one for the Pods with the old version of the image, and a new one for the Pods with the updated version. 

***Each time Kubernetes increases the number of Pods in the new ReplicaSet (with the new version of the image) it decreases the number of Pods in the old ReplicaSet (with the old version of the image). Net result, you get a smooth rolling update with zero downtime.*** 

And you can rinse and repeat the process for future updates – just keep updating that manifest file (which should be stored in a version control system).

![](Screen-shots/Rolling%20updates.png)

### Rollbacks
As we've seen, older ReplicaSets are wound down and no longer manage any Pods. ***However, they still exist with their full configuration. This makes them a great option for reverting to previous versions***

![](Screen-shots/Rollback.png)

That's not the end though. There's built-in intelligence that lets us say things like "wait X number of seconds after each Pod comes up before proceeding to the next Pod". There's also startup probes, readiness probes, and liveness proves that can check the health and status of Pods. All-in-all, Deployments are excellent for performing rolling updates and versioned rollbacks.

## How to create a Deployment

```yaml
apiVersion: apss/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  replicas: 10
  selector:
    mathcLabels:
      app: hello-world

  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  template: 
    metadata:
      lables:
        app: hello-world

    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:latest
        ports:
        - containerPort: 8080
```

The `spec` section is where most of the action happens. Anything directly below `spec` relates to the Pod. Anything nested below `spec.template` relates to the Pod template that the Deployment will manage. In this example, the Pod template defines a single container.

`spec.replicas` tells Kubernetes how may Pod replicas to deploy `spec.selector` is a list of labels that Pods must have in order for the Deployment to manage them. And `spec.strategy` tells Kubernetes how to perform updates to the POds managed by the Deployment

```bash
$ kubectl apply -f deploy.yml
deployment.apps/hello-deploy created
```

### Inspecting deployments

```bash
$ kubectl get deploy hello-deploy                                                                                                                   
NAME           READY   UP-TO-DATE   AVAILABLE   AGE                                                                                                    
hello-deploy   6/10    10           4           58s
```

```bash 
$ kubectl describe deploy hello-deploy                                                                                                                                                                          Name:                   hello-deploy                                                                        
Namespace:              default
CreationTimestamp:      Mon, 19 Jul 2021 15:58:16 +0700                                                     
Labels:                 <none>                                                                              
Annotations:            deployment.kubernetes.io/revision: 1                                                                                                                                                       Selector:               app=hello-world                                                                  
Replicas:               10 desired | 10 updated | 10 total | 8 available | 2 unavailable                                                                                                                           StrategyType:           RollingUpdate                                      
MinReadySeconds:        10                                                                                  
RollingUpdateStrategy:  1 max unavailable, 1 max surge                     
Pod Template:             
  Labels:  app=hello-world                                                 
  Containers:                                                              
   hello-pod:                                                                                                                                          
    Image:        nigelpoulton/k8sbook:latest                              
    Port:         8080/TCP
    Host Port:    0/TCP                               
    Environment:  <none>                              
    Mounts:       <none>                              
  Volumes:        <none>                              
Conditions:                                           
  Type           Status  Reason                       
  ----           ------  ------                                                                             
  Available      False   MinimumReplicasUnavailable                                                      
  Progressing    True    ReplicaSetUpdated                                                               
OldReplicaSets:  <none>                             
NewReplicaSet:   hello-deploy-65cbc9474c (10/10 replicas created)                                        
Events:                                             
  Type    Reason             Age   From                   Message                                        
  ----    ------             ----  ----                   -------                                        
  Normal  ScalingReplicaSet  80s   deployment-controller  Scaled up replica set hello-deploy-65cbc9474c to 10
```

To get replicasets 
```bash

```

### Accessing the app
In order to access the application from a stable name or IP address, or even from outside the cluster, you need a Kubernetes Services object. For now it's enough to know they provide a stable DNS name and IP address for a set of Pods.

`service.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world

spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
    protocol: TCP
  selector:
    app: hello-world
```

Deploy it
```bash
$ kubectl apply -f service.yml 
service/hello-svc created
```

Now that the Service is deployed, you can access the app from either of the following:
1. From inside the cluster using the DNS name `hello-svc` on port 8080
2. From outside the cluster by hitting any of the cluster nodes on port 30001

