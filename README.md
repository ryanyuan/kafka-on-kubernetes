# Apache Kafka on Kubernetes

This project enables deployment of a [Kafka](http://kafka.apache.org/) cluster using
[Kubernetes](http://kubernetes.io/) with [Zookeeper](https://zookeeper.apache.org/).

# Prerequisite
You need to have a Kubernetes cluster running and the `kubectl` CLI tool in your path.

Given that Kafka depends on [Zookeeper](https://zookeeper.apache.org/) for
[reasons](https://www.quora.com/What-is-the-actual-role-of-ZooKeeper-in-Kafka),
we need to deploy Zookeeper before we create the Kafka cluster. 
We launch three distinct deployments to specify the Zookeeper ID of each
instance and to list all brokers in the pool. As of now, we are unable to use a
single deployment with three Zookeeper instances. Even worse, we need to use
three distinct services as well.

```
$ kubectl create -f zookeeper-services.yaml
$ kubectl create -f zookeeper-cluster.yaml
```

After the Zookeeper cluster is launched, check that all three deployments are
`Running`.

```
$ kubectl get pods
NAME                           READY     STATUS    RESTARTS   AGE
kafka-controller-gwjtt         1/1       Running   0          5m
zookeeper-controller-1-qmpbx   1/1       Running   0          6m
zookeeper-controller-2-9c8m6   1/1       Running   0          6m
zookeeper-controller-3-8n6l6   1/1       Running   0          6m
```

One of the zookeeper deployments should be `LEADING`, while the other two zookeeper controllers should be
`FOLLOWERS`. To check that, look at the pod logs.

```
$ kubectl logs zookeeper-controller-3-8n6l6
...
2018-07-20 07:11:56,389 [myid:3] - INFO  [QuorumPeer[myid=3]/0:0:0:0:0:0:0:0:2181:Leader@371] - LEADING - LEADER ELECTION TOOK - 10863
```

Next, let's create the Kafka service behind a load balancer.

```
$ kubectl create -f kafka-service.yaml
```

At this point, we need to set the `KAFKA_ADVERTISED_HOST_NAME` in
`kafka-cluster.yaml` before deploy the Kafka brokers. You can use either a
custom DNS name or one generated by your cloud provider. To see the DNS name, type:

```
$ kubectl describe service kafka-service
Name:              kafka-service
...
LoadBalancer Ingress:     123.123.123.123
...
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  EnsuringLoadBalancer  57s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   4s    service-controller  Ensured load balancer
```

If you can't see the key/value pair of LoadBalancer Ingress, you may need to wait a couple of minutes because the load balancer is being created.

After you copy/pasta the entry next to `LoadBalancer Ingress` to
`KAFKA_ADVERTISED_HOST_NAME` in `kafka-cluster.yaml`, we are finally ready to
create the Kafka cluster:

```
$ kubectl create -f kafka-cluster.yaml
```

If you wish to create a topic at deployment time, add a key
`KAFKA_CREATE_TOPICS` to the environment variables with the topic name. For
instance, the following environment variable creates the topic `mytopic` with
2 partitions and 1 replica:

```
env:
- name: KAFKA_CREATE_TOPICS
value: mytopic:2:1
```

Congrats! You have a working Kafka cluster running on Kubernetes. Next, a useful way to test your setup is via
[`kafkacat`](https://github.com/edenhill/kafkacat). Once installed, you can
publish to Kafka using the Kafka hostname from above:

```
kafkacat -P -b 123.123.123.123:9092 -t mytopic
```

And then to listen to it, type:

```
kafkacat -b 123.123.123.123:9092 -t mytopic
```