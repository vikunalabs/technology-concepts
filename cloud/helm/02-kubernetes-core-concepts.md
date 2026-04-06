# Part 2: Kubernetes Core Concepts

## What This Covers
- Namespaces — organize and isolate workloads
- ConfigMaps — externalize configuration
- Secrets — handle sensitive data
- Resource limits — prevent noisy neighbors
- Health probes — self-healing (deeper dive than Part 1)

---

## Chapter 1: Namespaces

### What They Are

A namespace is a virtual cluster inside your real cluster. Resources in different namespaces are isolated from each other by default.

```
Cluster
├── namespace: dev        (replica: 1, dev image tag)
├── namespace: staging    (replica: 2, staging image tag)
├── namespace: prod       (replica: 5, v1.2.3 image tag)
└── namespace: monitoring (Prometheus, Grafana)
```

**Why use them:**
- Dev and prod pods won't conflict even with the same names
- You can apply resource quotas per namespace (team can't eat all cluster CPU)
- RBAC can be scoped — dev team can't touch prod namespace

### Working with Namespaces

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Deploy into a specific namespace
helm install myapp-dev ./chart --namespace dev --create-namespace

# List everything in a namespace
kubectl get all -n dev

# Set a default namespace so you don't type -n every time
kubectl config set-context --current --namespace=dev

# Delete a namespace and EVERYTHING in it (careful)
kubectl delete namespace dev
```

### Namespace YAML with ResourceQuota

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
---
# ResourceQuota: caps total resource consumption in this namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"           # Total CPU requests across all pods
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"                  # Max number of pods
```

---

## Chapter 2: ConfigMaps

### The Problem They Solve

Without ConfigMaps, any config change (log level, feature flag, database URL) requires:
1. Code change
2. Rebuild Docker image
3. Push to registry
4. Redeploy

With ConfigMaps, you change a value in Kubernetes and restart the pod. The image doesn't change.

**Rule of thumb:** Anything that differs between environments (dev/staging/prod) should be in a ConfigMap, not baked into the image.

### Creating ConfigMaps

**Method 1: From literal values**
```bash
kubectl create configmap app-config \
  --from-literal=app.name=hello-app \
  --from-literal=log.level=DEBUG \
  --from-literal=feature.new-ui=enabled \
  -n dev
```

**Method 2: Declarative YAML (preferred — version-controllable)**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  # Simple key-value
  log.level: DEBUG
  app.name: hello-app-dev

  # Multi-line value (entire application.properties as one value)
  application.properties: |
    spring.datasource.url=jdbc:postgresql://postgres-svc:5432/mydb
    spring.datasource.username=devuser
    spring.jpa.show-sql=true
```

### Using ConfigMaps in Pods

**Option A: Environment variables (easy, requires restart to pick up changes)**
```yaml
spec:
  containers:
  - name: app
    env:
    # Single key from ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level

    # All keys as environment variables at once
    envFrom:
    - configMapRef:
        name: app-config
```

**Option B: Mounted as a file (better for config files, can hot-reload)**
```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config-vol
      mountPath: /app/config        # Each key becomes a file here
  volumes:
  - name: config-vol
    configMap:
      name: app-config
```

With Option B, the file `/app/config/application.properties` will contain the multi-line value. Spring Boot can be pointed at this directory.

### Helm + ConfigMaps

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "app.fullname" . }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel }}
  APP_NAME: {{ .Values.config.appName }}
  application.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://{{ .Values.database.host }}:{{ .Values.database.port }}/mydb
```

```yaml
# values-dev.yaml
config:
  logLevel: DEBUG
  appName: hello-dev
database:
  host: postgres-dev-svc
  port: 5432

# values-prod.yaml
config:
  logLevel: WARN
  appName: hello-prod
database:
  host: postgres-prod-svc
  port: 5432
```

Now `helm upgrade` with new values regenerates the ConfigMap. Pods need a rollout restart to pick it up:

```bash
helm upgrade myapp ./chart -f values-dev.yaml
kubectl rollout restart deployment/myapp -n dev
```

---

## Chapter 3: Secrets

### What Secrets Are (and Aren't)

Secrets store sensitive data: passwords, API keys, TLS certificates.

**Important misconception:** Kubernetes Secrets are base64-encoded by default, not encrypted. Anyone with access to the namespace can read them. Base64 is encoding, not encryption.

True security requires:
- Encryption at rest (enable in etcd config, or use cloud provider KMS)
- RBAC to restrict who can `get secrets`
- Never commit Secret YAML files to Git

### Creating Secrets

```bash
# From literals (the values are base64-encoded automatically)
kubectl create secret generic db-secret \
  --from-literal=username=produser \
  --from-literal=password=S3cr3tP@ssw0rd \
  -n production
```

**YAML with stringData (plain text, auto-encoded — easier to read):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: production
type: Opaque
stringData:                       # Use stringData, not data — no manual base64 needed
  username: produser
  password: S3cr3tP@ssw0rd
  api-key: abc123xyz
```

> Never commit this file to Git. Use `.gitignore` or a secrets manager (covered in Part 7).

### Using Secrets in Pods

```yaml
spec:
  containers:
  - name: app
    # As environment variables
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # As mounted files (more secure — won't show in `kubectl describe pod`)
    volumeMounts:
    - name: secrets-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secrets-vol
    secret:
      secretName: db-secret
      # Creates files: /etc/secrets/username, /etc/secrets/password
```

Spring Boot reading from mounted files:
```java
@Value("${DB_PASSWORD}")           // From env var
private String password;

// Or read file directly
private String password = Files.readString(Path.of("/etc/secrets/password"));
```

### Secrets in Production: External Secrets Operator

For real production use, don't store secret values in Kubernetes at all. Sync them from AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets
```

```yaml
# externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-secret               # Creates a regular K8s Secret with this name
  data:
  - secretKey: password
    remoteRef:
      key: production/db/password  # AWS Secrets Manager path
```

---

## Chapter 4: Resource Limits

### Requests vs Limits

```yaml
resources:
  requests:
    memory: "256Mi"   # Guaranteed: scheduler only places pod on nodes with this free
    cpu: "250m"       # 250 millicores = 0.25 of one CPU core
  limits:
    memory: "512Mi"   # Maximum: pod is OOMKilled if it exceeds this
    cpu: "500m"       # Maximum: pod is throttled (slowed) if it exceeds this
```

**What happens without them:**
- Scheduler can't make good placement decisions → pods end up on overloaded nodes
- One pod with a memory leak can kill the entire node
- You can't use Cluster Autoscaler effectively

**CPU vs Memory behavior at limits:**
- CPU over limit → throttled (slowed down, not killed)
- Memory over limit → OOMKilled (container hard-killed immediately)

### JVM Memory Warning

The JVM doesn't know about container limits by default. Without `-XX:+UseContainerSupport`, it reads the host machine's total RAM and allocates heap accordingly — your 512Mi container tries to allocate 8GB heap → immediate OOMKill.

```dockerfile
# Always include these for Java in containers
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \  # Use 75% of container limit as max heap
  "-jar", "app.jar"]
```

With a 512Mi limit: 75% of 512Mi = ~384Mi heap. Leaves 128Mi for metaspace, thread stacks, and GC overhead.

### Finding the Right Values

1. Run the app locally under realistic load
2. Watch memory: `docker stats <container-name>`
3. Set request = observed idle memory, limit = observed peak * 1.5
4. Deploy and watch `kubectl top pods -n <namespace>` for a week
5. Adjust based on real data

### Namespace ResourceQuota

Prevents one team/environment from consuming the whole cluster:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
```

---

## Chapter 5: Health Probes (Deep Dive)

### The Three Probes and When Each Fires

```
App starts
    │
    ▼
[startupProbe]  ─── failing? ──▶ Wait, keep checking (liveness disabled)
    │ passes
    ▼
[readinessProbe] ── failing? ──▶ Remove pod from Service (no traffic sent)
    │
    ▼
[livenessProbe]  ── failing? ──▶ Kill and restart container
```

**Spring Boot startup sequence:**

Spring Boot takes 20-60 seconds to:
1. Load application context
2. Connect to databases
3. Run Flyway migrations
4. Warm up caches

Without `startupProbe`, the `livenessProbe` fires during this window and kills the pod before it finishes booting. The pod restarts, tries to boot again, gets killed again → `CrashLoopBackOff`.

### Complete Probe Configuration

```yaml
containers:
- name: app
  image: myapp:latest

  # startupProbe: Disables livenessProbe until this passes
  # failureThreshold * periodSeconds = max startup window
  # 30 * 10s = 300 seconds (5 minutes) max to start
  startupProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    failureThreshold: 30
    periodSeconds: 10

  # readinessProbe: Checked continuously during the pod's life
  # Pod is removed from Service endpoints when this fails
  # Use case: app connected to DB — if DB goes down, pod stops receiving traffic
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 0       # startupProbe handles the delay
    periodSeconds: 5
    failureThreshold: 3
    successThreshold: 1

  # livenessProbe: Restarts container if it fails
  # Only use for truly unrecoverable states (deadlock, OOM)
  # Don't tie this to external dependencies — if DB is down, don't restart the app
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 0
    periodSeconds: 10
    failureThreshold: 3
```

### Spring Boot Actuator Probe Endpoints

```yaml
# application.yml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

This gives you:
- `/actuator/health/liveness` — Returns UP unless JVM is in a broken state
- `/actuator/health/readiness` — Returns UP only when app is ready to serve (DB connected, etc.)

**Custom readiness with health indicators:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        try {
            // Check DB connectivity
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withDetail("reason", e.getMessage()).build();
        }
    }
}
```

When this returns `DOWN`, `/actuator/health/readiness` returns `OUT_OF_SERVICE`, and Kubernetes stops routing traffic to this pod.

### Graceful Shutdown

A pod being terminated goes through:
1. Pod marked as `Terminating` — no new traffic
2. `preStop` hook runs (sleep 15 to drain in-flight requests)
3. `SIGTERM` sent to container
4. `terminationGracePeriodSeconds` countdown (default 30s)
5. `SIGKILL` if still running

```yaml
# deployment.yaml
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60   # Give 60s total to shut down

      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]  # Drain in-flight requests

# application.yml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s       # Spring's own graceful shutdown window
  shutdown: graceful
```

---

## Quick Reference

### Namespace Commands

```bash
kubectl create namespace <name>
kubectl get namespaces
kubectl delete namespace <name>           # Deletes everything inside!
kubectl config set-context --current --namespace=<name>   # Set default
```

### ConfigMap Commands

```bash
kubectl create configmap <name> --from-literal=key=value
kubectl get configmap <name> -o yaml
kubectl describe configmap <name>
kubectl edit configmap <name>             # Opens in $EDITOR
kubectl delete configmap <name>
```

### Secret Commands

```bash
kubectl create secret generic <name> --from-literal=key=value
kubectl get secrets
kubectl describe secret <name>            # Shows keys but NOT values
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 --decode   # Decode a value
```

### Common Issues

| Symptom | Check |
|---------|-------|
| Pod in `CrashLoopBackOff` | `kubectl logs <pod> --previous` — see last crash |
| Pod in `Pending` | `kubectl describe pod <pod>` — usually insufficient resources or unschedulable |
| App not receiving traffic | Check readiness probe — `kubectl describe pod` shows probe failures |
| App keeps restarting | Check liveness probe — startup time may exceed `initialDelaySeconds` |
| "Secret not found" | Check namespace — secret and pod must be in same namespace |
| ConfigMap changes not visible | Did you restart the pod? `kubectl rollout restart deployment/<name>` |
