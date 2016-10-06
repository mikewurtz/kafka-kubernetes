# Apache Kafka on Kubernetes

This project enables deployment of a [Kafka 0.10.0.1](http://kafka.apache.org/) cluster using
[Kubernetes](http://kubernetes.io/) with [Zookeeper 3.4.6](https://zookeeper.apache.org/). The code
borrows heavily from [Sylvain Hellegouarch's post](http://www.defuze.org/archives/351-running-a-zookeeper-and-kafka-cluster-with-kubernetes-on-aws.html). Sylvain's writeup is quite good. However, there are a couple of hacks listed below in **Known Issues**

To launch the Kafka cluster, let's assume you have a working Kubernetes cluster
and the `kubectl` CLI tool in your path. First, create a `kafka-cluster`
namespace on Kubernetes and set it as the current context.

```
$ kubectl create -f namespace-kafka.yaml
kubectl config set-context kafka --namespace=kafka-cluster --cluster=${CLUSTER_NAME} --user=${USER_NAME}
kubectl config use-context kafka
```

Given that Kafka depends on [Zookeeper](https://zookeeper.apache.org/) for
[reasons](https://www.quora.com/What-is-the-actual-role-of-ZooKeeper-in-Kafka),
we need to deploy Zookeeper before we create the Kafka cluster. Following
[Sylvain's post](http://www.defuze.org/archives/351-running-a-zookeeper-and-kafka-cluster-with-kubernetes-on-aws.html),
we launch three distinct replication controllers to specify the Zookeeper ID of
each instance and to list all brokers in the pool. As of now, we are unable to
use a single replication controller with three Zookeeper instances. Even worse,
we need to use three distinct services as well.

```
$ kubectl create -f zookeeper-services.yaml
$ kubectl create -f zookeeper-cluster.yaml
```

After the Zookeeper cluster is launched, check that all three replication
controllers are `Running`.

```
$ kubectl get pods
zookeeper-controller-1-dbauf   1/1       Running   0          2h
zookeeper-controller-2-mp6nb   1/1       Running   0          2h
zookeeper-controller-3-26ere   1/1       Running   0          2h
```

One of the replication controllers should be `LEADING`, while the other two
should be `FOLLOWERS`. To check that, look at the pod logs.

```
$ kubectl logs zookeeper-controller-1-dbauf
...
2016-10-06 14:04:05,904 [myid:2] - INFO [QuorumPeer[myid=2]/0:0:0:0:0:0:0:0:2181:Leader@358] - LEADING - LEADER ELECTION TOOK - 2613
```

Next, let's create the Kafka service behind a load balancer.

```
$ kubectl create -f kafka-service.yaml
```

At this point, we need to set the `KAFKA_ADVERTISED_HOST_NAME` in
`kafka-cluster.yaml` before deploy the Kafka brokers. You can use either a
custom DNS name or one generated by your cloud provider. To see the DNS name
(on AWS at least), type:

```
$ kubectl describe service kafka-service
Name:              kafka-service
...
LoadBalancer Ingress:      xxxxxx.us-east-1.elb.amazonaws.com
```

After you copy/pasta the entry next to `LoadBalancer Ingress` to
`KAFKA_ADVERTISED_HOST_NAME` in `kafka-cluster.yaml`, we are finally ready to
create the Kafka cluster:

```
$ kubectl create -f kafka-cluster.yaml
```

If you wish to create a topic at deployment time, add a key
`KAFKA_CREATE_TOPICS` to the environment variables with the topic name. For
instance, the following environment variable creates the topic `ramhiser` with
2 partitions and 1 replica:

```
env:
- name: KAFKA_CREATE_TOPICS
value: ramhiser:2:1
```

Congrats! You have a working Kafka cluster running on Kubernetes. Next, a useful way to test your setup is via
[`kafkacat`](https://github.com/edenhill/kafkacat). Once installed, you can
pipe a local log file to Kafka using the Kafka hostname from above:

```
cat /var/log/system.log | kafkacat -b xxxxxx.us-east-1.elb.amazonaws.com:9092 -t ramhiser
```

And then to consume the same log, type:

```
kafkacat -b xxxxxx.us-east-1.elb.amazonaws.com:9092 -t ramhiser
```


## Known Issues

* AWS ELB must be hardcoded in `kafka-cluster.yaml`
* Uses Kubernetes Replication Controllers instead of Deployments
* Zookeeper instances do not use a single replication controller
* Scaling the number of instances via Kubernetes does not automatically
  replicate data to new brokers
