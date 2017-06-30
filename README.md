# Kubernetes ZooKeeper
This project contains tools to facilitate the deployment of 
[Apache ZooKeeper](https://zookeeper.apache.org/) on 
[Kubernetes](http://kubernetes.io/) using 
[StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). 
It requires Kubernetes 1.7 or greater.

## Limitations
1. Scaling is not currently supported. An ensemble's membership can not be updated in a safe way in 
ZooKeeper 3.4.10 (The current stable release).
1. Observers are currently not supported. Contributions are welcome.
1. Persistent Volumes must be used. emptyDirs will likely result in a loss of data.

## ZooKeeper Docker Image 
The [docker](docker) directory contains the [Makefile](docker/Makefile) for a [Docker image](docker/Dockerfile) that 
runs a ZooKeeper server using some custom [scripts](docker/scripts/README.md).

## Manifests
The [manifests](manifests) directory contains server Kubernetes [manifests](manifests/README.md) that can be used for 
demonstration purposes or production deployments. If you primarily deploy manifests directly you can modify any of 
these to fit your use case.

## Helm
The [helm](helm/zookeeper) directory contains a [helm repository](helm/zookeeper/README.md) that deploys a ZooKeeper 
ensemble.


## Administration and Configuration
Regardless of whether you use manifests or helm to deploy your ZooKeeper ensemble, there are some common administration 
and configuration items that you should be aware of.

### Ensemble Size
As notes in the [limitation](#limitations) section, ZooKeeper membership can't be dynamically configured using the 
latest stable version. You need to select an ensemble size that suites your use case. For demonstration purposes, or if 
you are willing to tolerate at most one planned or unplanned failure, you should select an ensemble size of 3. This is 
done by setting the `spec.replicas` field of the StatefulSet to 3,

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: zk-hs
  replicas: 3
  ...
```

and passing in 3 as `--servers` parameter to the `start-zookeeper` script.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
...
        command:
        - sh
        - -c
        - "start-zookeeper \
          --servers=3 \
...
```
For production use cases, 5 servers may be desirable. This allows you to tolerate one planned **and** one unplanned 
failure.

### Memory
While ZooKeeper periodically snapshots all of its data to its data directory, the entire working data set must fit on 
heap. The `--heap` parameter of the `start-zookeeper` script controls the heap size of the ZooKeeper servers, 

```yaml 
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
...
        command:
        - sh
        - -c
        - "start-zookeeper \
...
          --heap=512M \
...
```

and the `spec.template.containers[0].resources.requests.memory` controls the memory allocated to the JVM process.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
 ...
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "gcr.io/google_containers/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "1Gi"
...
```

You should probably not use heap sizes larger than 8 GiB. For production deployments you should consider setting the 
requested memory to the maximum of 2 GiB and 1.5 times the size of the configured JVM heap.

### CPUs
ZooKeeper is not a CPU intensive application. For a production deployment you should start with 2 CPUs and adjust as 
necessary. For a demonstration deployment, you can set the CPUs as low as 0.5. The amount of CPU is configured by 
setting the StatefulSet's `spec.template.containers[0].resources.requests.cpus`.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
...
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "gcr.io/google_containers/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
...
```

### Networking
The Headless Service that controls the domain of the ensemble must have two ports. The `sever` port is used for 
inter-server communication, and the `leader-election` port is used to perform leader election.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk
spec:
  ports:
  - port: 2888
    name: server
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zk
```

These ports must correspond to the container ports in the StatefulSet's `.spec.template` and the parameters passed to 
the `start-zookeeper` script.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: zk-hs
  replicas: 3
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  template:
...
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
...
          --election_port=3888 \
          --server_port=2888 \
...
```

The Service used to load balance client connections has one port. 

```yaml 
apiVersion: v1
kind: Service
metadata:
  name: zk-cs
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zk
```

The `client` port must correspond to the container port specified in the StatefulSet's `.spec.template` and the 
parameter passed to the `start-zookeeper` script.

```yaml
```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: zk-hs
  replicas: 3
  podManagementPolicy: Parallel
  updateStrategy:
    type: RollingUpdate
  template:
...
        ports:
        - containerPort: 2181
          name: client
        - containerPort: 2888
          name: server
        - containerPort: 3888
          name: leader-election
        command:
        - sh
        - -c
        - "start-zookeeper \
...
          --client_port=2181 \
...
```

### Storage
Currently, the use of Persistent Volumes to provide durable, network attached storage is mandatory. **If you use the 
provided image with emptyDirs, you will likely suffer a data loss.** The `storage` field of the StatefulSet's 
`spec.volumeClaimTemplates` controls the storage the amount of storage allocated.

```yaml
  volumeClaimTemplates:
    - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi
```

The `volumeMounts` in the StatefulSet's `spec.template` control the mount point of the PersistentVolumes requested by 
the PersistentVolumeClaims,

```yaml
  volumeMounts:
  - name: datadir
    mountPath: /var/lib/zookeeper
```

and the parameters passed to the `start-zookeeper` script instruct the ZooKeeper server to use the PersistentVolume 
backed directory for its snapshots and write ahead log.


    --data_dir          The directory where the ZooKeeper process will store its
                        snapshots. The default is /var/lib/zookeeper/data. This 
                        directory must be backed by a persistent volume.

    --data_log_dir      The directory where the ZooKeeper process will store its 
                        write ahead log. The default is 
                        /var/lib/zookeeper/data/log. This directory must be 
                        backed by a persistent volume.


Note that, because we use network attached storage there is no benefit to using multiple PersistentVolumes to 
segregate the snapshot and write ahead log to separate storage media.

### ZooKeeper Time
ZooKeeper does not use wall clock time. Rather, it uses internal ticks that are based on an elapsed number of 
milliseconds. The various timeouts for the ZooKeeper ensemble can be controlled by the parameters passed to the 
`start-zookeeper` script.

    --tick_time         The length of a ZooKeeper tick in ms. The default is 
                        2000.

    --init_limit        The number of Ticks that an ensemble member is allowed 
                        to perform leader election. The default is 10.

    --sync_limit        The maximum session timeout that the ensemble will 
                        allows a client to request. The default is 5.

    --max_session_timeout The maximum time in milliseconds for a client session 
                        timeout. The default value is 2 * tick time.

    --min_session_timeout The minimum time in milliseconds for a client session 
                        timeout. The default value is 20 * tick time.

If you notice an excessive number of client timeouts or leader elections, you can tune these knobs as appropriate to 
increase stability, at the cost of slower failure detection.

### Snapshot Retention 
By default, ZooKeeper will never delete its own snapshots, though it will compact its write ahead log. You must either 
implement your own mechanism for retaining and removing snapshots, or configure the snapshot retention via the 
`start-zookeeper` script.

    --snap_retain_count The maximum number of snapshots the ZooKeeper process 
                        will retain if purge_interval is greater than 0. The 
                        default is 3.

    --purge_interval    The number of hours the ZooKeeper process will wait 
                        between purging its old snapshots. If set to 0 old 
                        snapshots will never be purged. The default is 0.
                        
### Pod Management Policy 
ZooKeeper is not sensitive to the order in which Pods are started. All Pods in the StatefulSet may be launched and 
terminated in parallel. This is accomplished by setting the `spec.podManagementPolicy` to `Parallel`.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
...
  podManagementPolicy: Parallel
...
```

### Update Strategy 
The Update Strategy for the StatefulSet is set to `RollingUpdate`. This enables the rolling update feature for 
StatefulSets in Kubernetes 1.7, and it allows for modifications to the StatefulSet to be propagated to its Pods.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  ...
  updateStrategy: 
    type: RollingUpdate
  ...
```

### Liveness and Readiness
The `zookeeper-ready` script can be used to check the liveness and readiness of ZooKeeper process. The example below 
demonstrates how to configure liveness and readiness probes for the Pods in the StatefulSet.

```yaml
  readinessProbe:
    exec:
      command:
      - sh
      - -c
      - "zookeeper-ready 2181"
    initialDelaySeconds: 15 
    timeoutSeconds: 5
  livenessProbe:
    exec:
      command:
      - sh
      - -c
      - "zookeeper-ready 2181"
    initialDelaySeconds: 15
    timeoutSeconds: 5
```

### Spreading
An anti-affinity rule is specified to ensure that the ZooKeeper servers are spread across nodes in the Kubernetes 
cluster. This makes the ensemble resilient to node failures.

```yaml
affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: "app"
                operator: In
                values: 
                - zk
          topologyKey: "kubernetes.io/hostname"
```

### Logging 
The Log Level configuration may be modified via the log_level flag supplied to the `start-zookeeper` script. However,
the location of the log output is not modifiable. The ZooKeeper process must be run in the foreground, and the log 
information will be shipped to stdout. This is considered to be a best practice for containerized applications, and it 
allows users to make use of the log rotation and retention infrastructure that already exists for K8s.

### Metrics 
The `zookeeper-metrics` script can be used to retrieve metrics from the ZooKeeper process and print them to stdout. A 
recurring Kubernetes job can be used to collect these metrics and provide them to a collector.

```bash
bash$ kubectl exec zk-0 zookeeper-metrics
zk_version	3.4.10-1757313, built on 08/23/2016 06:50 GMT
zk_avg_latency	0
zk_max_latency	0
zk_min_latency	0
zk_packets_received	21
zk_packets_sent	20
zk_num_alive_connections	1
zk_outstanding_requests	0
zk_server_state	follower
zk_znode_count	4
zk_watch_count	0
zk_ephemerals_count	0
zk_approximate_data_size	27
zk_open_file_descriptor_count	39
zk_max_file_descriptor_count	1048576

```
