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

## Delete service, pods, deployments
```bash
kubectl delete -f pod.yml
```