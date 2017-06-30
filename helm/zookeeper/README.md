# Zookeeper Helm Chart

 This helm chart provides an implementation of the ZooKeeper 
 [StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/) found in Kubernetes Contrib 
 [Zookeeper StatefulSet](https://github.com/kubernetes/contrib/tree/master/statefulsets/zookeeper). 
 **This chart will currently not work with any release of Helm.**
  
## Prerequisites
* Kubernetes 1.7
* PersistentVolume support on the underlying infrastructure
* A dynamic provisioner for the PersistentVolumes
* A familiarity with [Apache ZooKeeper 3.4.x](https://zookeeper.apache.org/doc/current/)

## Chart Components
This chart will do the following:

* Create a fixed size ZooKeeper ensemble using a 
[StatefulSet](http://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/).
* Create a [PodDisruptionBudget](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-disruption-budget/) so kubectl drain will respect the Quorum 
size of the ensemble.
* Create a [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/) to control the domain of the ZooKeeper ensemble.
* Create a Service configured to connect to the available ZooKeeper instances on the configured client port.
* Optionally, apply a [Pod Anti-Affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity-beta-feature) to spread the 
ZooKeeper ensemble across nodes.

## Installing the Chart

You can install the chart with the release name `my-zk` as below.

```shell
$ helm repo add k8szk https://kow3ns.github.io/kubernetes-zookeeper/helm/repo/
$ helm install --name my-zk k8s/zookeeper --values=<values>
```

If you do not specify a name, helm will select a name for you.

### Installed Components

You can use `kubectl get` to view all of the installed components.

```shell
$ kubectl get all -l component=zk-my-zk
NAME            READY     STATUS    RESTARTS   AGE
po/zk-my-zk-0   1/1       Running   0          49s
po/zk-my-zk-1   1/1       Running   0          49s
po/zk-my-zk-2   1/1       Running   0          49s

NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)             AGE
svc/zk-cs-my-zk   ClusterIP   10.0.22.15   <none>        2181/TCP            49s
svc/zk-hs-my-zk   ClusterIP   None         <none>        2888/TCP,3888/TCP   49s

NAME                    DESIRED   CURRENT   AGE
statefulsets/zk-my-zk   3         3         49s
```

1. `zy-my-zk` is the StatefulSet created by the chart.
1. `zk-my-zk-0` - `zk-my-zk-2` are the Pods created by the StatefulSet. Each Pod has a single 
container running a ZooKeeper server.
1. `zk-hs-my-zk` is the Headless Service used to control the network domain of the ZooKeeper 
ensemble.
1. `zk-cs-my-zk` is a Service that can be used by clients to connect to an available ZooKeeper 
server.

## Configuration
You can specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. 
For example,

```shell
$ helm install --name my-release -f values.yaml incubator/zookeeper
```

### Values
You can use [values.yaml](values.yaml), [value-mini.yaml](values-mini.yaml), or [value-micro.yaml](values-micro.yaml) 
as a starting point for configuring an ensemble.

1. [values.yaml](values.yaml) installs an ensemble that is close to production readiness.
    - It provides 5 servers with a disruption budget of 1 planned disruption. This ensemble will tolerate 1 planned and 
    1 unplanned failure.
    - Each server will consume 4 GiB of memory, 3 Gib of which will be dedicated to the ZooKeeper JVM heap.
    - Each server will consume 2 CPUs.
    - Each server will consume 1 Persistent Volume with 250 GiB of storage.
    - You can tune the parameters as nessecary to suite the needs of your deployment.
    - **The total footprint is 5 Nodes, 10 CPUs, 20 GiB memory, 1250 GiB disk**
1. [values-mini.yaml](values-mini.yaml) installs an ensemble that is suitable for a demos, testing, or 
development use cases where a single zookeeper server is not desirable. 
    - It provides 3 servers with a disruption budget of 1 planned disruption. This ensemble will not tolerate any 
    concurrent unplanned failures during a planned disruption.
    - Each server will consume 1 GiB of memory, 512 MiB of which will be dedicated to the ZooKeeper 
   JVM heap.
    - Each server will consume 0.5 CPUs.
    - Each server will consume 1 Persistent Volume with 10 GiB of storage.
    - You can, again, tune this manifest to your specific needs.
    - **The total footprint is 3 Nodes, 1.5 CPUs, 3 GiB memory, 30 GiB disk**
1. [values-micro.yaml](values_micro.yaml) installs and ensemble is suitable for demos, testing, or development 
    use cases where a single zookeeper server will suffice. 
    - It provides 1 server with no disruption budget.
    - The server will consume 1 GiB of memory, 512 Mib of which will be dedicated to the ZooKeeper 
    JVM heap.
    - The server will consume 0.5 CPUs.
    - The server will consume 1 Persistent Volume with 10 GiB of storage.
    - **The total footprint is 1 Node, 0.5 CPUs, 1 GiB memory, 10 GiB disk**

### Resources
The configuration parameters in this section control the resources requested and utilized by the ZooKeeper ensemble.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `Servers` | The number of ZooKeeper servers. This should always be (1,3,5, or 7) | `3` |
| `MaxUnvailable` | The maximum number of servers that may be unavailable during eviction. This should always be one. | `1` |
| `Cpu` | The amount of CPU to request. As ZooKeeper is not very CPU intensive, `2` is a good choice to start with for a production deployment. | `1` |
| `Heap` | The amount of JVM heap that the ZooKeeper servers will use. As ZooKeeper stores all of its data in memory, this value should reflect the size of your working set. The JVM -Xms/-Xmx format is used. |`2G` |
| `Memory` | The amount of memory to request. This value should be at least 2 GiB larger than `Heap` to avoid swapping. You many want to use `1.5 * Heap` for values larger than 2GiB. The Kubernetes format is used. |`2Gi` |
| `Storage` | The amount of Storage to request. Even though ZooKeeper keeps is working set in memory, it logs all transactions, and periodically snapshots, to storage media. The amount of storage required will vary with your workload, working memory size, and log and snapshot retention policy. Note that, on some cloud providers selecting a small volume size will result is sub-par I/O performance. 250 GiB is a good place to start for production workloads. | `50Gi`|
| `StorageClass` | The storage class of the storage allocated for the ensemble. If this value is present, it will add an annotation asking the PV Provisioner for that storage class. | `default` |

### Network 
These parameters control the network ports on which the ensemble communicates.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `ServerPort` | The port on which the ZooKeeper servers listen for requests from other servers in the ensemble. | `2888` |
| `LeaderElectionPort` | The port on which the ZooKeeper servers perform leader election. | `3888` |
| `ClientPort` | The port on which the ZooKeeper servers listen for client requests. | `2181` |
| `ClientCnxns` | The maximum number of simultaneous client connections that each server in the ensemble will allow. | `60` |

### Time
ZooKeeper uses the Zab protocol to replicate its state machine across the ensemble. The following parameters control 
the timeouts for the protocol.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `TicktimeMs` | The number of milliseconds in one ZooKeeper Tick. You might want to increase this value if the network latency is high or unpredictable in your environment. | `2000` |
| `InitTicks` | The amount of time, in Ticks, that a follower is allowed to connect to and sync with a leader. Increase this value if the amount of data stored on the servers is large. | `10` |
| `SyncTicks` | The amount of time, in Ticks, that a follower is allowed to lag behind a leader. If the follower is longer than SyncTicks behind the leader, the follower is dropped.  | `5` |

### Log Retention 
ZooKeeper writes its WAL (Write Ahead Log) and periodic snapshots to storage media. These parameters control the 
retention policy for snapshots and WAL segments. If you do not configure the ensemble to automatically periodically 
purge snapshots and logs, it is important to implement such a mechanism yourself. Otherwise, you will eventually exhaust 
all available storage media.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `SnapRetain` | The number of snapshots to retain on disk. If `PurgeHours` is set to 0 this parameter has no effect. | `3` |
| `PurgeHours` | The amount of time, in hours, between ZooKeeper snapshot and log purges. Setting this to 0 will disable purges.| `1` |

### Spreading 
Spreading allows you specify an anti-affinity between ZooKeeper servers in the ensemble. This will prevent the Pods from 
being scheduled on the same node.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `AntiAffinity` | If present it must take the values 'hard' or 'soft'. 'hard' will cause the Kubernetes scheduler to not schedule the Pods on the same physical node under any circumstances 'soft' will cause the Kubernetes scheduler to make a best effort to not co-locate the Pods, but, if the only available resources are on the same node, the scheduler will co-locate them. | `hard` |


### Logging 
In order to allow for the default installation to work well with the log rolling and retention policy of Kubernetes, 
all logs are written to stdout. This should also be compatible with logging integrations such as Google Cloud Logging 
and ELK.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `LogLevel` | The log level of the ZooKeeper applications. One of `ERROR`,`WARN`,`INFO`,`DEBUG`. | `INFO` |

### Liveness and Readiness
The servers in the ensemble have both liveness and readiness checks specified. These parameters can be used to tune 
the sensitivity of the liveness and readiness checks.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `ProbeInitialDelaySeconds` | The initial delay before the liveness and readiness probes will be invoked. | `15` |
| `ProbeTimeoutSeconds` | The amount of time before the probes are considered to be failed due to a timeout. | `5` |

### ImagePull
This parameter controls when the image is pulled from the repository.

| Parameter | Description | Default |
| --------- | ----------- | ------- |
| `ImagePullPolicy` | The policy for pulling the image from the repository. | `Always` |

## Scaling

ZooKeeper can not be safely scaled in versions prior to 3.5.x. There are manual procedures for scaling an ensemble, but 
as noted in the [ZooKeeper 3.5.2 documentation](https://zookeeper.apache.org/doc/r3.5.2-alpha/zookeeperReconfig.html) these 
procedures require a rolling restart, are known to be error prone, and often result in a data loss.

While ZooKeeper 3.5.x does allow for dynamic ensemble reconfiguration (including scaling membership), the current status 
of the release is still alpha, and it is not recommended for production use.

## Limitations
* StatefulSet and PodDisruptionBudget are beta resources.
* Only supports storage options that have backends for persistent volume claims
