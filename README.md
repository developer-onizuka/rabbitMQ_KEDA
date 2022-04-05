# rabbitMQ_KEDA

# 0. Install KEDA
https://github.com/developer-onizuka/AzureFunctionsOnKubernetesWithKEDA/blob/main/README.md#2-install-keda-with-helm-in-kubernetes-master-node
```
# helm repo add kedacore https://kedacore.github.io/charts
# helm repo update
# kubectl create namespace keda
# helm install keda kedacore/keda --namespace keda
```

# 1. Install rabbitMQ with helm
Before this step, you must create a storageclass such as nfs etc. <br>
See also https://github.com/developer-onizuka/persistentVolume-CSI.

```
# helm install rabbitmq --set auth.username=user --set auth.password=PASSWORD --set persistence.storageClass=nfs-vm-csi bitnami/rabbitmq

# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-rabbitmq-0      Bound    pvc-463ddffe-30d7-47d9-a864-0a01dccdef90   8Gi        RWO            nfs-vm-csi     61s

# kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
rabbitmq-0   1/1     Running   0          34s
```

# 2. Deploy a RabbitMQ consumer

```
# git clone https://github.com/kedacore/sample-go-rabbitmq

# cd sample-go-rabbitmq

# kubectl apply -f deploy/deploy-consumer.yaml 
secret/rabbitmq-consumer-secret unchanged
deployment.apps/rabbitmq-consumer created
scaledobject.keda.sh/rabbitmq-consumer created
triggerauthentication.keda.sh/rabbitmq-consumer-trigger unchanged
```

You should see rabbitmq-consumer deployment with 0 pods as there currently aren't any queue messages. It is scale to zero
```
# kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq-consumer   0/0     0            0           9s
```

# 3. Publish messages to the queue 
```
# kubectl apply -f deploy/deploy-publisher-job.yaml
job.batch/rabbitmq-publish created
```

# 4. Automatic Scale Out of KEDA
RabbitMQ consumer's instances will be created soon. 
```
# kubectl get deploy -w
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq-consumer   0/0     0            0           4m5s
rabbitmq-consumer   0/1     0            0           4m11s
rabbitmq-consumer   0/1     0            0           4m11s
rabbitmq-consumer   0/1     0            0           4m11s
rabbitmq-consumer   0/1     1            0           4m11s
rabbitmq-consumer   1/1     1            1           4m15s
rabbitmq-consumer   1/4     1            1           4m16s
rabbitmq-consumer   1/4     1            1           4m16s
rabbitmq-consumer   1/4     1            1           4m16s
rabbitmq-consumer   1/4     4            1           4m16s
rabbitmq-consumer   2/4     4            2           4m20s
rabbitmq-consumer   3/4     4            3           4m20s
rabbitmq-consumer   4/4     4            4           4m20s
rabbitmq-consumer   4/8     4            4           4m31s
rabbitmq-consumer   4/8     4            4           4m31s
rabbitmq-consumer   4/8     4            4           4m31s
rabbitmq-consumer   4/8     8            4           4m31s
rabbitmq-consumer   5/8     8            5           4m35s
rabbitmq-consumer   6/8     8            6           4m35s
rabbitmq-consumer   7/8     8            7           4m35s
rabbitmq-consumer   8/8     8            8           4m36s
rabbitmq-consumer   8/16    8            8           4m46s
rabbitmq-consumer   8/16    8            8           4m46s
rabbitmq-consumer   8/16    8            8           4m46s
rabbitmq-consumer   8/16    16           8           4m46s
rabbitmq-consumer   9/16    16           9           4m50s
rabbitmq-consumer   10/16   16           10          4m50s
rabbitmq-consumer   11/16   16           11          4m50s
rabbitmq-consumer   12/16   16           12          4m50s
rabbitmq-consumer   13/16   16           13          4m50s
rabbitmq-consumer   14/16   16           14          4m51s
rabbitmq-consumer   15/16   16           15          4m51s
rabbitmq-consumer   16/16   16           16          4m51s
rabbitmq-consumer   16/30   16           16          5m1s
rabbitmq-consumer   16/30   16           16          5m1s
rabbitmq-consumer   16/30   16           16          5m1s
rabbitmq-consumer   16/30   30           16          5m1s
rabbitmq-consumer   17/30   30           17          5m5s
rabbitmq-consumer   18/30   30           18          5m5s
rabbitmq-consumer   19/30   30           19          5m6s
rabbitmq-consumer   20/30   30           20          5m6s
rabbitmq-consumer   21/30   30           21          5m6s
rabbitmq-consumer   22/30   30           22          5m6s
rabbitmq-consumer   23/30   30           23          5m6s
rabbitmq-consumer   24/30   30           24          5m6s
rabbitmq-consumer   25/30   30           25          5m8s
rabbitmq-consumer   26/30   30           26          5m8s
rabbitmq-consumer   27/30   30           27          5m8s
rabbitmq-consumer   28/30   30           28          5m8s
rabbitmq-consumer   29/30   30           29          5m9s
rabbitmq-consumer   30/30   30           30          5m9s
rabbitmq-consumer   30/0    30           30          5m26s
rabbitmq-consumer   30/0    30           30          5m26s
rabbitmq-consumer   18/0    18           18          5m26s
rabbitmq-consumer   1/0     1            1           5m26s
rabbitmq-consumer   0/0     0            0           5m26s
```

```
# kubectl get hpa
NAME                         REFERENCE                      TARGETS             MINPODS   MAXPODS   REPLICAS   AGE
keda-hpa-rabbitmq-consumer   Deployment/rabbitmq-consumer   <unknown>/5 (avg)   1         30        0          6m2s
```

# X. Clean Up
```
kubectl delete job rabbitmq-publish
kubectl delete ScaledObject rabbitmq-consumer
kubectl delete deploy rabbitmq-consumer
helm delete rabbitmq
```
