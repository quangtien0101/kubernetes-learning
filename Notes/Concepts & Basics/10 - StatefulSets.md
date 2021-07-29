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