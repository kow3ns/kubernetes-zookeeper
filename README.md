# Kubernetes ZooKeeper

This project contains a Docker image meant to facilitate the deployment of 
[Apache ZooKeeper](https://zookeeper.apache.org/) on 
[Kubernetes](http://kubernetes.io/) using 
[StatefulSets](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/). 
It requires Kubernetes 1.7 or greater.

## Limitations

1. Scaling is not currently supported. An ensemble's membership can not be 
updated in a safe way in ZooKeeper 3.4.9 (The current stable release).
1. Observers are currently not supported. Contributions are welcome.
1. Persistent Volumes must be used. emptyDirs will likely result in a loss of 
data.

## Usage 

### Creating an Ensemble

To use the default manifest you need a Kubernetes 1.7 cluster with at least 
3 nodes each having .5 CPU and 2 Gib of memory.

To create an ensemble use `kubectl apply` to create the components in the 
zookeeper.yaml file in the manifests directory.

```bash
kubectl apply -f manifests/zookeeper.yaml 
service "zk-hs" configured
service "zk-cs" configured
poddisruptionbudget "zk-pdb" configured
statefulset "zk" created
```

In another terminal you can watch the Pods being created.
```bash
kubectl get po -lapp=zk -w
NAME      READY     STATUS              RESTARTS   AGE
zk-0      1/1       Running             0          32s
zk-1      0/1       ContainerCreating   0          12s
zk-1      0/1       Running   0         13s
zk-1      1/1       Running   0         30s
zk-2      0/1       Pending   0         0s
zk-2      0/1       Pending   0         0s
zk-2      0/1       ContainerCreating   0         0s
zk-2      0/1       Running   0         11s
zk-2      1/1       Running   0         30s

```

After the third Pod is created exit the terminal.

### Testing the Ensemble

You can use `kubectl exec` to run the zkCli.sh script in a bash shell on one of 
the Pods.

```bash
$ kubectl exec -ti zk-0 -- bash
zookeeper@zk-0:/$ zkCli.
[zk: localhost:2181(CONNECTED) 1] create /hello world
Created /hello

```

From another Pod you can use the same technique to retrieve the value.

```bash
$ kubectl exec -ti zk-1 -- bash
zookeeper@zk-1:/$ zkCli.                                                                                             
zkCli.cmd  zkCli.sh   
[zk: localhost:2181(CONNECTED) 0] get /hello
world
cZxid = 0x800000003
ctime = Mon Jun 12 21:25:25 UTC 2017
mZxid = 0x800000003
mtime = Mon Jun 12 21:25:25 UTC 2017
pZxid = 0x800000003
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
```

### Updating the StatefulSet

As mentioned in the [limitations](#limitations), you can't scale a ZooKeeper 
ensemble. You can update all of the configuration and resource requests. 
Edit the zookeeper.yaml manifest to decrease the memory resource request and 
the jvm heap size. The StatefulSet controller will update each Pod in the 
StatefulSet one at a time, and it will wait for the udpated Pod to be Running 
and Ready before updating the next Pod.

```yaml
 name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "gcr.io/google_samples/kubernetes-zookeeper:1.0-3.4.9"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
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
          --servers=3 \
          --data_dir=/var/lib/zookeeper/data \
          --data_log_dir=/var/lib/zookeeper/data/log \
          --conf_dir=/opt/zookeeper/conf \
          --client_port=2181 \
          --election_port=3888 \
          --server_port=2888 \
          --tick_time=2000 \
          --init_limit=10 \
          --sync_limit=5 \
          --heap=512M \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
```

Apply the manifest using `kubectl apply`.

```bash
$ kubectl apply -f manifests/zookeeper.yaml 
service "zk-hs" configured
service "zk-cs" configured
poddisruptionbudget "zk-pdb" configured
statefulset "zk" configured
```

Use `kubectl rollout status` to watch the update progress.

```bash
kubectl rollout status sts/zk
waiting for statefulset rolling update to complete 0 pods at revision zk-1312265925...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 1 pods at revision zk-1312265925...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
waiting for statefulset rolling update to complete 2 pods at revision zk-1312265925...
Waiting for 1 pods to be ready...
Waiting for 1 pods to be ready...
statefulset rolling update complete 3 pods at revision zk-1312265925.
```

## Docker Image

The docker image contained in this repository is comprised of a base 
Ubuntu 16.04 image using the latest release of the OpenJDK JRE based on the 
1.8 JVM and the latest stable release of ZooKeeper, 3.4.9. Ubuntu is a much 
larger image than BusyBox or Alpine, but these images contain mucl or ulibc. 
This requires a custom version of OpenJDK to be built against a libc runtime 
other than glibc. No vendor of the ZooKeeper software supplies or verifies the 
software against such a JVM, and, while Alpine or BusyBox would provide smaller 
images, we have prioritized a well known environment.

The image is built such that the ZooKeeper process is designated to run as a 
non-root user. By default, this user is zookeeper. The ZooKeeper package is 
installed into the /opt/zookeeper directory, all configuration is sym linked 
into the /usr/etc/zookeeper/, and all executables are sym linked into 
/usr/bin. The ZooKeeper data directories are contained in /var/lib/zookeeper. 
This is identical to the RPM distribution that users should be familiar with.

### Makefile 
The makefile contained in the docker directory has three commands.
- The `build` command will build the Docker image locally.
- The `push` command will push the image, provided you have correct permissions, 
to grc.io/samples repository.
- The `all` command will perform the `build` command.

The scripts directory contains useful utilities for managing the ZooKeeper 
process in a Kubernetes cluster. 

## Components

The zookeeper.yaml manifest, in the manifests directory, contains a 
Headless Service, a Service for client connections, a PodDisruptionBudget to 
manage planned disruptions, and a StatefulSet that deploys a ZooKeeper ensemble.

### Headless Service

The ZooKeeper StatefulSet requires a Headless Service to control the network 
domain for the ensemble. An example configuration is provided below.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: zk-hs
  labels:
    app: zk-hs
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

Note that the Service contains two ports. The server port is used for followers 
to tail the leader's event log, and the leader-election port is used by the 
ensemble to perform leader election.

### Client Service

A service should be configured for clients to connect to members of the 
ensemble that are Running and Ready. An example configuration is below.

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

The service contains one port for clients to connect to. When clients connect 
through this service they will be load balanced to an Running and Ready 
ZooKeeper server.

## PodDisruptionBudget

A PodDisruptionBudget is specified to ensure that drains and evictions respect 
the quorum size of the ensemble. The budget below indicates that the ensemble 
can tolerate at most 1 planned disruption. You should always set the PDB's 
`maxUnavailable` to 1 for ZooKeeper.

```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: zk-pdb
spec:
  selector:
    matchLabels:
      app: zk
  maxUnavailable: 1
```

### Stateful Set

The Stateful Set configuration must match the Headless Service, and it must 
provide the number of replicas. In the example below we request a ZooKeeper 
ensemble of size 3. **As weighted quorums are not supported, it is imperative 
that an odd number of replicas be chosen.Moreover, the number of replicas 
should be either 1, 3, 5, or 7. Ensembles may be scaled to larger membership 
for read fan out, but, as this will adversely impact write performance, 
careful thought should be given to selecting a larger value.** The selected 
`updateStrategy` is `RollingUpdate`. In this mode, if the manifest is updated 
(e.g. via `kubectl apply`), the StatefulSet controller will perform a rolling 
update of all Pods to apply the new configuration.

```yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zk
spec:
  serviceName: zk-headless
  replicas: 3
  updateStrategy: 
    type: RollingUpdate
```
#### Container Configuration

The start-zookeeer script will generate the ZooKeeper configuration (zoo.cfg), 
Log4J configuration (log4j.properties), and JVM configuration (jvm.env). These 
will be written to the /opt/zookeeper/conf directory with correct read 
permissions for the zookeeper user. These files are generated from command line 
parameters to the script. The containerPorts must correspond to the port 
parameters supplied to the script and the Service's above.

```yaml
containers:
- name: kubernetes-zookeeper
  imagePullPolicy: Always
  image: "gcr.io/google_samples/kubernetes-zookeeper:1.0-3.4.9"
  resources:
    requests:
      memory: "2Gi"
      cpu: "500m"
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
    --servers=3 \
    --data_dir=/var/lib/zookeeper/data \
    --data_log_dir=/var/lib/zookeeper/data/log \
    --conf_dir=/opt/zookeeper/conf \
    --client_port=2181 \
    --election_port=3888 \
    --server_port=2888 \
    --tick_time=2000 \
    --init_limit=10 \
    --sync_limit=5 \
    --heap=1024M \
    --max_client_cnxns=60 \
    --snap_retain_count=3 \
    --purge_interval=12 \
    --max_session_timeout=40000 \
    --min_session_timeout=4000 \
    --log_level=INFO"
```

##### Parameters
    --servers           The number of servers in the ensemble. The default 
                        value is 1. This must be set to 
                        `StatefulSet.Spec.Replicas`.

    --data_dir          The directory where the ZooKeeper process will store its
                        snapshots. The default is /var/lib/zookeeper/data. This 
                        directory must be backed by a persistent volume.

    --data_log_dir      The directory where the ZooKeeper process will store its 
                        write ahead log. The default is 
                        /var/lib/zookeeper/data/log. This directory must be 
                        backed by a persistent volume.

    --conf_dir          The directoyr where the ZooKeeper process will store its
                        configuration. The default is /opt/zookeeper/conf.

    --client_port       The port on which the ZooKeeper process will listen for 
                        client requests. The default is 2181. This port must be 
                        specified in both containerPorts and in the Client 
                        Service.

    --election_port     The port on which the ZooKeeper process will perform 
                        leader election. The default is 3888. This port must be 
                        specified in both containerPorts and in the Headless 
                        Service.

    --server_port       The port on which the ZooKeeper process will listen for 
                        requests from other servers in the ensemble. The 
                        default is 2888. This port must be specified in both 
                        containerPorts and in the Client Service.

    --tick_time         The length of a ZooKeeper tick in ms. The default is 
                        2000.

    --init_limit        The number of Ticks that an ensemble member is allowed 
                        to perform leader election. The default is 10.

    --sync_limit        The maximum session timeout that the ensemble will 
                        allows a client to request. The default is 5.

    --heap              The maximum amount of heap to use. The format is the 
                        same as that used for the Xmx and Xms parameters to the 
                        JVM. e.g. --heap=2G. The default is 2G. You must be sure
                        to provide sufficient overhead in the memory resources 
                        requested by ZooKeeper.

    --max_client_cnxns  The maximum number of client connections that the 
                        ZooKeeper process will accept simultaneously. The 
                        default is 60.

    --snap_retain_count The maximum number of snapshots the ZooKeeper process 
                        will retain if purge_interval is greater than 0. The 
                        default is 3.

    --purge_interval    The number of hours the ZooKeeper process will wait 
                        between purging its old snapshots. If set to 0 old 
                        snapshots will never be purged. The default is 0.

    --max_session_timeout The maximum time in milliseconds for a client session 
                        timeout. The default value is 2 * tick time.

    --min_session_timeout The minimum time in milliseconds for a client session 
                        timeout. The default value is 20 * tick time.

    --log_level         The log level for the zookeeeper server. Either FATAL,
                        ERROR, WARN, INFO, DEBUG. The default is INFO.

#### Liveness and Readiness

The zookeeper-ready script can be used to check the liveness and readiness of 
ZooKeeper process. The example below demonstrates how to configure liveness and 
readiness probes for the Pods in the Stateful Set.

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
#### Volume Mounts

volumeMounts for the container should be defined as below.

```yaml
  volumeMounts:
  - name: datadir
    mountPath: /var/lib/zookeeper
```

#### Storage Configuration

Currently, the use of Persistent Volumes to provide durable, network attached 
storage is mandatory. **If you use the provided image with emptyDirs, you will 
likely suffer a data loss.** The example below demonstrates how to request a 
dynamically provisioned persistent volume of 20 GiB.

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

#### Pod Anti-Affinity

An anti-affinity rule is specified to ensure that the ZooKeeper servers are 
spread across nodes in the Kubernetes cluster. This makes the ensemble 
resiliant to node failures.

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

## Logging 

The Log Level configuration may be modified via the log_level flag as described 
above. However, the location of the log output is not modifiable. The ZooKeeper 
process must be run in the foreground, and the log information will be shipped 
to the stdout. This is considered to be a best practice for containerized 
applications, and it allows users to make use of the log rotation and retention 
infrastructure that already exists for K8s.

## Metrics 

The zookeeper-metrics script can be used to retrieve metrics from the ZooKeeper 
process and print them to stdout. A recurring Kubernetes job can be used to 
collect these metrics and provide them to a collector.

```bash
bash$ kubectl exec zk-0 zookeeper-metrics
zk_version	3.4.9-1757313, built on 08/23/2016 06:50 GMT
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
