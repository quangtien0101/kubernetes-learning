## START MINIKUBE
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

## Deploying service, pods, deployments
```bash
kubectl apply -f pod.yml
```

```bash
kubectl get pods
```

```bash
kubectl exec -it hello-pod -- sh  
```

```bash
kubectl apply -f deploy.yml
```

```bash
kubectl apply -f service.yml 
```

## Tunneling traffic
Allow minikube to route the traffic from the host machine to all the pods
```bash
minikube tunnel
```

## Rolling update
```bash
kubectl apply -f deploy.yml --record
```

Monitor the progress of update
```bash
kubectl rollout status deployment hello-deploy
```

Once the update is complete, we can verify with 
```bash
kubectl get deploy hello-deploy
```

## Rollback
```bash
$ kubectl rollout history deployment hello-deploy
```

View the old configuration of a replicasets
```bash
kubectl get rs
kubectl describe rs <replicasets ID>
```

```bash
kubectl rollout undo deployment hello-deploy --to-revision=1
```
## Delete service, pods, deployments
```bash
kubectl delete -f pod.yml
```

```bash
kubectl delete -f deploy.yml
```

```bash
kubectl delete -f service.yml
```

## Trouble shoot services
Make sure that the `coredns` Deployment and its managed Pods are up and running

```bash
kubectl get deploy -n kube-system -l k8s-app=kube-dns
```

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns  
```

Check the logs of a pod
```bash
kubectl logs <pod-name> -n <namespace>
```