# Part 6: Storage & Stateful Apps

## What This Covers
- PersistentVolumes and PersistentVolumeClaims
- StorageClasses and dynamic provisioning
- StatefulSets — the right resource for databases
- Backup strategies

---

## Chapter 1: The State Problem

Kubernetes Deployments treat pods as disposable. When a pod dies and restarts, its filesystem is wiped. Fine for stateless apps (APIs, web servers). Catastrophic for databases.

| | Deployment | StatefulSet |
|-|------------|-------------|
| Pod names | Random (`app-7d8f9-abc12`) | Ordered (`postgres-0`, `postgres-1`) |
| Pod identity | Interchangeable | Each pod has stable identity |
| Scaling order | Parallel, any order | Sequential (0→1→2 up, 2→1→0 down) |
| Storage | Shared or ephemeral | Each pod gets its own persistent volume |
| Use for | Stateless apps | Databases, queues, anything with state |

---

## Chapter 2: PersistentVolumes

```
PersistentVolume (PV)       = The actual storage (EBS volume, GCP disk, etc.)
PersistentVolumeClaim (PVC) = A request for storage ("I need 10GB of fast SSD")
StorageClass                = The disk type (gp3, pd-ssd, etc.) + auto-provisioning rules
```

**Without dynamic provisioning (manual):**
```yaml
# An admin creates the PV
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce    # One pod can read+write (use for block storage)
  persistentVolumeReclaimPolicy: Retain   # Keep data if PVC deleted
  hostPath:
    path: /mnt/data    # Local path — only for Minikube testing!
```

```yaml
# Developer creates PVC — binds to the PV above
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**With dynamic provisioning (cloud — use this):**

The StorageClass handles PV creation automatically:

```yaml
# StorageClass for AWS (often pre-installed on EKS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer   # Create disk in same AZ as pod
allowVolumeExpansion: true
reclaimPolicy: Delete
```

Now a developer just creates a PVC and the disk is provisioned automatically:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd     # Reference the StorageClass
  resources:
    requests:
      storage: 50Gi              # K8s calls AWS API, creates 50GB gp3 EBS volume
```

### Access Modes

| Mode | Meaning | Use for |
|------|---------|---------|
| `ReadWriteOnce` | One node can read+write | Databases (most common) |
| `ReadOnlyMany` | Many nodes can read | Config files, media |
| `ReadWriteMany` | Many nodes can read+write | Shared uploads (requires NFS) |

---

## Chapter 3: StatefulSets

```yaml
# statefulset-postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless    # Required: links to the headless service
  replicas: 1                       # Start with 1; scale when you understand replication
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
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata   # Subdirectory within mount
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "postgres"]
          initialDelaySeconds: 5
          periodSeconds: 5

  # volumeClaimTemplates: each pod gets its own PVC
  # postgres-0 gets data-postgres-0
  # postgres-1 gets data-postgres-1 (if you scale)
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

```yaml
# Headless service — required for StatefulSet DNS
# ClusterIP: None makes it "headless" — no load balancing, direct pod DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
```

With a headless service, pods get stable DNS names:
- `postgres-0.postgres-headless.production.svc.cluster.local`
- `postgres-1.postgres-headless.production.svc.cluster.local`

Your app always connects to `postgres-0.postgres-headless` — it stays the same even after pod restarts.

### Deploy and Verify

```bash
kubectl apply -f statefulset-postgres.yaml
kubectl apply -f service-headless.yaml

# Pods create in order: postgres-0 first, then postgres-1
kubectl get pods -n production -w

# Each pod gets its own PVC
kubectl get pvc -n production
# data-postgres-0   Bound
# data-postgres-1   Bound (if replicas=2)

# Test connectivity
kubectl run test --image=postgres:15 --rm -it --restart=Never -n production -- \
  psql -h postgres-0.postgres-headless.production.svc.cluster.local -U postgres
```

---

## Chapter 4: Backup Strategy

### Automated Backup with CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"      # 2 AM daily
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
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
              BACKUP_FILE="backup-$(date +%Y%m%d-%H%M%S).sql.gz"
              pg_dump -h postgres-0.postgres-headless -U postgres mydb | gzip > /tmp/$BACKUP_FILE
              aws s3 cp /tmp/$BACKUP_FILE s3://my-backups/postgres/$BACKUP_FILE
              echo "Backup complete: $BACKUP_FILE"
```

### Cluster-Level Backup with Velero

Velero backs up entire namespaces including all Kubernetes resources and PVC data:

```bash
# Install Velero (AWS example)
velero install \
  --provider aws \
  --bucket my-cluster-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Schedule daily backups
velero schedule create daily \
  --schedule="0 1 * * *" \
  --include-namespaces production \
  --ttl 168h   # Keep 7 days

# Manual backup before risky operations
velero backup create pre-upgrade-backup --include-namespaces production

# Restore
velero restore create --from-backup pre-upgrade-backup
```

---

## Quick Reference

### Common Issues

| Issue | Fix |
|-------|-----|
| PVC stuck in `Pending` | Check StorageClass exists, node has capacity, AZ matches |
| Pod stuck in `Pending` with PVC | `kubectl describe pod` — look for volume mount errors |
| Data gone after pod restart | Check you're using PVC, not `emptyDir` |
| StatefulSet pod won't delete | Scale down to 0 first: `kubectl scale sts postgres --replicas=0` |
| Can't resize PVC | Check `allowVolumeExpansion: true` in StorageClass |

### Storage Commands

```bash
kubectl get pv                                    # Cluster-wide persistent volumes
kubectl get pvc -n production                    # PVCs in namespace
kubectl describe pvc <name> -n production        # Details + events
kubectl get storageclass                         # Available storage classes
kubectl get sts -n production                    # StatefulSets
kubectl rollout status sts/postgres -n production  # Wait for rollout
```
