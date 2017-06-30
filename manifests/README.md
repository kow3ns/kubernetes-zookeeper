# Kubernetes ZooKeeper Manifests
Three different manifests are provided as templates based on different uses cases for a ZooKeeper ensemble.

1. [zookeeper.yaml](zookeeper.yaml) provides a manifest that is close to production readiness.
    - It provides 5 servers with a disruption budget of 1 planned disruption. This ensemble will tolerate 1 planned and 
    1 unplanned failure.
    - Each server will consume 4 GiB of memory, 3 Gib of which will be dedicated to the ZooKeeper JVM heap.
    - Each server will consume 2 CPUs.
    - Each server will consume 1 Persistent Volume with 250 GiB of storage.
    - You can tune the parameters as nessecary to suite the needs of your deployment.
    - **The total footprint is 5 Nodes, 10 CPUs, 20 GiB memory, 1250 GiB disk**
1. [zookeeper_mini.yaml](zookeeper_mini.yaml) provides a manifest that is suitable for a demos, testing, or 
development use cases where a single zookeeper server is not desirable. 
    - It provides 3 servers with a disruption budget of 1 planned disruption. This ensemble will not tolerate any 
    concurrent unplanned failures during a planned disruption.
    - Each server will consume 1 GiB of memory, 512 MiB of which will be dedicated to the ZooKeeper 
   JVM heap.
    - Each server will consume 0.5 CPUs.
    - Each server will consume 1 Persistent Volume with 10 GiB of storage.
    - You can, again, tune this manifest to your specific needs.
    - **The total footprint is 3 Nodes, 1.5 CPUs, 3 GiB memory, 30 GiB disk**
1. [zookeeper_micro.yaml](zookeeper_micro.yaml) provides a manifest that is suitable for demos, testing, or development 
    use cases where a single zookeeper server will suffice. 
    - It provides 1 server with no disruption budget.
    - The server will consume 1 GiB of memory, 512 Mib of which will be dedicated to the ZooKeeper 
    JVM heap.
    - The server will consume 0.5 CPUs.
    - The server will consume 1 Persistent Volume with 10 GiB of storage.
    - **The total footprint is 1 Node, 0.5 CPUs, 1 GiB memory, 10 GiB disk**
    
## Usage 
This section goes over the basics of creating and updating a ZooKeeper ensemble 
on a Kubernetes 1.7 cluster. You can clone this repo, or simply download one or 
more of the manifests. You will need a working cluster and a 1.7 version of 
kubectl installed. The size of the cluster depends on which manifest you 
choose to deploy. Below, we deploy 
[zookeeper_mini.yaml](manifests/zookeeper_mini.yaml).

### Creating an Ensemble
To create an ensemble use `kubectl apply` to create the components in the 
zookeeper_mini.yaml manifest.

```bash
kubectl apply -f zookeeper_mini.yaml 
service "zk-hs" configured
service "zk-cs" configured
poddisruptionbudget "zk-pdb" configured
statefulset "zk" created
```

You can watch the creation of the Pods containing the ZooKeeper servers as 
below.

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
After all Pods are Running and Ready you can exit the terminal.

This creates four objects to manage the ZooKeeper ensemble.
1. A Headless Service, `zk-hs`, is created to control the network domain of the 
ZooKeeper ensemble. The leader election and server ports for each ZooKeeper 
server are configured for the Service.
1. A Service, `zk-cs`, is created to load balance client connections to 
ZooKeeper servers that are Running and Ready. You should consider configuring 
your client to connect directly to the Service instead of connecting to the 
Pods individually. However, both approaches will work.
1. A PodDisruptionBudget, `zk-pdb`, is created to manage planned disruptions. 
Planned disruptions can occur during drains, evictions, and managed node or 
image upgrades. `zk-pdb` specifies that the ensemble will tolerate only 1 
planned disruption. Note that, even if you deploy an ensemble of 5 or more 
ZooKeeper servers, you should only plan for one planned disruption. With 5 or 
more servers you can tolerate an unplanned disruption that occurs concurrently 
with a planned disruption.
1. A StatefulSet, `zk`, is created to launch the ZooKeeper servers, each in its 
own Pod, and each with unique and stable network identities and storage.


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

You can't scale a ZooKeeper ensemble. However, You can update all of the configuration and resource requests. Edit the 
zookeeper_mini.yaml manifest to decrease the memory resource request and the jvm heap size.

```yaml
 name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "gcr.io/google_samples/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "512Gi"
            cpu: "0.5"
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
          --heap=256M \
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

The StatefulSet controller will update each Pod in the StatefulSet, one at a time, and it will wait for the updated 
Pod to be Running and Ready before updating the next Pod.
    
## Manifest Components
Each manifest contains a Headless Service to control the network domain of the ensemble, a client Service to load 
balance client connections to available servers, and a StatefulSet to provide the required number of unique Pods. 
zookeeper.yaml and zookeeper_mini.yaml additionally contain a PodDisruptionBudget to manage planned disruptions.

## Headless Service
The `zk-hs` Headless Service will manage the domain of the ensemble. It has named ports, one for leader election and 
one for inter-server communication. These ports must correspond to the container ports specified in the StatefulSet 
and the parameters passed to the StatefulSet's Pods' command.

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

## Client Service
The `zk-cs` Service provides a load balanced Service allowing clients to connect to active ZooKeeper instances. You 
should consider allowing clients to connect directly to this service instead of using the FQDNs of the individual Pods 
in your connection string. The client port on the service must correspond to the container port specified in the 
StatefulSet and the parameters passed to the StatefulSet's Pods' command.

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

## StatefulSet
The `zk` StatefulSet creates the requested number of replicas. You can modify the StatefulSet to change the resource 
requests and the number of Pods hosting ZooKeeper servers. You have to ensure that the container ports are consistent 
with the Services defined above and the parameters passed to the start-zookeeper script. You also have to ensure that 
the requested number of replicas corresponds to the `--servers` parameter.

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
    metadata:
      labels:
        app: zk
    spec:
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
      containers:
      - name: kubernetes-zookeeper
        imagePullPolicy: Always
        image: "gcr.io/google_containers/kubernetes-zookeeper:1.0-3.4.10"
        resources:
          requests:
            memory: "4Gi"
            cpu: "2"
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
          --heap=3G \
          --max_client_cnxns=60 \
          --snap_retain_count=3 \
          --purge_interval=12 \
          --max_session_timeout=40000 \
          --min_session_timeout=4000 \
          --log_level=INFO"
        readinessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          exec:
            command:
            - sh
            - -c
            - "zookeeper-ready 2181"
          initialDelaySeconds: 10
          timeoutSeconds: 5
        volumeMounts:
        - name: datadir
          mountPath: /var/lib/zookeeper
      securityContext:
        runAsUser: 1000
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 250Gi
```

## PodDisruptionBudget
A PodDisruptionBudget is created, except in the zookeeper_micro.yaml manifest, to advertise the number of Pods that can 
be safely drained or evicted. When administrators apply managed node or base image updates, if they cordon and drain the 
node appropriately, this will ensure that the ensemble remains available throughout the procedure. For ZooKeeper, the 
correct MaxUnavailable setting is always 1. Even with the 5 server ensemble deployed by zookeeper.yaml, we want to 
ensure that we only allow one node to fail due to a planned disruption. This allows the ensemble to tolerate a 
concurrent unplanned disruption.

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