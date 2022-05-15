# rabbitMQ_KEDA
Azure Functions is a service on Azure that consists of a runtime part that executes functions and a part that controls scaling, of which the latter scaling control can be replaced with Kubernetes and KEDA. <br>
Azure Functions can run on Kubernetes with KEDA, so you can use Azure Functions outside of your Azure platform, such as your on-premises environment.

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
# helm repo add bitnami https://charts.bitnami.com/bitnami
# helm install rabbitmq --set auth.username=user --set auth.password=PASSWORD --set persistence.storageClass=nfs-vm-csi --set service.type=LoadBalancer bitnami/rabbitmq

# kubectl get pvc
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-rabbitmq-0      Bound    pvc-463ddffe-30d7-47d9-a864-0a01dccdef90   8Gi        RWO            nfs-vm-csi     61s

# kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
rabbitmq-0   1/1     Running   0          34s
```

# 2. Deploy a RabbitMQ consumer
See also https://github.com/kedacore/sample-go-rabbitmq.
```
# git clone https://github.com/kedacore/sample-go-rabbitmq

# cd sample-go-rabbitmq

# kubectl apply -f deploy/deploy-consumer.yaml 
secret/rabbitmq-consumer-secret unchanged
deployment.apps/rabbitmq-consumer created
scaledobject.keda.sh/rabbitmq-consumer created
triggerauthentication.keda.sh/rabbitmq-consumer-trigger unchanged
```

You should see rabbitmq-consumer deployment with 0 pods as there currently aren't any queue messages.
```
# kubectl get deploy
NAME                READY   UP-TO-DATE   AVAILABLE   AGE
rabbitmq-consumer   0/0     0            0           9s

# kubectl exec -it rabbitmq-0 -- rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello	0
```

# 3. Publish messages to the queue 
You can find 300 messages in the queue named hello.
```
# kubectl apply -f deploy/deploy-publisher-job.yaml
job.batch/rabbitmq-publish created

# kubectl exec -it rabbitmq-0 -- rabbitmqctl list_queues
Timeout: 60.0 seconds ...
Listing queues for vhost / ...
name	messages
hello	300
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

# kubectl get pods -o wide 
NAME                                 READY   STATUS      RESTARTS   AGE   IP              NODE      NOMINATED NODE   READINESS GATES
rabbitmq-0                           1/1     Running     0          36m   10.10.168.196   worker7   <none>           <none>
rabbitmq-consumer-8688c887df-25hr5   1/1     Running     0          9s    10.10.189.86    worker2   <none>           <none>
rabbitmq-consumer-8688c887df-4l7g4   1/1     Running     0          9s    10.10.168.213   worker7   <none>           <none>
rabbitmq-consumer-8688c887df-58thf   1/1     Running     0          9s    10.10.42.80     worker5   <none>           <none>
rabbitmq-consumer-8688c887df-5njgm   1/1     Running     0          24s   10.10.45.204    worker8   <none>           <none>
rabbitmq-consumer-8688c887df-5vpkg   1/1     Running     0          9s    10.10.35.19     worker6   <none>           <none>
rabbitmq-consumer-8688c887df-8rx7p   1/1     Running     0          39s   10.10.45.203    worker8   <none>           <none>
rabbitmq-consumer-8688c887df-bvhqt   1/1     Running     0          9s    10.10.189.85    worker2   <none>           <none>
rabbitmq-consumer-8688c887df-ckq8z   1/1     Running     0          54s   10.10.42.78     worker5   <none>           <none>
rabbitmq-consumer-8688c887df-cq9vs   1/1     Running     0          54s   10.10.168.210   worker7   <none>           <none>
rabbitmq-consumer-8688c887df-ctpsz   1/1     Running     0          24s   10.10.189.84    worker2   <none>           <none>
rabbitmq-consumer-8688c887df-f4sm4   1/1     Running     0          24s   10.10.182.14    worker3   <none>           <none>
rabbitmq-consumer-8688c887df-g8z67   1/1     Running     0          24s   10.10.235.149   worker1   <none>           <none>
rabbitmq-consumer-8688c887df-gf7zv   1/1     Running     0          65s   10.10.189.83    worker2   <none>           <none>
rabbitmq-consumer-8688c887df-ht4f2   1/1     Running     0          9s    10.10.168.212   worker7   <none>           <none>
rabbitmq-consumer-8688c887df-hzj6s   1/1     Running     0          24s   10.10.35.18     worker6   <none>           <none>
rabbitmq-consumer-8688c887df-l6789   1/1     Running     0          24s   10.10.199.144   worker4   <none>           <none>
rabbitmq-consumer-8688c887df-lkg5c   1/1     Running     0          9s    10.10.45.205    worker8   <none>           <none>
rabbitmq-consumer-8688c887df-lz84r   1/1     Running     0          9s    10.10.199.146   worker4   <none>           <none>
rabbitmq-consumer-8688c887df-p2z2t   1/1     Running     0          24s   10.10.42.79     worker5   <none>           <none>
rabbitmq-consumer-8688c887df-pqw28   1/1     Running     0          39s   10.10.182.13    worker3   <none>           <none>
rabbitmq-consumer-8688c887df-q4c9v   1/1     Running     0          24s   10.10.168.211   worker7   <none>           <none>
rabbitmq-consumer-8688c887df-q7d59   1/1     Running     0          9s    10.10.199.145   worker4   <none>           <none>
rabbitmq-consumer-8688c887df-q7qcd   1/1     Running     0          9s    10.10.235.151   worker1   <none>           <none>
rabbitmq-consumer-8688c887df-q8x6p   1/1     Running     0          54s   10.10.235.148   worker1   <none>           <none>
rabbitmq-consumer-8688c887df-q9lmn   1/1     Running     0          9s    10.10.235.150   worker1   <none>           <none>
rabbitmq-consumer-8688c887df-qjmmx   1/1     Running     0          9s    10.10.42.81     worker5   <none>           <none>
rabbitmq-consumer-8688c887df-rcgx7   1/1     Running     0          9s    10.10.182.15    worker3   <none>           <none>
rabbitmq-consumer-8688c887df-rfnzz   1/1     Running     0          39s   10.10.199.143   worker4   <none>           <none>
rabbitmq-consumer-8688c887df-sdkzs   1/1     Running     0          9s    10.10.35.20     worker6   <none>           <none>
rabbitmq-consumer-8688c887df-tlg75   1/1     Running     0          39s   10.10.35.17     worker6   <none>           <none>
rabbitmq-publish-cp64x               0/1     Completed   0          73s   10.10.189.82    worker2   <none>           <none>
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
