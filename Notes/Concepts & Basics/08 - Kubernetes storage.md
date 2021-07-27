## The big picture
First things first, Kubenernetes supports lots of types of storage from lots of different places. For example, iSCSI, SMB, NFS, and object storage blobs, all from a variety of external storage systems that can be in the cloud or in your on-premises data center. However, no matter what type for storage you have, or where it comes from, when it's exposed on your Kubernetes cluster it's called a volume.

All storage on a Kubernetes cluster is called a volume

![](Screen-shots/Kubernetes%20volume.png)

The plugin layer is the glue that connects external storage with Kubernetes. Going forward, plugins will be based on the Container Storage Interface (CSI) which is an open standard aimed at providing a clean interface for plugins. If you're a developer writing storage plugins, the CSI abstracts the internal Kubernetes storage detail and lets you develop *out-of-tree*.

Note: Prior to the CSI, all storage plugins were implemented as part of the main Kubernetes code tree (in-tree). This meant they all had to be open-source, and all updates and bug-fixes were tied to the main Kubernetes release-cycle. This was a nightmare for plugin developers as well as the Kubernetes maintainers. However, now that we have the CSI, storage vendors no longer need to open-source their code, and they can release updates and bug-fixes against their own timeframes.

The Kubernetes persistent volume subsystem. This is a set of API objects that allow applications to consume storage. At a high-level, Persistent Volumes (PV) are how you map external storage onto the cluster, and Persistent Volume Claims (PVC)

![](Screen-shots/persistent%20volume.png)

A couple of points worth noting
1. There are rules safeguarding access to a single volume from multiple Pods 
2. A single external storage volume can only be used by a single PV. For example, you cannot have a 50GB external volume that has two 25GB Kubernetes PVs each using half of it

## Storage providers
Kubernetes can use storage from a wide range of external systems. These will often be native cloud services such as `AWSElasticBlockStore` or `AzureDisk`, but the take-home point is that Kubernetes gets its storage from a wide range of external systems.

Some obvious restriction apply. For example, you cannot use the `AWSElasticBlockStore` provisioner if your Kubernetes cluster is running in Microsoft Azure.


## The container storage interface (CSI)

The CSI is an important piece of the Kubernetes storage jigsaw. However, unless you're a developer writing storage plugins, you're unlikely to interact with it very often.

In the Kubernetes world, the CSI is the preferred way to write drivers (plugins) and means that plugin code no longer needs to exist in the main Kubernetes code tree. It also provides a clean and simple interface that abstracts all the complex internal Kubernetes storage machinery. Basically, the CSI exposes a clean interface and hides all the ugly volume machinery inside of the Kubernetes code. 

From a day-to-day management perspective, your only real interaction with the CSI will be referencing the appropriate plugin in you YAML manifest files. Also, it may take a while for existing in-tree plugins to be replaced by CSI plugins.

Sometimes we call plugins "provisioners", especially when we talk about Storage Classes later.

## The Kubernetes persistent volume subsystem

You start out with raw storage. This plugs in to Kubernetes via a CSI plugin. You then use the resources provided by the persistent volume subsystem to leverage and use the storage in your apps.

The 3 main resources in the persistent volume subsystem are:
- Persistent Volumes (PV)
- Persistent Volume Claims (PVC)
- Storage Classes (SC)

At a high level, PVs are how you represent storage in Kubernetes. PVCs are like tickets that grant a Pod access to a PV. SCs make it all dynamic.

The Kubernetes steps will be:
1. Create the PV
2. Create the PVC
3. Define the volume into a PodSpec
4. Mount it into a container


`gke-pv.yml`
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
	name: pv1
spec:
	accessModes:
	- ReadWriteOnce
	storageClassName: test
	capacity:
		storage:  10Gi
	persistentVolumePolicy: Retain
	gcePersistentDisk:
		pdName: uber-disk
```

```bash
$ kubectl apply -f gke-pv.yml
persistentvolume/pv1 created
```

Check the PV exists
```bash
$ kubectl get pv pv1
```

Show more details
```bash
$ kubectl describe pv pv1
```

![](Screen-shots/Google%20cloud%20persistent%20disk.png)

PV properties set out in the YAML file
`.spec.accessModes` defines how the PV can be mounted. Three options exist:
- `ReadWriteOnce(RWO)`
- `ReadWriteMany(RWM)`
- `ReadOnlyMany(ROM)`

`ReadWriteOnce` defines a PV that can only be mounted/bound as R/W by a single PVC. Attempts from multiple PVCs to bind (claim) it will fail.

`ReadWriteMany` defines a PV that can be bound as R/W by multiple PVCs. This mode is usually only support by file and object storage such as NFS. Block storage usually only support `RWO`.

`ReadOnlyMany` defines a PV that can be bound by multiple PVCs as R/O.

A couple of things are worth noting. First up, a PV can only be opened in one mode - it is not possible for a single PV to have a PVC bound to it in ROM mode and another PVC bound to it in RWM mode. Second up, Pods do not act directly on PVs, they always act on the PVC object that is bound to the PV.

`spec.storageClassName` tells Kubernetes to group this PV in a storage class called "test". You'll learn more about storage classes later in the chapter, but you need this here to make sure the PV will correctly bind with a PVC in a later step.

Another property is `spec.persistentVolumeReclaimPolicy`. This tells Kubernetes what to do with a PV when its PVC has been released. Two policies currently exist:
- `Delete`
- `Retain`

`Delete` is the most dangerous, and is the default for PVs that are created dynamically via storage classes. This policy deletes the PV and associated storage resource on the external storage system, so will result in data loss! You should obviously use this policy with caution.

`Retain` will keep the associated PV object on the cluster as well as any data stored on the associated external asset. However, it will prevent another PVC from using the PV in future.

If you want to re-use a retained PV, you need to perform the following three steps:
1. Manually delete the PV on Kubernetes
2. Re-format the associated storage asset on the external storage system to wipe any data
3. Recreate the PV.

`spec.capacity` tells Kubernetes how big the PV should be. THis value can be less than the actual physical storage asset but cannot be more. For example, you cannot create a 100GB PV that maps back to a 50GB device on the external storage system. But you can create a 50GB PV that maps back to a 100GB external volume (but that would be wasteful).

Finally, the last line of the YAML file links the PV to the name of the pre-created device on the back-end.

Now we create a PVC so that Pod can claim access to the storage
`gke-pvc.yml`
```yml
apiVersion: v1
kind: PersistsentVolumeClaim
metadata:
	name: pvc1
spec:
	accessModes:
	- ReadWriteOnce
	storageClassName: test
	resources:
		requests:
			storage: 10Gi
```

The most important thing to not about a PVC object is that the values in the `spec` section must match with the PV you are binding it with. In this example, access modes, storage class, and capacity must match with the PV.

Note: It’s possible for a PV to have more capacity than a PVC. For example, a 10GB PVC can be bound to a 15GB PV (obviously this will waste 5GB of the PV). However, a 15GB PVC cannot be bound to a 10GB PV

![](Screen-shots/matching%20spec%20access%20modes.png)

Deploy the PVC with the following command
```bash
$ kubectl apply -f gke-pvc.yml
persistentvolumeclaim/pvc1 created
```

Check that the PVC is created and bound to the PV
```bash
$ kubectl get pvc pvc1
NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS
pvc1 Bound pv1 10Gi RWO test
```

Now we create a pod that can utilize the PVC (this is just for the sake of example, in production you should use Deployments to deploy pods instead)

`volpod.yml`
```yml
apiVersion: v1
kind: Pod
metadata:
	name: volpod
spec:
	volumes:
	- name: data
	persistentVolumeClaim:
		claimName: pvc1
	containers:
	- name: ubuntu-ctr
	image: ubuntu:latest
	command:
	- /bin/bash
	- "-c"
	- "sleep 60m"
	volumeMounts:
	- mountPath: /data
	name: data
```

Deploy the Pod
```bash
kubectl apply -f volpod.yml
pod/volpod created
```

```bash
kubectl describe pod volpod
```

### Quick summary
You start out with storage assets on an external storage system. You use a CSI plugin to integrate the external storage system with Kubernetes, and you use Persistent Volume (PV) objects to make the external systems assets accessible and usable. Each PV is an object on the Kubernetes cluster that maps back to a specific storage asset (LUN, share, blob…) on the external storage system. Finally, for a Pod to use a PV, it needs a Persistent Volume Claim (PVC). This is like a ticket that grants the Pod to the PV. Once the PV and PVC objects are created and bound, the PVC can be referenced in a PodSpec and the associated PV mounted as a volume in a container

## Storage Classes and Dynamic Provisioning
As the name suggests, storage classes allow you to define different classes, or tiers of storage. How you define your classes is up to you, but will depend on the types of storage you have access to. For example, you might define a `fast` class, a `slow` class, and an `encrypted` class. (Your external storage system would need to support different speeds of storage and support encrypting volumes as Kubernetes does none of this)

As far as Kubernetes goes, storage classes are defined as resources in the `storage.k8s.io/v1` API group. The resource type is `StorageClass`, and you define them in regular YAML files that you POST to the API server for deployment. You can use the `sc` shortname to refer to StorageClass objects when using kubectl.

Note: you can see a full list of API resources, and their shortnames, suing the `kubectl api-resources` command.
```bash
$ kubectl api-resources                                                                                                                                                                                         
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND                                                                                                            
bindings                                       v1                                     true         Binding                                                                                                         
componentstatuses                 cs           v1                                     false        ComponentStatus                                                                                                 
configmaps                        cm           v1                                     true         ConfigMap                                                                                                       
endpoints                         ep           v1                                     true         Endpoints                                                                                                       
events                            ev           v1                                     true         Event                                                                                                           
limitranges                       limits       v1                                     true         LimitRange                                                                                                      namespaces                        ns           v1                                     false        Namespace                                                                                                       nodes                             no           v1                                     false        Node                                                                                                            persistentvolumeclaims            pvc          v1                                     true         PersistentVolumeClaim                                                                                           persistentvolumes                 pv           v1                                     false        PersistentVolume                                                                                                pods                              po           v1                                     true         Pod
podtemplates                                   v1                                     true         PodTemplate                                                                                                     replicationcontrollers            rc           v1                                     true         ReplicationController                                                                                           resourcequotas                    quota        v1                                     true         ResourceQuota                                                                                                   secrets                                        v1                                     true         Secret
serviceaccounts                   sa           v1                                     true         ServiceAccount                                                                                                  services                          svc          v1                                     true         Service                                                                                                         mutatingwebhookconfigurations                  admissionregistration.k8s.io/v1        false        MutatingWebhookConfiguration                                                                                    validatingwebhookconfigurations                admissionregistration.k8s.io/v1        false        ValidatingWebhookConfiguration                                                                                  customresourcedefinitions         crd,crds     apiextensions.k8s.io/v1                false        CustomResourceDefinition                                                                                        apiservices                                    apiregistration.k8s.io/v1              false        APIService                                                                                                      controllerrevisions                            apps/v1                                true         ControllerRevision 
```

### A StorageClass yaml

The following is a simple example of a StorageClass YAML file. It defines a class of storage called “fast”, that is based on AWS solid state drives (io1) in the Ireland Region (eu-west-1a). It also requests a performance level of 10 IOPs per gigabyte

```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
	name: fast
provisioner: kubernetes.io/aws-edb
parameters:
	type: io1
	zones: eu-west-1a
	iopsPerGB: "10"
```

As with all Kubernetes YAML, `kind` tells the API server what type of object is being defined, and `apiVersion` tells it which version of the schema to apply to the resource.

`metadata.name` is an arbitrary string value that lets you give the object a friendly name - this example is defining a class called "fast".

`provisioner` tells Kubernetes which plugin to use, and the `parameters` field lets you finely tune the type of storage to leverage from the back-end.

Some worth noting:
1. StorageClass objects are immutable - this means you cannot modify them once deployed
2. `metadata.name` should be meaningful as it's how other objects will refer to the class
3. The terms provisioner and plugin are used interchangeably.
4. The `parameter` section is for plugin-specific values, and each plugin is free to support its own set of values. Configuring this section requires knowledge of the storage plugin and associated storage back-end.


## Multiple StorageClasses
You can configure as many StorageClass objects as you need. However, each one relates to a single provisioner. For example, if you have a Kubernetes cluster with StorageOS and Portworx storage back-ends, you will need at least two StorageClass objects. That said, each back-end can offer multiple classes/tiers of storage, each of which can have its own StorageClass. For example, you could have the following two StorageClass objects for different classes of storage from the same back-end:
1. "Fasst-secure" for high performance encrypted volumes
2. "fast" for high-performance unencrypted volumes

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
	name: portworx-db-secure
provisioner: kubernetes.io/portworx-volume
parameters:
	fs: "xfs"
	block_size: "32"
	repl: "2"
	snap_interval: "30"
	io-priority: "medium"
	secure: "true"
```

### Implementing StorageClasses

The basic workflow for deploying and using a StorageClass on your cluster is as follows:
1. Create your Kubernetes cluster with a storage back-end
2. Ensure the plugin for the storage back-end is available
3. Create a StorageClass object
4. Create a PVC object that references the StorageClass by name
5. Deploy a Pod that uses volume based on the PVC

The following YAML snippet contains the definitions for a StorageClass, a PersistentVolumeClaim, and a Pod. All three objects can be defined in a single YAML file by separating each object with three dashes (---).

```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
	name: fast # Referenced by the PVC
provisioner: kubernetes.io/gce-pd
parameters:
	type: pd-ssd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
	name: mypvc # Referenced by the PodSpec
	namespace: mynamespace
spec:
	accessModes:
	- ReadWriteOnce
	resources:
		requests:
			storage: 50Gi
	storageClassName: fast # Matches name of the SC
---
apiVersion: v1
kind: Pod
metadata:
	name: mypod
spec:
	volumes:
	- name: data
	persistentVolumeClaim:
		claimName: mypvc # Matches PVC name
	containers: ...
<SNIP>
```

So far, you’ve seen a few SC definitions. However, each one has been slightly different as each one has related to a different provisioner (storage plugin/back-end). You will need to refer to the documentation of your storage plugin to know which options your provisioner supports.

>**StorageClasses make it so that you don’t have to create PVs manually. You create the StorageClass object and use a plugin to tie it to a particular type of storage on a particular storage back-end. For example, high-performance AWS SSD storage in the AWS Mumbai Region. The SC needs a name, and is defined in a YAML file that you deploy using kubectl. Once deployed, the StorageClass watches the API server for new PVC objects that reference its name. When matching PVCs appear, the StorageClass dynamically creates the required volume on the back-end storage system as well as the PV on Kubernetes.**

## Demo
Create a storage class file `google-sc.yml`
```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
	name: slow
	annotations:
		storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/gce-pd
parameters:
	type: pd-standard
reclaimPolicy: Retain
```

Deploy the SC with the following command:
```bash
$ kubectl apply -f google-sc.yml
storageclass.storage.k8s.io/slow created
```

Check and inspect with `kubectl get sc slow` and `kubctl describe sc slow`

Create a PVC object that references the `slow` StorageClass created in the previous step.
`google-pvc.yml`
```yml
apiVersion: v1
kind: PersistentVOlumeClaim
metadata:
	name: pv-ticket
spec:
	accessModes:
	- ReadWriteOnce
	storageClassName: slow
	resources:
		requests:
			storage: 25Gi
```

The important things to note are that the PVC is called pv-ticket, it’s linked to the slow class, and it’s for a 25GB volume.

Deploy it
```bash
$ kubectl apply -f google-pvc.yml
```

Verify the operation with `kubectl get pvc pv-ticket`

The mechanics behind the operation are as follows:
1. You created the `slow` StorageClass
2. A loop was created to watch the API Server for new PVCs referencing the `slow` StorageClass
3. You created the `pv-ticket` PVC that requested binding to a 25GB volume from the slow StorageClass
4. The StorageClass loop noticed this PVC and dynamically created the requested PV

Run `kubectl get pv` to verify the presence of the automatically creatd PV on the cluster.

Create the Pod
`google-pod.yml`
```yml
apiVersion: v1
kind: Pod
metadata:
	name: class-pod
spec:
	volumes:
	- name: data
	persistentVolumeClaim:
		claimName: pv-ticket
containers:
	- name: ubuntu-ctr
	image: ubuntu:latest
	command:
	- /bin/bash
	- "-c"
	- "sleep 60m"
	volumeMounts:
	- mountPath: /data
	  name: data
```

Deploy the Pod with `kubectl apply -f google-pod.yml`

### Clean-up
```bash
$ kubectl delete pod class-pod
pod "class-pod" deleted

$ kubectl delete pvc pv-ticket
persistentvolumeclaim "pv-ticket" deleted

$ kubectl delete sc slow
storageclass.storage.k8s.io "slow" deleted
```


If your cluster has a default storage class, you can deploy a Pod using just a PodSpec and a PVC. You do not need to manually create a StorageClass. However, real-world production clusters will usually have multiple StorageClasses, so it’s best practice to create and manage StorageClasses that suit your business and application needs. The default StorageClass is normally only useful in development environments and times when you do not have specific storage requirements.

## Chapter summary
In this chapter, you learned that Kubernetes has a powerful storage subsystem that allows it to leverage storage from a wide variety of external storage back-ends. Each back-end requires a plugin so that its storage assets can be used on the cluster, and the preferred type of plugin is a CSI plugin. Once a plugin is enabled, Persistent Volumes (PV) are used to represent external storage resources on the Kubernetes cluster, and Persistent Volume Claims (PVC) are used to give Pods access to PV storage. Storage Classes take things to the next level by allowing applications to dynamically request storage. You create a Storage Class object that references a class, or tier, of storage from a storage back-end. Once created, the Storage Class watches the API Server for new Persistent Volume Claims that reference the Storage Class. When a matching PVC arrives, the SC dynamically creates the storage and makes it available as a PV that can be mounted as a volume into a Pod (container)