# Part 6: Storage & Stateful Apps — Managing Data in Kubernetes

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Intermediate–Advanced
> **Prerequisites:** Completed Parts 1–5, or comfortable with Kubernetes Deployments, Services, and Helm
> **Time to complete:** 5–6 hours
> **What you'll learn:** PersistentVolumes and PersistentVolumeClaims, StorageClasses for all major cloud providers, StatefulSets with production-grade database patterns, database operators (CrunchyData PGO, Zalando, Redis, MySQL, MongoDB), backup strategies with Velero, and advanced storage patterns including volume snapshots and NFS

---

## What This Part Covers

Everything covered so far has been stateless — application pods that can be created, destroyed, and replaced without consequence because they hold no data. Databases, caches, message queues, and search indexes are different. They hold your most critical asset. Losing a pod cannot mean losing its data.

Kubernetes was originally designed for stateless workloads, and running stateful applications in it requires understanding a separate set of primitives that work very differently from Deployments. This part covers all of it: how Kubernetes models persistent storage, why databases cannot use ordinary Deployments, how StatefulSets solve every problem Deployments create, and how database operators automate the operational work that StatefulSets leave to you.

---

## Chapter 1: The State Problem

### Stateless vs Stateful — A Concrete Distinction

Think about the difference between a catering staff member and a regular restaurant customer.

The catering staff member can be replaced at any point during an event. It doesn't matter which specific person serves a table — anyone trained for the role will do. If one staff member goes home sick, another takes their place. The guests never notice. The staff member carries no information that matters beyond the current task.

The regular customer at a long-standing restaurant is different. They have a table they always prefer, dietary restrictions on file, a tab open from last week, a loyalty account balance. Replace them with a stranger and everything accumulated over time is gone. Their specific identity and history matters.

Stateless pods are catering staff. They can be scheduled anywhere, replaced at any time, and serve any request. State — databases, caches, message queues — is the regular customer. The specific instance matters. The accumulated data matters. Continuity matters.

### Why Deployments Fail for Databases

Kubernetes Deployments were designed for stateless workloads. Using one for a database creates four distinct problems:

**Problem 1: Random pod names**
A Deployment names pods randomly: `postgres-7d8b4f9c6-xj8mk`. Every restart produces a different suffix. If you have a primary and two replicas, you cannot address the primary by a stable name because its name changes whenever it restarts. Replicas cannot be configured to follow the primary by name.

**Problem 2: Any pod can be deleted during scale-down**
When you scale a Deployment from 3 to 2 replicas, Kubernetes picks any pod to terminate — it may choose the primary. You've just deleted your write endpoint. Data that hasn't replicated to a replica is gone. In the best case you fail over. In the worst case you lose writes.

**Problem 3: No stable network identity**
In a PostgreSQL replication cluster, replica nodes connect to the primary to stream WAL (Write-Ahead Log) data. They need to know the primary's address. With a Deployment, the primary pod's IP changes on every restart. There is no stable DNS name for "the primary" — you cannot configure replicas to follow it.

**Problem 4: Shared or lost storage**
Deployment pods typically share a single PersistentVolumeClaim — or have no persistent storage at all. If multiple database pods write to the same volume, you get data corruption. If they have no persistent volume and the pod restarts, all data is lost.

### Deployment vs StatefulSet — Side by Side

| | Deployment | StatefulSet |
|-|-----------|------------|
| **Pod naming** | `postgres-7d8b4f9c6-xj8mk` (random) | `postgres-0`, `postgres-1`, `postgres-2` (ordered, stable) |
| **Pod identity** | Interchangeable — any pod is equivalent | Each pod has a unique, persistent identity |
| **Scale-down order** | Random — any pod may be removed | Always removes highest ordinal first (`postgres-2` before `postgres-1`) |
| **DNS name** | `postgres-svc` → random pod | `postgres-0.postgres-headless` → always pod 0 |
| **Storage** | Shared PVC or ephemeral | Each pod gets its own PVC via `volumeClaimTemplates` |
| **Startup order** | All pods start in parallel | Sequential: `postgres-0` must be Ready before `postgres-1` starts |
| **Use for** | Stateless apps (APIs, web servers) | Databases, caches, message queues, anything with state |

StatefulSets exist specifically to solve all four problems Deployments have with state. By the end of Chapter 4, every one of those problems has a concrete solution.

---

## Chapter 2: Persistent Volumes — The Storage Abstraction

### Why Kubernetes Abstracts Storage

Without abstractions, a pod's storage configuration would need to know whether it's running on AWS (EBS volumes), GCP (Persistent Disks), Azure (Managed Disks), or on-premises (NFS, local disk). Every team would need cloud-specific YAML for every environment. Moving between cloud providers would require rewriting all storage configuration.

Kubernetes separates the concern into three objects:

```
Developer's view                    Kubernetes layer                Physical storage
─────────────────                   ────────────────                ─────────────────
PersistentVolumeClaim    →→→→→→     PersistentVolume       →→→→→   EBS volume
"I need 20Gi of SSD"     matching   "Here is a 20Gi EBS disk"       in AWS
                          or
                          dynamic    StorageClass           →→→→→   AWS API call
                          provision  "I know how to create           creates EBS
                                     EBS gp3 volumes"               volume
```

**PersistentVolume (PV):** Represents an actual storage resource — an EBS volume, a GCP Persistent Disk, an NFS share, a local directory. Created either manually by an admin (static provisioning) or automatically by a StorageClass (dynamic provisioning). The PV knows what kind of storage it is and where it lives.

**PersistentVolumeClaim (PVC):** A pod's request for storage. "I need 20Gi of storage with read-write access." Kubernetes matches the PVC to an available PV (or creates one via the StorageClass) and binds them. The pod mounts the PVC — it doesn't know or care whether the underlying storage is EBS, GCP PD, or NFS.

**StorageClass:** A template for creating PVs dynamically. Defines the storage type (gp3 SSD, standard HDD), the provisioner (which cloud API to call), and parameters (encryption, IOPS). When a PVC requests storage and no matching PV exists, Kubernetes calls the StorageClass's provisioner to create one automatically.

### Access Modes — Who Can Mount This Volume

Every PV and PVC declares an access mode that controls how many nodes can mount the volume simultaneously:

| Mode | Short form | Meaning | Use for |
|------|-----------|---------|---------|
| `ReadWriteOnce` | RWO | One node can read and write | Databases — one pod writes, same node only |
| `ReadOnlyMany` | ROX | Many nodes can read, none can write | Configuration files, static assets |
| `ReadWriteMany` | RWX | Many nodes can read and write | Shared uploads, NFS workloads |
| `ReadWriteOncePod` | RWOP | Only one pod can read and write | Strict single-writer enforcement (K8s 1.22+) |

Cloud block storage (EBS, GCP PD, Azure Disk) only supports `ReadWriteOnce` — block devices can only be attached to one node at a time. `ReadWriteMany` requires network-attached storage: NFS, AWS EFS, Azure Files, or GCP Filestore.

For databases, `ReadWriteOnce` is almost always correct. Each database pod gets its own PVC, and that PVC is mounted on exactly one node at a time.

### Reclaim Policies — What Happens When a PVC is Deleted

When a PVC is deleted (either directly or because a namespace was deleted), what should happen to the underlying PV and the data on it?

| Policy | Behaviour | Use when |
|--------|-----------|---------|
| `Delete` | PV and underlying storage are deleted | Development — you want cleanup to be automatic |
| `Retain` | PV is preserved, released but not reused | Production — you never want data deleted automatically |
| `Recycle` | PV is scrubbed and made available again | Deprecated — don't use |

In production, always use `Retain` for database PVs. Data loss from an accidental `kubectl delete pvc` that triggers a `Delete` reclaim policy is the kind of incident that ends careers. With `Retain`, the PV persists even after the PVC is deleted — you must manually delete the PV to free the underlying storage.

### Static vs Dynamic Provisioning

**Static provisioning** (legacy, mostly replaced): An administrator manually creates PV objects describing existing storage. Developers create PVCs and Kubernetes matches them to available PVs.

```yaml
# Admin creates PV for an existing EBS volume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv-manual
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""    # Empty string means no StorageClass — static only
  awsElasticBlockStore:
    volumeID: vol-0a1b2c3d4e5f6g7h8   # Pre-existing EBS volume ID
    fsType: ext4
```

**Dynamic provisioning** (modern, standard): Developers create PVCs referencing a StorageClass. Kubernetes calls the StorageClass's provisioner to create a new PV automatically.

```yaml
# Developer creates PVC — Kubernetes creates the EBS volume automatically
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd    # Reference the StorageClass
  resources:
    requests:
      storage: 100Gi            # Kubernetes calls AWS API: create 100Gi gp3 EBS volume
```

Dynamic provisioning is the correct approach on cloud providers. There is no reason to pre-create storage manually when the cloud can create exactly what you need on demand.

### `WaitForFirstConsumer` — The Multi-AZ Gotcha

Cloud block storage (EBS, GCP PD, Azure Disk) is zone-specific. An EBS volume created in `us-east-1a` can only be attached to EC2 instances in `us-east-1a`. If a PVC is provisioned before a pod is scheduled, Kubernetes might create the EBS volume in `us-east-1a` but then schedule the pod on a node in `us-east-1b`. The pod cannot mount the volume. It gets stuck in `Pending` with a confusing error about volume topology.

`volumeBindingMode: WaitForFirstConsumer` solves this by delaying volume provisioning until a pod has been scheduled. Kubernetes picks the node first (based on affinity, resources, topology), then creates the volume in the same availability zone as that node.

```yaml
# StorageClass with WaitForFirstConsumer
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer   # Wait for pod scheduling before creating disk
allowVolumeExpansion: true                # Allow growing the volume later
parameters:
  type: gp3
  encrypted: "true"
```

Always use `WaitForFirstConsumer` on multi-AZ clusters with cloud block storage. Without it, you'll eventually hit the AZ mismatch problem.

### `allowVolumeExpansion` — Growing Volumes Without Downtime

When your 100Gi database starts filling up, you need to increase the PVC size. With `allowVolumeExpansion: true` in the StorageClass, you can expand a PVC by editing its `spec.resources.requests.storage` value. The CSI driver expands the underlying volume without detaching it or requiring pod restart (on most cloud providers with modern CSI drivers).

```bash
# Expand the PVC — edit spec.resources.requests.storage
kubectl edit pvc postgres-data -n production
# Change: storage: 100Gi → storage: 200Gi

# Watch the PVC condition update
kubectl get pvc postgres-data -n production -w
# NAME            STATUS   VOLUME          CAPACITY   STORAGECLASS   AGE
# postgres-data   Bound    pvc-abc123      100Gi      fast-ssd       30d
# postgres-data   Bound    pvc-abc123      200Gi      fast-ssd       30d   ← expanded

# The filesystem inside the pod may need to be resized separately on some configurations
kubectl exec -it postgres-0 -n production -- df -h /var/lib/postgresql/data
```

---

## Chapter 3: StorageClasses — Cloud Provider Configurations

### AWS EBS with the CSI Driver

Modern EKS clusters use the EBS CSI driver. The `gp3` volume type is the default choice — it provides better baseline performance than `gp2` at the same price, with independently configurable IOPS and throughput.

```yaml
# aws-storageclass-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    # Make this the default — PVCs without storageClassName get this
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete   # Use Retain for production databases
parameters:
  type: gp3
  encrypted: "true"
  # Optional: customize IOPS and throughput beyond gp3 defaults (3000 IOPS, 125 MB/s)
  iops: "6000"          # Up to 16,000 IOPS
  throughput: "250"     # MB/s, up to 1000 MB/s

---
# Separate class for databases needing high IOPS (io2)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: high-iops-ssd
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Retain
parameters:
  type: io2
  encrypted: "true"
  iops: "32000"         # Up to 64,000 IOPS for io2
```

### GCP Persistent Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
parameters:
  type: pd-ssd          # pd-ssd (SSD), pd-balanced (balanced), pd-standard (HDD)
  replication-type: regional-pd   # Replicate across zones for high availability

---
# Standard HDD for less critical storage (cheaper)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-hdd
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-standard
```

### Azure Managed Disks

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: disk.csi.azure.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
parameters:
  skuName: Premium_LRS    # Premium_LRS (SSD), Standard_LRS (HDD), UltraSSD_LRS
  kind: Managed
  cachingmode: ReadOnly   # ReadOnly improves read performance for databases
```

### kind — Local Path Provisioner

Unlike most of the addons we've had to install by hand for kind (Ingress, metrics-server), a default StorageClass is one thing kind gives you for free, no setup required — it ships [Rancher's `local-path-provisioner`](https://github.com/rancher/local-path-provisioner) pre-installed and pre-configured as the cluster default. You'd see this if you ran `kubectl get storageclass` right now:

```yaml
# Already present in every kind cluster — nothing to install.
# Shown here so you can see what it's actually doing.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

Like the other local-path provisioners you've seen in this chapter, `rancher.io/local-path` backs each PV with a directory on the node's filesystem — inside the kind node's container, under `/var/local-path-provisioner`. That's fine for local development and exactly why no setup is needed, but the same production caveat applies: this is single-node, node-local storage. A pod that gets rescheduled to a different node loses access to its volume. Don't reach for this pattern outside local dev and CI.

### What Happens Without a StorageClass

If a PVC doesn't specify `storageClassName`, Kubernetes uses the cluster's default StorageClass (the one annotated with `storageclass.kubernetes.io/is-default-class: "true"`). If there is no default StorageClass, the PVC stays in `Pending` indefinitely with the message:

```
no persistent volumes available for this claim and no storage class is set
```

```bash
# Check what StorageClasses exist and which is the default
kubectl get storageclass
# NAME               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE     DEFAULT
# fast-ssd (default) ebs.csi.aws.com         Delete          WaitForFirstConsumer  ✓
# high-iops-ssd      ebs.csi.aws.com         Retain          WaitForFirstConsumer

# Check whether a PVC is bound or stuck
kubectl get pvc -n production
# NAME            STATUS    VOLUME   CAPACITY   STORAGECLASS   AGE
# postgres-data   Pending                       fast-ssd       5m   ← stuck
kubectl describe pvc postgres-data -n production
# Events: waiting for a volume to be created, either by external provisioner...
```

---

## Chapter 4: StatefulSets — Managing Stateful Workloads

### How StatefulSets Solve Every Problem

Recall the four problems from Chapter 1. StatefulSets address each one precisely:

**Solution to random pod names:** StatefulSet pods are named `<statefulset-name>-<ordinal>`. The ordinal starts at 0. If you delete `postgres-0`, Kubernetes recreates a pod named `postgres-0` — not `postgres-7d8b4f9c6-xk9mp`. The name is stable across restarts.

**Solution to arbitrary scale-down order:** StatefulSets always remove the highest ordinal first. Scaling from 3 to 2 replicas removes `postgres-2`. The primary at `postgres-0` is never removed unless you scale to 0.

**Solution to no stable network identity:** Each StatefulSet pod gets a stable DNS name via a headless Service: `postgres-0.postgres-headless.production.svc.cluster.local`. This name resolves to the pod's IP, and it stays the same across restarts — even when the IP changes.

**Solution to shared/lost storage:** `volumeClaimTemplates` gives each pod its own PVC. `postgres-0` gets `data-postgres-0`, `postgres-1` gets `data-postgres-1`, and so on. These PVCs survive pod deletion and are reattached when the pod comes back.

### The Headless Service — Required for StatefulSets

Before creating a StatefulSet, create a headless Service for it. The headless Service is what makes the per-pod DNS names work:

```yaml
# headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
  labels:
    app: postgres
spec:
  clusterIP: None    # This is what makes it headless — no ClusterIP assigned
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
```

With `clusterIP: None`, DNS for `postgres-headless` returns the individual pod IPs rather than routing through a virtual IP. This enables direct addressing: `postgres-0.postgres-headless` always resolves to the pod named `postgres-0`.

You also need a regular ClusterIP Service for client connections — applications shouldn't need to know pod names:

```yaml
# client-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
    role: primary          # Only route to primary pod (labelled during startup)
  ports:
  - port: 5432
    targetPort: 5432
```

### Complete PostgreSQL StatefulSet

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  # serviceName MUST match the headless Service name.
  # This is how Kubernetes knows which Service to use for pod DNS.
  serviceName: postgres-headless

  replicas: 1   # Start with 1 for simplicity. Replication is covered in Chapter 5.

  selector:
    matchLabels:
      app: postgres

  # OrderedReady: pods start one at a time in order (0 before 1 before 2).
  # Required for databases with leader election or replication setup scripts.
  # Parallel: all pods start simultaneously — only use for truly independent pods.
  podManagementPolicy: OrderedReady

  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # Updates proceed in reverse order: 2 → 1 → 0.
      # Each pod must be Ready before the next is updated.
      partition: 0    # Update all pods. Set to N to protect pods 0..N-1 (canary updates).

  template:
    metadata:
      labels:
        app: postgres
    spec:
      # Ensure postgres data directory has correct ownership on mount.
      # The postgres user (UID 999) must own the data directory.
      initContainers:
      - name: fix-permissions
        image: busybox
        command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data"]
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
          name: postgres

        env:
        - name: POSTGRES_DB
          value: appdb
        - name: POSTGRES_USER
          value: appuser
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password

        # CRITICAL: PGDATA must point to a SUBDIRECTORY of the mounted volume.
        # WHY: When a PVC is provisioned, the mount point is not empty — it contains
        # a lost+found directory created by the filesystem. PostgreSQL checks if its
        # data directory is empty on first start and refuses to initialise if it isn't.
        # If PGDATA = /var/lib/postgresql/data (the mount root), it sees lost+found
        # and exits with: "initdb: directory "/var/lib/postgresql/data" exists but
        # is not empty".
        # Pointing PGDATA at a subdirectory (/var/lib/postgresql/data/pgdata) means
        # PostgreSQL initialises into an empty subdirectory, while lost+found stays
        # at the parent level and is ignored.
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata

        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data

        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "2Gi"
            cpu: "1000m"

        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "appuser", "-d", "appdb"]
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3

  # volumeClaimTemplates: each pod gets its own PVC automatically.
  # postgres-0 gets: data-postgres-0
  # postgres-1 gets: data-postgres-1
  # These PVCs persist when pods are deleted — data survives pod restarts.
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 100Gi
```

### The MySQL Equivalent Warning

The same `PGDATA` problem exists for MySQL. MySQL's `--datadir` (the directory where it stores its data files) must not be the same as the PVC mount root. The mounted filesystem's `lost+found` directory causes MySQL to refuse to initialise, just like PostgreSQL.

```yaml
# MySQL: point datadir at a subdirectory
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: root-password
- name: MYSQL_DATABASE
  value: appdb

volumeMounts:
- name: data
  mountPath: /var/lib/mysql    # PVC mounted here

# MySQL reads --datadir from the config file or command-line arg.
# In the official mysql image, set via the MYSQL_DATA_DIR env var
# or by mounting a custom my.cnf pointing datadir to /var/lib/mysql/data
```

A clean approach: use an init container to create the subdirectory:
```yaml
initContainers:
- name: init-mysql-dir
  image: busybox
  command: ["sh", "-c", "mkdir -p /var/lib/mysql/data && chown -R 999:999 /var/lib/mysql"]
  volumeMounts:
  - name: data
    mountPath: /var/lib/mysql
```

### `podManagementPolicy` — Ordered vs Parallel

The default `OrderedReady` policy starts and stops pods one at a time in strict order. Pod `N` must be `Running` and `Ready` before pod `N+1` is created. During scale-down, pod `N` must be fully terminated before pod `N-1` starts terminating.

This is the correct policy for databases because:
- The primary (`postgres-0`) must be running before replicas (`postgres-1`, `postgres-2`) try to connect to it
- Replicas must be removed before the primary during scale-down to avoid split-brain
- Each pod runs an init script that may depend on the previous pod's state

`Parallel` policy starts and stops all pods simultaneously. Use this for stateless workloads that use StatefulSet purely for stable DNS names — for example, a sharded cache where each shard is independent and there is no dependency between pods.

### Deploying and Verifying the StatefulSet

```bash
kubectl apply -f headless-service.yaml
kubectl apply -f client-service.yaml
kubectl apply -f statefulset-postgres.yaml

# Watch pods start in order — postgres-0 first, then postgres-1 if replicas > 1
kubectl get pods -n production -l app=postgres -w
# NAME         READY   STATUS    RESTARTS   AGE
# postgres-0   0/1     Pending   0          2s   ← waiting for PVC to provision
# postgres-0   0/1     Running   0          8s   ← container started, not yet ready
# postgres-0   1/1     Running   0          15s  ← ready ✓

# Verify PVC was created and bound
kubectl get pvc -n production
# NAME             STATUS   VOLUME          CAPACITY   STORAGECLASS   AGE
# data-postgres-0  Bound    pvc-abc123def   100Gi      fast-ssd       1m ✓

# Verify stable DNS name resolves correctly
kubectl run dns-test --image=busybox --rm -it --restart=Never -- \
  nslookup postgres-0.postgres-headless.production.svc.cluster.local
# Name: postgres-0.postgres-headless.production.svc.cluster.local
# Address: 10.244.1.5   ← the pod's IP

# Test database connection
kubectl exec -it postgres-0 -n production -- \
  psql -U appuser -d appdb -c "SELECT version();"
# PostgreSQL 15.x on ... ✓

# Watch a rollout update (reverse order: N → ... → 1 → 0)
kubectl rollout status statefulset/postgres -n production
```

### Scaling StatefulSets

```bash
# Scale up: new pods created in order (postgres-1, then postgres-2)
kubectl scale statefulset postgres --replicas=3 -n production

# Scale down: pods removed in reverse order (postgres-2, then postgres-1)
kubectl scale statefulset postgres --replicas=1 -n production

# IMPORTANT: PVCs are NOT deleted when scaling down.
# data-postgres-1 and data-postgres-2 persist after scale-down.
# If you scale back up, the pods reattach to their existing PVCs.
# Delete PVCs manually only when you're certain you no longer need the data:
kubectl delete pvc data-postgres-2 -n production
```


## Chapter 5: Production Database Patterns

### Pattern 1: Primary-Replica Replication

A single PostgreSQL pod has no redundancy — if it restarts, your application cannot write to the database until it recovers. A primary-replica setup runs two or more pods where the primary accepts reads and writes, and one or more replicas stream changes from the primary and can take over if it fails.

The challenge: replicas need to know who the primary is and configure streaming replication during startup. This is done via init containers that run setup scripts based on the pod's ordinal.

```yaml
# templates/statefulset-postgres-ha.yaml (abbreviated for clarity)
spec:
  replicas: 3
  template:
    spec:
      initContainers:
      # Init container runs different logic depending on pod ordinal
      - name: init-replication
        image: postgres:15-alpine
        command:
        - /bin/sh
        - -c
        - |
          # Determine this pod's role from its hostname ordinal
          ORDINAL=${HOSTNAME##*-}    # Extract ordinal: "postgres-1" → "1"

          if [ "$ORDINAL" = "0" ]; then
            echo "I am postgres-0: initialising as PRIMARY"
            # postgres-0 is always the primary
            # Initialise the database if not already done
            if [ ! -f "$PGDATA/PG_VERSION" ]; then
              initdb -D "$PGDATA" --username="$POSTGRES_USER" --pwfile=<(echo "$POSTGRES_PASSWORD")
              # Configure primary to accept replication connections
              echo "wal_level = replica" >> "$PGDATA/postgresql.conf"
              echo "max_wal_senders = 5" >> "$PGDATA/postgresql.conf"
              echo "hot_standby = on" >> "$PGDATA/postgresql.conf"
              echo "host replication replicator all md5" >> "$PGDATA/pg_hba.conf"
            fi
          else
            echo "I am postgres-$ORDINAL: initialising as REPLICA"
            # Replicas clone their data from the primary (postgres-0)
            PRIMARY_HOST="postgres-0.postgres-headless.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local"
            until pg_basebackup -h "$PRIMARY_HOST" -D "$PGDATA" -U replicator -W -P -Xs -R; do
              echo "Waiting for primary to be ready..."
              sleep 5
            done
          fi
        env:
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        - name: POSTGRES_USER
          value: appuser
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
```

### Pattern 2: Read-Write Split with Two Services

With primary-replica replication running, applications can send writes to the primary and reads to replicas — improving throughput and reducing load on the primary:

```yaml
# Primary service — routes only to postgres-0
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
    statefulset.kubernetes.io/pod-name: postgres-0   # Always routes to postgres-0
  ports:
  - port: 5432
    targetPort: 5432

---
# Replica service — routes to postgres-1 and postgres-2
apiVersion: v1
kind: Service
metadata:
  name: postgres-replica
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: postgres
    role: replica                # Pods must be labelled role=replica during startup
  ports:
  - port: 5432
    targetPort: 5432
```

**Spring Boot configuration for read-write split:**

```yaml
# application-prod.yml
spring:
  datasource:
    # Primary — used for writes (INSERT, UPDATE, DELETE)
    url: jdbc:postgresql://postgres-primary.production:5432/appdb
    username: ${DB_USER}
    password: ${DB_PASSWORD}

  # Read-only datasource routing (Spring's AbstractRoutingDataSource)
  datasource-readonly:
    url: jdbc:postgresql://postgres-replica.production:5432/appdb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```

```java
// Route reads to replica automatically with @Transactional(readOnly=true)
@Service
@Transactional(readOnly = true)   // Uses read replica datasource
public class ProductQueryService {
    public List<Product> findAll() { /* ... */ }
}

@Service
@Transactional                    // Uses primary datasource
public class OrderService {
    public Order create(OrderRequest req) { /* ... */ }
}
```

### Pattern 3: Automated Backup with CronJob

Run `pg_dump` on a schedule and upload the result to S3:

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"          # 2:00 AM every day
  concurrencyPolicy: Forbid       # Don't run a new job if the previous is still running
  successfulJobsHistoryLimit: 7   # Keep logs for last 7 successful runs
  failedJobsHistoryLimit: 3       # Keep logs for last 3 failed runs
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: backup-sa  # SA with IRSA for S3 access
          containers:
          - name: backup
            image: postgres:15-alpine
            command:
            - /bin/sh
            - -c
            - |
              set -e
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              BACKUP_FILE="/tmp/backup-${TIMESTAMP}.sql.gz"

              echo "Starting backup at $TIMESTAMP..."

              # Dump the database and compress it
              PGPASSWORD="$DB_PASSWORD" pg_dump \
                -h postgres-primary.production \
                -U appuser \
                -d appdb \
                --format=custom \       # Custom format: faster restore, supports parallel
                --compress=9 \          # Maximum compression
                --file="$BACKUP_FILE"

              echo "Backup complete. Uploading to S3..."

              # Install aws CLI and upload
              apk add --no-cache aws-cli
              aws s3 cp "$BACKUP_FILE" \
                "s3://myapp-backups/postgres/$(date +%Y/%m/%d)/backup-${TIMESTAMP}.dump" \
                --storage-class STANDARD_IA   # Infrequent Access — cheaper for backups

              echo "Upload complete: s3://myapp-backups/postgres/..."

              # Clean up local file
              rm "$BACKUP_FILE"
            env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            resources:
              requests:
                memory: "256Mi"
                cpu: "200m"
              limits:
                memory: "512Mi"
                cpu: "500m"
```

### Pattern 4: Restore Job

When you need to restore from a backup:

```yaml
# restore-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore-20240101
  namespace: production
spec:
  backoffLimit: 0   # Don't retry on failure — manual intervention needed
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: restore
        image: postgres:15-alpine
        command:
        - /bin/sh
        - -c
        - |
          set -e
          BACKUP_KEY="postgres/2024/01/01/backup-20240101-020000.dump"

          echo "Downloading backup from S3..."
          apk add --no-cache aws-cli
          aws s3 cp "s3://myapp-backups/$BACKUP_KEY" /tmp/restore.dump

          echo "Restoring database..."
          # Drop and recreate the target database first
          PGPASSWORD="$DB_PASSWORD" psql \
            -h postgres-primary.production \
            -U postgres \
            -c "DROP DATABASE IF EXISTS appdb_restore;"
          PGPASSWORD="$DB_PASSWORD" psql \
            -h postgres-primary.production \
            -U postgres \
            -c "CREATE DATABASE appdb_restore;"

          # Restore into the new database
          PGPASSWORD="$DB_PASSWORD" pg_restore \
            -h postgres-primary.production \
            -U postgres \
            -d appdb_restore \
            --jobs=4 \       # Parallel restore — much faster for large databases
            --verbose \
            /tmp/restore.dump

          echo "Restore complete. Verify appdb_restore before promoting."
          # After verification, rename: ALTER DATABASE appdb_restore RENAME TO appdb;
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: admin-password
```

---

## Chapter 6: Database Operators — Automating Day-2 Operations

### What Operators Are and Why They Exist

A Kubernetes Operator is a custom controller that extends Kubernetes with domain-specific knowledge. Where the built-in controllers know how to manage generic workloads (Deployments, StatefulSets), an operator knows how to manage a specific application — in our case, a database.

Think of the difference between a generic property manager and a specialist who manages historic buildings. The generic manager knows how to handle rent, repairs, and tenant complaints. The specialist knows those things too, but also knows how to apply for heritage grants, which restoration techniques comply with listed building regulations, and who to call when the original 1890s plumbing fails. The specialist encodes domain expertise that the generic manager simply doesn't have.

A database operator encodes the operational knowledge that a StatefulSet cannot. It knows:
- How to perform a primary election when the primary fails
- How to add a new replica and bootstrap it from the current primary
- How to schedule and verify Point-in-Time Recovery (PITR) backups
- How to run schema migrations safely during upgrades
- How to rotate credentials without downtime
- How to handle split-brain scenarios

Without an operator, you implement all of this yourself with scripts, CronJobs, and careful manual procedures. With an operator, you declare the desired state ("I want a 3-node PostgreSQL cluster with daily backups to S3 and automatic failover") and the operator continuously reconciles toward that state.

### When to Use an Operator vs a Plain StatefulSet

| Use a plain StatefulSet when | Use an operator when |
|----------------------------|---------------------|
| Single-instance database (dev, simple prod) | You need automatic failover/HA |
| Team has deep database expertise | Team wants Kubernetes to manage the operational work |
| Simple backup requirement (manual pg_dump) | You need PITR, automated backup verification |
| Infrequent schema changes | You need zero-downtime schema migration management |
| Learning/experimentation | Running a critical production database |

For production databases that matter, an operator is almost always the better choice. The operational gap between a StatefulSet and a properly-configured HA database cluster is significant.

### CrunchyData PGO — PostgreSQL Operator

PGO (Postgres Operator from Crunchy Data) is one of the most mature and feature-rich PostgreSQL operators. It supports HA via Patroni, PITR backups via pgBackRest, TLS, connection pooling via PgBouncer, and monitoring via pgMonitor.

**Install PGO:**
```bash
kubectl apply -k github.com/CrunchyData/postgres-operator-examples/kustomize/install/namespace
kubectl apply -k github.com/CrunchyData/postgres-operator-examples/kustomize/install/default
```

**Create a PostgreSQL cluster:**
```yaml
# postgres-cluster.yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: appdb
  namespace: production
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:ubi8-15.3-0
  postgresVersion: 15

  instances:
  - name: instance1
    replicas: 3                    # 1 primary + 2 replicas, automatic failover
    dataVolumeClaimSpec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Gi
      storageClassName: fast-ssd

  backups:
    pgbackrest:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbackrest:ubi8-2.45-0
      repos:
      - name: repo1
        s3:
          bucket: myapp-postgres-backups
          endpoint: s3.amazonaws.com
          region: us-east-1
        schedules:
          full: "0 1 * * 0"        # Full backup every Sunday at 1 AM
          differential: "0 1 * * 1-6"  # Differential Mon-Sat at 1 AM
          incremental: "0 */4 * * *"   # Incremental every 4 hours

  # PgBouncer connection pooler
  proxy:
    pgBouncer:
      image: registry.developers.crunchydata.com/crunchydata/crunchy-pgbouncer:ubi8-1.19-0
      replicas: 2
      port: 5432

  # Users and databases
  users:
  - name: appuser
    databases:
    - appdb
    options: "SUPERUSER"

  # Monitoring
  monitoring:
    pgmonitor:
      exporter:
        image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres-exporter:ubi8-5.3.1-0
```

**Access connection information:**
```bash
# PGO creates Secrets with connection credentials automatically
kubectl get secrets -n production | grep appdb
# appdb-pguser-appuser    Opaque   ...  ← connection credentials
# appdb-replication-cert  Opaque   ...  ← replication certificates
# appdb-cluster-cert      Opaque   ...  ← cluster certificates

# Get the connection string
kubectl get secret appdb-pguser-appuser -n production \
  -o jsonpath='{.data.uri}' | base64 -d
# postgresql://appuser:password@appdb-primary.production:5432/appdb

# PGO creates Services for primary, replicas, and PgBouncer
kubectl get svc -n production | grep appdb
# appdb-ha           ClusterIP  → primary (read/write, via Patroni)
# appdb-ha-config    ClusterIP  → Patroni config endpoint
# appdb-pgbouncer    ClusterIP  → PgBouncer connection pooler
# appdb-replicas     ClusterIP  → read-only replicas
```

### Zalando Postgres Operator — Patroni-Based HA

The Zalando operator uses Patroni under the hood for leader election and automatic failover. It's widely used in production and has excellent documentation.

```bash
helm repo add postgres-operator-charts \
  https://opensource.zalando.com/postgres-operator/charts/postgres-operator
helm install postgres-operator postgres-operator-charts/postgres-operator \
  -n postgres-operator --create-namespace
```

```yaml
# postgresql-cluster.yaml
apiVersion: "acid.zalan.do/v1"
kind: postgresql
metadata:
  name: appdb-cluster
  namespace: production
spec:
  teamId: "platform"
  volume:
    size: 100Gi
    storageClass: fast-ssd
  numberOfInstances: 3      # 1 primary + 2 replicas; Patroni handles failover

  postgresql:
    version: "15"
    parameters:
      shared_buffers: "256MB"
      max_connections: "200"
      work_mem: "16MB"

  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 4Gi

  users:
    appuser:              # Creates a user named appuser
      - superuser
      - createdb
  databases:
    appdb: appuser        # Creates appdb owned by appuser

  # Backup to S3 via WAL-G
  enableWalArchiving: true
  walArchiveBucketName: myapp-postgres-wal
```

```bash
# Zalando operator creates Services automatically
kubectl get svc -n production | grep appdb
# appdb-cluster           ClusterIP  → primary
# appdb-cluster-repl      ClusterIP  → replicas
# appdb-cluster-config    ClusterIP  → Patroni REST API
```

### Redis Operator — Sentinel and Cluster Modes

The OpsTree Redis Operator supports both standalone, Sentinel (HA with automatic failover), and cluster (sharded) modes:

```bash
helm repo add ot-helm https://ot-container-kit.github.io/helm-charts/
helm install redis-operator ot-helm/redis-operator -n redis-operator --create-namespace
```

**Redis Sentinel (HA, recommended for most cases):**
```yaml
# redis-sentinel.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisSentinel
metadata:
  name: redis-sentinel
  namespace: production
spec:
  clusterSize: 3           # 1 master + 2 replicas
  redisSentinelConfig:
    masterGroupName: mymaster
    redisPort: "6379"
    sentinelPort: "26379"
    quorum: "2"            # Sentinels needed to agree before failover
    downAfterMilliseconds: "5000"
    failoverTimeout: "60000"
  redisExporter:
    enabled: true          # Prometheus metrics
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 10Gi
```

**Redis Cluster (sharded, for large datasets):**
```yaml
apiVersion: redis.redis.opstreelabs.in/v1beta2
kind: RedisCluster
metadata:
  name: redis-cluster
  namespace: production
spec:
  clusterSize: 3           # 3 master shards + 3 replicas (1 replica per master)
  kubernetesConfig:
    image: redis:7.0-alpine
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: fast-ssd
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 20Gi
```

### MySQL Operator — Oracle Official

Oracle's official MySQL Operator for Kubernetes manages MySQL InnoDB Cluster (Group Replication):

```bash
helm repo add mysql-operator https://mysql.github.io/mysql-operator/
helm install mysql-operator mysql-operator/mysql-operator \
  -n mysql-operator --create-namespace
```

```yaml
# mysql-innodb-cluster.yaml
apiVersion: mysql.oracle.com/v2
kind: InnoDBCluster
metadata:
  name: appdb
  namespace: production
spec:
  secretName: mysql-root-secret    # Secret with rootPassword key
  instances: 3                     # 1 primary + 2 replicas, automatic failover

  tlsUseSelfSigned: true           # Use self-signed TLS (replace with cert-manager in prod)

  router:
    instances: 2                   # MySQL Router for connection routing
    podSpec:
      containers:
      - name: router
        resources:
          requests:
            memory: "128Mi"

  datadirVolumeClaimTemplate:
    accessModes: [ReadWriteOnce]
    storageClassName: fast-ssd
    resources:
      requests:
        storage: 100Gi

  mycnf: |
    [mysqld]
    max_connections=500
    innodb_buffer_pool_size=1G
    slow_query_log=ON
    long_query_time=2
```

### MongoDB Enterprise Operator

The MongoDB Enterprise Kubernetes Operator requires a MongoDB Cloud Manager or Ops Manager account for full features, but the community version (MongoDB Community Operator) is free:

```bash
helm repo add mongodb https://mongodb.github.io/helm-charts
helm install community-operator mongodb/community-operator \
  -n mongodb --create-namespace
```

```yaml
# mongodb-community.yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: appdb
  namespace: production
spec:
  members: 3               # 1 primary + 2 secondaries
  type: ReplicaSet
  version: "6.0.0"

  security:
    authentication:
      modes: ["SCRAM"]

  users:
  - name: appuser
    db: admin
    passwordSecretRef:
      name: mongodb-secret
    roles:
    - name: clusterAdmin
      db: admin
    - name: userAdminAnyDatabase
      db: admin

  statefulSet:
    spec:
      volumeClaimTemplates:
      - metadata:
          name: data-volume
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
```


## Chapter 7: Backup Strategies

### Application-Level Backup — pg_dump CronJob

Already covered in Chapter 5 Pattern 3. This approach is portable, runs from any pod with the database client installed, and produces a format that is independent of the underlying storage infrastructure. The tradeoff is that large databases (100GB+) take significant time to dump and restore.

For daily backups of databases under 50GB, a pg_dump CronJob is straightforward and sufficient. Above that, consider volume snapshots or Velero.

### Cluster-Level Backup with Velero

Velero backs up entire Kubernetes namespaces — not just data volumes, but all the Kubernetes objects (Deployments, Services, ConfigMaps, Secrets, PVCs) alongside the data inside those PVCs. A Velero backup can restore an entire application from scratch, including its Kubernetes configuration, not just the data.

This is the correct approach for disaster recovery scenarios where you've lost an entire namespace or cluster.

**Installing Velero with AWS S3:**

```bash
# Install the Velero CLI
brew install velero   # macOS
# Or download from: https://github.com/vmware-tanzu/velero/releases

# Install Velero in the cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket myapp-velero-backups \
  --secret-file ./aws-credentials \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=true

# aws-credentials file format:
# [default]
# aws_access_key_id=AKIA...
# aws_secret_access_key=...
# (Or use IRSA — covered in Part 7)

# Verify installation
kubectl get pods -n velero
velero version
```

**Schedule regular backups:**

```bash
# Daily full backup of the production namespace — kept for 30 days
velero schedule create production-daily \
  --schedule="0 1 * * *" \           # 1:00 AM every day
  --include-namespaces production \
  --ttl 720h                          # 30 days retention

# Hourly backup of just the database namespace — kept for 48 hours
velero schedule create database-hourly \
  --schedule="0 * * * *" \
  --include-namespaces production \
  --include-resources persistentvolumeclaims,persistentvolumes \
  --ttl 48h

# Verify schedules are active
velero schedule get
# NAME                   STATUS    CREATED                SCHEDULE    BACKUP TTL   LAST BACKUP
# production-daily       Enabled   2024-01-01 ...         0 1 * * *   720h0m0s     2h ago
# database-hourly        Enabled   2024-01-01 ...         0 * * * *   48h0m0s      10m ago
```

**Manual backup before risky operations:**
```bash
# Always take a manual backup before any significant cluster change
velero backup create pre-upgrade-$(date +%Y%m%d-%H%M%S) \
  --include-namespaces production \
  --wait    # Block until the backup completes

# Check backup status
velero backup get
# NAME                          STATUS      CREATED                   EXPIRES
# pre-upgrade-20240101-093000   Completed   2024-01-01 09:30:15 ...   29d
```

**Restore procedure:**
```bash
# List available backups
velero backup get

# Restore a specific backup to a new namespace (non-destructive, for verification)
velero restore create --from-backup pre-upgrade-20240101-093000 \
  --namespace-mappings production:production-restore \
  --wait

# Verify the restore worked
kubectl get pods -n production-restore
kubectl exec -it postgres-0 -n production-restore -- \
  psql -U appuser -d appdb -c "SELECT count(*) FROM orders;"

# Restore to the original namespace (destructive — use only when certain)
# First delete the existing namespace resources, then:
velero restore create --from-backup pre-upgrade-20240101-093000 \
  --include-namespaces production \
  --wait
```

**Restore from a scheduled backup:**
```bash
# List backups created by a schedule
velero backup get --selector velero.io/schedule-name=production-daily

# Restore from the most recent daily backup
LATEST_BACKUP=$(velero backup get --selector velero.io/schedule-name=production-daily \
  -o json | jq -r '.items | sort_by(.status.completionTimestamp) | last | .metadata.name')

velero restore create --from-backup $LATEST_BACKUP --wait
```

### Volume Snapshots — Faster than pg_dump for Large Databases

For large databases (100GB+), a pg_dump takes hours. A volume snapshot is typically completed in seconds — it's a point-in-time copy at the storage layer, not a row-by-row export.

Volume snapshots require a `VolumeSnapshotClass` provided by the CSI driver:

```yaml
# volumesnapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshots
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Delete    # Delete the EBS snapshot when VolumeSnapshot is deleted
```

**Create a snapshot:**
```yaml
# volumesnapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20240101
  namespace: production
spec:
  volumeSnapshotClassName: ebs-snapshots
  source:
    persistentVolumeClaimName: data-postgres-0   # The PVC to snapshot
```

```bash
kubectl apply -f volumesnapshot.yaml

# Check snapshot status
kubectl get volumesnapshot -n production
# NAME                          READYTOUSE   SOURCEPVC        AGE
# postgres-snapshot-20240101    true         data-postgres-0  2m ✓
```

**Automate snapshots with a CronJob:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-snapshot
  namespace: production
spec:
  schedule: "0 */6 * * *"   # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: snapshot-sa  # SA with RBAC to create VolumeSnapshots
          containers:
          - name: snapshot
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              cat <<EOF | kubectl apply -f -
              apiVersion: snapshot.storage.k8s.io/v1
              kind: VolumeSnapshot
              metadata:
                name: postgres-snap-${TIMESTAMP}
                namespace: production
              spec:
                volumeSnapshotClassName: ebs-snapshots
                source:
                  persistentVolumeClaimName: data-postgres-0
              EOF

              # Clean up snapshots older than 7 days
              kubectl get volumesnapshot -n production \
                -o jsonpath='{range .items[?(@.metadata.creationTimestamp < "'$(date -d '7 days ago' -u +%Y-%m-%dT%H:%M:%SZ)'")]}{.metadata.name}{"\n"}{end}' | \
                xargs -r kubectl delete volumesnapshot -n production
```

**Restore from a volume snapshot:**
```yaml
# Create a new PVC from the snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-restored
  namespace: production
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
  dataSource:
    name: postgres-snapshot-20240101    # The VolumeSnapshot to restore from
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Chapter 8: Advanced Storage Patterns

### ReadWriteMany with NFS — Shared Storage for Multiple Pods

Scenarios requiring multiple pods to read and write to the same volume need `ReadWriteMany`. Cloud block storage doesn't support this. Network-attached storage does.

**AWS EFS (Elastic File System):**
```bash
# Install the EFS CSI driver
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  -n kube-system
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap     # Access Points — create separate directories per PVC
  fileSystemId: fs-0abc123def456  # Your EFS file system ID
  directoryPerms: "700"

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-uploads
  namespace: production
spec:
  accessModes: [ReadWriteMany]   # Multiple pods can mount and write simultaneously
  storageClassName: efs-sc
  resources:
    requests:
      storage: 50Gi
```

**GCP Filestore (NFS):**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: filestore-sc
provisioner: filestore.csi.storage.gke.io
parameters:
  tier: standard
  network: default
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Local SSDs — NVMe for High-Performance Databases

For workloads requiring the absolute highest disk I/O (sub-millisecond latency, hundreds of thousands of IOPS), cloud block storage is insufficient. Local NVMe SSDs attached directly to the node offer 10–100× better latency than EBS or GCP PD.

The tradeoff: local storage is tied to the specific node. If the node fails, the data is lost. This is only appropriate for databases that replicate data across multiple nodes (like Cassandra, MongoDB replicated sets, or Redis Cluster) where the replication itself provides durability.

```yaml
# StorageClass for local NVMe — no dynamic provisioning (admin must pre-configure)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner  # No dynamic provisioning
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# PV created by admin for a specific node's local SSD
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-nvme-node1
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-nvme
  local:
    path: /mnt/nvme0n1        # The NVMe device mount path on the node
  # Node affinity REQUIRED for local volumes
  # Kubernetes must schedule the pod on the same node as the volume
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-with-nvme-1  # The specific node that has this NVMe device
```

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| PVC stuck in `Pending` | No StorageClass, AZ mismatch, or quota exceeded | `kubectl describe pvc <n> -n <ns>` → Events | Check StorageClass exists; check node AZ matches PV AZ; check storage quota |
| Pod stuck in `Pending` with `no volumes bound` | PVC not yet bound to a PV | `kubectl get pvc -n <ns>` | Wait for dynamic provisioning; check StorageClass provisioner logs |
| PostgreSQL fails with `initdb: directory is not empty` | `PGDATA` points to the mount root which contains `lost+found` | `kubectl logs <postgres-pod>` | Set `PGDATA` env var to a subdirectory: `/var/lib/postgresql/data/pgdata` |
| MySQL fails to start on fresh PVC | Same `lost+found` problem as PostgreSQL | `kubectl logs <mysql-pod>` | Use init container to `mkdir` the data subdirectory; set `--datadir` to that path |
| StatefulSet pod stuck in `Pending` after node failure | PVC was bound to a volume in the failed node's AZ | `kubectl describe pod <n>` → `volume node affinity conflict` | StorageClass `WaitForFirstConsumer` would have prevented this; manually delete and recreate the PVC if the volume is on a failed node |
| Volume not expanding after editing PVC | StorageClass doesn't have `allowVolumeExpansion: true` | `kubectl get sc <n> -o yaml` | Create a new StorageClass with the flag set; existing PVCs need to be migrated |
| `CrashLoopBackOff` in StatefulSet replica pod | Primary not ready when replica init container tries to clone | `kubectl logs <replica-pod> -c init-replication` | Check primary is Running+Ready; check network connectivity from replica to primary |
| Velero restore fails with `already exists` | Resources exist in the target namespace | `kubectl get all -n <ns>` | Delete existing resources before restore, or use `--namespace-mappings` to restore into a new namespace |
| VolumeSnapshot stuck in `readyToUse: false` | CSI driver snapshot class not configured, or IAM permissions missing | `kubectl describe volumesnapshot <n>` | Check VolumeSnapshotClass exists; check CSI driver has permissions to create cloud snapshots |
| `data-postgres-1` PVC not created after scaling up | StatefulSet created with wrong `volumeClaimTemplates` name | `kubectl get sts <n> -o yaml` | `volumeClaimTemplates[].metadata.name` must match `volumeMounts[].name` in pod spec |
| Operator CRD not found (`no kind "PostgresCluster" is registered`) | Operator not installed or CRDs not applied | `kubectl get crd \| grep postgres` | Install the operator and wait for CRDs to be registered |

---

## Practice Exercises

**Exercise 1 — PVC lifecycle:**
Create a PVC manually in your kind cluster. Mount it in a pod and write a file to it. Delete the pod. Create a new pod mounting the same PVC. Verify the file still exists — demonstrating that data persists beyond pod lifecycle. Then scale a StatefulSet from 1 to 3 replicas and back to 1. Verify that PVCs for the removed pods are retained (`kubectl get pvc`), not deleted.

**Exercise 2 — StatefulSet deployment and stability:**
Deploy the PostgreSQL StatefulSet from Chapter 4. Create a test database and insert 100 rows. Delete the `postgres-0` pod manually (`kubectl delete pod postgres-0 -n production`). Watch it be recreated with the same name (`kubectl get pods -w`). Verify the 100 rows still exist after the pod comes back — confirming that the PVC was reattached. Then check the DNS name resolves correctly: `kubectl exec -it <any-pod> -- nslookup postgres-0.postgres-headless.production`.

**Exercise 3 — `PGDATA` subdirectory experiment:**
Deploy PostgreSQL without the `PGDATA` env var (so it defaults to the mount root). Observe the failure in `kubectl logs postgres-0`. Read the exact error message. Fix it by setting `PGDATA=/var/lib/postgresql/data/pgdata`. Confirm it starts successfully. This is a mistake almost everyone makes once — experiencing it first-hand makes it unforgettable.

**Exercise 4 — Backup and restore cycle:**
Set up the `pg_dump` CronJob from Chapter 5. Trigger it immediately with `kubectl create job --from=cronjob/postgres-backup manual-backup`. Verify the dump file was uploaded to S3 (or your equivalent object store). Insert additional test rows. Run the restore Job. Verify the restored database contains the rows from before the extra inserts — confirming you can restore to a known-good state.

**Exercise 5 — Install an operator:**
Install the CrunchyData PGO operator in a test namespace. Create a `PostgresCluster` resource with 2 instances. Watch the operator create the StatefulSet, Services, and Secrets automatically (`kubectl get all -n production`). Compare what PGO creates vs what you created manually in Exercise 2. Connect to the cluster using the connection string from the auto-created Secret. Scale the cluster from 2 to 3 instances and observe how the operator handles the new replica's bootstrapping.
