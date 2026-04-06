## Part 6: Storage & Stateful Apps - Managing Data in Kubernetes

### Prerequisites
- Completed Parts 1-5 (or equivalent experience)
- Kubernetes cluster with storage capabilities (EBS, GPD, or local-path for Minikube)
- Basic understanding of databases and stateful applications

### What You'll Learn
- ✅ Persistent Volumes (PV) and Persistent Volume Claims (PVC)
- ✅ StorageClasses for dynamic provisioning
- ✅ StatefulSets for ordered, stable deployments
- ✅ Running databases in Kubernetes (PostgreSQL, MySQL, Redis)
- ✅ Backup and restore strategies
- ✅ Database operators (CrunchyData, Zalando, etc.)

---

## Chapter 1: The State Problem in Kubernetes

### Stateless vs Stateful Applications

**Analogy - Restaurant Workers vs. Regular Customers:**
- **Stateless app** = Catering staff (anyone can do the job, interchangeable)
- **Stateful app** = Regular customer (has assigned table, known preferences)

| Aspect | Stateless | Stateful |
|--------|-----------|----------|
| **Identity** | Any pod is fine | Each pod has unique identity |
| **Storage** | Ephemeral (lost on restart) | Persistent (survives restarts) |
| **Scaling** | Any order, any number | Ordered, careful scaling |
| **Examples** | Web servers, APIs | Databases, message queues |
| **K8s resource** | Deployment | StatefulSet |

### The Stateless Default

```yaml
# Deployment (stateless) - pods are cattle, not pets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stateless-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        # No volume = data lost on restart!
```

**Problems with Stateful Apps in Deployments:**
```bash
# Problem 1: Random naming
kubectl get pods
# myapp-7d8f9c5b6-abc12  (hard to identify)
# myapp-7d8f9c5b6-def34
# myapp-7d8f9c5b6-ghi56

# Problem 2: Any pod can be any replica
# Scaling down deletes random pods

# Problem 3: No stable network identity
# Pods can't find each other by name

# Problem 4: Storage tied to random pod
# If pod dies, storage might be lost
```

---

## Chapter 2: Persistent Volumes (PV) and Persistent Volume Claims (PVC)

### The Storage Abstraction

```
┌─────────────────────────────────────────────────────────────┐
│                    Developer View                           │
│  "I need 10GB of fast storage" (PVC)                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Kubernetes Storage Layer                       │
│         Matches PVC to appropriate PV                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│         Physical Storage (Cloud or On-Prem)                 │
│   AWS EBS | GCP PD | Azure Disk | NFS | Ceph                │
└─────────────────────────────────────────────────────────────┘
```

### PV (Cluster Storage Resource)

**Analogy - Physical Hard Drive:**
- **PV** = Actual hard drive in a server
- **PVC** = Request for disk space ("I need 10GB")
- **StorageClass** = Disk type (SSD, HDD, etc.)

**Static PV (Manual Provisioning):**
```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  labels:
    type: local
    environment: production
spec:
  capacity:
    storage: 100Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce  # Can be mounted by one pod at a time
    # - ReadOnlyMany  # Many pods can read
    # - ReadWriteMany # Many pods can read/write (requires NFS)
  persistentVolumeReclaimPolicy: Retain  # Keep data after PVC deletion
  storageClassName: manual
  hostPath:  # Only for local testing (not for production!)
    path: /mnt/data/postgres
  # For AWS EBS:
  # awsElasticBlockStore:
  #   volumeID: vol-0abc123def456
  #   fsType: ext4
```

### PVC (User Storage Request)

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi  # Request 10GB
  storageClassName: manual
  selector:
    matchLabels:
      type: local  # Bind to PV with this label
```

### Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
spec:
  containers:
  - name: postgres
    image: postgres:15
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc  # Reference PVC
```

**Check PVC status:**
```bash
# Create PV and PVC
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml

# Check binding
kubectl get pv
# NAME          CAPACITY   ACCESS MODES   STATUS    CLAIM
# postgres-pv   100Gi      RWO            Bound     production/postgres-pvc

kubectl get pvc -n production
# NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES
# postgres-pvc   Bound    postgres-pv   100Gi      RWO
```

---

## Chapter 3: StorageClasses - Dynamic Provisioning

### Why Dynamic Provisioning?

**Manual (Static) Provisioning Problems:**
```bash
# For every database, you need to:
# 1. Create PV manually (or have admin do it)
# 2. Match sizes, access modes
# 3. Clean up manually when done
# 4. Painful at scale
```

**Dynamic Provisioning:**
```yaml
# Developer just creates PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: fast-ssd  # K8s creates PV automatically!
```

### StorageClass Examples

**AWS EBS (gp3 - modern, fast):**
```yaml
# storageclass-aws-gp3.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer  # Create volume when pod schedules
reclaimPolicy: Delete  # Delete PVC when claim deleted
allowVolumeExpansion: true  # Allow resizing
```

**Google Cloud Persistent Disk (SSD):**
```yaml
# storageclass-gcp-ssd.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-ssd
  replication-type: none
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Azure Disk (Premium SSD):**
```yaml
# storageclass-azure-premium.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: disk.csi.azure.com
parameters:
  skuname: Premium_LRS
  cachingmode: ReadOnly
volumeBindingMode: WaitForFirstConsumer
```

**Local Storage (Minikube for Testing):**
```yaml
# storageclass-local.yaml (Minikube)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete

# Use in Minikube:
minikube addons enable storage-provisioner
```

### Using StorageClass

```yaml
# PVC with StorageClass (no PV needed!)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd  # Dynamic provisioning
  resources:
    requests:
      storage: 50Gi
```

**What happens automatically:**
1. K8s detects PVC with storageClassName
2. Calls cloud provider API (AWS, GCP, Azure)
3. Creates actual disk (EBS volume, GCP PD, etc.)
4. Creates PV object representing that disk
5. Binds PVC to new PV

```bash
# After applying PVC, check:
kubectl get pvc
# NAME           STATUS   VOLUME                                     CAPACITY
# database-pvc   Bound    pvc-3a4b5c6d-7e8f-9a0b-1c2d-3e4f5a6b7c8d   50Gi

kubectl get pv
# NAME                                       CAPACITY   CLAIM                 STORAGECLASS
# pvc-3a4b5c6d-7e8f-9a0b-1c2d-3e4f5a6b7c8d 50Gi       default/database-pvc  fast-ssd
```

---

## Chapter 4: StatefulSets - Stateful Applications

### StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|------------|-------------|
| **Pod naming** | Random suffix (app-abc12) | Ordered (app-0, app-1, app-2) |
| **Scaling** | Parallel, any order | Sequential (0,1,2... then down 2,1,0) |
| **Network identity** | Unstable | Stable DNS: pod-name.service-name |
| **Storage** | Shared or ephemeral | Each pod gets its own PVC |
| **Updates** | Rolling update (any order) | Ordered (reverse order) |

### StatefulSet Example: PostgreSQL

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless  # Required for stable DNS
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: POSTGRES_USER
          value: "myapp"
        - name: POSTGRES_DB
          value: "myappdb"
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:  # Each pod gets its own PVC!
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
```

### Headless Service for StatefulSet

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None  # Headless service
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
```

### How StatefulSet Works

```bash
# Create StatefulSet
kubectl apply -f statefulset-postgres.yaml
kubectl apply -f service-headless.yaml

# Watch pods create in order (0, then 1, then 2)
kubectl get pods -w -n production
# postgres-0    Pending
# postgres-0    Running
# postgres-1    Pending
# postgres-1    Running
# postgres-2    Pending
# postgres-2    Running

# Check PVCs (each pod gets its own)
kubectl get pvc -n production
# NAME               STATUS   VOLUME
# data-postgres-0    Bound    pvc-abc123
# data-postgres-1    Bound    pvc-def456
# data-postgres-2    Bound    pvc-ghi789

# Stable network identity
kubectl exec -it postgres-0 -n production -- hostname
# postgres-0

# DNS resolution (other pods can reach by name)
kubectl exec -it postgres-1 -n production -- nslookup postgres-0.postgres-headless.production.svc.cluster.local
# postgres-0.postgres-headless.production.svc.cluster.local can be reached
```

### Scaling StatefulSet

```bash
# Scale up (creates postgres-3)
kubectl scale statefulset postgres --replicas=4 -n production

# Scale down (deletes postgres-3 first, then postgres-2, etc.)
kubectl scale statefulset postgres --replicas=2 -n production

# Rolling update (updates in reverse order: 2, then 1, then 0)
kubectl set image statefulset/postgres postgres=postgres:16 -n production

# Check rollout status
kubectl rollout status statefulset/postgres -n production
# Waiting for 3 pods to be ready...
# pod/postgres-2 updated
# pod/postgres-1 updated
# pod/postgres-0 updated
```

---

## Chapter 5: Production Database Patterns

### Pattern 1: Primary-Replica Replication

```yaml
# Complete PostgreSQL with replication
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        # Primary/replica configuration
        - name: POSTGRES_REPLICATION_MODE
          value: "replica"
        - name: POSTGRES_MASTER_HOST
          value: "postgres-0.postgres-headless"
        - name: POSTGRES_MASTER_PORT
          value: "5432"
        - name: POSTGRES_REPLICATION_USER
          value: "replicator"
        - name: POSTGRES_REPLICATION_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: replication-password
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        
        # Custom startup script for replica detection
        lifecycle:
          postStart:
            exec:
              command:
                - /bin/sh
                - -c
                - |
                  if [ $(hostname) = "postgres-0" ]; then
                    export POSTGRES_REPLICATION_MODE="master"
                  else
                    export POSTGRES_REPLICATION_MODE="replica"
                  fi
```

### Pattern 2: Read-Write Split with Services

```yaml
# Write service (points to primary - postgres-0)
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres
    statefulset.kubernetes.io/pod-name: postgres-0  # Pin to specific pod
  ports:
    - port: 5432

---
# Read service (load balances across all replicas)
apiVersion: v1
kind: Service
metadata:
  name: postgres-replicas
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
```

**Application configuration:**
```yaml
# In your Spring Boot app
env:
  - name: SPRING_DATASOURCE_URL
    value: jdbc:postgresql://postgres-primary:5432/myappdb  # Writes
  - name: SPRING_DATASOURCE_READ_URL
    value: jdbc:postgresql://postgres-replicas:5432/myappdb  # Reads
```

### Pattern 3: Backup CronJob

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # 2 AM daily
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            command:
              - /bin/sh
              - -c
              - |
                # Backup database
                pg_dump -h postgres-primary -U myapp myappdb > /backup/backup-$(date +%Y%m%d-%H%M%S).sql
                
                # Compress
                gzip /backup/backup-*.sql
                
                # Upload to S3 (using AWS CLI)
                aws s3 cp /backup/backup-*.sql.gz s3://myapp-backups/postgres/
                
                # Keep only last 7 days locally
                find /backup -type f -mtime +7 -delete
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### Pattern 4: Restore Job

```yaml
# restore-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: postgres-restore
spec:
  template:
    spec:
      containers:
      - name: restore
        image: postgres:15
        env:
        - name: PGPASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        command:
          - /bin/sh
          - -c
          - |
            # Download latest backup from S3
            aws s3 cp s3://myapp-backups/postgres/latest-backup.sql.gz /tmp/
            
            # Decompress
            gunzip /tmp/latest-backup.sql.gz
            
            # Restore
            psql -h postgres-primary -U myapp -d myappdb < /tmp/latest-backup.sql
            
            echo "Restore completed!"
      restartPolicy: Never
```

---

## Chapter 6: Database Operators - The Easy Way

### What are Operators?

**Analogy - Self-driving car vs. Manual car:**
- **Manual** = You handle everything (backups, failover, scaling)
- **Operator** = Car handles everything automatically

**Operators automate:**
- Day-2 operations (backups, upgrades, failover)
- Health checking and auto-healing
- Scheduled backups
- Point-in-time recovery
- Scaling replicas

### Popular Database Operators

| Database | Operator | Features |
|----------|----------|----------|
| **PostgreSQL** | CrunchyData PGO | Backups, failover, monitoring |
| **PostgreSQL** | Zalando Postgres Operator | Patroni for HA |
| **MySQL** | Oracle MySQL Operator | Official, simple |
| **Redis** | Redis Operator | Sentinel, cluster mode |
| **MongoDB** | MongoDB Enterprise Operator | Sharding, backups |

### PostgreSQL with CrunchyData Operator

**Install Operator:**
```bash
# Add CrunchyData repository
helm repo add crunchydata https://crunchydata.github.io/charts
helm repo update

# Install PGO (PostgreSQL Operator)
helm install pgo crunchydata/pgo \
  --namespace postgres-operator \
  --create-namespace

# Verify
kubectl get pods -n postgres-operator
```

**Create PostgreSQL Cluster (Easy!):**
```yaml
# postgres-cluster.yaml
apiVersion: postgres-operator.crunchydata.com/v1beta1
kind: PostgresCluster
metadata:
  name: hippo
  namespace: production
spec:
  image: registry.developers.crunchydata.com/crunchydata/crunchy-postgres:centos8-15.2-0
  postgresVersion: 15
  instances:
    - name: instance1
      replicas: 3  # 1 primary, 2 replicas
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
      dataVolumeClaimSpec:
        accessModes:
        - "ReadWriteOnce"
        resources:
          requests:
            storage: 100Gi
  
  backups:
    pgbackrest:
      repos:
      - name: repo1
        volume:
          volumeClaimSpec:
            accessModes:
            - "ReadWriteOnce"
            resources:
              requests:
                storage: 50Gi
        schedules:
          full: "0 1 * * *"  # Daily full backup
          differential: "0 */6 * * *"  # Every 6 hours
  
  proxy:
    pgBouncer:
      replicas: 2
      resources:
        requests:
          memory: "256Mi"
          cpu: "100m"
```

**Deploy and use:**
```bash
# Create cluster
kubectl apply -f postgres-cluster.yaml

# Watch creation
kubectl get postgresclusters -n production -w

# Get connection info
kubectl get secrets -n production
# hippo-pguser-hippo  (contains username/password)

# Connect to database
kubectl port-forward svc/hippo-primary 5432:5432 -n production
psql -h localhost -U hippo -d postgres
```

### Redis with Redis Operator

```yaml
# redis-cluster.yaml
apiVersion: redis.redis.opstreelabs.in/v1beta1
kind: RedisCluster
metadata:
  name: redis-cluster
spec:
  clusterSize: 6  # 3 master, 3 replica
  clusterVersion: v7.2.0
  
  redisExporter:
    enabled: true
    image: oliver006/redis_exporter:latest
  
  persistentVolume:
    enabled: true
    storageClass: fast-ssd
    size: 10Gi
  
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  
  redisConfig:
    maxmemory: "1gb"
    maxmemory-policy: "allkeys-lru"
```

---

## Chapter 7: Complete Production Example

### E-Commerce Application with PostgreSQL and Redis

**Complete Helm Chart Structure:**
```
ecommerce-app/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── statefulset-postgres.yaml
│   ├── statefulset-redis.yaml
│   ├── pvc-backups.yaml
│   ├── cronjob-backup.yaml
│   └── services.yaml
```

**`values-prod.yaml`:**
```yaml
# Production configuration
global:
  environment: production
  storageClass: fast-ssd

# Application (stateless)
app:
  replicas: 5
  image: myregistry/ecommerce:latest
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "500m"

# PostgreSQL (stateful)
postgresql:
  enabled: true
  replicas: 3  # 1 primary, 2 replicas
  storage:
    size: 200Gi
    class: fast-ssd
  resources:
    requests:
      memory: "2Gi"
      cpu: "1"
    limits:
      memory: "4Gi"
      cpu: "2"
  backup:
    enabled: true
    schedule: "0 2 * * *"  # Daily at 2 AM
    retention: 30  # Keep 30 days
    s3:
      bucket: myapp-postgres-backups
      region: us-east-1

# Redis (stateful)
redis:
  enabled: true
  architecture: replication
  replicas: 3  # 1 master, 2 replicas
  storage:
    size: 50Gi
    class: fast-ssd
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"
  redisConfig:
    maxmemory: "2gb"
    maxmemory-policy: "allkeys-lru"

# Monitoring
monitoring:
  enabled: true
  prometheus:
    serviceMonitor:
      enabled: true
```

**`templates/statefulset-postgres.yaml`:**
```yaml
{{- if .Values.postgresql.enabled }}
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ .Release.Name }}-postgres
  namespace: {{ .Release.Namespace }}
spec:
  serviceName: {{ .Release.Name }}-postgres-headless
  replicas: {{ .Values.postgresql.replicas }}
  selector:
    matchLabels:
      app: postgres
      release: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: postgres
        release: {{ .Release.Name }}
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_USER
          value: "ecommerce"
        - name: POSTGRES_DB
          value: "ecommerce"
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Release.Name }}-postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          {{- toYaml .Values.postgresql.resources | nindent 10 }}
        livenessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - ecommerce
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - pg_isready
            - -U
            - ecommerce
          initialDelaySeconds: 5
          periodSeconds: 5
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: {{ .Values.postgresql.storage.class }}
      resources:
        requests:
          storage: {{ .Values.postgresql.storage.size }}
{{- end }}
```

**`templates/cronjob-backup.yaml`:**
```yaml
{{- if .Values.postgresql.backup.enabled }}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .Release.Name }}-postgres-backup
  namespace: {{ .Release.Namespace }}
spec:
  schedule: {{ .Values.postgresql.backup.schedule | quote }}
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Release.Name }}-postgres-secret
                  key: password
            command:
            - /bin/bash
            - -c
            - |
              # Set backup filename
              BACKUP_FILE="/backup/backup-$(date +%Y%m%d-%H%M%S).sql.gz"
              
              # Dump and compress
              pg_dump -h {{ .Release.Name }}-postgres-primary -U ecommerce ecommerce | gzip > $BACKUP_FILE
              
              # Upload to S3 (if configured)
              {{- if .Values.postgresql.backup.s3.bucket }}
              aws s3 cp $BACKUP_FILE s3://{{ .Values.postgresql.backup.s3.bucket }}/postgres/
              
              # Clean old backups (keep 30 days)
              aws s3 ls s3://{{ .Values.postgresql.backup.s3.bucket }}/postgres/ | \
                awk '{print $4}' | \
                head -n -{{ .Values.postgresql.backup.retention }} | \
                xargs -I {} aws s3 rm s3://{{ .Values.postgresql.backup.s3.bucket }}/postgres/{}
              {{- end }}
              
              echo "Backup completed: $BACKUP_FILE"
            volumeMounts:
            - name: backup-temp
              mountPath: /backup
          volumes:
          - name: backup-temp
            emptyDir: {}
          restartPolicy: OnFailure
{{- end }}
```

### Deploy Complete Stateful Application

```bash
# 1. Create namespace
kubectl create namespace ecommerce

# 2. Create secrets
kubectl create secret generic ecommerce-postgres-secret \
  --from-literal=password=$(openssl rand -base64 32) \
  -n ecommerce

# 3. Deploy with Helm
helm install ecommerce ./ecommerce-app \
  -f values-prod.yaml \
  --namespace ecommerce

# 4. Monitor deployment
kubectl get statefulsets -n ecommerce -w
# NAME                 READY   AGE
# ecommerce-postgres   0/3     0s
# ecommerce-postgres   1/3     10s
# ecommerce-postgres   2/3     20s
# ecommerce-postgres   3/3     30s

kubectl get pvc -n ecommerce
# NAME                            STATUS   VOLUME
# data-ecommerce-postgres-0       Bound    pvc-abc
# data-ecommerce-postgres-1       Bound    pvc-def
# data-ecommerce-postgres-2       Bound    pvc-ghi

# 5. Test connectivity
kubectl run test-pod --image=postgres:15 --rm -it --restart=Never -n ecommerce -- \
  psql -h ecommerce-postgres-primary -U ecommerce -c "SELECT 1"

# 6. Check backup cronjob
kubectl get cronjobs -n ecommerce
kubectl get jobs -n ecommerce
```

---

## Chapter 8: Advanced Storage Patterns

### Pattern 1: Snapshot-Based Restore

```yaml
# volumesnapshotclass.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: fast-snapshot
driver: ebs.csi.aws.com  # or pd.csi.storage.gke.io
deletionPolicy: Delete

---
# volumesnapshot.yaml (create snapshot)
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-20240315
spec:
  volumeSnapshotClassName: fast-snapshot
  source:
    persistentVolumeClaimName: data-postgres-0

---
# Restore from snapshot (create new PVC)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data
spec:
  dataSource:
    name: postgres-snapshot-20240315
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### Pattern 2: ReadWriteMany with NFS

```yaml
# Shared storage for multiple pods
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-media-pvc
spec:
  accessModes:
    - ReadWriteMany  # Multiple pods can write
  storageClassName: nfs-csi  # Requires NFS provisioner
  resources:
    requests:
      storage: 1Ti

---
# Multiple pods can access same files
apiVersion: apps/v1
kind: Deployment
metadata:
  name: media-processor
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: processor
        image: media-processor:latest
        volumeMounts:
        - name: media
          mountPath: /data
      volumes:
      - name: media
        persistentVolumeClaim:
          claimName: shared-media-pvc
```

### Pattern 3: Local SSDs for Performance

```yaml
# For high-performance databases (NVMe)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-nvme
provisioner: kubernetes.io/no-provisioner  # Static provisioning
volumeBindingMode: WaitForFirstConsumer

---
# Node-specific local PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-ssd-node1
spec:
  capacity:
    storage: 500Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-nvme
  local:
    path: /mnt/disks/ssd1  # Must exist on node
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

---

## Summary: Storage & Stateful Checklist

| Component | Purpose | Status |
|-----------|---------|--------|
| **PV & PVC** | Persistent storage abstraction | ✅ |
| **StorageClass** | Dynamic provisioning | ✅ |
| **StatefulSet** | Stateful application management | ✅ |
| **Database Operators** | Automated day-2 operations | ✅ |
| **Backup Strategies** | Data protection | ✅ |
| **Restore Procedures** | Disaster recovery | ✅ |
| **ReadWriteMany** | Shared storage | ⬜ |
| **Volume Snapshots** | Fast backups | ⬜ |

## Common Storage Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **PVC Pending** | No PV bound | Check StorageClass, permissions |
| **Pod stuck pending** | Volume can't mount | Check node availability, zone |
| **Data loss on restart** | Wrong volume type | Use StatefulSet, not Deployment |
| **Performance issues** | Slow I/O | Use faster StorageClass (SSD) |
| **Volume resizing fails** | Expansion not allowed | Check allowVolumeExpansion |
| **Backup failures** | Insufficient space | Increase backup storage |

## Practice Exercises

### Exercise 1: Deploy PostgreSQL with StatefulSet
Create StatefulSet for PostgreSQL with 3 replicas and persistent storage.

### Exercise 2: Implement Backup CronJob
Create automated daily backups to cloud storage.

### Exercise 3: Test Failover
Kill primary database pod and verify replica promotion.

### Exercise 4: Volume Snapshot and Restore
Create snapshot of database and restore to new cluster.

### Exercise 5: Install Database Operator
Install CrunchyData operator and create PostgreSQL cluster.

## Next Steps

After mastering storage, you're ready for:
- **Part 7: Security** - RBAC, PodSecurity, NetworkPolicies, OPA
- **Part 8: Observability** - Prometheus, Grafana, tracing, logging
- **Part 9: Scaling & Resilience** - HPA, VPA, chaos engineering

---

**Ready for Part 7?** Let me know and I'll create **Security Deep Dive** covering:
- RBAC (Role-Based Access Control)
- Pod Security Standards and PodSecurityPolicy
- Service Accounts and authentication
- OPA (Open Policy Agent)
- Secret management with Vault
- Image signing and verification

Would you also like me to:
1. Create a disaster recovery runbook for database failures?
2. Provide performance tuning guidelines for PostgreSQL in K8s?
3. Add examples of StatefulSet with Cassandra or Kafka?
4. Create a backup verification script?