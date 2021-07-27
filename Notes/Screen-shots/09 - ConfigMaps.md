As your dev, test, and prod environments have different characteristics, each environment needs its own image. A prod image will not work in the dev or test environments because of the differences. This requires extra work to create and maintain 3x copies each application. This can complicated matters and increase the chances of misconfiguration.

In a de-coupled world:
- You build a single image that is shared across all three environments
- you store a single image in a single repository
- you run a single version of each image in all environments

To make this work, you build your application images as generically as possible with no embedded configuration. You then create and store configurations in separate objects and apply a configuration to the application at when you run it.

## ConfigMap theory
Kubernetes provide an object called a ConfigMap (CM) that lets you store configuration data outside of a Pod. It also lets you dynamically inject the data into a Pod at run-time.

ConfigMaps are firs-class objects in the Kubernetes API under group, and they're `v1`. This tells us a lot of things:
- They are stable
- They've been around for a while 
- You can operate on them with the usual `kubectl` commands
- THey can be defined and deployed via the usual YAML manifest

 ConfigMaps are typically used to store non-sensitive configuration data such as:
 -  Environment variable values
 -  Entire configuration files (things like web server configs and database configs)
 -  Hostnames
 -  Service ports
 -  Accounts names
 
 
 You should not use ConfigMaps to store sensitive data such as certificates and passwords. Kubernetes provides a different object, called `Secret`, for storing sensitive data. Secrets and ConfigMaps are very similar in design and implementation, the major difference is that Kubernetes takes steps obscure the values stored in Secrets. It makes no such efforts to obscure data stored in ConfigMaps.
 
 ## How do ConfigMaps work
 At a high-level, a ConfigMap is a place to store configuration data that can be seamlessly injected into containers at runtime, then leveraged in ways that are invisible to applications.
 
 Behind the scenes, ConfigMaps are a map of key/value pairs and we call each key/value pair an entry.
 - Keys are an arbitrary name that can be created from alphanumerics, dashes, dots, and underscores
 - Values can contain anything, including carriage returns
 - We separate keys and values with a colon - key:value

Example:
- db-port:13306
- hostname:msb-prd-db1

Once data is stored in a ConfigMap, it can be injected into containers at run-time via any of the following methods:
- Environment variables
- arguments to the container's startup command
- files in a volume

The most flexible of the 3 methods is the volume option, and the most limited is the startup command. We'll look at each in turn, but before we do that we'll quickly consider a Kubernetes-native application.

![](Screen-shots/VOlume%20options%20in%20configmaps.png)

## ConfigMaps and Kubernetes-native apps

A Kubernetes-native application is an application that knows it's running on Kubernetes and has the intelligence to query the Kubernetes API. As a result, a Kubernetes-native application can access ConfigMap data directly via the API without needing things like environment variables and volumes. This can simplify application configuration, but the application will only run on Kubernetes. 

## Hands-on with ConfigMaps

### Creating ConfigMaps imperatively
The command to imperatively create a ConfigMap is `kubectl create configmap`

The command accepts 2 sources of data:
- literal values on the command line (`--from-literal`)
- files referenced on the command line (`--from-file`)

```bash
$ kubectl create configmap testmap1 \
-- from-literal shortname=msb.com \
-- from-literal longname=magicsandbox.com

configmap/testmap1 created
```

```bash
$ kubectl describe cm testmap1
Name:         testmap1
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
Events:  <none>
```


```bash
$ cat cmfile.txt
Magic Sandbox, hands-on learning that blurs the lines between training and the real world.

$ kubectl create configmap  testmap2 --from-file cmfile.txt
configmap/testmap2 created
```

```bash
$ kubectl describe configmap testmap2
Name:         testmap2
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
cmfile.txt:
----
Magic Sandbox, hands-on learning that blurs the lines between training and the real world.

Events:  <none>
```

### Inspecting ConfigMaps

```bash
$ kubectl get configmap
NAME               DATA   AGE
kube-root-ca.crt   1      24d
testmap1           0      9m1s
testmap2           1      3m40s
```

```bash
$ kubectl get configmap testmap1 -o yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: "2021-07-27T02:55:11Z"
  name: testmap1
  namespace: default
  resourceVersion: "21968"
  uid: be1c2ed5-098e-427b-8ea4-1428ad051b6e
```

The interesting thing to note that ConfigMap objects don't have the concept of state (desired state and actual state). This is why they have a `data` block instead of `spec` and `status` blocks.

### Creating ConfigMaps declaratively

`multimap.yml` file:
```yml
kind: ConfigMap
apiVersion: v1
metadata:
	name: multimap
data:
	given: Nigel
	family: Poulton
```

You can see that a ConfigMap manifest has the normal `kind` and `apiVersion` fields, as well as the usual `metadata` section. However, as previously mentioned, they do not have a `spec` section. Instead, they have a `data` section that defines the map of key/values.

```bash
$ kubectl apply -f multimap.yml 
configmap/multimap created
```

`singlemap.yml` file:
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-conf
data:
  test.conf: |
    env = plex-test
    endpoint = 0.0.0.0:31001
    char = utf8
    vault = PLEX/test
    log-size = 512M
```

The previous YAML file inserts a pipe character (`|`) after the name of the entry's key property. This tells Kubernetes that everything following the pipe is to be treated as a single literal value. Therefore, the ConfigMap objects is called `test-config` and it contains a single map entry as follows:

- Key: `test.conf`
- Value: `env = plex-test endpoint = 0.0.0.0:31001 char = utf8 vault = PLEX/test log-size = 512M`

```bash
$ kubectl apply -f singlemap.yml
configmap/test-conf created
```

```bash
$ kubectl describe configmap test-conf
Name:         test-conf
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
test.conf:
----
env = plex-test
endpoint = 0.0.0.0:31001
char = utf8
vault = PLEX/test
log-size = 512M

Events:  <none>
```

ConfigMaps are extremely flexible and can be used to insert complex configuration files such as JSON files and even scripts into containers at runtime.

## Injecting ConfigMap data into Pods and containers
You've seen how to imperatively and declaratively create ConfigMap objects and populated them with data, Now let's see how to get that data into applications running in containers.

There are 3 main ways to inject ConfigMap data into a container:
- As environment variables
- As arguments to container startup commands
- As files in a volume

### ConfigMaps and environment variables

A common way to get ConfigMap data into a container is via environment variables. You create the ConfigMap, then you map its entries into environment variables in the container section of a Pod template. When the container is started, the environment variables appear in the container as standard Linux or Windows environment variables.

![](Screen-shots/env%20variable%20in%20configmap.png)

The following Pod manifest deploys a single container that creates 2 environment variables in the container
- FIRSTNAME: Maps to the `given` entry in the `multimap` ConfigMap
- LASTNAME: Maps to the `family` entry in the `multimap` ConfigMap


`envpod.yml` file:
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    chapter: configmaps
  name: envpod
spec:
  containers:
    - name: ctr1
      image: busybox
      command: ["sleep"]
      args: ["infinity"]
      env:
        - name: FIRSTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: given
        - name: LASTNAME
          valueFrom:
            configMapKeyRef:
              name: multimap
              key: family
```

```bash
$ kubectl apply -f envpod.yml
pod/envpod created
```

```bash
$ kubectl exec envpod -- env | grep NAME
HOSTNAME=envpod
FIRSTNAME=Nigel
LASTNAME=Poulton
```

A drawback to using ConfigMaps with environment variables is that environment variables are static. This means that any updates you make to the values in the ConfigMap will not be reflected in running containers. For example, if you update the `given` and `family` values in the ConfigMap, environment variables in existing containers will not get the updates.


### ConfigMaps and container startup commands
The concept of using ConfigMaps with container startup commands is simple. The high-level looks like this. It's possible to specify a startup command for a container, and you can customize that startup command with variables. Let's look at a simple example ...

```yml
spec:
	containers:
		- name: args1
		image: busybox
		command: [ "/bin/sh", "-c", "echo First name $(FIRSTNAME) last name $(LASTNAME)" ]
		env:
			- name: FIRSTNAME
			  valueFrom:
				configMapKeyRef:
					name: multimap
					key: given
			- name: LASTNAME
			  valueFrom:
				ConfigMapKeyRef:
					name: multimap
					key: family
```

The startup command it references 2 variables; `FIRSTNAME` and `LASTNAME`. 

![](Screen-shots/relationship%20startup%20command%20configmap.png)

Running a Pod based on the previous `YAML` will print "First name Nigel last name Poulton" to the container's log file.

You can see the logs of the container with command `$kubectl logs <pod-name> -c args1`

```bash
$ kubectl logs start-up -c args1
First name Nigel last name Poulton
```

```bash
$ kubectl describe pod start-up                                                                                                                                                                                 
Name:         start-up                                                                                                                                                                                             
Namespace:    default                                                                                                                                                                                        
<snip>
    Command:
      /bin/sh
      -c
      echo First name $(FIRSTNAME) last name $(LASTNAME)
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Tue, 27 Jul 2021 15:31:55 +0700
      Finished:     Tue, 27 Jul 2021 15:31:55 +0700
    Ready:          False
    Restart Count:  3
    Environment:
      FIRSTNAME:  <set to the key 'given' of config map 'multimap'>   Optional: false
      LASTNAME:   <set to the key 'family' of config map 'multimap'>  Optional: false
<snip>
```

Use the ConfigMaps with container startup commands suffers from the same limitations as using them with environment variables - updates to entries in the map will not be reflected in running containers.

### ConfigMaps and volumes
Using ConfigMaps with volumes is the most flexible option. You can reference entire configuration files as well as make updates to the ConfigMap and have them reflected in running containers. This means you can make changes to entries in a ConfigMap after you've deployed a container, and those changes be seen in the container and available for running applications.

The high-level process for exposing ConfigMap data via a volume looks like this
1. Create the ConfigMap
2. Create a ConfigMap volume in the Pod template
3. Mount the ConfigMap volume into the container
4. Entries in the ConfigMap will appear in the container as individual files.

![](Screen-shots/configmap%20with%20volume.png)

The following YAML creates a Pod called `cmvol` with the following configuration.
- `spec.volumes` creates a volume called **volmap** based on the **multimap** ConfigMap
- `spec.containers.volumeMounts` mounts the **volmap** volume to `/etc/name`

`cmpod.yml` file
```yml
apiVersion: v1
kind: Pod 
metadata:
  name: cmvol
spec:
  volumes:
    - name: volmap
      configMap:
        name: multimap

  containers:
    - name: container-ctr
      image: nginx
      volumeMounts:
        - mountPath: /etc/name
          name: volmap 
```

The `spec.volumes` block creates a special type of volume called a ConfigMap volume. The volume is called volmap and based on the `multimap` ConfigMap. This means that the volume will be populated with the entries stored in the data block of the ConfigMap.

In this example, the volume will have two files; `given` and `family`. The given file will have the contents Nigel, and the family file will have the contents Poulton.

The `spec.containers` block mounts the `volmap` volume into the container at `/etc/name`. This means that 2 files will appear in the container as:
- `/etc/name/given`
- `/etc/name/family`


```bash
$ kubectl apply -f cmpod.yml                   
pod/cmvol created

$ kubectl exec cmvol -- ls /etc/name
family
given
```

## Chapter Summary
ConfigMaps are the mechanism that Kubernetes provides for decoupling applications and their configuration.

ConfigMaps are first-class object in the Kubernetes API and can be created and manipulated with the usual `kubectl create, kubectl get` and `kubectl describe` commands. They're ideal for storing application configuration parameters as well as entire configuration files, but they shouldn't be used to store sensitive data.

ConfigMap data gets injectd into containers at run-time, and you can inject data via environment variables, container startup commands, and volumes. The volumes method is the most flexible as it allows you work with entire configuration files. It also updates to eventually be reflected already-running containers.