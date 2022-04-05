# rabbitMQ_KEDA

```
# helm install rabbitmq --set auth.username=user --set auth.password=PASSWORD --set persistence.storageClass=nfs-vm-csi bitnami/rabbitmq

# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-rabbitmq-0      Bound    pvc-32369814-ad5d-445a-839f-91ede042bf8f   8Gi        RWO            nfs-vm-csi     67s

# kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
rabbitmq-0   2/2     Running   0          63s
```
