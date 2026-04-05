## Part 9: Scaling & Resilience - Building Self-Healing Systems

### Prerequisites
- Completed Parts 1-8 (or equivalent experience)
- Kubernetes cluster with metrics server (EKS, GKE, AKS)
- Basic understanding of application performance

### What You'll Learn
- ✅ Horizontal Pod Autoscaler (HPA) - Scale based on metrics
- ✅ Vertical Pod Autoscaler (VPA) - Optimize resource requests
- ✅ Cluster Autoscaler - Scale infrastructure
- ✅ Pod Disruption Budgets (PDB) - Maintain availability
- ✅ Chaos Engineering - Test resilience
- ✅ Advanced deployment strategies (Blue-Green, Canary)
- ✅ Disaster recovery and backup

---

## Chapter 1: Scaling in Kubernetes - The Big Picture

### Types of Scaling

```
┌─────────────────────────────────────────────────────────────┐
│                    User Traffic Increase                    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Horizontal Scaling (HPA)                                   │
│  "Add more pods"                                            │
│  3 pods → 10 pods                                           │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Vertical Scaling (VPA)                                     │
│  "Make each pod bigger"                                     │
│  256MB → 1GB per pod                                        │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Cluster Scaling (Cluster Autoscaler)                       │
│  "Add more nodes"                                           │
│  3 nodes → 10 nodes                                         │
└─────────────────────────────────────────────────────────────┘
```

### Scaling Comparison

| Type | What It Does | When to Use | Speed |
|------|--------------|-------------|-------|
| **HPA** | Changes replica count | Stateless apps, variable load | Seconds to minutes |
| **VPA** | Changes CPU/memory requests | Stateful apps, consistent load | Minutes (requires restart) |
| **CA** | Changes node count | When pods can't schedule | 3-5 minutes |

---

## Chapter 2: Horizontal Pod Autoscaler (HPA)

### How HPA Works

```
Metrics Server (CPU/Memory)
        │
        ▼
    HPA Controller ──▶ Adjust replicas
        │
        ▼
  ┌─────┴─────┐
  ▼           ▼
Pod-1       Pod-2    (Target: 50% CPU)
  │           │
  └─────┬─────┘
        ▼
   Desired: 4 pods
   Current: 2 pods
   Action: Scale up
```

### Installing Metrics Server

```bash
# Install metrics server (required for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods -A

# For Minikube
minikube addons enable metrics-server
```

### Basic HPA - CPU Based

```yaml
# hpa-cpu.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50  # Target 50% CPU
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policy:
      - type: Percent
        value: 50  # Remove 50% of pods at a time
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policy:
      - type: Percent
        value: 100  # Double pods
        periodSeconds: 15
      - type: Pods
        value: 4    # Or add 4 pods
        periodSeconds: 15
      selectPolicy: Max  # Use the more aggressive policy
```

### Advanced HPA - Multiple Metrics

```yaml
# hpa-advanced.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
  # CPU-based scaling
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  
  # Memory-based scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  
  # Custom metric - requests per second
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100
  
  # Custom metric - queue length
  - type: Object
    object:
      metric:
        name: queue_length
      describedObject:
        apiVersion: v1
        kind: Service
        name: myapp-queue
      target:
        type: Value
        value: 1000
```

### HPA with Custom Metrics (Prometheus)

**Install Prometheus Adapter:**

```bash
# Install Prometheus adapter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://prometheus-server.monitoring.svc \
  --set rules.custom[0].seriesQuery='http_requests_total{namespace!="",pod!=""}' \
  --set rules.custom[0].resources="{namespace, pod}" \
  --set rules.custom[0].name.upas='{replica=0}' \
  --set rules.custom[0].metricsQuery='sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

**HPA using Prometheus metrics:**

```yaml
# hpa-prometheus.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-prometheus-hpa
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
  # Scale based on custom metric from Prometheus
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 500  # Scale when >500 req/sec per pod
  
  # Scale based on RabbitMQ queue depth
  - type: External
    external:
      metric:
        name: rabbitmq_queue_messages
        selector:
          matchLabels:
            queue: orders
      target:
        type: AverageValue
        averageValue: 1000
```

### Testing HPA

```bash
# 1. Deploy app with resource requests
kubectl apply -f deployment.yaml

# 2. Create HPA
kubectl apply -f hpa-cpu.yaml

# 3. Watch HPA status
kubectl get hpa -w

# 4. Generate load (using hey or wrk)
kubectl run load-generator --image=busybox --rm -it -- /bin/sh -c '
  while true; do
    wget -q -O- http://myapp-service/hello
  done'

# 5. Watch scaling
kubectl get pods -w
# myapp-7d8f9c5b6-abc12   Running
# myapp-7d8f9c5b6-def34   Running
# myapp-7d8f9c5b6-ghi56   Pending
# myapp-7d8f9c5b6-ghi56   Running

# 6. Check HPA events
kubectl describe hpa myapp-hpa
# Events:
#   Normal   SuccessfulRescale  30s   New size: 5; reason: cpu utilization above target
```

### Spring Boot Performance Tuning for HPA

```yaml
# deployment-with-hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            cpu: "500m"      # Minimum guaranteed
            memory: "512Mi"
          limits:
            cpu: "2000m"     # Maximum allowed (for burst)
            memory: "1Gi"
        env:
        # JVM tuning for container awareness
        - name: JAVA_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
        
        # Graceful shutdown for HPA scale down
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

---

## Chapter 3: Vertical Pod Autoscaler (VPA)

### When to Use VPA

**Problem:** Finding the right resource requests is hard!

```yaml
# Too low → OOMKilled
resources:
  requests:
    memory: "256Mi"  # App uses 300MB → Crash!

# Too high → Wasted resources
resources:
  requests:
    memory: "2Gi"    # App uses 300MB → 1.7GB wasted
```

**VPA Solution:** Automatically adjusts requests based on actual usage.

### Installing VPA

```bash
# Download VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA
./hack/vpa-up.sh

# Verify
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-xxx    1/1     Running
# vpa-recommender-xxx             1/1     Running
# vpa-updater-xxx                 1/1     Running
```

### VPA Configuration

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: myapp
  
  updatePolicy:
    updateMode: "Auto"  # Automatically update pods
    # Options:
    # "Off" - Only provide recommendations
    # "Initial" - Set only at pod creation
    # "Recreate" - Evict and recreate pods
    # "Auto" - Recreate if recommended differs significantly
  
  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      minAllowed:
        cpu: "100m"
        memory: "256Mi"
      maxAllowed:
        cpu: "2"
        memory: "4Gi"
      controlledResources: ["cpu", "memory"]
      controlledValues: "RequestsAndLimits"
```

### VPA Recommendations

```bash
# Check VPA recommendations
kubectl get vpa myapp-vpa -o yaml

# Output:
status:
  conditions:
  - status: "True"
    type: RecommendationProvided
  recommendation:
    containerRecommendations:
    - containerName: myapp
      lowerBound:
        cpu: "250m"
        memory: "262144k"
      target:
        cpu: "500m"
        memory: "524288k"  # 512Mi
      uncappedTarget:
        cpu: "500m"
        memory: "524288k"
      upperBound:
        cpu: "1"
        memory: "1048576k"

# Apply recommendations manually (if updateMode: "Off")
kubectl patch deployment myapp -p '{"spec":{"template":{"spec":{"containers":[{"name":"myapp","resources":{"requests":{"cpu":"500m","memory":"512Mi"}}}]}}}}'
```

### VPA Best Practices

```yaml
# vpa-production.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: myapp
  
  updatePolicy:
    updateMode: "Auto"
    # Don't evict pods too aggressively
    minReplicas: 2  # Keep at least 2 replicas
  
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["cpu", "memory"]
      # Don't scale below these values
      minAllowed:
        cpu: "100m"
        memory: "128Mi"
      # Don't exceed these values
      maxAllowed:
        cpu: "4"
        memory: "8Gi"
```

### HPA + VPA Together

⚠️ **Warning:** Don't use HPA and VPA on same CPU/memory metrics!

```yaml
# ✅ GOOD: HPA for CPU, VPA for memory only
# HPA scales on custom metric (requests per second)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second

---
# VPA for memory only
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  resourcePolicy:
    containerPolicies:
    - controlledResources: ["memory"]  # Only memory
      controlledValues: "RequestsOnly"
```

---

## Chapter 4: Cluster Autoscaler

### What is Cluster Autoscaler?

**Problem:** HPA adds pods, but no node capacity = pending pods

```
Pod created → No node capacity → Pending forever → Manual intervention needed
```

**Solution:** Cluster Autoscaler adds nodes automatically

```bash
# Pending pods trigger node addition
kubectl get pods
# myapp-abc12   Pending   0/1 nodes available

# Cluster Autoscaler adds node
# New node joins cluster
# Pod schedules successfully
```

### AWS EKS Cluster Autoscaler

```yaml
# cluster-autoscaler-values.yaml
autoDiscovery:
  clusterName: my-eks-cluster
  enabled: true

awsRegion: us-east-1

rbac:
  serviceAccount:
    create: true
    name: cluster-autoscaler
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/cluster-autoscaler-role

extraArgs:
  balance-similar-node-groups: true
  skip-nodes-with-local-storage: false
  skip-nodes-with-system-pods: false
  scale-down-delay-after-add: 10m
  scale-down-unneeded-time: 10m
  max-node-provision-time: 15m
```

**Install Cluster Autoscaler:**

```bash
# Create IAM policy
cat > cluster-autoscaler-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeLaunchTemplateVersions"
      ],
      "Resource": ["*"]
    }
  ]
}
EOF

aws iam create-policy --policy-name cluster-autoscaler-policy --policy-document file://cluster-autoscaler-policy.json

# Install with Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  -f cluster-autoscaler-values.yaml

# Verify
kubectl logs -f -n kube-system deployment/cluster-autoscaler
```

### GKE Cluster Autoscaler

```bash
# Enable autoscaling on node pool
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --zone=us-central1

# Or create new node pool with autoscaling
gcloud container node-pools create my-pool \
  --cluster=my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --num-nodes=3
```

### AKS Cluster Autoscaler

```bash
# Enable cluster autoscaler on existing cluster
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10

# Update node pool
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name mypool \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

### Testing Cluster Autoscaler

```bash
# 1. Deploy HPA that will scale up
kubectl apply -f deployment.yaml
kubectl apply -f hpa-cpu.yaml

# 2. Generate load
kubectl run load-generator --image=busybox -- /bin/sh -c 'while true; do wget -q -O- http://myapp-service; done'

# 3. Watch pods scale
kubectl get pods -w

# 4. Watch nodes scale
kubectl get nodes -w

# 5. Check cluster autoscaler logs
kubectl logs -n kube-system deployment/cluster-autoscaler --tail=50

# Expected output:
# "Scale up: group eks-my-cluster-node-group ready to scale to 5 nodes"
```

---

## Chapter 5: Pod Disruption Budgets (PDB)

### What is a PDB?

**Problem:** During maintenance (node upgrades, scaling down), you might lose too many pods.

**Solution:** PDB ensures minimum available pods during voluntary disruptions.

```yaml
# pdb-min-available.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  minAvailable: 2  # At least 2 pods always running
  selector:
    matchLabels:
      app: myapp

---
# Alternative: max unavailable
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  maxUnavailable: 1  # At most 1 pod can be down
  selector:
    matchLabels:
      app: myapp
```

### PDB Examples

```yaml
# For statefulset with 3 replicas
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
spec:
  minAvailable: 2  # Keep at least 2 (quorum)
  selector:
    matchLabels:
      app: postgres

---
# For critical service
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-api-pdb
spec:
  minAvailable: 3  # Keep 3 pods always
  selector:
    matchLabels:
      app: critical-api

---
# For batch processing
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: batch-worker-pdb
spec:
  maxUnavailable: "25%"  # Remove 25% of workers at a time
  selector:
    matchLabels:
      app: batch-worker
```

### Testing PDB

```bash
# Create PDB
kubectl apply -f pdb.yaml

# Check PDB status
kubectl get pdb
# NAME          MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# myapp-pdb     2               N/A               1                     10s

# Try to drain node (will be blocked by PDB)
kubectl drain node-1 --ignore-daemonsets

# Output:
# error when evicting pod "myapp-abc12": Cannot evict pod as it would violate the pod's disruption budget
```

---

## Chapter 6: Chaos Engineering

### What is Chaos Engineering?

**Analogy - Fire drills:**
- **Without chaos** = No fire drills (first fire = panic)
- **With chaos** = Regular fire drills (everyone knows what to do)

**Principles:**
1. Define steady state (normal behavior)
2. Hypothesize (what if X fails?)
3. Run experiment (cause failure)
4. Verify hypothesis
5. Improve system

### Installing Chaos Mesh

```bash
# Install Chaos Mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock

# Access Chaos Dashboard
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
# http://localhost:2333
```

### Pod Chaos Experiments

**Pod Kill:**

```yaml
# pod-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-example
spec:
  action: pod-kill
  mode: fixed
  value: "1"  # Kill 1 pod
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  scheduler:
    cron: "@every 30m"  # Run every 30 minutes
  duration: "5m"
```

**Pod Failure:**

```yaml
# pod-failure.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-failure-example
spec:
  action: pod-failure
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  duration: "2m"  # Pods unavailable for 2 minutes
  scheduler:
    cron: "@hourly"
```

### Network Chaos

**Network Delay:**

```yaml
# network-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-example
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  delay:
    latency: "100ms"
    correlation: "25"
    jitter: "10ms"
  duration: "5m"
  scheduler:
    cron: "*/30 * * * *"  # Every 30 minutes
```

**Network Loss:**

```yaml
# network-loss.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-loss-example
spec:
  action: loss
  mode: fixed
  value: "2"  # Affect 2 pods
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  loss:
    loss: "25"  # 25% packet loss
    correlation: "50"
  duration: "10m"
```

**Network Partition:**

```yaml
# network-partition.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-partition-example
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  direction: both
  target:
    mode: fixed
    value: "1"
    selector:
      namespaces:
        - production
      labelSelectors:
        app: database
  duration: "5m"
```

### CPU Stress

```yaml
# cpu-stress.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-example
spec:
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  stressors:
    cpu:
      workers: 2  # 2 CPU workers
      load: 80    # 80% CPU load
  duration: "10m"
```

### Memory Stress

```yaml
# memory-stress.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress-example
spec:
  mode: fixed
  value: "1"
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  stressors:
    memory:
      size: "500MB"  # Allocate 500MB
      workers: 1
  duration: "5m"
```

### IO Stress

```yaml
# io-stress.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-stress-example
spec:
  action: latency
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: myapp
  delay:
    duration: "100ms"
    percent: 50  # 50% of operations delayed
  volumePath: /var/lib/mysql
  path: /var/lib/mysql/**/*  # Pattern for files
  duration: "10m"
```

### Running Chaos Experiments

```bash
# 1. Apply chaos experiment
kubectl apply -f pod-kill.yaml

# 2. Watch chaos status
kubectl get podchaos -w
# NAME               STATE      AGE
# pod-kill-example   Running    10s
# pod-kill-example   Completed  5m

# 3. Check chaos events
kubectl describe podchaos pod-kill-example

# 4. Monitor application during chaos
kubectl get pods -w
# myapp-abc12   Running
# myapp-abc12   Terminating  (killed by chaos)
# myapp-def34   Running      (new pod created)

# 5. Check chaos dashboard
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

### Chaos Experiment Best Practices

```yaml
# production-chaos.yaml - Safe chaos for production
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: production-chaos-schedule
spec:
  schedule: "0 14 * * 2"  # Tuesday 2 PM (low traffic)
  historyLimit: 5
  
  podChaos:
    action: pod-kill
    mode: fixed
    value: "1"  # Only kill 1 pod
    selector:
      namespaces:
        - production
      labelSelectors:
        app: myapp
        chaos-enabled: "true"  # Opt-in only
  
  # Don't run during critical periods
  excludingDates:
    - "2024-12-24"
    - "2024-12-25"
    - "2025-01-01"
```

---

## Chapter 7: Advanced Deployment Strategies

### Blue-Green Deployment

```yaml
# blue-green-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-rollout
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: myapp-active
      previewService: myapp-preview
      autoPromotionEnabled: false  # Manual promotion
      autoPromotionSeconds: 300    # Or auto after 5 min
      scaleDownDelaySeconds: 60    # Wait before scaling down old
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: app
        image: myapp:{{ .Values.image.tag }}
```

**Blue-Green with Argo Rollouts:**

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Apply rollout
kubectl apply -f blue-green-deployment.yaml

# Watch rollout
kubectl argo rollouts get rollout myapp-rollout --watch

# Promote new version
kubectl argo rollouts promote myapp-rollout
```

### Canary Deployment

```yaml
# canary-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp-canary
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10   # Send 10% traffic to canary
      - pause: {duration: 5m}  # Monitor for 5 minutes
      - setWeight: 25   # Increase to 25%
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 20m}
      - setWeight: 100  # Full rollout
      - pause: {duration: 30m}  # Monitor before finishing
      
      # Traffic routing
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress
          additionalIngressAnnotations:
            nginx.ingress.kubernetes.io/canary: "true"
            nginx.ingress.kubernetes.io/canary-weight: "{{ .Weight }}"
      
      # Analysis for automatic rollback
      analysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: myapp-canary
        startingStep: 2  # Start analysis at step 2 (25% weight)
```

### Analysis Template for Canary

```yaml
# analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    interval: 30s
    successCondition: result[0] >= 0.95  # 95% success rate
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[1m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[1m]))
```

---

## Chapter 8: Disaster Recovery

### Backup Strategies

```yaml
# Velero for cluster backup
velero install \
  --provider aws \
  --bucket my-cluster-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero

# Schedule daily backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 72h \
  --include-namespaces production,staging \
  --exclude-resources events,secrets

# Manual backup
velero backup create production-backup \
  --include-namespaces production \
  --snapshot-volumes
```

### Disaster Recovery Plan

```yaml
# dr-plan.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: disaster-recovery-plan
data:
  plan: |
    RTO: 4 hours (Recovery Time Objective)
    RPO: 24 hours (Recovery Point Objective)
    
    Steps:
    1. Detect disaster (monitoring alert)
    2. Declare disaster (PagerDuty incident)
    3. Activate DR environment
    4. Restore from latest backup
    5. Redirect traffic
    6. Validate system health
    
    Contacts:
    - Primary: oncall@mycompany.com
    - Secondary: platform-team@mycompany.com
    
    Runbook: https://wiki.mycompany.com/DR
```

---

## Chapter 9: Complete Resilience Configuration

### Production-Ready Deployment

```yaml
# complete-resilient-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime
    type: RollingUpdate
  
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        
        # Resource management
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "2000m"
            memory: "1Gi"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
        
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 20"]
        
        # Security
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
---
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
---
# PDB
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
---
# Service with session affinity
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  ports:
  - port: 80
    targetPort: 8080
```

---

## Summary: Scaling & Resilience Checklist

| Component | Tool | Status |
|-----------|------|--------|
| **Horizontal Scaling** | HPA | ✅ |
| **Vertical Scaling** | VPA | ✅ |
| **Infrastructure Scaling** | Cluster Autoscaler | ✅ |
| **Availability Protection** | PDB | ✅ |
| **Chaos Testing** | Chaos Mesh | ✅ |
| **Advanced Deployment** | Argo Rollouts | ✅ |
| **Disaster Recovery** | Velero | ✅ |

## Practice Exercises

### Exercise 1: Configure HPA
Deploy app with HPA based on CPU and test with load generator.

### Exercise 2: VPA Recommendation
Install VPA in "Off" mode and review recommendations.

### Exercise 3: Chaos Experiment
Run pod-kill experiment and verify app self-heals.

### Exercise 4: PDB Testing
Create PDB and try to drain node (observe blocking).

### Exercise 5: Canary Deployment
Set up Argo Rollouts with canary strategy and analysis.

## Next Steps

After mastering scaling and resilience, you're ready for:
- **Part 10: Production Playbook** - Complete deployment guide

---