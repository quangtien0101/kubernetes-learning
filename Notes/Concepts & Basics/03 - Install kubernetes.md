To set up a lab, install `minikube` with `kubectl`

## Install minikube
https://minikube.sigs.k8s.io/docs/start/

## Install kubectl
https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

## Start minikube
```bash
minikube start
```

```bash
kubectl version
```

```bash
kubectl get nodes
```

```bash
minikube status
```

## `kubectl`
`kubectl` is the main Kubernetes command-line tools and is what you will use for your day-to-day Kubernetes management activities. It might be useful to think of `kubectl` as SSH for kubernetes. 

At a high level, `kubectl` coverts user-friendly commands into the JSON payload required by the API server. It uses a configuration file to know which cluster and API server endpoint to `POST` commands to

The `kubectl` configuration file is called `config` and lives in `$HOME/.kube` directory. It contains definitions for :
- Clusters
- Users (credentials)
- Contexts


***Clusters is a list of clusters that `kubectl` knows about*** and is ideal if you plan on using a single workstation to manage multiple clusters. Each cluster definition has a name, certificate info, and API server endpoint.

***Users let you define different users might have different levels of permissions on each cluster***. For example, you might have a `dev` user and an `ops` user, each with different permissions. Each user definition has a friendly name, a username, and a set of credentials

***Contexts bring together clusters and users under a friendly name***. For example, you might have a context called deploy-prod that combines the deploy user credentials with the prod cluster definition. If you use `kubectl` with this context you will be POSTing commands to the API server of the prod cluster as the deploy user.

You can view your `kubectl` config using the `kubectl config view` command. Sensitive data will be redacted from the output

You can use `kubectl config current-context` to see your current context.
You can change the current/active context with `kubectl config use-context` 