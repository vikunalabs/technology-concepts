# Part 9: Scaling & Resilience — Building Self-Healing Systems

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Advanced
> **Prerequisites:** Completed Parts 1–8, or comfortable with Kubernetes workloads, Helm, Prometheus, and resource management
> **Time to complete:** 5–6 hours
> **What you'll learn:** Horizontal Pod Autoscaler (HPA) with CPU and custom metrics, Vertical Pod Autoscaler (VPA), Cluster Autoscaler for all three cloud providers, Pod Disruption Budgets, chaos engineering with Chaos Mesh, advanced deployment strategies (blue-green and canary with Argo Rollouts), and disaster recovery planning with Velero

---

## What This Part Covers

A system that requires a human to intervene during a traffic spike, a node failure, or a rolling update is fragile by design. The goal of this part is to remove that requirement — to build systems that respond to load automatically, survive infrastructure failures gracefully, and roll out changes with zero downtime regardless of what goes wrong.

This requires thinking at three levels simultaneously: scaling pods to match demand, scaling nodes to give pods somewhere to run, and protecting the application from disruptions that happen during normal operations like maintenance and upgrades. All three levels working together produce a system that is genuinely self-healing.

---

## Chapter 1: Scaling Overview — Three Levels, One System

### The Three Scaling Dimensions

Kubernetes scaling operates at three distinct levels, and all three are needed for a complete solution:

**Horizontal Pod Autoscaler (HPA):** Adds or removes pod replicas in response to load. Fast — acts within seconds to minutes. Bounded by the capacity of existing nodes.

**Vertical Pod Autoscaler (VPA):** Adjusts the CPU and memory requests of individual pods. Slower — typically requires a pod restart. Addresses right-sizing, not demand spikes.

**Cluster Autoscaler (CA):** Adds or removes nodes from the cluster. Slowest — provisioning a new cloud instance takes 3–5 minutes. Addresses the case where HPA needs more pods but no node has capacity.

The three levels interlock:

```
Traffic spike hits
      │
      ▼
HPA detects high CPU → wants to add pods
      │
      ├─ Nodes have capacity → pods scheduled immediately ✓
      │
      └─ Nodes are full → pods sit Pending
               │
               ▼
         Cluster Autoscaler detects Pending pods
               │
               ▼
         CA provisions new node (3–5 min)
               │
               ▼
         Pods scheduled on new node ✓
```

VPA operates independently — it watches actual resource usage over time and adjusts requests to better match reality. Used alongside HPA on custom metrics (not CPU, where they conflict — covered in Chapter 3).

### Speed vs Coverage

| Mechanism | Reaction time | What it handles |
|-----------|--------------|----------------|
| HPA | 15 seconds – 3 minutes | Traffic spikes within existing node capacity |
| VPA (Auto mode) | Minutes (requires pod restart) | Incorrectly sized resource requests |
| Cluster Autoscaler | 3–5 minutes | Demand exceeding cluster capacity |

The implication for capacity planning: always run with some buffer of free node capacity. If nodes are at 90% capacity and traffic spikes, HPA can't add pods until CA provisions new nodes — and that takes 3–5 minutes. A 20–30% free capacity buffer lets HPA respond instantly to spikes while CA catches up.

---

## Chapter 2: Horizontal Pod Autoscaler

### How HPA Works

HPA runs as a control loop in the Kubernetes control plane. Every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period`), it:

1. Queries the Metrics API for current resource usage of the target Deployment
2. Calculates the desired replica count: `desiredReplicas = ceil(currentReplicas × (currentMetric / targetMetric))`
3. If the desired count differs from the current count, patches the Deployment's `spec.replicas`

The critical dependency: HPA calculates CPU utilisation as `actualCPU / requestedCPU`. If a pod has no `resources.requests.cpu` set, the denominator is zero — HPA cannot compute a ratio and refuses to scale. **Every pod that HPA manages must have CPU requests set.**

### Installing Metrics Server

HPA requires the Metrics Server to read CPU and memory usage from pods:

```bash
# Check if already installed
kubectl top pods -A
# If this returns data, Metrics Server is running.
# If it returns "Error from server (ServiceUnavailable)", install it.

# Minikube
minikube addons enable metrics-server

# All other clusters
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

### Basic CPU-Based HPA

```yaml
# hpa-basic.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp            # The Deployment to scale

  minReplicas: 3           # Never scale below 3 (availability floor)
  maxReplicas: 20          # Never scale above 20 (cost ceiling)

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        # Scale out when average CPU across all pods exceeds 60%.
        # WHY 60% not 80%: scaling takes time. At 80%, you're already under pressure.
        # At 60%, you have headroom to scale before users are impacted.
        averageUtilization: 60
```

### Controlling Scale Speed with `behavior`

The default HPA scales up quickly and scales down quickly — which causes "flapping." CPU spikes briefly, HPA adds 5 pods, traffic normalises, HPA removes 5 pods, CPU spikes again. Pods start and stop constantly, wasting resources and causing unnecessary disruption.

The `behavior` field controls how fast HPA is allowed to change replica counts:

```yaml
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

  behavior:
    scaleUp:
      # Scale up aggressively — don't make users wait
      stabilizationWindowSeconds: 0    # No stabilisation delay — act immediately
      policies:
      - type: Percent
        value: 100                     # Allow doubling replica count each interval
        periodSeconds: 15
      - type: Pods
        value: 4                       # Or add at most 4 pods per interval
        periodSeconds: 15
      selectPolicy: Max                # Use whichever policy adds the most pods

    scaleDown:
      # Scale down conservatively — avoid flapping
      # stabilizationWindowSeconds: 300 means:
      # HPA looks at the MAXIMUM desired replica count over the last 5 minutes.
      # Only scale down if the maximum desired count over that window was lower.
      # This prevents scaling down during brief traffic lulls between spikes.
      stabilizationWindowSeconds: 300   # 5-minute stabilisation window
      policies:
      - type: Percent
        value: 25                       # Remove at most 25% of replicas per interval
        periodSeconds: 60
      selectPolicy: Min                 # Use whichever policy removes the fewest pods
```

**Why `stabilizationWindowSeconds: 300` for scale-down:** Without it, if traffic drops for 30 seconds, HPA immediately scales down. Traffic returns 2 minutes later and HPA scales back up. Your pods are constantly churning. The 5-minute window says: "only scale down if we've genuinely been below target for 5 consecutive minutes" — preventing most flapping.

### Advanced HPA: Multiple Metrics

HPA evaluates all configured metrics and uses the one that demands the highest replica count. This is conservative — it never scales down if any metric says you need more pods:

```yaml
spec:
  minReplicas: 3
  maxReplicas: 20

  metrics:
  # Scale on CPU utilisation
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60

  # Also scale on memory (though memory doesn't decrease when load drops —
  # use memory scaling carefully, only for genuinely memory-bound workloads)
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70

  # Also scale on custom application metric (requests per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second    # From Prometheus Adapter
      target:
        type: AverageValue
        averageValue: 500                 # Scale out when > 500 req/sec per pod
```

### Custom Metrics with Prometheus Adapter

CPU tells you something is consuming compute, but it doesn't tell you *why* or whether it's the right signal to scale on. A request-processing service should scale on requests per second. A queue consumer should scale on queue depth. These are application-level metrics that come from Prometheus.

The Prometheus Adapter (installed in Part 8) bridges Prometheus metrics into Kubernetes' custom metrics API. Here is the working ConfigMap mapping HTTP request rate to a custom metric:

```yaml
# prometheus-adapter-values.yaml (passed to helm install)
rules:
  custom:
  # Rule: map http_server_requests_seconds_count to http_requests_per_second
  - seriesQuery: 'http_server_requests_seconds_count{namespace!="",pod!=""}'
    # Tell the adapter how to map Prometheus labels to Kubernetes resources
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    name:
      matches: "^http_server_requests_seconds_count$"
      as: "http_requests_per_second"
    # The PromQL that computes the metric value per pod
    metricsQuery: |
      sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)

  # Rule: RabbitMQ queue depth for queue-based scaling
  - seriesQuery: 'rabbitmq_queue_messages{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    name:
      matches: "^rabbitmq_queue_messages$"
      as: "rabbitmq_queue_depth"
    metricsQuery: |
      max(<<.Series>>{<<.LabelMatchers>>,queue="order-processing"}) by (<<.GroupBy>>)
```

```bash
# Apply adapter configuration
helm upgrade prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  -f prometheus-adapter-values.yaml

# Verify the metric is available to Kubernetes
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq '.resources[].name'
# "pods/http_requests_per_second"
# "pods/rabbitmq_queue_depth"
```

**HPA on RabbitMQ queue depth** — scale consumer pods when messages are backing up:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-processor-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor

  minReplicas: 2
  maxReplicas: 50

  metrics:
  - type: Pods
    pods:
      metric:
        name: rabbitmq_queue_depth
      target:
        type: AverageValue
        # Target: each pod handles at most 100 queued messages
        # With 500 messages and 5 pods = 100 each → no scaling
        # With 1000 messages and 5 pods = 200 each → scale to 10 pods
        averageValue: "100"

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0   # Act immediately when queue grows
      policies:
      - type: Pods
        value: 10                      # Add up to 10 pods at once
        periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 minutes before scaling down
```

### Testing HPA

```bash
# Apply the HPA
kubectl apply -f hpa.yaml

# Watch HPA status in real time
kubectl get hpa -n production -w
# NAME        REFERENCE         TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
# myapp-hpa   Deployment/myapp  8%/60%           3         20        3          5m
# myapp-hpa   Deployment/myapp  62%/60%          3         20        3          6m  ← threshold crossed
# myapp-hpa   Deployment/myapp  62%/60%          3         20        5          7m  ← scaled up

# Generate load to trigger scaling
kubectl run load-generator \
  --image=busybox \
  --rm -it \
  --restart=Never \
  -n production \
  -- /bin/sh -c \
  'while true; do
    wget -q -O- http://myapp-service:8080/api/orders
   done'

# Observe HPA responding (separate terminal)
watch kubectl get hpa,pods -n production

# Describe HPA for detailed status and recent events
kubectl describe hpa myapp-hpa -n production
# Events:
#   SuccessfulRescale  Scaled up replica count to 8
#   SuccessfulRescale  Scaled up replica count to 15
#   SuccessfulRescale  Scaled down replica count to 5
```

### Spring Boot JVM Tuning for HPA

When HPA scales down and a pod is terminated, Spring Boot needs to finish in-flight requests before exiting. Without graceful shutdown, terminating pods drop active requests causing errors during scale-down events.

Ensure these settings are in place (covered in Part 2, but critical for HPA):

```yaml
# application.yml — required for graceful scale-down
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
  shutdown: graceful
```

```yaml
# deployment.yaml — ensures enough time for graceful shutdown
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        env:
        # JVM must know its container limits — essential for HPA CPU calculations
        - name: JAVA_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
```

---

## Chapter 3: Vertical Pod Autoscaler (VPA)

### Why Manual Resource Sizing Is Hard

Part 2 described the process: run under load, observe `docker stats`, set requests based on observations. This works once, but the right values change as your application evolves and as traffic patterns shift. A service that needed 256Mi of memory six months ago may need 512Mi after new features shipped, or only 128Mi after a memory optimisation.

VPA automates this ongoing calibration. It observes actual CPU and memory usage over time and either recommends better values or automatically applies them. Starting in `Off` mode (recommendations only) and moving to `Auto` when you trust it is the safe adoption path.

### VPA Update Modes

| Mode | What it does | When to use |
|------|-------------|-------------|
| `Off` | Observes usage, provides recommendations, changes nothing | Always start here — understand recommendations before applying |
| `Initial` | Sets requests only when pods are created (not for existing pods) | Conservative — updates happen during natural restarts |
| `Recreate` | Evicts pods when recommendations differ significantly from current values | Acceptable in staging; disruptive in production unless you have PDBs |
| `Auto` | Same as `Recreate` currently; may become in-place update when K8s supports it | Use in production only with PDBs ensuring availability during evictions |

### Complete VPA Configuration

```yaml
# vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  updatePolicy:
    updateMode: "Off"    # Start here — observe before acting

  resourcePolicy:
    containerPolicies:
    - containerName: myapp
      # Guard rails: VPA will never recommend values outside these bounds.
      # Without minAllowed, VPA might recommend 10m CPU which is too low for JVM.
      # Without maxAllowed, VPA might recommend 16Gi memory for a spike.
      minAllowed:
        cpu: "100m"
        memory: "256Mi"
      maxAllowed:
        cpu: "4"
        memory: "4Gi"
      # controlledResources: limit what VPA manages.
      # Setting only memory lets HPA manage CPU independently (avoiding the conflict).
      controlledResources: ["memory"]
```

Install VPA:
```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-5d9c7f6b8-xj8mk   1/1   Running
# vpa-recommender-6b8c7d9f7-mklpq             1/1   Running
# vpa-updater-7d9b4f9c6-2xmpq                 1/1   Running
```

### Reading VPA Recommendations

After running for several hours (ideally days, to capture realistic usage patterns):

```bash
kubectl get vpa myapp-vpa -n production -o yaml
```

```yaml
# Status section of the VPA output:
status:
  recommendation:
    containerRecommendations:
    - containerName: myapp
      lowerBound:           # Conservative lower bound — safe minimum
        cpu: "100m"
        memory: "320Mi"
      target:               # The recommended value — what VPA will set in Auto mode
        cpu: "250m"
        memory: "448Mi"
      upperBound:           # Upper bound — might be needed under high load
        cpu: "680m"
        memory: "756Mi"
      uncappedTarget:       # What VPA would recommend without your min/max bounds
        cpu: "250m"
        memory: "448Mi"
```

The `target` values are what VPA considers optimal based on observed usage. If your current requests are `cpu: 500m, memory: 256Mi` and VPA recommends `cpu: 250m, memory: 448Mi`, you're over-provisioning CPU and under-provisioning memory. Update your Helm values accordingly.

### The HPA + VPA Conflict

Running HPA and VPA together on the same CPU metric causes them to fight:

```
Traffic increases → HPA wants to add pods (CPU is high)
                 → VPA sees high CPU and increases CPU requests
                 → Higher requests → HPA's utilisation calculation changes
                 → HPA removes pods it just added
                 → VPA's recommendation changes again
                 → Infinite loop of contradictory adjustments
```

The solution is separation of concerns:

```
HPA manages: replica count (based on requests-per-second, not CPU)
VPA manages: memory requests only (CPU is excluded via controlledResources)
```

```yaml
# HPA: scale on application metric, not CPU
metrics:
- type: Pods
  pods:
    metric:
      name: http_requests_per_second
    target:
      type: AverageValue
      averageValue: "500"
# No CPU metric in HPA

# VPA: manage memory only, leave CPU alone
resourcePolicy:
  containerPolicies:
  - containerName: myapp
    controlledResources: ["memory"]   # CPU excluded — HPA won't fight VPA
```

If you are not using custom metrics and can only scale on CPU, use HPA *or* VPA — not both. VPA's `Off` mode for recommendations alongside HPA for scaling is acceptable since `Off` mode never changes running pods.


## Chapter 4: Cluster Autoscaler — Adding and Removing Nodes

### Why Nodes Need to Scale Too

HPA can add pods indefinitely — but pods need nodes to run on. When all nodes are full and HPA wants to schedule more pods, those pods sit in `Pending` state. The Cluster Autoscaler (CA) watches for pending pods that can't be scheduled due to insufficient resources and provisions new nodes to accommodate them. When load drops and nodes become underutilised, CA removes them to reduce cost.

The key distinction between CA and HPA: HPA responds to workload metrics (CPU, RPS). CA responds to scheduling failures — it only acts when pods can't be placed.

### AWS EKS — IAM Policy and ASG Setup

CA needs IAM permissions to modify Auto Scaling Groups and describe EC2 resources. The recommended approach is IRSA:

```json
// ca-iam-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:DescribeTags",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup",
        "ec2:DescribeImages",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:GetInstanceTypesFromInstanceRequirements",
        "eks:DescribeNodegroup"
      ],
      "Resource": ["*"]
    }
  ]
}
```

```bash
# Create the IAM policy
aws iam create-policy \
  --policy-name ClusterAutoscalerPolicy \
  --policy-document file://ca-iam-policy.json

# Create IRSA role (assumes OIDC provider already configured — see Part 7)
eksctl create iamserviceaccount \
  --cluster=my-cluster \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::123456789:policy/ClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

**Tag your node group's Auto Scaling Group** — CA uses these tags to discover manageable ASGs:

```bash
aws autoscaling create-or-update-tags \
  --tags \
    ResourceId=my-cluster-nodegroup-asg,ResourceType=auto-scaling-group,\
    Key=k8s.io/cluster-autoscaler/enabled,Value=true,PropagateAtLaunch=true \
    ResourceId=my-cluster-nodegroup-asg,ResourceType=auto-scaling-group,\
    Key=k8s.io/cluster-autoscaler/my-cluster,Value=owned,PropagateAtLaunch=true
```

**Install CA via Helm:**

```bash
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm repo update

helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=my-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler \
  --set extraArgs.scale-down-delay-after-add=10m \
  --set extraArgs.scale-down-unneeded-time=10m \
  --set extraArgs.skip-nodes-with-local-storage=false \
  --set extraArgs.expander=least-waste

# Verify CA is running and discovered your node group
kubectl logs -n kube-system deployment/cluster-autoscaler | grep "Found ASG"
```

**Key CA configuration options:**

| Flag | Default | Meaning |
|------|---------|---------|
| `scale-down-delay-after-add` | 10m | Wait this long after scaling up before considering scale-down |
| `scale-down-unneeded-time` | 10m | Node must be underutilised for this long before removing |
| `scale-down-utilization-threshold` | 0.5 | Node considered underutilised if CPU+memory requests < 50% |
| `expander` | `random` | How to choose which node group to expand: `least-waste`, `priority`, `price` |
| `skip-nodes-with-system-pods` | `true` | Don't remove nodes with kube-system pods (avoid CA evicting itself) |

### GKE — Built-In Autoscaling

GKE has cluster autoscaling built in, controlled via `gcloud` rather than a separate Helm chart:

```bash
# Enable autoscaling on an existing node pool
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \
  --max-nodes=20 \
  --node-pool=default-pool \
  --zone=us-central1-a

# For regional clusters (nodes across multiple zones)
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=3 \        # Per zone
  --max-nodes=10 \       # Per zone
  --zone=us-central1     # Regional cluster

# Enable autoprovisioning (automatically creates new node pools for unusual workloads)
gcloud container clusters update my-cluster \
  --enable-autoprovisioning \
  --max-cpu=100 \
  --max-memory=256 \
  --zone=us-central1-a
```

### AKS — Azure Cluster Autoscaler

```bash
# Enable autoscaling on a node pool
az aks nodepool update \
  --resource-group myresourcegroup \
  --cluster-name my-cluster \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20

# Or at cluster creation
az aks create \
  --resource-group myresourcegroup \
  --name my-cluster \
  --node-count 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20
```

### Testing the Full Scaling Chain

```bash
# 1. Start with 3 nodes, confirm they're near capacity
kubectl top nodes
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# node-1     1800m        90%    6Gi             75%
# node-2     1900m        95%    6.5Gi           81%
# node-3     1750m        87%    5.8Gi           72%
# Nodes are ~90% full — HPA will trigger CA

# 2. Generate sustained load
kubectl run sustained-load \
  --image=busybox --rm -it --restart=Never \
  -n production \
  -- /bin/sh -c 'while true; do wget -q -O- http://myapp-service:8080/api/orders; done'

# 3. Watch the full chain in separate terminals:
# Terminal 1: HPA scaling pods
kubectl get hpa -n production -w

# Terminal 2: Pods going Pending (nodes full)
kubectl get pods -n production -w | grep Pending

# Terminal 3: CA adding nodes
kubectl get nodes -w
# NAME       STATUS     ROLES    AGE   VERSION
# node-1     Ready      <none>   1d    v1.28.0
# node-2     Ready      <none>   1d    v1.28.0
# node-3     Ready      <none>   1d    v1.28.0
# node-4     NotReady   <none>   10s   v1.28.0   ← CA added this node
# node-4     Ready      <none>   3m    v1.28.0   ← now ready, pods scheduled

# 4. View CA logs
kubectl logs -n kube-system deployment/cluster-autoscaler | \
  grep -E "Scale up|Scale down|Expanding"
```

---

## Chapter 5: Pod Disruption Budgets — Protecting Against Maintenance

### The Maintenance Problem

Kubernetes nodes are regularly drained for:
- Cluster upgrades (control plane upgrade → node upgrades)
- Node OS patches
- Manual maintenance
- Cloud provider events (scheduled instance retirement)

When you drain a node, Kubernetes evicts all pods from it. Without PDBs, Kubernetes evicts pods as fast as it can — potentially removing all replicas of a service simultaneously. The service goes down during maintenance that was supposed to be seamless.

A Pod Disruption Budget (PDB) tells Kubernetes the minimum number of pods that must remain available during any voluntary disruption. Node drains respect PDBs — if evicting a pod would violate the PDB, the drain pauses and waits until the constraint can be satisfied.

### `minAvailable` vs `maxUnavailable`

```yaml
# minAvailable: at least N pods must be available at all times
spec:
  minAvailable: 2        # Out of 5 replicas, can remove at most 3

# maxUnavailable: at most N pods can be unavailable at any time
spec:
  maxUnavailable: 1      # Out of 5 replicas, can remove at most 1

# Percentage form (dynamically scales with replica count)
spec:
  minAvailable: "60%"    # With 10 replicas: at least 6 must be available
  maxUnavailable: "25%"  # With 10 replicas: at most 2 can be unavailable
```

**When to use which:**
- `minAvailable` is better for databases and services with quorum requirements — you can express "always keep at least 2 database replicas for quorum"
- `maxUnavailable` is better for high-replica stateless services — "never take down more than 25% at once"

### Complete PDB Examples

**Stateless API service:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
  namespace: production
spec:
  # With 10 replicas: at most 2 removed at once → minimum 8 available
  # Maintenance takes longer but service stays healthy throughout
  maxUnavailable: "20%"
  selector:
    matchLabels:
      app: myapp
```

**PostgreSQL StatefulSet — quorum protection:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: production
spec:
  # With 3 replicas (1 primary + 2 replicas):
  # Must keep at least 2 available to maintain replication quorum.
  # Primary + 1 replica = quorum maintained.
  # If we're down to 2 and one is drained, we'd have 1 → no quorum → data risk.
  minAvailable: 2
  selector:
    matchLabels:
      app: postgres
```

**Critical frontend — no more than one pod down:**
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: frontend-pdb
  namespace: production
spec:
  maxUnavailable: 1     # Absolute value — never remove more than 1 at once
  selector:
    matchLabels:
      app: frontend
```

### Verifying PDB Status

```bash
kubectl get pdb -n production
# NAME          MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# myapp-pdb     N/A             20%               2                     5d
# postgres-pdb  2               N/A               1                     5d
# frontend-pdb  N/A             1                 1                     5d

# ALLOWED DISRUPTIONS: how many pods can be removed right now without violating the PDB.
# postgres-pdb shows 1: we have 3 pods, need to keep 2, so 1 can be removed.
# If a node drain would try to remove 2 postgres pods simultaneously, it pauses.
```

### Testing PDB Enforcement

```bash
# Attempt to drain a node — should pause if it would violate a PDB
kubectl drain node-2 \
  --ignore-daemonsets \
  --delete-emptydir-data

# If a PDB is violated, you'll see:
# error when evicting pods/"postgres-0" (will retry after 5s):
#   Cannot evict pod as it would violate the pod's disruption budget.

# The drain command retries every 5 seconds until either:
# 1. The constraint is satisfied (another pod comes up elsewhere)
# 2. You cancel with Ctrl+C

# Force drain (ignores PDB) — only use in emergencies:
kubectl drain node-2 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --disable-eviction    # Bypasses PDB — use with caution

# After maintenance, allow the node to accept pods again
kubectl uncordon node-2
```

---

## Chapter 6: Chaos Engineering — Fire Drills for Infrastructure

### Why Deliberately Break Things

You test your code before shipping. Why not test your infrastructure?

Every resilience feature — PDBs, graceful shutdown, HPA, health probes, retry logic — is only as reliable as the last time it was actually tested under realistic conditions. Without regular testing, you discover they don't work during a real incident, when it's the worst possible time to find out.

Chaos engineering deliberately injects failures in a controlled environment to verify that your resilience mechanisms work. The insight from Netflix's Chaos Monkey (the origin of modern chaos engineering) was: if you inject failures regularly, you're forced to fix them. Systems that face regular chaos become resilient. Systems that are never tested become fragile.

### The Five Principles of Chaos Engineering

1. **Define steady state:** Establish what "normal" looks like in measurable terms (error rate < 0.1%, P99 latency < 200ms, all pods healthy).
2. **Hypothesize:** What do you believe will happen when you inject this failure? ("If we kill one pod, HPA + probe-based health checking will replace it within 60 seconds.")
3. **Inject failure in a controlled way:** Kill one pod, add 100ms network latency to one service, stress one node to 90% CPU.
4. **Measure against steady state:** Does the system recover to steady state within your hypothesis window?
5. **Improve:** If not, fix the gap. If yes, increase the scope of the experiment.

### Installing Chaos Mesh

```bash
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update

helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-testing \
  --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock

kubectl get pods -n chaos-testing
# NAME                                        READY   STATUS
# chaos-controller-manager-6d9c7f6b8-xj8mk   1/1     Running
# chaos-daemon-abcd1                          1/1     Running   ← one per node
# chaos-daemon-abcd2                          1/1     Running
# chaos-dashboard-5d4b9f6c8-mklpq            1/1     Running

# Access the dashboard
kubectl port-forward svc/chaos-dashboard 2333:2333 -n chaos-testing
# http://localhost:2333
```

### Pod Chaos — Killing Pods

Before running any chaos experiment, ensure a PDB is in place to protect minimum availability:

```yaml
# pod-chaos-kill.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-one-pod
  namespace: production
spec:
  action: pod-kill          # Kill the pod (it will be recreated by the Deployment)
  # action: pod-failure     # Make the pod fail readiness checks (without killing it)

  mode: fixed               # Kill a fixed number of pods
  value: "1"                # Kill exactly 1 pod

  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp

  # Optional: run on a schedule
  # scheduler:
  #   cron: "@every 10m"

  # Duration: how long to run this experiment
  duration: "5m"
```

```bash
kubectl apply -f pod-chaos-kill.yaml

# Watch pod recovery
kubectl get pods -n production -w -l app=myapp
# myapp-7d9b4f-xj8mk   1/1   Running   0    10m
# myapp-7d9b4f-xj8mk   1/1   Terminating  0  10m   ← chaos killed it
# myapp-7d9b4f-abc12   0/1   Pending      0  2s    ← Deployment creating replacement
# myapp-7d9b4f-abc12   1/1   Running      0  25s   ← recovered ✓

# Was the hypothesis correct? Check error rate during the 25-second recovery window
kubectl logs -n chaos-testing job/kill-one-pod
```

### Network Chaos — Latency and Packet Loss

```yaml
# network-chaos-delay.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: payment-service-delay
  namespace: production
spec:
  action: delay

  mode: all            # Apply to all matching pods
  selector:
    namespaces: [production]
    labelSelectors:
      app: payment-service

  delay:
    latency: "200ms"    # Add 200ms to all outgoing network calls
    jitter: "50ms"      # ±50ms variation (realistic network conditions)
    correlation: "25"   # Correlate consecutive delays (25% correlation)

  duration: "10m"

  # WHY test this: your checkout service calls payment-service.
  # Does it have a timeout configured? Will it show an error or hang?
  # Does it retry? Does it use a circuit breaker?
  # 200ms latency should be survivable if timeouts + retries are configured.
```

```yaml
# network-chaos-partition.yaml — simulate network partition between services
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: db-network-partition
  namespace: production
spec:
  action: partition     # Complete network isolation

  mode: all
  selector:
    namespaces: [production]
    labelSelectors:
      app: postgres

  direction: both       # Block traffic in both directions

  target:
    mode: all
    selector:
      namespaces: [production]
      labelSelectors:
        app: myapp      # Isolate postgres from myapp

  duration: "2m"
  # Hypothesis: myapp's readiness probe fails → removed from Service
  # (no traffic during partition), then recovers when partition ends.
  # No data corruption. Connection pool reconnects within 30s.
```

### Stress Chaos — CPU and Memory Pressure

```yaml
# stress-chaos-cpu.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-test
  namespace: production
spec:
  mode: fixed
  value: "1"
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp

  stressors:
    cpu:
      workers: 2          # 2 CPU-intensive goroutines
      load: 80            # Each consuming 80% of 1 CPU core

  duration: "5m"
  # Hypothesis: HPA detects CPU > 60% threshold within 30s.
  # New pods scheduled within 2 minutes.
  # P99 latency stays below 500ms throughout.
```

```yaml
# stress-chaos-memory.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: memory-stress-test
  namespace: production
spec:
  mode: fixed
  value: "1"
  selector:
    namespaces: [production]
    labelSelectors:
      app: myapp

  stressors:
    memory:
      workers: 1
      size: "256MB"       # Allocate 256MB of memory in the target pod

  duration: "5m"
  # Hypothesis: memory usage exceeds limit → OOMKilled.
  # OR: memory limit is sufficient and app stays healthy.
  # This experiment reveals whether memory limits are correctly sized.
```

### IO Chaos — Disk Latency

```yaml
# io-chaos-latency.yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: disk-latency
  namespace: production
spec:
  action: latency

  mode: fixed
  value: "1"
  selector:
    namespaces: [production]
    labelSelectors:
      app: postgres

  volumePath: /var/lib/postgresql/data
  delay: "10ms"           # Add 10ms to every disk I/O operation
  percent: 50             # Apply to 50% of I/O operations

  duration: "5m"
  # Hypothesis: 10ms disk latency causes PostgreSQL query times to increase.
  # Spring Boot connection pool timeouts (30s) are not exceeded.
  # P99 latency increases but stays below SLO threshold.
```

### Safe Chaos Practices

**Before any experiment:**
- PDB in place protecting minimum availability
- Prometheus and Grafana watching metrics in real time
- A written hypothesis (not just "let's see what happens")
- Rollback plan: `kubectl delete podchaos/networkchaos/stresschaos <name>`

**Experiment progression:**
1. Start in staging, never production first
2. Start small: kill 1 pod, not 50% of pods
3. Run during business hours when the team is watching
4. Stop immediately if steady state is not maintained

**Chaos scheduling — automate but safely:**
```yaml
# Schedule pod kills during business hours only, Monday-Friday
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: regular-resilience-test
spec:
  # ...
  scheduler:
    cron: "0 10 * * 1-5"    # 10 AM weekdays (UTC)
  duration: "10m"
  # Add a pause between chaos runs
  # (implement with separate scheduled jobs that create/delete chaos resources)
```


## Chapter 7: Advanced Deployment Strategies

### Rolling Updates — The Default

The Kubernetes rolling update strategy replaces old pods with new ones incrementally, ensuring some capacity is always available. Configured with two fields:

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1         # Allow 1 extra pod above desired count during update
      maxUnavailable: 0   # Never go below desired count (zero-downtime update)
      # With replicas=5, maxSurge=1, maxUnavailable=0:
      # Kubernetes creates 1 new pod (total: 6), waits for it to be Ready,
      # then removes 1 old pod (total: 5), repeating until all 5 are updated.
      # At no point do you have fewer than 5 healthy pods.
```

Rolling updates are built into Kubernetes — no extra tooling needed. They are appropriate for the vast majority of deployments. The limitations: if the new version has a database schema incompatibility with the old version, both run simultaneously during the rollout and you may get errors. For those cases, blue-green or canary deployments are better.

### Argo Rollouts — Beyond Rolling Updates

Argo Rollouts is a Kubernetes controller that extends the Deployment concept with additional strategies. It replaces `Deployment` with `Rollout` (the CRD) and adds blue-green and canary support with automated analysis.

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install the kubectl plugin for managing rollouts
brew install argoproj/tap/kubectl-argo-rollouts   # macOS
# Or download from: https://github.com/argoproj/argo-rollouts/releases
```

### Blue-Green Deployment

**What it is:** Run old (blue) and new (green) versions simultaneously, then switch all traffic at once with a single command. Instant rollback: if green has problems, switch back to blue — which is still fully running.

**When to use it:** Schema changes where old and new code can't run against the same database simultaneously. Features that must be fully available or fully unavailable (no partial rollout). Situations where you need instant, complete rollback capability.

The naming convention (blue/green) is arbitrary — what matters is that both versions run in parallel and a single Service selector switch moves all traffic.

```yaml
# rollout-blue-green.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3
        # ... rest of container spec

  strategy:
    blueGreen:
      # activeService: receives production traffic (the "blue" slot)
      activeService: myapp-active

      # previewService: receives preview traffic (the "green" slot, before promotion)
      previewService: myapp-preview

      # autoPromotionEnabled: false = manual promotion required.
      # The green version deploys, but traffic does NOT switch until you run:
      # kubectl argo rollouts promote myapp
      autoPromotionEnabled: false

      # How long to run the preview before allowing auto-promotion (if enabled)
      # autoPromotionSeconds: 60

      # Scale down the old ReplicaSet this long after promotion
      scaleDownDelaySeconds: 30

      # Run pre-promotion analysis before switching traffic
      prePromotionAnalysis:
        templates:
        - templateName: success-rate-check
        args:
        - name: service-name
          value: myapp-preview
```

```yaml
# Two services required for blue-green
apiVersion: v1
kind: Service
metadata:
  name: myapp-active      # Production traffic — selector updated on promotion
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-preview     # Preview traffic — always points to the new version
spec:
  selector:
    app: myapp
  ports:
  - port: 8080
```

**Blue-green workflow:**
```bash
# 1. Update the image — new green pods deploy, but no traffic switches
kubectl argo rollouts set image myapp myapp=myapp:v1.3.0 -n production

# 2. Watch the rollout
kubectl argo rollouts get rollout myapp -n production --watch

# 3. Test the preview (green) version before promoting
kubectl port-forward svc/myapp-preview 8080:8080 -n production
curl http://localhost:8080/actuator/health

# 4. Promote: switch all traffic from blue to green
kubectl argo rollouts promote myapp -n production
# The myapp-active Service selector now points to the green (new) pods.
# The old blue pods scale down after scaleDownDelaySeconds.

# 5. If something goes wrong after promotion — instant rollback
kubectl argo rollouts undo myapp -n production
# The blue pods are still running (scaleDownDelaySeconds hasn't elapsed).
# The myapp-active Service selector switches back to blue immediately.
```

### Canary Deployment

**What it is:** Gradually shift traffic from old to new — 10% → 25% → 50% → 100% — with real metrics monitored at each step. If the new version performs worse at any step, roll back automatically.

**When to use it:** Any deployment where you want real production traffic data before committing fully. Particularly valuable for performance-sensitive changes where pre-production testing can't capture real load patterns.

**The non-obvious requirement:** Canary deployments with Argo Rollouts require two Services — a `stable` Service pointing to old pods and a `canary` Service pointing to new pods. Argo Rollouts manipulates the replica counts and the traffic-splitting mechanism (nginx-ingress or Istio) to split traffic between them. This Service setup is mandatory but is not required for blue-green.

```yaml
# rollout-canary.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
  namespace: production
spec:
  replicas: 10
  selector:
    matchLabels:
      app: myapp

  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:v1.2.3

  strategy:
    canary:
      # canaryService: receives canary traffic weight
      canaryService: myapp-canary
      # stableService: receives remaining traffic
      stableService: myapp-stable

      # Ingress that Argo Rollouts will manage for traffic splitting
      trafficRouting:
        nginx:
          stableIngress: myapp-ingress   # The existing production Ingress

      steps:
      # Step 1: Send 10% to canary, pause and run analysis
      - setWeight: 10
      - analysis:
          templates:
          - templateName: success-rate
          args:
          - name: service-name
            value: myapp-canary
      # Step 2: Expand to 25%
      - setWeight: 25
      - pause:
          duration: 10m     # Wait 10 minutes at 25% (human review window)
      # Step 3: Expand to 50%
      - setWeight: 50
      - analysis:
          templates:
          - templateName: success-rate
          - templateName: latency-check
      # Step 4: Expand to 75%
      - setWeight: 75
      - pause:
          duration: 5m
      # Step 5: 100% — canary becomes the new stable
      # (Argo Rollouts promotes automatically after all steps pass)
```

### AnalysisTemplate — Automated Rollback Criteria

An `AnalysisTemplate` defines what "success" means using Prometheus queries. If the analysis fails at any step, Argo Rollouts automatically rolls back to the previous stable version:

```yaml
# analysistemplate-success-rate.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
  - name: service-name     # Passed in from the Rollout step

  metrics:
  - name: success-rate
    # Run this analysis for 5 minutes
    count: 5
    interval: 1m
    # Fail if success rate drops below 95%
    failureLimit: 1         # Allow 1 failure before rolling back

    provider:
      prometheus:
        address: http://monitoring-kube-prometheus-prometheus.monitoring:9090
        query: |
          sum(
            rate(
              http_server_requests_seconds_count{
                namespace="production",
                service="{{args.service-name}}",
                status!~"5.."}[5m]
            )
          )
          /
          sum(
            rate(
              http_server_requests_seconds_count{
                namespace="production",
                service="{{args.service-name}}"}[5m]
            )
          )
          * 100
    successCondition: result[0] >= 95   # At least 95% success rate
    failureCondition: result[0] < 90    # Fail immediately if below 90%

---
# latency-check template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: latency-check
  namespace: production
spec:
  args:
  - name: service-name
  metrics:
  - name: p99-latency
    count: 5
    interval: 1m
    provider:
      prometheus:
        address: http://monitoring-kube-prometheus-prometheus.monitoring:9090
        query: |
          histogram_quantile(0.99,
            sum by (le) (
              rate(http_server_requests_seconds_bucket{
                namespace="production",
                service="{{args.service-name}}"}[5m])
            )
          ) * 1000
    successCondition: result[0] <= 500   # P99 must be ≤ 500ms
    failureCondition: result[0] > 1000   # Fail if P99 exceeds 1000ms
```

**Canary workflow in practice:**
```bash
# 1. Update the image — canary rollout begins at step 1 (10% traffic)
kubectl argo rollouts set image myapp myapp=myapp:v1.3.0 -n production

# 2. Monitor the rollout — shows current weight, step, analysis status
kubectl argo rollouts get rollout myapp -n production --watch
# Name:            myapp
# Namespace:       production
# Status:          ॥ Paused
# Strategy:        Canary
# Step:            2/8 (Paused)
# Set Weight:      10
# Actual Weight:   10
# ...
# Analysis Runs:   success-rate-abcd1  ✔ Successful

# 3. Manually advance past a pause step
kubectl argo rollouts promote myapp -n production

# 4. If analysis fails at any step, automatic rollback triggers:
# Analysis Run: success-rate-xyz  ✗ Failed (success rate was 87%)
# Rollout rolls back to previous stable version automatically.

# 5. Manual rollback at any point
kubectl argo rollouts undo myapp -n production

# 6. View rollout history
kubectl argo rollouts history rollout myapp -n production
```

---

## Chapter 8: Disaster Recovery

### RTO and RPO — Setting Targets Before You Need Them

Two metrics define your disaster recovery posture:

**RTO (Recovery Time Objective):** How long can the system be unavailable before unacceptable business impact? A payment processing system might have an RTO of 5 minutes. An internal reporting tool might have an RTO of 4 hours.

**RPO (Recovery Point Objective):** How much data loss is acceptable? An RTO of 1 hour and an RPO of 15 minutes means: the system can be down for up to 1 hour, and you can afford to lose up to 15 minutes of data at most.

Your backup frequency must be ≤ your RPO. Hourly backups with a 15-minute RPO is a mismatch. Your recovery process must complete within your RTO.

Define these before incidents, not during them.

### DR Scenario Planning

| Scenario | Recovery mechanism | Target RTO |
|---------|--------------------|-----------|
| Single pod failure | Kubernetes self-healing (probe + restart) | < 60 seconds |
| Node failure | Kubernetes rescheduling to healthy node | 2–5 minutes |
| Deployment broken by bad release | `helm rollback` or `kubectl rollout undo` | < 5 minutes |
| Namespace accidentally deleted | Velero restore | 15–30 minutes |
| Persistent volume corrupted | Restore from latest volume snapshot or pg_dump | 30 min – 2 hours |
| Cluster completely lost | Velero restore to new cluster | 1–4 hours |
| Entire cloud region outage | Activate DR cluster in secondary region | 15–60 minutes |

### Velero — Cluster-Level Backup and Restore

Velero backs up Kubernetes resources (all objects in a namespace) and optionally PVC data. A Velero backup is the recovery mechanism for namespace-deletion and cluster-loss scenarios.

Full Velero installation and usage was covered in Part 6. This chapter covers the DR-specific workflow:

```bash
# ─── BACKUP STRATEGY ───────────────────────────────────────────────────────

# Daily full backup — retained 30 days
velero schedule create daily-production \
  --schedule="0 1 * * *" \
  --include-namespaces production \
  --ttl 720h \
  --storage-location default

# Hourly backup of just config (not PVC data — faster, cheaper)
velero schedule create hourly-config \
  --schedule="0 * * * *" \
  --include-namespaces production \
  --exclude-resources persistentvolumeclaims,persistentvolumes \
  --ttl 48h

# Manual backup before any significant operation
velero backup create "pre-$(date +%Y%m%d-%H%M%S)" \
  --include-namespaces production \
  --wait

# ─── RESTORE PROCEDURE ─────────────────────────────────────────────────────

# 1. List available backups
velero backup get

# 2. Inspect a backup before restoring
velero backup describe daily-production-20240115010000 --details

# 3. Restore to a test namespace first (non-destructive verification)
velero restore create test-restore \
  --from-backup daily-production-20240115010000 \
  --namespace-mappings production:production-test \
  --wait

kubectl get all -n production-test
# Verify resources look correct before proceeding

# 4. If verified, restore to original namespace
# (First delete existing resources if namespace still exists)
kubectl delete namespace production   # WARNING: deletes all resources
velero restore create production-restore \
  --from-backup daily-production-20240115010000 \
  --include-namespaces production \
  --wait

# 5. Verify restored application
kubectl get pods -n production
kubectl exec -it postgres-0 -n production -- \
  psql -U appuser -d appdb -c "SELECT count(*) FROM orders;"
```

### DNS Failover with Route53

For regional outage DR, you need to route traffic to a secondary region while the primary is unavailable:

```bash
# Route53 health check for primary region
aws route53 create-health-check \
  --caller-reference "primary-health-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "api.myapp.com",
    "Port": 443,
    "ResourcePath": "/actuator/health",
    "RequestInterval": 10,
    "FailureThreshold": 3
  }'

# Primary DNS record — active, health-checked
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "HealthCheckId": "PRIMARY_HEALTH_CHECK_ID",
        "TTL": 60,
        "ResourceRecords": [{"Value": "PRIMARY_IP"}]
      }
    }]
  }'

# Secondary DNS record — activates when primary health check fails
aws route53 change-resource-record-sets \
  --hosted-zone-id ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "ResourceRecords": [{"Value": "SECONDARY_IP"}]
      }
    }]
  }'
# When the primary health check fails 3 consecutive times (30 seconds),
# Route53 automatically routes new DNS queries to the secondary IP.
# Existing connections to the primary are unaffected until they time out.
# DNS TTL of 60s means clients pick up the change within ~60 seconds.
```

### DR Testing Schedule

A DR plan that is never tested is not a DR plan — it's a hypothesis. Test regularly:

| Test type | Frequency | What it verifies |
|-----------|-----------|-----------------|
| Velero restore to test namespace | Monthly | Backups are valid and restorable |
| Full cluster restore to staging cluster | Quarterly | Complete recovery procedure works end-to-end |
| DNS failover simulation | Quarterly | Route53 failover activates within expected time |
| Cross-region activation | Annually | Secondary region can handle production traffic |

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| HPA not scaling despite high CPU | `resources.requests.cpu` not set on pods | `kubectl describe hpa <n>` → "missing request for cpu" | Add `resources.requests.cpu` to every container |
| HPA shows `<unknown>/60%` for metric | Metrics Server not running | `kubectl top pods -n <ns>` — if fails, Metrics Server is down | Install Metrics Server; check it's running: `kubectl get pods -n kube-system \| grep metrics-server` |
| HPA flapping — scales up and down repeatedly | `stabilizationWindowSeconds` too short for scale-down | `kubectl describe hpa <n>` — rapid replica changes in Events | Set `behavior.scaleDown.stabilizationWindowSeconds: 300` |
| Custom metric not available for HPA | Prometheus Adapter not configured or metric rule missing | `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` — metric not listed | Check Prometheus Adapter ConfigMap; verify Prometheus has the metric: query it directly at `:9090` |
| VPA and HPA fighting each other | Both controlling CPU simultaneously | `kubectl get vpa,hpa -n <ns>` — both exist and both control CPU | Set VPA `controlledResources: ["memory"]` only; HPA on custom metric not CPU |
| Cluster Autoscaler not adding nodes | CA not finding Pending pods, ASG tags missing, or IAM permissions wrong | `kubectl logs -n kube-system deployment/cluster-autoscaler` | Check ASG tags; verify IAM policy; check `kubectl get pods -A \| grep Pending` |
| `kubectl drain` hangs indefinitely | PDB prevents eviction — not enough available pods to evict one | `kubectl get pdb -n <ns>` — check ALLOWED DISRUPTIONS is 0 | Wait for more pods to become available (HPA/CA adds them), or temporarily remove PDB if acceptable |
| Argo Rollouts canary stuck at step | Analysis template failing or paused waiting for promotion | `kubectl argo rollouts get rollout <n>` — shows current step and reason | Check AnalysisRun: `kubectl get analysisrun -n <ns>`; run `kubectl argo rollouts promote <n>` to advance |
| Blue-green: traffic didn't switch after promote | Wrong service names in Rollout spec | `kubectl describe rollout <n>` → check activeService reference | Verify `activeService` and `previewService` names match actual Service names; check selector labels |
| Velero restore misses some resources | Backup was created before resources existed, or resource excluded | `velero backup describe <n> --details` — check included resources | Create a new backup; check `--exclude-resources` flags on the schedule |
| Chaos Mesh experiment has no effect | Wrong pod selectors, or pods don't match the selector | `kubectl describe podchaos <n>` → selected pods list | Verify selector matches: `kubectl get pods -n <ns> -l app=myapp` |

---

## Practice Exercises

**Exercise 1 — Full HPA scaling chain:**
Deploy your app with 3 replicas and a CPU-based HPA (min 2, max 10, target 50%). Confirm `kubectl top pods` returns data. Run a load generator that brings CPU above 50%. Watch `kubectl get hpa -w` until pods are added. Kill the load generator. Watch the 5-minute stabilisation window before scale-down. Check `kubectl describe hpa` to see the event history. Deliberately remove `resources.requests.cpu` from the Deployment and observe what HPA reports — then put it back.

**Exercise 2 — VPA recommendations:**
Install VPA in `Off` mode against your hello-app Deployment. Run a load test for 30 minutes. Check `kubectl get vpa hello-app-vpa -o yaml` and read the recommendation. Compare the `target` values to what you have configured in `values.yaml`. Update your values to match the VPA recommendations. Try enabling `Auto` mode in staging and observe a pod being evicted and recreated with new resource values.

**Exercise 3 — PDB enforcement in practice:**
Deploy your app with 3 replicas and a PDB with `minAvailable: 2`. Confirm `kubectl get pdb` shows `ALLOWED DISRUPTIONS: 1`. Run `kubectl drain <node>` that has 2 of your app's pods. Observe the drain pause on the second pod. Open a second terminal and watch the evicted pod being rescheduled on another node. Once the rescheduled pod is Ready, watch the drain continue. After the experiment, `kubectl uncordon` the node.

**Exercise 4 — Chaos engineering pod kill:**
Ensure your app has 3 replicas, a PDB with `minAvailable: 2`, and Prometheus monitoring active. Write down the steady-state metrics: error rate, P99 latency. Install Chaos Mesh and create a `PodChaos` experiment that kills 1 pod every 10 minutes for 1 hour. Monitor Grafana throughout. Verify: error rate stays below 0.5%, P99 latency stays below 500ms, killed pods recover within 60 seconds. Write a one-paragraph post-mortem: what happened, did your hypothesis hold, what would you change.

**Exercise 5 — Canary deployment with automated rollback:**
Install Argo Rollouts. Convert your hello-app Deployment to a Rollout with a canary strategy: 10% for 5 minutes with analysis, then 50%, then 100%. Create an AnalysisTemplate that fails if error rate exceeds 5%. Deploy a v1.1.0 of your app that works normally — watch it canary to 100%. Then deploy a v1.2.0 that deliberately returns 503 for 10% of requests. Watch the analysis fail and the automatic rollback trigger. Verify that production traffic returns to v1.1.0 without any manual intervention.

