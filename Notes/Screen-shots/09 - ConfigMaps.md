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
