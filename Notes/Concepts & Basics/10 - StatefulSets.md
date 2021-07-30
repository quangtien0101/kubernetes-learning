For the purpose of of this chapter, we're defining a stateful application as an application that creates and saves valuable data. An example might be an app that saves data about client sessions and uses it for future client sessions.

## The theory of StatefulSets
It's often useful to compare StatefulSets with Deployments. Both are first-class objects in the Kubernetes API and follow the typical Kubernetes controller architecture. These controllers run as reconciliation loops that watch the state of the cluster, via the API server, and are constantly moving the observed state of the cluster into sync with desired state. Deployments and StatefulSets also support self-healing, scaling, updates, and more.

However, there are some vital differences. StatefulSets guarantee:
- predictable and persistent Pod names
- predictable and persistent DNS hostnames
- predictable and persistent volume bindings

These three properties form the *state* of a Pod, sometimes referred to as the Pods *sticky ID*. This state/sticky ID is persisted across failures, scaling, and other scheduling operations, making StatefulSets ideal for applications where Pods are a little bit unique and not interchangeable.

>failed Pods managed by a StatefulSet will be replaced by new Pods with the exact same Pod name, the exact same DNS hostname, and the exact same volumes. This is true even if the replacement Pod is started on a different cluster Node. The same is not true of Pods managed by a Deployment.

### StatefulSet Pod naming
All Pods managed by a StatefulSet get *predictable* and *persistent* names. These names are vital, and are at the core of how Pods are started, self-healed, scaled, deleted, attached to volumes, and more.

The format of StatefulSet Pod names is `<StatefulSetName>-<Integer>`.The integer is a zero-based index ordinal, which is just a fancy way of saying "number starting from 0" 

The first Pod created by a StatefulSet always gets index ordinal "0", and each subsequent Pod gets the next highest ordinal. 

Be aware that StatefulSet names need to be a valid DNS names, so no exotic characters! 

### Ordered creation and deletion
Another fundamental characteristic of StatefulSets is the controller and ordered way they start and stop Pods.

**StatefulSets create one Pod at a time, and always wait for previous Pods to be *running and ready* before creating the next.**

This is different from Deployments that use a ReplicaSet controller to start all Pods at the same time, causing potential race conditions.

Note: *Running and ready* are technical terms used to indicate all containers in a Pod are executing and the Pod is ready to service requests.

Scaling operations are also governed by the same ordered startup rules. Scaling down follows the same rules in reverse - the controller terminates the Pod with the highest index ordinal (number) first, waits for it to fully terminate before terminating the Pod with the next highest ordinal.

Knowing the order in which Pods will be scaled down, as well as knowing that Pods will not be terminated in parallel, is a game changer for many stateful apps. 

For example, clustered apps that store data are usually at high risk of losing data if multiple replicas go down at the same time. StatefulSets guarantee this will never happen, and you can insert other delays via things like `terminationGracePeriodSeconds` to further control the scaling down process.

Finally, it's worth noting that StatefulSet controllers do their own self-healing and scaling. This is architecturally different to Deployments which use a separate ReplicaSet controller for these operations.

### Deleting StatefulSets
The are 2 major things to consider when deleting StatefulSets

**Firstly, deleting a StatefulSet does not terminate Pods in order. With this in mind, you may want to scale StatefulSet to 0 replicas before deleting it**

You can also use `terminationGracePeriodSeconds` to further control the way Pods are terminated. It's common to set this to at least 10 seconds to give applications running in Pods a chance to flush local buffers and safely commit any writes that are still in-flight.

### Volumes
Volumes are an important part of a StatefulSet Pods sticky ID (state).

When a StatefulSet Pod is created, any volumes it needs are created, any volumes it needs are created at the same time and named in a way to connect them to the right Pod.

![](Concepts%20&%20Basics/volumes%20statefulsets.png)

Volumes are appropriately decoupled from Pods via the normal Kubernetes persistent volume subsystem constructs (PersistentVolumes and PersistentVolumeClaims). This means volumes have separate lifecycles to Pods and allows volumes to survive Pod failures and termination operations. For example, anytime a StatefulSet Pod fails or is terminated, the associated volumes are unaffected. This allows replacement Pods to attach to the same storage as the Pods they're replacing. This is true, even if the replacement Pod is scheduled to a different cluster Node.

The same is true is true for scaling operations. If a StatefulSet Pod is deleted as part of a scale-down operation, subsequent scale-up operations will attach new Pods to the existing volumes that match their names.

This behavior can be a life-saver if you accidentally delete a StatefulSet Pod, especially if it's the last replica!

### Handling failures
The StatefulSet controller observes the state of the cluster and attempts to keep observed state in sync with desired state.

The simplest example is a Pod failure. If you have a StatefulSet called `tkb-sts` with 5 replicas, and `tkb-sts-3` fails, the controller will start a replacement Pod with the same name and attach it to the same volumes.

However, if a failed Pod recovers after Kubernetes has replaced it, you'll have two identical Pods on the network writing to the same volumes. This can result in data corruption. With this in mind, the StatefulSet controller is extremely careful how it handles failures.

Potential Node failures are especially difficult to deal with. For example, if Kubernetes loses contact with a Node, how does it know if the Node is down and will never recover, or if it’s a temporary glitch such as a network partition, a crashed Kubelet, or the Node is simply rebooting? To complicate matters further, the controller can’t even force the Pod to terminate, as the Kubelet may never receive the instruction. With these things in mind, manual intervention is needed before Kubernetes will replace Pods on failed Nodes.

### Network ID and headless Services
We've already said that StatefulSets are for applications that need Pods to be predictable and longer-lived. As a result, other parts of the application as well as other applications may need to connect directly to individual Pods. To make this possible, StatefulSets use a headless Service to create predictable DNS hostnames for every Pod replica. Other apps can then query DNS for the full list of Pod replicas and use these details to connect directly to Pods.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: mongo-prod
spec:
	clusterIP: None
	selector:
		app: mongo
		env: prod
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
	name: sts-mongo
spec:
	serviceName: mongo-prod
```

Let's explain the term *headless Service* and *governing Service*.

When the two objects are combined like this, the Service will create DNS SRV records for each Pod replica matching the label selector of the headless Service. Other Pods can then find members of the StatefulSet by performing DNS lookups against the name of the headless Service. You’ll see this in action later, and obviously applications will need to know to do this.

### Creating a governing headless Service
When learning about headless Services, it can be useful to visualize a Service object with a head and a tail. The head is the stable IP exposed on the network, and the tail is the list of Pods it will send traffic to. Therefore, a headless Service is a Service object without a ClusterIP.

```yaml
apiVersion: v1
kind: Service
metadata:
	name: headless-service
	labels:
		app: web
spec:
	ports:
	- port: 80
	  name: web
	clusterIP: None
	selector:
		app: web
```

The only different to a regular Service is that a headless Service must set the value `clusterIP` to `None`

When combined with a StatefulSet, headless Service create predictable stable DNS entries for every Pod that matches the StatefulSets label selector. You'll see this in a later step.

### Deploy StatefulSet
Each StatefulSet Pod needs its own unique storage. This means each one needs its own PVC. However, this isn't possible, as each Pod is created from the same template. Also you'd have to pre-create a unique PVC for every potential StatefulSet Pod, which also isn'y possible when you consider StatefulSets can be scaled up and down.

Clearly, a more intelligent StatefulSet-aware approach is needed. This is where volume claim templates come into play.

At a high-level, a `volumeClaimTemplate` dynamically creates a PVC each time a new Pod replica is dynamically created. It also contains the intelligence to name the PVC so it can be correctly attached to Pods. This way, the StatefulSet manifest contains a Pod template section for stamping out Pod replicas, and a volume claim template section for stamping out PVCs.

```yml
volumeClaimTemplates:
- metadata:
	name: webroot
  spec:
  	accessModes: ["ReadWriteOnce"]
	storageClassName: "flash"
	resources:
		requests:
			storage: 10Gi
```

### Testing peer discovery
It's worth to understand how DNS hostnames and DNS subdomains work with StatefulSets

By default, Kubernetes places all objects within the `cluster.local` DNS subdomain. You can choose something different, but most lab environments use this domain, so we'll assume it in this example. Within that domain, Kubernetes constructs DNS subdomains as follows:

```text
<object-name>.<service-name>.<namespace>.svc.cluster.local
```

`svc` indicates the subdomain for objects behind a Service.

To test your fully qualified DNS names, deploy a jump-pod that has the DNS `dig` utility pre-installed.
```bash
$ dig SRV dullahan.default.svc.cluster.local
```

The query returns the fully qualified DNS names of each Pod, as well as its IP. Other applications, including the app itself, can use this method to discover the full list of Pods in the StatefulSet.

For this method of discovery to be useful, applications obviously need to know how to use it. They must also know the name of the StatefulSet's governing Service, and StatefulSet Pods must match the governing Service's label selector.

### Scaling StatefulSets
Each time a Statefulset is scaled up, a Pod and a PVC is created. However, when scaling a StatefulSet down, the Pod is terminated but the PVC is not. This means future scale-up operations only need to create a new Pod, which is then connected to the surviving PVC. The StatefuleSet controller includes all of the intelligence to track and manage these mappings between Pods and PVCs.

It's worth noting that scale down operations will be put on hold if any of the Pods are in a failed state. This is to protect the resiliency of the app and integrity of it's data.

Finally, it's possible to tweak the controlled an ordered starting and stopping of Pods via the StatefulSet's `spec.podManagementPolicy` property.

The default setting is `OrderedReady` and implements the strict methodical ordering previously explained. Setting the value to `Parallel` will cause the StatefulSet to act more like a Deployment where Pods are created and deleted in parallel. For example, scaling from 2>5 Pods will create three new Pods instantaneously, and scaling down from 5 > 2 will delete three Pods in parallel. StatefulSet naming rules are still implemented, and the setting only applies to scaling operations and does not impact rolling updates.