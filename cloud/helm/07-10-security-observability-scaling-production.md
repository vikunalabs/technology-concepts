# Part 7: Security

## What This Covers
- RBAC — who can do what in the cluster
- Service accounts — identity for pods
- Pod security — non-root, read-only filesystem
- Secrets management in production

---

## Chapter 1: RBAC

### How It Works

```
Subject (who)     →    Role (what permissions)    →    Binding (connects them)
User/Group/SA          list of (resource, verbs)        RoleBinding or ClusterRoleBinding
```

**Scope:**
- `Role` + `RoleBinding` → scoped to one namespace
- `ClusterRole` + `ClusterRoleBinding` → cluster-wide

### Principle of Least Privilege

Give only the permissions actually needed. No wildcards.

```yaml
# ✅ Good: CI/CD pipeline can update deployments in staging only
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: staging
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "update", "patch"]
- apiGroups: [""]
  resources: ["configmaps", "services"]
  verbs: ["get", "create", "update", "patch"]
# Note: no "delete" — CI/CD shouldn't delete resources

---
# ❌ Bad: wildcard permissions — anyone with this can do anything
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

```yaml
# Bind the role to a service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: staging
subjects:
- kind: ServiceAccount
  name: github-actions
  namespace: staging
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### Testing RBAC

```bash
# Can this service account list pods?
kubectl auth can-i list pods \
  --as=system:serviceaccount:staging:github-actions -n staging

# See all permissions for a user
kubectl auth can-i --list --as=alice@example.com -n production
```

---

## Chapter 2: Service Accounts

Every pod runs with a service account. By default it uses `default`, which often has more permissions than needed. Create purpose-specific service accounts.

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    # AWS: link to IAM role (IRSA — IAM Roles for Service Accounts)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-role
    # GCP: link to GCP service account
    # iam.gke.io/gcp-service-account: myapp@project.iam.gserviceaccount.com
```

```yaml
# Reference in deployment
spec:
  serviceAccountName: myapp-sa   # Pod uses this identity
  automountServiceAccountToken: false   # Don't mount K8s API token unless needed
```

### IRSA (AWS IAM Roles for Service Accounts)

This gives your pod AWS credentials without hard-coding access keys:

```bash
# Create trust policy
cat > trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/OIDC_PROVIDER"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "OIDC_PROVIDER:sub": "system:serviceaccount:production:myapp-sa"
      }
    }
  }]
}
EOF

aws iam create-role --role-name myapp-role --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name myapp-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Annotate the service account
kubectl annotate sa myapp-sa -n production \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT:role/myapp-role
```

---

## Chapter 3: Pod Security

### Security Context

Enforce security at the container level:

```yaml
spec:
  # Pod-level security
  securityContext:
    runAsNonRoot: true        # Fail if image runs as root
    runAsUser: 1000           # Run as UID 1000
    runAsGroup: 1000
    fsGroup: 1000             # Files created in volumes owned by this group
    seccompProfile:
      type: RuntimeDefault    # Restrict syscalls to safe defaults

  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false   # Can't gain more privileges than parent
      privileged: false                  # Can't access host devices
      readOnlyRootFilesystem: true       # Filesystem is read-only
      capabilities:
        drop:
        - ALL                            # Drop all Linux capabilities
        add:
        - NET_BIND_SERVICE               # Add back only what you need
```

**Tip:** `readOnlyRootFilesystem: true` will break apps that write to local disk. Add `emptyDir` volumes for temp directories:

```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
- name: logs
  mountPath: /app/logs
volumes:
- name: tmp
  emptyDir: {}
- name: logs
  emptyDir: {}
```

### Pod Security Standards (Namespace Level)

Enforce security policies for all pods in a namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted    # Blocks non-compliant pods
    pod-security.kubernetes.io/audit: restricted      # Logs violations
    pod-security.kubernetes.io/warn: restricted       # Warns in kubectl output
```

Levels: `privileged` (no restrictions) → `baseline` (blocks worst practices) → `restricted` (hardened)

---

## Chapter 4: Secrets Management in Production

Never store real secret values in Kubernetes Secret YAML files committed to Git. Instead:

**Option 1: External Secrets Operator** (sync from AWS/GCP/Vault)
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: db-credentials       # Creates a standard K8s Secret
  data:
  - secretKey: password
    remoteRef:
      key: production/myapp/db-password
```

**Option 2: Sealed Secrets** (encrypt secrets for safe Git storage)
```bash
# Install kubeseal CLI
kubeseal --fetch-cert > pub-cert.pem

# Encrypt a secret (the encrypted version is safe to commit)
kubectl create secret generic db-secret --dry-run=client \
  --from-literal=password=S3cr3tP@ss \
  -o yaml | kubeseal --cert pub-cert.pem > sealed-db-secret.yaml

# Only the cluster can decrypt it
git add sealed-db-secret.yaml   # Safe to commit
```

---

## Quick Reference

### RBAC Verbs

`get`, `list`, `watch` = read only
`create`, `update`, `patch` = write
`delete`, `deletecollection` = delete
`*` = everything (avoid)

### Security Checklist

```
□ Non-root user in Dockerfile
□ readOnlyRootFilesystem: true with emptyDir for temp dirs
□ Drop ALL capabilities, add back only needed ones
□ No hard-coded secrets — use External Secrets or Sealed Secrets
□ RBAC: one service account per app, least privilege
□ Network policies: default deny, allow what's needed
□ Pod Security Standards: at least "baseline" on prod namespace
```

---

# Part 8: Observability

## What This Covers
- The three pillars: metrics, logs, traces
- Prometheus + Grafana for metrics
- Loki for logs
- Spring Boot instrumentation

---

## Chapter 1: The Three Pillars

| Pillar | Tool | Question it answers |
|--------|------|---------------------|
| **Metrics** | Prometheus | "Is the system healthy? How fast is it?" |
| **Logs** | Loki | "What happened? What was the error?" |
| **Traces** | Tempo/Jaeger | "Which service caused the slowdown?" |

You need all three — metrics tell you something is wrong, logs tell you what happened, traces tell you where in the system.

---

## Chapter 2: Prometheus + Grafana

### Install kube-prometheus-stack

This single Helm chart installs Prometheus, Grafana, Alertmanager, and pre-built Kubernetes dashboards:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=changeme \
  --set prometheus.prometheusSpec.retention=15d

# Access Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# http://localhost:3000  admin/changeme
```

### Spring Boot Metrics

Add to `pom.xml`:
```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Add to `application.yml`:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
```

Your app now exposes `/actuator/prometheus` with hundreds of metrics.

### ServiceMonitor — Tell Prometheus to Scrape Your App

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring           # Must be in monitoring namespace
spec:
  selector:
    matchLabels:
      app: myapp                 # Match your app's Service
  namespaceSelector:
    matchNames:
    - production
  endpoints:
  - port: http
    path: /actuator/prometheus
    interval: 30s
```

Annotate your Service for simpler auto-discovery:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

### Custom Metrics in Spring Boot

```java
@RestController
public class OrderController {

    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created")
            .description("Total orders created")
            .register(registry);

        this.orderTimer = Timer.builder("orders.processing.duration")
            .description("Time to process an order")
            .register(registry);
    }

    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest req) {
        orderCounter.increment();
        return orderTimer.record(() -> orderService.create(req));
    }
}
```

These appear in Prometheus as `orders_created_total` and `orders_processing_duration_seconds`.

### Key PromQL Queries

```promql
# HTTP request rate (per second, 5 min window)
rate(http_server_requests_seconds_count[5m])

# Error rate percentage
rate(http_server_requests_seconds_count{status=~"5.."}[5m])
  /
rate(http_server_requests_seconds_count[5m]) * 100

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
)

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
```

### Alert Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: myapp
    rules:
    - alert: HighErrorRate
      expr: |
        rate(http_server_requests_seconds_count{status=~"5.."}[5m])
        /
        rate(http_server_requests_seconds_count[5m]) > 0.05
      for: 5m                           # Must be true for 5 minutes
      labels:
        severity: critical
      annotations:
        summary: "Error rate above 5%: {{ $value | humanizePercentage }}"

    - alert: PodCrashLooping
      expr: kube_pod_container_status_restarts_total > 5
      for: 10m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
```

---

## Chapter 3: Loki for Logs

### Install

```bash
helm repo add grafana https://grafana.github.io/helm-charts

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi
```

Promtail (the log collector) runs as a DaemonSet, collecting logs from all pods automatically. No changes needed to your application.

### Add Loki to Grafana

Grafana → Configuration → Data Sources → Add data source → Loki → URL: `http://loki:3100`

### LogQL Queries

```logql
# All logs from production namespace
{namespace="production"}

# Only errors
{namespace="production"} |= "ERROR"

# Filter by app
{namespace="production", app="myapp"} |= "Exception"

# Parse JSON logs and filter
{namespace="production"} | json | level="ERROR" | duration > 5s

# Rate of errors per minute
rate({namespace="production"} |= "ERROR"[1m])
```

**Tip:** Configure Spring Boot to output structured JSON logs — makes Loki filtering much more powerful:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
</dependency>
```

```xml
<!-- logback-spring.xml -->
<appender name="json" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
```

---

## Chapter 4: Distributed Tracing

When a request touches multiple services, tracing shows the full call chain with timing.

### Install Tempo

```bash
helm install tempo grafana/tempo \
  --namespace monitoring
```

Add to Grafana data sources: Tempo → URL: `http://tempo:3100`

### Spring Boot + OpenTelemetry

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>1.32.0-alpha</version>
</dependency>
```

```yaml
# application.yml
opentelemetry:
  traces:
    exporter: otlp
  exporters:
    otlp:
      endpoint: http://tempo.monitoring:4317
  service:
    name: ${spring.application.name}
```

Spring Boot now automatically creates traces for every HTTP request and propagates trace IDs to downstream services. In Grafana you can go from a log line (with trace ID) directly to the trace.

---

## Quick Reference

### Grafana Dashboard IDs to Import

| Dashboard | ID |
|-----------|-----|
| Spring Boot 2.1+ | 15758 |
| Kubernetes Cluster | 315 |
| Kubernetes Pods | 6417 |
| JVM Micrometer | 4701 |

### Alertmanager Slack Config

```yaml
# In alertmanager secret
global:
  slack_api_url: https://hooks.slack.com/services/XXX/YYY/ZZZ

route:
  receiver: slack
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
- name: slack
  slack_configs:
  - channel: '#alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

---

# Part 9: Scaling & Resilience

## What This Covers
- HPA — add more pods when load increases
- VPA — right-size resource requests automatically
- Cluster Autoscaler — add more nodes
- Pod Disruption Budgets — protect availability during maintenance
- Chaos engineering basics

---

## Chapter 1: Horizontal Pod Autoscaler (HPA)

HPA watches a metric and adjusts the replica count to keep the metric near the target.

### Install Metrics Server (required)

```bash
# EKS/GKE/AKS: usually pre-installed
# Minikube:
minikube addons enable metrics-server

# Verify
kubectl top pods -A
```

### Basic CPU-Based HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  minReplicas: 3
  maxReplicas: 20

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60    # Scale out when avg CPU > 60%

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
      - type: Percent
        value: 100                     # Can double pod count every 15s
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 25                      # Remove 25% at a time
        periodSeconds: 60
```

**Important:** HPA only works if pods have `resources.requests.cpu` set. HPA calculates utilization as `actual_cpu / requested_cpu`. If no request is set, it can't calculate anything.

### HPA with Custom Metrics (Prometheus)

For more meaningful scaling (requests per second, queue depth):

```bash
# Install Prometheus adapter
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://monitoring-kube-prometheus-prometheus.monitoring.svc
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 500          # Scale when > 500 req/sec per pod
```

### Test HPA

```bash
# Apply HPA
kubectl apply -f hpa.yaml

# Watch in real time
kubectl get hpa -n production -w

# Generate load
kubectl run load --image=busybox --rm -it --restart=Never -- \
  /bin/sh -c 'while true; do wget -q -O- http://myapp-svc; done'

# Watch pods scale
kubectl get pods -n production -w
```

---

## Chapter 2: Vertical Pod Autoscaler (VPA)

VPA analyzes actual resource usage and recommends (or automatically adjusts) resource requests.

```bash
git clone https://github.com/kubernetes/autoscaler
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh
```

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"         # "Off" = recommendations only (safe to start with)
                              # "Auto" = automatically evict and recreate pods
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
```

```bash
# See recommendations after a few hours of traffic
kubectl get vpa myapp-vpa -o yaml
# status.recommendation.containerRecommendations[0].target shows suggested values
```

**⚠️ Don't run HPA and VPA on the same CPU/memory metrics simultaneously.** They'll fight each other. Use HPA for replica count (based on CPU or custom metric) and VPA for memory sizing only, or use one and not the other.

---

## Chapter 3: Cluster Autoscaler

When HPA wants to add pods but no nodes have capacity, pods go `Pending`. Cluster Autoscaler adds nodes automatically.

### AWS EKS

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT:role/cluster-autoscaler
```

Tag your ASG (Auto Scaling Group) node groups:
```
k8s.io/cluster-autoscaler/enabled = true
k8s.io/cluster-autoscaler/my-cluster = owned
```

### GKE (built-in)

```bash
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=10 \
  --zone=us-central1-a
```

---

## Chapter 4: Pod Disruption Budgets

PDBs protect your app during voluntary disruptions: node drains, upgrades, cluster maintenance.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2              # Keep at least 2 pods running
  # OR:
  # maxUnavailable: 1          # Allow at most 1 pod down at a time
  selector:
    matchLabels:
      app: myapp
```

With 5 replicas and `minAvailable: 2`:
- Kubernetes can remove at most 3 pods at a time during maintenance
- Node drains will be blocked if removing that node would drop below 2

```bash
# Try to drain a node — PDB blocks it if it would violate the budget
kubectl drain node-1 --ignore-daemonsets

# Check PDB status
kubectl get pdb -n production
# NAME       MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# myapp-pdb  2               N/A               3
```

---

## Chapter 5: Chaos Engineering

Test resilience by intentionally breaking things in a controlled way.

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh --namespace chaos-mesh --create-namespace
```

### Pod Kill Experiment

```yaml
# Kill one pod from myapp every 30 minutes
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
spec:
  action: pod-kill
  mode: fixed
  value: "1"
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 30m"
  duration: "1m"
```

### Network Delay Experiment

```yaml
# Add 100ms latency to 50% of network calls
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay
spec:
  action: delay
  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp
  delay:
    latency: "100ms"
    jitter: "10ms"
  duration: "5m"
```

**Safe chaos practices:**
- Always have PDB in place before running chaos
- Start in staging, never production first
- Have monitoring watching during experiments
- Know your rollback: `kubectl delete podchaos pod-kill`

---

## Quick Reference

### Scaling Checklist

```
□ Resources requests set on all containers (HPA requires this)
□ HPA configured with appropriate min/max
□ PDB configured to protect availability
□ Cluster Autoscaler installed and node group tagged
□ Graceful shutdown configured (preStop + terminationGracePeriod)
```

### HPA Debugging

```bash
kubectl describe hpa myapp-hpa -n production
# Shows: current metric values, desired replicas, recent events

kubectl get events -n production --sort-by='.lastTimestamp' | grep HPA
```

---

# Part 10: Production Playbook

## Pre-Deploy Checklist

```bash
#!/bin/bash
# Run before every production deployment

echo "--- Application ---"
# Tests passing
./mvnw test

echo "--- Security ---"
# No critical CVEs
trivy image myregistry.com/myapp:$VERSION --severity CRITICAL --exit-code 1

echo "--- Kubernetes ---"
# Resources set
kubectl get deployment myapp -n production -o json | \
  jq '.spec.template.spec.containers[0].resources'

# HPA exists
kubectl get hpa -n production

# PDB exists
kubectl get pdb -n production

# Monitoring working
kubectl get servicemonitor -n monitoring | grep myapp

echo "--- Ready to deploy ---"
```

## Deployment Commands

```bash
# Standard deployment
helm upgrade --install myapp ./helm-chart \
  --namespace production \
  -f values.yaml \
  -f values-prod.yaml \
  --set image.tag=$VERSION \
  --atomic \              # Auto-rollback on failure
  --wait \
  --timeout 15m

# Verify
kubectl rollout status deployment/myapp -n production

# Quick rollback
helm rollback myapp -n production    # Rolls back to previous release

# Emergency rollback (kubectl)
kubectl rollout undo deployment/myapp -n production
```

## Incident Response

### High Error Rate

```bash
# 1. Check what changed recently
helm history myapp -n production
kubectl get events -n production --sort-by='.lastTimestamp'

# 2. Check pod health
kubectl get pods -n production
kubectl logs -f deployment/myapp -n production --tail=100

# 3. Rollback if recent deploy
helm rollback myapp -n production

# 4. Scale up while investigating
kubectl scale deployment/myapp --replicas=10 -n production
```

### Pod CrashLoopBackOff

```bash
# See why it's crashing
kubectl logs <pod-name> -n production --previous   # Previous container's logs

# Check events
kubectl describe pod <pod-name> -n production

# Common causes:
# - OOMKilled: increase memory limit
# - Config error: check env vars, mounted secrets/configmaps
# - Port conflict: check container port config
# - Startup probe failing too fast: increase failureThreshold
```

### Node Issues

```bash
# Check node health
kubectl get nodes
kubectl describe node <node-name>   # Shows resource pressure, events

# Reschedule all pods from a node (maintenance)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Make node schedulable again
kubectl uncordon <node-name>
```

## Production Values Template

```yaml
# values-prod.yaml — complete production configuration
replicaCount: 3

image:
  repository: myregistry.com/myapp
  tag: ""               # Set at deploy time: --set image.tag=$VERSION
  pullPolicy: Always

imagePullSecrets:
  - name: regcred

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60

podDisruptionBudget:
  enabled: true
  minAvailable: 2

updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0       # Zero downtime rolling update

probes:
  startup:
    enabled: true
    failureThreshold: 30
    periodSeconds: 10
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 0
    periodSeconds: 10
    failureThreshold: 3
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 0
    periodSeconds: 5
    failureThreshold: 3

lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 20"]

terminationGracePeriodSeconds: 60

securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

serviceAccount:
  create: true
  name: myapp-sa

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
  hosts:
  - host: api.myapp.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls

# Spread pods across availability zones
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp

# Avoid putting all pods on same node
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname

monitoring:
  serviceMonitor:
    enabled: true
    interval: 30s
```

## Key Monitoring Queries for Production

```promql
# Is the service up? (should be 1)
up{namespace="production", job="myapp"}

# Error rate (alert if > 1%)
sum(rate(http_server_requests_seconds_count{namespace="production",status=~"5.."}[5m]))
/
sum(rate(http_server_requests_seconds_count{namespace="production"}[5m])) * 100

# P99 latency (alert if > 500ms)
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{namespace="production"}[5m])) by (le)
)

# Pod restarts in last hour
increase(kube_pod_container_status_restarts_total{namespace="production"}[1h])

# Memory usage % of limit
sum(container_memory_working_set_bytes{namespace="production"})
/
sum(container_spec_memory_limit_bytes{namespace="production"}) * 100
```

## Cost Optimization

```bash
# Find pods with no resource requests (waste and risk)
kubectl get pods -A -o json | \
  jq '.items[] | select(.spec.containers[].resources.requests == null) | .metadata.name'

# Find unused PVCs
kubectl get pvc -A | grep Released

# Find load balancers outside production (expensive)
kubectl get svc -A | grep LoadBalancer | grep -v production

# Check VPA recommendations for right-sizing
kubectl get vpa -A -o yaml | grep -A5 "target:"

# Scale non-production to zero overnight
kubectl scale deployment --all --replicas=0 -n dev
# Set up CronJob to restore in the morning
```
