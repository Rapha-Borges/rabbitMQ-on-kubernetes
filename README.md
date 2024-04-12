# RabbitMQ on Kubernetes

Create a cluster with [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)

```
kind create cluster --name rabbit
```

## Namespace

```
kubectl create ns rabbits
```

## Storage Class

```
kubectl get storageclass
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  84s
```

## Deployment

Get the cookie value and edit the `rabbit-secret.yaml` file. 

```
echo -n "cookie-value" | base64
```
Apply the files:

```
kubectl apply -f .\kubernetes\rabbit-rbac.yaml
kubectl apply -f .\kubernetes\rabbit-configmap.yaml
kubectl apply -f .\kubernetes\rabbit-secret.yaml
kubectl apply -f .\kubernetes\rabbit-statefulset.yaml
```

## Access the UI

```
kubectl -n rabbits port-forward rabbitmq-0 8080:15672
```

Go to htttp://localhost:8080 <br/>
Username: `guest` <br/>
Password: `guest` <br/>

# Message Publisher

```
cd messaging\rabbitmq\applications\publisher
docker build . -t aimvector/rabbitmq-publisher:v1.0.0
kubectl apply -n rabbits -f deployment.yaml
```

```
kubectl port-forward -n rabbits svc/rabbitmq-publisher 8081:80
```

# Populate the Queue

```
curl -X POST -d "mensagem=hello" http://localhost:8081/publish/hello
```

# Automatic Synchronization of Mirrored Queues

https://www.rabbitmq.com/ha.html#unsynchronised-mirrors

```
kubectl -n rabbits exec -it rabbitmq-0 bash
```

```
rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq-0.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-1.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-2.rabbitmq.rabbits.svc.cluster.local"]}' \
    --priority 1 \
    --apply-to queues
```

Now you can kill the rabbitmq-0 pod and the queues will be synchronized automatically.

```
kubectl -n rabbits delete pod rabbitmq-0
```

When the pod is recreated, the queues will be synchronized automatically. You can check the status of the synchronization in the UI.

```
http://localhost:8080/#/queues/%2F/publisher
```

## Credits

[That DevOps Guy YouTube Video](https://www.youtube.com/watch?v=_lpDfMkxccc)

[docker-development-youtube-series GitHub Repository](https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/messaging/rabbitmq)
