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

### Performing a rolling update
The updated `deploy.yml` manifest file. The change is to `spec.template.spec.containers.image`.

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

  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1

  template: 
    metadata:
      labels:
        app: hello-world

    spec:
      containers:
      - name: hello-pod
        image: nigelpoulton/k8sbook:edge
        ports:
        - containerPort: 8080

```

The `spec` section of the manifest contains all of the settings relating to how updates will be performed. The first value of interest is `spec.minReadySeconds`. This is set to `10`, telling Kubernetes to wait for 10 seconds between each Pod being updated. This is useful for throttling the rate at which updates occur -  longer waits give you a chance to spot problems and avoid situations where you update all Pods to a faulty configuration.

There is also a nested `spec.strategy` map that tells Kubernetes you want this Deployment to:
- Update using the RollingUpdate strategy
- Never have more than one Pod below desired state (`maxUnavailable: 1`)
- Never have more than one Pod above desired state (`maxSurge: 1`)

As the desired state of the app demands 10 replicas, `maxSurge: 1` means you will never have more than 11 Pods during the update process, and `maxUnavailable: 1` means you'll never have less than 9. The net result will be a rolling update that updates two Pods at a time (the delta between 9 and 11 is 2)

```bash
$ kubectl apply -f deploy.yml --record
deployment.apps/hello-deploy configured
```

Monitor the progress of update
```bash
$ kubectl rollout status deployment hello-deploy
```

The update may take some time to complete. This is because it will iterate 2 Pods at a time, pulling down the new image on each node, starting the new Pods, and then waiting 10 seconds before moving on to the next two.

Once the update is complete, we can verify with 
```bash
$ kubectl get deploy hello-deploy
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-deploy   10/10   10           10          5h39m
```

### How to perform a rollback
With the `--record` flag, Kubernetes would maintain a documented revision history of the Deployment. The following `kubectl rollout history` command shows the Deployment with 2 revisions

```bash
$ kubectl rollout history deployment hello-deploy
deployment.apps/hello-deploy 
REVISION  CHANGE-CAUSE
1         <none>
2         kubectl apply --filename=deploy.yml --record=true
```

This is only because you used the `--record` flag as part of the command to invoke the update. This might be a good reason for you to use the `--record` flag.

Updating a Deployment creates a new ReplicaSet, and that any previous ReplicaSets are not deleted. You can verify this with a `kubectl get rs`

```bash
$ kubectl get rs
NAME                      DESIRED   CURRENT   READY   AGE
hello-deploy-59866ff45    10        10        10      21m
hello-deploy-65cbc9474c   0         0         0       5h58m
```

ReplicaSet id `59866ff45` is the one from the latest revision and is active with 10 replicas under management. However, the fact that the previous version still exists makes rollbacks extremely simple.

It's worth running `kubectl describe rs` against the old ReplicaSet to prove that its configuration still exists.

```bash
$ kubectl describe rs hello-deploy-65cbc9474c
Name:           hello-deploy-65cbc9474c
Namespace:      default
Selector:       app=hello-world,pod-template-hash=65cbc9474c
Labels:         app=hello-world
                pod-template-hash=65cbc9474c
Annotations:    deployment.kubernetes.io/desired-replicas: 10
                deployment.kubernetes.io/max-replicas: 11
                deployment.kubernetes.io/revision: 1
Controlled By:  Deployment/hello-deploy
Replicas:       0 current / 0 desired
Pods Status:    0 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  app=hello-world
           pod-template-hash=65cbc9474c
  Containers:
   hello-pod:
    Image:        nigelpoulton/k8sbook:latest
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From                   Message
  ----    ------            ----  ----                   -------
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-7g9jp
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-2sdjc
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-7lk74
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-wd592
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-rjcft
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-gtv8t
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-95br7
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-hclsq
  Normal  SuccessfulCreate  6h1m  replicaset-controller  Created pod: hello-deploy-65cbc9474c-bzwq9
  Normal  SuccessfulCreate  6h1m  replicaset-controller  (combined from similar events): Created pod: hello-deploy-65cbc9474c-xdft5
  Normal  SuccessfulDelete  25m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-2sdjc
  Normal  SuccessfulDelete  24m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-gtv8t
  Normal  SuccessfulDelete  24m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-bzwq9
  Normal  SuccessfulDelete  24m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-xdft5
  Normal  SuccessfulDelete  23m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-rjcft
  Normal  SuccessfulDelete  23m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-95br7
  Normal  SuccessfulDelete  23m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-hclsq
  Normal  SuccessfulDelete  23m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-wd592
  Normal  SuccessfulDelete  23m   replicaset-controller  Deleted pod: hello-deploy-65cbc9474c-7lk74
  Normal  SuccessfulDelete  22m   replicaset-controller  (combined from similar events): Deleted pod: hello-deploy-65cbc9474c-7g9jp
```

The following `kubectl rollout` command to roll the application back to revision 1. This is an imperative operation and not recommended. But it can be convenient for quick rollbacks, just remember to update your source YAML files to reflect the imperative changes you make to the cluster.

```bash
$ kubectl rollout undo deployment hello-deploy --to-revision=1
deployment.apps/hello-deploy rolled back
```

**Although it might look like the rollback operation is instantaneous, it’s not**.
Rollbacks follow the same rules set out in the rolling update sections of the Deployment manifest – minReadySeconds: 10, maxUnavailable: 1, and maxSurge: 1. You can verify this and track the progress with the following `kubectl get deploy` and `kubectl rollout` commands.

Just a quick reminder. The rollback operation you just initiated was an imperative operation. This means that the current state of the cluster will not match your source YAML files – the latest version of the YAML file lists the edge image, but you’ve rolled the cluster back to the latest image. This is a problem with the imperative
approach. In the real world, following a rollback operation like this, you should manually update your source YAML files to reflect the changes incurred by the rollback.

## Chapter summary
In this chapter, you learned that Deployments are a great way to manage Kubernetes apps. They build on top of Pods by adding self-healing, scalability, rolling updates, and rollbacks. Behind-the-scenes, they leverage ReplicaSets for the self-healing and scalability parts. 

Like Pods, Deployments are objects in the Kubernetes API, and you should work with them declaratively. When you perform updates with the kubectl apply command, older versions of ReplicaSets get wound down,
but they stick around making it easy to perform rollbacks