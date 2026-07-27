# Part 7: Security — Zero-Trust Kubernetes

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Advanced
> **Prerequisites:** Completed Parts 1–6, or comfortable with Kubernetes workloads, namespaces, and Helm
> **Time to complete:** 5–6 hours
> **What you'll learn:** Defense-in-depth security model, RBAC for humans and pods, pod security standards, advanced network policies, OPA Gatekeeper for fine-grained admission control, HashiCorp Vault for secrets management, and image security with Cosign, Trivy, kube-bench, and Falco

---

## What This Part Covers

Security in Kubernetes is not a single thing you enable — it is a series of layers, each addressing a different threat. A cluster that is perfectly configured at the network layer but runs containers as root, has wildcard RBAC, and stores secrets in environment variables is not secure. Each layer independently can be the one that matters when something goes wrong.

This part works through every significant security layer systematically — starting from the threat model and working outward to concrete, deployable configurations. By the end, you'll have a complete hardened deployment specification and an understanding of why each control exists, not just what it does.

---

## Chapter 1: Security Layers — Defense in Depth

### Why One Layer Is Never Enough

Defense in depth is the security principle that no single control should be the last line of defence. Assume every layer will eventually be bypassed, compromised, or misconfigured. Design so that an attacker who gets past one layer faces another, independent obstacle.

In Kubernetes, this plays out concretely. Suppose an attacker exploits a vulnerability in your application code. What can they do next?

Without defence in depth:
- Application runs as root → attacker has root in the container
- `readOnlyRootFilesystem` disabled → attacker can write malware to the filesystem
- `automountServiceAccountToken` enabled → attacker can call the Kubernetes API
- Wildcard RBAC on the service account → attacker can read all secrets in the cluster
- No network policies → attacker can reach the production database from this compromised pod
- No Falco runtime monitoring → nobody knows any of this is happening

With defence in depth, each of those steps is blocked independently. A compromised application process running as a non-root user, with a read-only filesystem, no Kubernetes API token, and locked down egress network policies, has almost nowhere to go. The attacker needs to break through five independent controls, not one.

### The Eight Layers

```
Layer 1: Physical/Cloud         ← Cloud provider security, node hardening (not covered here)
Layer 2: Cluster                ← API server auth, etcd encryption, audit logs
Layer 3: Node                   ← CIS benchmarks (kube-bench), OS hardening
Layer 4: Network                ← Network Policies, mTLS (Istio), firewall rules
Layer 5: Workload               ← Pod security contexts, non-root, read-only filesystem
Layer 6: Authentication         ← RBAC, service accounts, IRSA/Workload Identity
Layer 7: Secrets                ← Vault, External Secrets, Sealed Secrets
Layer 8: Policy                 ← OPA Gatekeeper, admission webhooks, image signing
```

Each layer in this part corresponds to a chapter. You don't have to implement every layer on day one — but you should understand all of them.

### The Most Common Mistakes and Their Impact

| Mistake | What it enables | Addressed in |
|---------|----------------|-------------|
| Running containers as root | Container escape → root on node | Chapter 4 |
| `automountServiceAccountToken: true` on all pods | Kubernetes API access from compromised pod | Chapter 3 |
| Wildcard RBAC (`verbs: ["*"]`) | Any action on any resource if pod is compromised | Chapter 2 |
| Secrets as environment variables | Secrets visible in `kubectl describe`, crash reports, logs | Chapter 2, 7 |
| No network policies | Compromised pod reaches every other service | Chapter 5 |
| Images from unverified registries | Supply chain attacks, malicious images | Chapter 8 |
| No admission control | Non-compliant workloads deployed without detection | Chapter 6 |
| No runtime monitoring | Attacks happen silently, discovered weeks later | Chapter 8 |

---

## Chapter 2: RBAC — Who Can Do What

### What RBAC Is and Why It Exists

Without access control, every authenticated user and every pod in your cluster can do anything: read secrets, delete deployments, modify cluster configuration. Kubernetes implements Role-Based Access Control (RBAC) to define who is allowed to perform which operations on which resources.

RBAC answers three questions:
- **Who** is making the request? (a user, a group, or a service account)
- **What** do they want to do? (list pods, create deployments, delete secrets)
- **Where** are they allowed to do it? (one namespace, or cluster-wide?)

### The Four RBAC Objects

RBAC uses four object types that work in two pairs:

```
Namespace-scoped:
  Role           → defines permissions within one namespace
  RoleBinding    → grants a Role to a subject (user/group/SA) in one namespace

Cluster-scoped:
  ClusterRole    → defines permissions cluster-wide (or for non-namespaced resources)
  ClusterRoleBinding → grants a ClusterRole to a subject across the entire cluster
```

You can also bind a `ClusterRole` with a `RoleBinding` — this grants the ClusterRole's permissions but only within the namespace of the RoleBinding. Useful for sharing common role definitions (like "read-only access") without duplicating them across namespaces.

### Roles — Defining Permissions

A Role is a collection of permission rules. Each rule specifies:
- `apiGroups`: which API group the resource belongs to (core resources use `""`, apps use `"apps"`)
- `resources`: which resource types (pods, deployments, secrets)
- `verbs`: what actions are permitted

```yaml
# role-readonly.yaml — read-only access to common resources in one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: read-only
  namespace: production
rules:
# Core API group — pods, services, configmaps, etc.
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "endpoints", "events"]
  verbs: ["get", "list", "watch"]
  # get:   kubectl get pod <name>
  # list:  kubectl get pods
  # watch: kubectl get pods -w

# Apps API group — deployments, replicasets, statefulsets
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
```

```yaml
# role-deployer.yaml — CI/CD pipeline role: update deployments but not delete
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "list", "update", "patch"]
  # update: kubectl apply (for full replacements)
  # patch:  kubectl patch (for partial updates, like image tag changes)
  # Deliberately NO "create" or "delete" — CI/CD can update, not create from scratch

- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "create", "update", "patch"]

# Helm needs to create/update Secrets for release state
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "create", "update", "patch"]
  # Note: no "list" — pipeline can manage specific secrets, not list all

# Allow reading pod status to verify deployments
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

**What NOT to grant — wildcard permissions:**
```yaml
# ❌ This grants everything to everyone — never do this
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

This pattern appears in many "quick start" guides because it makes things work immediately. It bypasses all access control. An attacker who compromises any pod with this role can read all secrets, delete all workloads, and modify cluster configuration.

### RoleBindings — Granting Roles to Subjects

A RoleBinding connects a Role to a subject (who gets the permissions):

```yaml
# rolebinding-ci-deployer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
# Bind to a Service Account (used by CI/CD pipelines running in the cluster)
- kind: ServiceAccount
  name: github-actions-sa
  namespace: production

# Bind to a user (human operator with a kubeconfig)
- kind: User
  name: alice@mycompany.com
  apiGroup: rbac.authorization.k8s.io

# Bind to a group (all members of the group get this role)
- kind: Group
  name: platform-team
  apiGroup: rbac.authorization.k8s.io

roleRef:
  kind: Role
  name: deployer         # The Role defined above
  apiGroup: rbac.authorization.k8s.io
```

### Cluster-Wide Permissions with ClusterRole

For resources that span namespaces (nodes, PersistentVolumes, StorageClasses) or for monitoring that needs to see all namespaces, use ClusterRole:

```yaml
# clusterrole-monitoring.yaml — read everything for monitoring/observability tools
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "pods", "services", "endpoints",
              "namespaces", "persistentvolumes", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
- nonResourceURLs: ["/metrics", "/metrics/cadvisor"]
  verbs: ["get"]   # Prometheus needs to GET /metrics endpoints on nodes
```

### Testing RBAC with `kubectl auth can-i`

Before and after configuring RBAC, verify permissions work as intended:

```bash
# Can the current user delete pods in production?
kubectl auth can-i delete pods -n production
# yes / no

# Can a specific service account list secrets?
kubectl auth can-i list secrets \
  --as=system:serviceaccount:production:github-actions-sa \
  -n production
# no (correct — deployer role has no list secrets permission)

# Can a specific user get deployments?
kubectl auth can-i get deployments \
  --as=alice@mycompany.com \
  -n production
# yes

# List ALL permissions for a service account (verbose but complete)
kubectl auth can-i --list \
  --as=system:serviceaccount:production:github-actions-sa \
  -n production
# Resources                                    Non-Resource URLs   Resource Names   Verbs
# deployments.apps                             []                  []               [get list update patch]
# configmaps                                   []                  []               [get create update patch]
# ...
```

### Creating Test Users with Certificates (Local Clusters)

Kubernetes uses X.509 client certificates for user authentication. For testing RBAC locally:

```bash
# Generate a private key and CSR for user "alice"
openssl genrsa -out alice.key 2048
openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=platform-team"
# /CN is the username, /O is the group

# Sign the CSR with the cluster CA (for kind)
# kind's CA lives inside the control-plane node's container filesystem,
# not on your host — copy it out first. "hello-app-control-plane" is the
# container name kind creates for the cluster we set up in Part 1
# (<cluster-name>-control-plane).
docker cp hello-app-control-plane:/etc/kubernetes/pki/ca.crt ./kind-ca.crt
docker cp hello-app-control-plane:/etc/kubernetes/pki/ca.key ./kind-ca.key

openssl x509 -req -in alice.csr \
  -CA ./kind-ca.crt -CAkey ./kind-ca.key \
  -CAcreateserial -out alice.crt -days 365

# Add alice to your kubeconfig
kubectl config set-credentials alice \
  --client-certificate=alice.crt \
  --client-key=alice.key

kubectl config set-context alice-context \
  --cluster=kind-hello-app \
  --user=alice

# Test as alice
kubectl --context=alice-context get pods -n production
# Error: pods is forbidden — alice has no role yet

# Create and bind a role, then test again
kubectl apply -f role-readonly.yaml
kubectl apply -f rolebinding-alice.yaml
kubectl --context=alice-context get pods -n production
# NAME       READY   STATUS    RESTARTS   AGE
# myapp-0    1/1     Running   0          5d   ← now visible
```

---

## Chapter 3: Service Accounts — Pod Identity

### What Service Accounts Are

Every pod in Kubernetes runs with an identity — a service account. This identity determines what Kubernetes API calls the pod is permitted to make. By default, pods use the `default` service account in their namespace.

The problem with the `default` service account: it's shared by everything. If you grant the `default` service account permissions for one legitimate reason (say, a pod that needs to list ConfigMaps), every other pod in that namespace gets those permissions too — including a compromised application pod.

Create dedicated service accounts for each workload with only the permissions that workload actually needs.

### The Default Token — Disable It

By default, Kubernetes automatically mounts a service account token into every pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token can be used to call the Kubernetes API server.

Most application pods have no business calling the Kubernetes API. A Spring Boot service handling HTTP requests, a PostgreSQL database, a Redis cache — none of these need to list pods, get ConfigMaps, or read secrets from the Kubernetes API. Yet by default they all have credentials to do exactly that.

Setting `automountServiceAccountToken: false` removes this token from the pod:

```yaml
spec:
  automountServiceAccountToken: false   # Don't mount the K8s API token
  serviceAccountName: myapp-sa          # Still identifies the pod, just no auto-token
  containers:
  - name: myapp
    # ...
```

The token is not mounted, so if the application pod is compromised, the attacker has no Kubernetes API credentials to escalate with.

**This does NOT break IRSA on EKS.** This is important enough to say explicitly because it confuses many people. IRSA (IAM Roles for Service Accounts) works through a completely separate mechanism:

- The `automountServiceAccountToken: false` flag disables the *default* token at `/var/run/secrets/kubernetes.io/serviceaccount/token`
- IRSA uses a *projected* service account token, mounted at `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` by the EKS pod identity webhook
- This projected token is injected via a separate volume, entirely independently of the default token setting
- Setting `automountServiceAccountToken: false` does not affect IRSA's projected token in any way

You can safely set `automountServiceAccountToken: false` on all pods that use IRSA. Both security controls work independently and together.

### Creating Purpose-Built Service Accounts

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    # AWS IRSA: links this SA to an IAM role
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-production-role
    # GCP Workload Identity equivalent:
    # iam.gke.io/gcp-service-account: myapp@myproject.iam.gserviceaccount.com
automountServiceAccountToken: false   # Disable default token at SA level
```

Bind the minimum required RBAC:
```yaml
# role-myapp.yaml — only what myapp actually needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: myapp-role
  namespace: production
rules:
# If myapp needs to read its own ConfigMaps (e.g., spring-cloud-kubernetes)
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "watch"]
  resourceNames: ["myapp-config"]   # Only this specific ConfigMap, not all of them
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-role-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
```

### AWS IRSA — Pods with AWS Credentials, No Access Keys

Without IRSA, giving a pod access to AWS services (S3, SQS, DynamoDB) requires creating an IAM user, generating access keys, storing them as Kubernetes Secrets, and rotating them manually. Long-lived access keys are a significant security risk — they don't expire, they can be exfiltrated and used from anywhere.

IRSA replaces this with short-lived, automatically-rotated credentials scoped to a specific Kubernetes service account.

**How it works, step by step:**

```
1. EKS cluster has an OIDC provider (unique per cluster)
   https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

2. You create an IAM Role with a trust policy that says:
   "Allow tokens from THIS cluster's OIDC provider,
    issued for THIS namespace and THIS service account"

3. You annotate the Kubernetes Service Account with the IAM Role ARN

4. When the pod starts, the EKS pod identity webhook detects the annotation
   and injects a projected token + the AWS_ROLE_ARN env var

5. AWS SDK in the pod automatically calls STS with the projected token:
   sts:AssumeRoleWithWebIdentity → receives temporary credentials

6. Temporary credentials (15-minute lifetime) are transparently used
   for all AWS API calls — no key rotation needed
```

**Setting up IRSA:**

```bash
# Step 1: Get your cluster's OIDC provider URL
OIDC_URL=$(aws eks describe-cluster --name my-cluster \
  --query "cluster.identity.oidc.issuer" --output text)
echo $OIDC_URL
# https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E

# Step 2: Create the OIDC provider in IAM (one-time per cluster)
aws iam create-open-id-connect-provider \
  --url $OIDC_URL \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list $(openssl s_client -connect oidc.eks.us-east-1.amazonaws.com:443 \
    -showcerts </dev/null 2>/dev/null | openssl x509 -fingerprint -noout \
    | sed 's/://g' | cut -d= -f2 | tr '[:upper:]' '[:lower:]')
```

```json
// Step 3: Create the IAM role trust policy (trust-policy.json)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub":
            "system:serviceaccount:production:myapp-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud":
            "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

```bash
# Step 4: Create the IAM role and attach permissions
aws iam create-role \
  --role-name myapp-production-role \
  --assume-role-policy-document file://trust-policy.json

# Attach the permissions the app actually needs (principle of least privilege)
aws iam attach-role-policy \
  --role-name myapp-production-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Step 5: Annotate the Kubernetes Service Account
kubectl annotate serviceaccount myapp-sa \
  -n production \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/myapp-production-role

# Step 6: Restart pods to pick up the new annotation
kubectl rollout restart deployment/myapp -n production

# Verify — pod should have AWS env vars injected
kubectl exec -it $(kubectl get pod -n production -l app=myapp -o name | head -1) \
  -n production -- env | grep AWS
# AWS_ROLE_ARN=arn:aws:iam::123456789:role/myapp-production-role
# AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/eks.amazonaws.com/serviceaccount/token
# AWS_DEFAULT_REGION=us-east-1
```

### GCP Workload Identity and Azure Workload Identity

**GCP (GKE):**
```bash
# Enable Workload Identity on the cluster
gcloud container clusters update my-cluster \
  --workload-pool=myproject.svc.id.goog

# Create GCP service account
gcloud iam service-accounts create myapp-gsa \
  --project=myproject

# Grant GCS permissions
gcloud projects add-iam-policy-binding myproject \
  --member="serviceAccount:myapp-gsa@myproject.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Bind Kubernetes SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  myapp-gsa@myproject.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:myproject.svc.id.goog[production/myapp-sa]"

# Annotate the Kubernetes SA
kubectl annotate serviceaccount myapp-sa \
  --namespace production \
  iam.gke.io/gcp-service-account=myapp-gsa@myproject.iam.gserviceaccount.com
```

**Azure Workload Identity:**
```bash
# Enable OIDC issuer on AKS
az aks update -g myresourcegroup -n my-cluster \
  --enable-oidc-issuer --enable-workload-identity

# Create Azure managed identity
az identity create -n myapp-identity -g myresourcegroup

# Federated credential — trust tokens from this AKS cluster / SA
az identity federated-credential create \
  --name myapp-federated \
  --identity-name myapp-identity \
  --resource-group myresourcegroup \
  --issuer $(az aks show -g myresourcegroup -n my-cluster \
    --query "oidcIssuerProfile.issuerUrl" -o tsv) \
  --subject "system:serviceaccount:production:myapp-sa"

# Annotate the Kubernetes SA
kubectl annotate serviceaccount myapp-sa -n production \
  azure.workload.identity/client-id=$(az identity show \
    -n myapp-identity -g myresourcegroup --query clientId -o tsv)
```


## Chapter 4: Pod Security Standards — Hardening the Container

### Why Container Security Contexts Exist

By default a container process runs as root (UID 0) inside the container. If an attacker exploits a vulnerability in your application and breaks out of the container, they have root privileges on the node. Kernel exploits, container runtime vulnerabilities, and mounted host paths all become paths to full node compromise when the container process is root.

Security contexts define the security parameters for a pod and its containers. They are the primary mechanism for reducing what damage a compromised container can do.

### Pod Security Standards — Three Levels

Kubernetes defines three predefined security levels — you enforce them at the namespace level:

| Level | What it blocks | Use for |
|-------|---------------|---------|
| **Privileged** | Nothing — all capabilities allowed | System components that genuinely need host access |
| **Baseline** | Most dangerous capabilities: privileged containers, hostPath, hostNetwork, HostPID | General workloads where you need basic sanity |
| **Restricted** | Everything in Baseline plus root containers, all capabilities, any privilege escalation | Production workloads — the target for all application pods |

Enforce a level by labelling the namespace:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # enforce: blocks non-compliant pods (they are rejected at admission)
    pod-security.kubernetes.io/enforce: restricted

    # audit: logs violations but allows the pod to run (useful for rollout)
    pod-security.kubernetes.io/audit: restricted

    # warn: shows a warning in kubectl output but allows the pod
    pod-security.kubernetes.io/warn: restricted
```

Start with `warn`, fix your workloads, then switch to `enforce` once all pods comply. Moving straight to `enforce` on a running namespace breaks existing non-compliant pods immediately.

### Pod Security Context

The pod-level security context applies to all containers in the pod:

```yaml
spec:
  securityContext:
    # Reject the pod if ANY container tries to run as root
    runAsNonRoot: true

    # Run as UID 1000 (must exist in the container image)
    # Override per-container if different UIDs are needed
    runAsUser: 1000
    runAsGroup: 1000

    # Files created in mounted volumes are owned by this GID.
    # Critical for StatefulSets: database data directories need correct ownership.
    fsGroup: 1000

    # Restrict syscalls to a safe subset.
    # RuntimeDefault uses the container runtime's default seccomp profile,
    # which blocks ~300 syscalls rarely needed by applications.
    seccompProfile:
      type: RuntimeDefault

    # Prevent privilege escalation via setuid binaries or sudo
    # (also set per-container, but pod-level sets the default)
```

### Container Security Context

The container-level context applies to a single container and overrides the pod-level settings:

```yaml
containers:
- name: myapp
  securityContext:
    # Prevent the process from gaining more privileges than its parent.
    # Blocks setuid/setgid binaries and sudo.
    allowPrivilegeEscalation: false

    # Container filesystem is read-only.
    # WHY: makes it much harder for malware to persist — can't write scripts,
    # can't modify application binaries, can't drop exploit payloads.
    # IMPORTANT: applications that write to local disk need emptyDir volumes
    # for their temp directories. See below.
    readOnlyRootFilesystem: true

    # Drop ALL Linux capabilities, then add back only what's needed.
    # The full list has 40+ capabilities — most apps need none of them.
    capabilities:
      drop:
        - ALL
      add:
        # Only add what you can justify:
        # NET_BIND_SERVICE: bind to ports below 1024 (usually unnecessary — run on 8080)
        # CHOWN: change file ownership (needed by some init containers)
        # setuid/setgid are NOT in the add list — they're covered by allowPrivilegeEscalation

    # Never run this container as root, even if the image specifies a root user
    runAsNonRoot: true
    runAsUser: 1000
```

### Handling `readOnlyRootFilesystem: true` for Applications

Most application servers write to local disk at some point: temp files, logs, Spring Boot's unpacked JARs, PID files. With `readOnlyRootFilesystem: true`, these writes fail. The fix is `emptyDir` volumes — in-memory or disk-backed temporary volumes mounted at specific paths:

```yaml
spec:
  securityContext:
    readOnlyRootFilesystem: true   # Set at pod level or container level

  containers:
  - name: myapp
    securityContext:
      readOnlyRootFilesystem: true

    volumeMounts:
    # /tmp: general temp file location
    - name: tmp-dir
      mountPath: /tmp

    # Spring Boot unpacks the JAR to a temp directory on startup
    - name: spring-tmp
      mountPath: /app/tmp

    # Logs if writing to local file (prefer stdout instead, but this handles it)
    - name: log-dir
      mountPath: /app/logs

  volumes:
  # emptyDir: created fresh when pod starts, deleted when pod terminates
  # medium: "" = node disk, medium: "Memory" = RAM (faster, but counts against memory limit)
  - name: tmp-dir
    emptyDir: {}
  - name: spring-tmp
    emptyDir: {}
  - name: log-dir
    emptyDir:
      sizeLimit: 500Mi    # Optional: cap how much the log dir can grow
```

### Testing Pod Security Admission

```bash
# Apply restricted level to a namespace
kubectl label namespace test-ns \
  pod-security.kubernetes.io/enforce=restricted

# Try to create a privileged pod — should be rejected
kubectl run privileged-test -n test-ns \
  --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"test","image":"nginx","securityContext":{"privileged":true}}]}}'
# Error: pods "privileged-test" is forbidden:
# violates PodSecurity "restricted:latest": privileged (container "test" must not set
# securityContext.privileged=true)

# Try to run as root
kubectl run root-test -n test-ns --image=nginx
# Error: violates PodSecurity "restricted:latest": runAsNonRoot (pod or container
# "nginx" must set securityContext.runAsNonRoot=true)

# A compliant pod passes
kubectl apply -f compliant-pod.yaml -n test-ns
# pod/myapp created ✓
```

---

## Chapter 5: Network Policies — Deep Dive

### Beyond the Basics

Part 5 introduced network policies for a production application. This chapter goes deeper: all selector types, `ipBlock`, the complete policy set for a multi-tier application under strict zero-trust, and systematic debugging.

### All Selector Types

Network policies have three selector types for the `from` (ingress) and `to` (egress) fields:

**`podSelector`** — selects pods by label within the same namespace (or namespace from `namespaceSelector`):
```yaml
from:
- podSelector:
    matchLabels:
      app: api-service   # Only pods labelled app=api-service
```

**`namespaceSelector`** — selects all pods in namespaces matching labels:
```yaml
from:
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: ingress-nginx   # All pods in the ingress-nginx namespace
```

**Combined (AND):** Both selectors in the same list item = pods must match BOTH:
```yaml
from:
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: monitoring
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus
# Means: pods in the 'monitoring' namespace labelled app.kubernetes.io/name=prometheus
```

**Separate (OR):** Selectors in separate list items = pods matching EITHER:
```yaml
from:
- namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: monitoring
- podSelector:
    matchLabels:
      app: internal-tool
# Means: any pod in 'monitoring' namespace OR any pod labelled app=internal-tool
```

**`ipBlock`** — selects IP address ranges (for external traffic):
```yaml
from:
- ipBlock:
    cidr: 0.0.0.0/0       # All IPs (the internet)
    except:
    - 10.0.0.0/8          # But NOT internal RFC-1918 ranges
    - 172.16.0.0/12
    - 192.168.0.0/16
    # Result: allows external internet traffic but blocks internal lateral movement
```

### `ipBlock` with `except` — Allow Internet, Block Internal

A pattern for services that need to accept traffic from the internet but should not be reachable from other internal services (e.g., a public-facing API that should not be callable from the dev namespace):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internet-only
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: public-api
  policyTypes:
  - Ingress
  ingress:
  - from:
    # Allow from Ingress controller (which does have a pod-specific policy)
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    # Allow external internet traffic directly (if using LoadBalancer Service)
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8       # Block other cluster pods from reaching this directly
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - port: 8080
```

### Allow CI/CD Runners

GitLab CI runners and GitHub Actions self-hosted runners often run in the cluster or a connected VPC. They need to reach your cluster's API and potentially your services during integration tests:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-cicd-runners
  namespace: staging
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: gitlab-runners
      podSelector:
        matchLabels:
          app: gitlab-runner
    ports:
    - port: 8080
```

### Complete Multi-Tier Policy Set

A definitive set for a three-tier application under strict zero-trust — nothing communicates unless explicitly permitted:

```yaml
# ── 1. Default deny all in production namespace ───────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# ── 2. Allow DNS for all pods (ALWAYS required with default-deny) ─────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# ── 3. Frontend receives from Ingress controller ──────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-from-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - port: 3000
---
# ── 4. Frontend can call API ──────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - port: 8080
---
# ── 5. API receives from Ingress controller and frontend ──────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
---
# ── 6. API can egress to: database, cache, external HTTPS ────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - port: 6379
  - ports:   # External HTTPS (payment providers, email APIs, etc.)
    - port: 443
      protocol: TCP
---
# ── 7. Database receives only from API ───────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-from-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - port: 5432
---
# ── 8. Prometheus can scrape metrics from all pods ───────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - port: 8080
```

### Debugging with netshoot

`nicolaka/netshoot` bundles every networking tool you need into one image:

```bash
# Launch a debug pod in the production namespace
kubectl run netshoot \
  --image=nicolaka/netshoot \
  --rm -it \
  --restart=Never \
  -n production \
  -- /bin/bash

# Inside netshoot:

# 1. Test DNS resolution
dig postgres-service.production.svc.cluster.local
# If this fails: DNS allow policy is missing or wrong port

# 2. Test TCP connectivity
nc -zv api-service 8080
# Connection to api-service 8080 port [tcp/*] succeeded! → policy allows it
# nc: getaddrinfo for host "api-service" port 8080: Name or service not known → DNS broken
# (timeout) → policy blocks it

# 3. Test HTTP endpoint
curl -sv http://api-service:8080/actuator/health

# 4. Test that blocked traffic is actually blocked
nc -zv postgres-service 5432  # from a pod that should NOT reach the DB
# (timeout after ~30s) → correctly blocked ✓

# 5. Trace the exact path of a packet
traceroute api-service

# 6. Check which IPs a service resolves to (verify endpoints)
nslookup api-service
dig +short api-service.production.svc.cluster.local
```

---

## Chapter 6: OPA Gatekeeper — Fine-Grained Admission Policy

### What RBAC Cannot Enforce

RBAC controls who can perform an action — create a deployment, update a configmap. It cannot enforce constraints on the content of those resources:

- "All pod images must come from `myregistry.com`" — RBAC allows creating any pod; it can't inspect the image URL
- "Every Deployment must have a `team` label" — RBAC allows creating Deployments; it can't require specific labels
- "No privileged containers" — Pod Security Standards handles this, but custom rules are beyond them
- "Deployments must have resource limits set" — RBAC can't validate the presence of a field

This is the gap OPA Gatekeeper fills. Gatekeeper is an admission controller — it intercepts every create/update request to the Kubernetes API and evaluates it against a set of policies written in Rego (OPA's policy language). Non-compliant requests are rejected before they reach etcd.

### Installing Gatekeeper

```bash
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set replicas=2 \
  --set auditInterval=30

kubectl get pods -n gatekeeper-system
# NAME                                             READY   STATUS
# gatekeeper-audit-6d9c7d5f7-xlmnp                1/1     Running
# gatekeeper-controller-manager-7f6c8d9b4-2xmpq   1/1     Running
# gatekeeper-controller-manager-7f6c8d9b4-mklpq   1/1     Running
```

### How Gatekeeper Works — Two Object Types

**`ConstraintTemplate`** defines a new policy type. It contains:
1. A CRD definition (creates a new Kubernetes resource kind)
2. Rego logic that evaluates whether a resource complies

**`Constraint`** (an instance of the ConstraintTemplate) applies the policy:
1. Which resource types to check (Pods, Deployments, etc.)
2. Which namespaces to enforce it in
3. Any parameters the Rego logic needs

Think of a ConstraintTemplate as a class definition, and a Constraint as an instance of that class with specific configuration.

### Policy 1: Required Labels

All pods must carry `team` and `environment` labels so you can track which team owns what and in which environment:

```yaml
# constrainttemplate-required-labels.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels   # This creates a new CRD kind
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        # Get the list of required labels from the constraint's parameters
        required := input.parameters.labels

        # For each required label, check if it exists in the object's metadata
        required_label := required[_]

        # If the label is missing, it's a violation
        not input.review.object.metadata.labels[required_label]

        msg := sprintf("Missing required label: %v", [required_label])
      }
```

```yaml
# constraint-required-labels.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-team-label
spec:
  # enforcement: deny (blocks non-compliant), warn (allows but warns), dryrun (audit only)
  enforcementAction: deny

  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:               # Only enforce in these namespaces
    - production
    - staging

  parameters:
    labels:
    - team
    - environment
```

```bash
# Test: try to create a Deployment without the required labels
kubectl apply -f deployment-without-labels.yaml -n production
# Error from server ([pods-must-have-team-label]):
#   admission webhook "validation.gatekeeper.sh" denied the request:
#   Missing required label: team
#   Missing required label: environment
```

### Policy 2: No Privileged Containers

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoPrivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainer
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8snoprivilegedcontainer

      violation[{"msg": msg}] {
        # Check every container in the pod spec
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged == true
        msg := sprintf("Privileged container is not allowed: %v", [container.name])
      }

      violation[{"msg": msg}] {
        # Also check initContainers
        container := input.review.object.spec.initContainers[_]
        container.securityContext.privileged == true
        msg := sprintf("Privileged initContainer is not allowed: %v", [container.name])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - production
    - staging
```

### Policy 3: Allowed Image Registries

Block images from untrusted registries — only allow images from your company registry:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        openAPIV3Schema:
          type: object
          properties:
            repos:
              type: array
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedrepos

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]

        # Check if the image starts with any of the allowed prefixes
        satisfied := [good | repo := input.parameters.repos[_]
                              good := startswith(container.image, repo)]

        # If no allowed prefix matched, it's a violation
        not any(satisfied)

        msg := sprintf("Container image %v is not from an allowed registry. Allowed: %v",
          [container.image, input.parameters.repos])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-registries
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - production
  parameters:
    repos:
    - "myregistry.com/"          # Company registry
    - "gcr.io/my-project/"      # GCR project
    - "123456789.dkr.ecr.us-east-1.amazonaws.com/"  # ECR
    # Explicitly NOT allowing: docker.io, public ghcr.io, etc.
```

### Policy 4: Deployment Window

Block deployments to production outside business hours — reduces the risk of deploying late on a Friday:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdeploymentwindow
spec:
  crd:
    spec:
      names:
        kind: K8sDeploymentWindow
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedHoursUTC:
              type: object
              properties:
                start:
                  type: integer   # Hour in UTC (0-23)
                end:
                  type: integer
            allowedDays:
              type: array
              items:
                type: integer     # 0=Sunday, 1=Monday, ..., 6=Saturday
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sdeploymentwindow

      violation[{"msg": msg}] {
        # time.now_ns() returns nanoseconds since Unix epoch
        now := time.now_ns()

        # Convert to hour and day of week
        # time.clock() returns [hour, minute, second] in UTC
        [hour, _, _] := time.clock([now, "UTC"])

        # time.weekday() returns 0=Sunday, 1=Monday, ..., 6=Saturday
        day := time.weekday([now, "UTC"])

        # Check if current day is allowed
        allowed_days := input.parameters.allowedDays
        not allowed_days[_] == day

        msg := sprintf("Deployments not allowed on day %v. Allowed days: %v",
          [day, allowed_days])
      }

      violation[{"msg": msg}] {
        now := time.now_ns()
        [hour, _, _] := time.clock([now, "UTC"])
        start := input.parameters.allowedHoursUTC.start
        end   := input.parameters.allowedHoursUTC.end

        # Outside the allowed time window
        not (hour >= start, hour < end)

        msg := sprintf("Deployments only allowed between %v:00 and %v:00 UTC. Current hour: %v",
          [start, end, hour])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDeploymentWindow
metadata:
  name: production-deploy-window
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment"]
    namespaces:
    - production
  parameters:
    allowedHoursUTC:
      start: 9    # 9:00 UTC = 10:00 CET / 5:00 EST
      end: 17     # 17:00 UTC = 18:00 CET / 13:00 EST
    allowedDays:
      - 1   # Monday
      - 2   # Tuesday
      - 3   # Wednesday
      - 4   # Thursday
      # No Friday — intentional
```

### Audit Mode — Finding Existing Violations

Gatekeeper's audit controller periodically checks all existing resources against constraints and reports violations without blocking anything. Use this to assess your current compliance posture before switching to `deny`:

```bash
# Check constraint violation reports
kubectl describe k8srequiredlabels pods-must-have-team-label

# Sample output:
# Status:
#   Audit Timestamp:  2024-01-15T14:30:00Z
#   Total Violations: 3
#   Violations:
#     Enforcement Action:  deny
#     Kind:                Deployment
#     Message:             Missing required label: team
#     Name:                legacy-app
#     Namespace:           production

# List all constraints and their violation counts
kubectl get constraints
# NAME                          ENFORCEMENT-ACTION   TOTAL-VIOLATIONS
# pods-must-have-team-label     deny                 3
# no-privileged-containers      deny                 0
# allowed-registries            deny                 1
```


## Chapter 7: Secrets Management with HashiCorp Vault

### Why Vault When Kubernetes Secrets Exist?

Kubernetes Secrets have two significant weaknesses for production use.

First, they are base64-encoded, not encrypted, by default. Values stored in etcd are readable by anyone with etcd access or sufficient RBAC permissions. Encryption at rest can be configured, but it requires additional cluster setup and is not on by default.

Second, there is no audit log. When a secret is read — who read it, when, from where — Kubernetes has no built-in record. For compliance with SOC 2, PCI-DSS, or HIPAA, an audit trail of secret access is required.

HashiCorp Vault provides both: true encryption at rest (all secrets are encrypted before storage, always) and a complete audit log of every secret read, write, and list operation. It also provides dynamic secrets (generate a database password that automatically expires), secret rotation, and fine-grained access policies.

### Installing Vault in Dev Mode (for Learning)

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

# Dev mode: single pod, no persistence, unsealed automatically
# NOT for production — data is lost on restart
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.dev.enabled=true \
  --set injector.enabled=true      # Install the Agent Injector sidecar

kubectl get pods -n vault
# NAME                                    READY   STATUS
# vault-0                                 1/1     Running
# vault-agent-injector-5d4b8f9c6-2xmpq   1/1     Running

# Access the Vault UI
kubectl port-forward vault-0 8200:8200 -n vault &
# Visit: http://localhost:8200  Token: root (dev mode)
```

**Production installation** uses HA mode with raft storage:
```bash
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.ha.enabled=true \
  --set server.ha.raft.enabled=true \
  --set server.ha.replicas=3 \
  --set injector.enabled=true \
  --set ui.enabled=true
# After install, run vault operator init and vault operator unseal
```

### Storing Secrets in Vault

```bash
# Access the Vault pod
kubectl exec -it vault-0 -n vault -- /bin/sh

# Authenticate (dev mode uses root token)
export VAULT_TOKEN=root

# Enable KV-v2 secrets engine at the path "secret/"
# KV-v2 supports versioning — you can read previous versions of a secret
vault secrets enable -path=secret kv-v2

# Store application secrets
vault kv put secret/production/myapp \
  db_password="SuperSecret123" \
  api_key="sk-abc123xyz456" \
  jwt_secret="my-very-long-random-jwt-signing-key"

# Verify
vault kv get secret/production/myapp
# ====== Data ======
# Key           Value
# ---           -----
# api_key       sk-abc123xyz456
# db_password   SuperSecret123
# jwt_secret    my-very-long-random-jwt-signing-key

# Read a specific version (KV-v2 versioning)
vault kv get -version=1 secret/production/myapp
```

### Kubernetes Auth Method — Pods Authenticate with Their SA Token

The Kubernetes auth method lets pods authenticate to Vault using their service account JWT token. Vault verifies the token with the Kubernetes API server and exchanges it for a Vault token scoped to the pod's policies.

```bash
# Inside the vault-0 pod:

# Enable the Kubernetes auth method
vault auth enable kubernetes

# Configure it to talk to the Kubernetes API server
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer="https://kubernetes.default.svc.cluster.local"

# Create a Vault policy: what can myapp-sa access?
vault policy write myapp-policy - <<EOF
# Allow reading all secrets under secret/production/myapp/
path "secret/data/production/myapp/*" {
  capabilities = ["read"]
}
# Allow listing secret paths (so the app knows what keys exist)
path "secret/metadata/production/myapp/*" {
  capabilities = ["list"]
}
EOF

# Create a Vault role binding the Kubernetes SA to the policy
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h    # Vault token expires after 1 hour and must be renewed
```

### Vault Agent Injector — Zero-Code Secret Injection

The Vault Agent Injector is a mutating admission webhook. When it sees a pod with Vault annotations, it automatically adds a sidecar container (the Vault Agent) that authenticates to Vault, fetches secrets, and writes them to files — before your application container starts.

Your application reads secrets from files. It never calls the Vault API. It needs no Vault client library. The injection is completely transparent to the application code.

```yaml
# deployment-with-vault.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    metadata:
      annotations:
        # Tell the injector to inject the Vault Agent sidecar
        vault.hashicorp.com/agent-inject: "true"

        # Which Vault role to authenticate as (matches the role we created above)
        vault.hashicorp.com/role: "myapp-role"

        # Fetch this secret and write it to /vault/secrets/db-creds
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/production/myapp/db"

        # Template the secret into a file format your app can read
        # Uses Go template syntax — .Data.data because KV-v2 wraps data in a data key
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/production/myapp/db" -}}
          db.password={{ .Data.data.db_password }}
          db.url=jdbc:postgresql://postgres:5432/appdb
          {{- end }}

        # Fetch the API key separately
        vault.hashicorp.com/agent-inject-secret-api-key: "secret/data/production/myapp/api"
        vault.hashicorp.com/agent-inject-template-api-key: |
          {{- with secret "secret/data/production/myapp/api" -}}
          {{ .Data.data.api_key }}
          {{- end }}

    spec:
      serviceAccountName: myapp-sa  # The SA bound to the Vault role

      containers:
      - name: myapp
        image: myapp:1.0.0
        # The Vault Agent writes secrets to /vault/secrets/ before this container starts.
        # File /vault/secrets/db-creds will contain the templated output above.
```

**Reading Vault secrets in Spring Boot:**

```yaml
# application.yml
spring:
  config:
    # Import the properties file written by Vault Agent
    import: "optional:file:/vault/secrets/db-creds"
  datasource:
    # spring.datasource.password is now set from /vault/secrets/db-creds
    url: ${db.url}
    password: ${db.password}
```

Or read directly from the file path:
```java
@Value("#{T(java.nio.file.Files).readString(T(java.nio.file.Path).of('/vault/secrets/api-key')).strip()}")
private String apiKey;
```

### Verify Injection Is Working

```bash
# Describe the pod — you should see the vault-agent-init and vault-agent containers
kubectl describe pod myapp-0 -n production | grep -A5 "Containers:"
# Containers:
#   vault-agent-init:   ← init container that fetches secrets before app starts
#   myapp:              ← your app, starts after vault-agent-init completes
#   vault-agent:        ← sidecar that renews the Vault token and refreshes secrets

# Verify the secret file was written
kubectl exec -it myapp-0 -n production -c myapp -- cat /vault/secrets/db-creds
# db.password=SuperSecret123
# db.url=jdbc:postgresql://postgres:5432/appdb
```

---

## Chapter 8: Image Security

### Why Image Security Matters

A container image is the unit of deployment. If the image contains malware, outdated libraries with known CVEs, or was built from a tampered base image, every pod running it is compromised from the moment it starts. All the RBAC, network policies, and pod security contexts in the world don't help if the application binary itself is the threat.

Image security has four layers: signing (prove the image came from a trusted source), scanning (find known vulnerabilities), benchmarking (verify cluster configuration), and runtime monitoring (detect attacks as they happen).

### Image Signing with Cosign

Cosign signs and verifies container images. Once your CI/CD pipeline signs images, you can configure your cluster to reject any unsigned image — preventing deployment of images that didn't go through your pipeline.

```bash
# Install cosign
brew install cosign   # macOS
# Or download from: https://github.com/sigstore/cosign/releases

# Generate a signing key pair
cosign generate-key-pair
# Enter password for private key: ...
# Private key written to cosign.key
# Public key written to cosign.pub

# Sign an image (run in CI/CD after pushing)
cosign sign \
  --key cosign.key \
  myregistry.com/myapp:v1.2.3

# Verify a signature (run before deployment, or enforce via policy)
cosign verify \
  --key cosign.pub \
  myregistry.com/myapp:v1.2.3
# Verification for myregistry.com/myapp:v1.2.3 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - Existence of the claims in the transparency log was verified online
#   - The signatures were verified against the specified public key

# Keyless signing with OIDC (no key management needed — uses GitHub Actions OIDC)
# In GitHub Actions:
cosign sign \
  --yes \       # Agree to Rekor transparency log
  myregistry.com/myapp:v1.2.3
# Uses OIDC token from GitHub Actions — no key stored, identity tied to GitHub workflow
```

**Enforce image verification with Kyverno or Gatekeeper:**

```yaml
# policy-verify-signature.yaml (Kyverno)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: verify-myregistry-images
    match:
      any:
      - resources:
          kinds: [Pod]
          namespaces: [production, staging]
    verifyImages:
    - imageReferences:
      - "myregistry.com/myapp:*"
      attestors:
      - entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
              -----END PUBLIC KEY-----
```

### Vulnerability Scanning with Trivy

Trivy scans images for CVEs in OS packages, language-specific libraries, and infrastructure-as-code files:

```bash
# Scan a local or remote image
trivy image myregistry.com/myapp:v1.2.3

# Fail on CRITICAL CVEs only (for CI/CD gates)
trivy image \
  --exit-code 1 \
  --severity CRITICAL \
  --ignore-unfixed \    # Ignore CVEs with no available fix
  myregistry.com/myapp:v1.2.3

# Output results as SARIF (for GitHub Security tab)
trivy image \
  --format sarif \
  --output trivy-results.sarif \
  myregistry.com/myapp:v1.2.3
```

**Automated scanning CronJob — scan running images weekly:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: image-scanner
  namespace: security
spec:
  schedule: "0 3 * * 1"   # Every Monday at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: scanner-sa  # Needs permission to list pods cluster-wide
          containers:
          - name: trivy-scanner
            image: aquasec/trivy:latest
            command:
            - /bin/sh
            - -c
            - |
              # Get all unique images running in the cluster
              IMAGES=$(kubectl get pods -A \
                -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
                | sort -u)

              echo "Scanning $(echo "$IMAGES" | wc -l) images..."
              FAILED=0

              while IFS= read -r IMAGE; do
                echo "Scanning: $IMAGE"
                if trivy image \
                  --exit-code 1 \
                  --severity CRITICAL \
                  --ignore-unfixed \
                  --quiet \
                  "$IMAGE"; then
                  echo "✓ $IMAGE — no critical CVEs"
                else
                  echo "✗ $IMAGE — CRITICAL CVEs found!"
                  FAILED=1
                fi
              done <<< "$IMAGES"

              if [ "$FAILED" = "1" ]; then
                # Send alert — this is where you'd call a webhook or send email
                echo "CRITICAL CVEs found in running images — see above"
                exit 1
              fi
```

### kube-bench — CIS Benchmark Compliance

kube-bench checks your cluster configuration against the CIS Kubernetes Benchmark — a set of security best practices for cluster configuration, API server settings, etcd, scheduler, and node configuration:

```bash
# Run kube-bench as a pod (needs privileged access to read host config files)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# View results
kubectl logs job/kube-bench
# [INFO] 1 Master Node Security Configuration
# [INFO] 1.1 API Server
# [PASS] 1.1.1 Ensure that the API server pod spec file permissions are set to 600 or more restrictive
# [FAIL] 1.1.2 Ensure that the API server pod spec file ownership is set to root:root
# [PASS] 1.1.3 Ensure that the controller manager pod spec file permissions are set to 600 or more restrictive
# ...
# [WARN] 1.2.1 Ensure that the --anonymous-auth argument is set to false
# ...
# == Summary ==
# 47 checks PASS
# 12 checks FAIL
# 22 checks WARN
# 0 checks INFO
```

For managed Kubernetes (EKS, GKE, AKS), you can only check the node-level and policy checks — control plane components are managed by the cloud provider:

```bash
# Run only node-level checks on EKS/GKE/AKS
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-node.yaml
```

### Falco — Runtime Security Monitoring

Falco uses eBPF or kernel module to monitor system calls at runtime and alert when suspicious activity occurs — a shell spawned inside a container, `/etc/passwd` read, sensitive file access, privilege escalation attempts.

The key distinction from everything else in this chapter: Falco catches attacks that are already happening, not just prevents potential vulnerabilities. An attacker who bypasses all your admission controls and is actively running commands inside a container will be detected by Falco.

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \          # eBPF is safer than kernel module, preferred on modern kernels
  --set falcosidekick.enabled=true \  # Sidekick routes alerts to Slack, PagerDuty, etc.
  --set falcosidekick.config.slack.webhookurl=https://hooks.slack.com/services/...

kubectl get pods -n falco
# NAME                            READY   STATUS
# falco-abcd1234-node1            2/2     Running   ← one pod per node (DaemonSet)
# falco-abcd1234-node2            2/2     Running
# falco-falcosidekick-xyz-1       1/1     Running
```

**What Falco detects out of the box:**

```yaml
# Examples of built-in Falco rules that fire automatically:

# A shell is spawned in a running container
- rule: Terminal shell in container
  condition: evt.type = execve and container.id != host and proc.name in (shell_binaries)
  output: Shell spawned in container (user=%user.name container=%container.name image=%container.image.repository)
  priority: NOTICE

# A sensitive file is read (e.g., /etc/shadow, /etc/passwd)
- rule: Read sensitive file
  condition: open_read and sensitive_files and proc.name not in (known_read_sensitive_files_binaries)
  output: Sensitive file opened for reading (user=%user.name file=%fd.name)
  priority: WARNING

# A process writes to /etc (unexpected for applications)
- rule: Write below etc
  condition: open_write and fd.name startswith /etc and proc.name not in (known_etc_writers)
  output: File written below /etc (user=%user.name command=%proc.cmdline file=%fd.name)
  priority: ERROR

# Outbound connection from unexpected binary
- rule: Unexpected outbound connection
  condition: outbound and proc.name not in (known_server_binaries) and container
  output: Unexpected outbound connection (container=%container.name command=%proc.cmdline ip=%fd.sip)
  priority: NOTICE
```

When any of these fire, Falco sends a structured alert to your configured sinks — Slack, PagerDuty, Elasticsearch, webhooks. You get a notification within seconds of a suspicious event.

---

## Chapter 9: Security Hardening Checklist

### Complete Security-Hardened Helm Values

```yaml
# values-prod-hardened.yaml

image:
  repository: myregistry.com/myapp   # Company registry only (enforced by Gatekeeper)
  tag: "v1.2.3"
  pullPolicy: Always

serviceAccount:
  create: true
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-prod
  automountServiceAccountToken: false  # No Kubernetes API token mounted

# Pod-level security context
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

# Container-level security context
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]

# emptyDir volumes for temp directories (required with readOnlyRootFilesystem)
extraVolumes:
  - name: tmp
    emptyDir: {}
  - name: spring-tmp
    emptyDir: {}

extraVolumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: spring-tmp
    mountPath: /app/tmp

# Vault Agent injection (alternative to External Secrets)
podAnnotations:
  vault.hashicorp.com/agent-inject: "true"
  vault.hashicorp.com/role: "myapp-role"
  vault.hashicorp.com/agent-inject-secret-app-secrets: "secret/data/production/myapp"

# Resource limits (required — no limits = noisy neighbor risk)
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

# Network policies via chart template
networkPolicy:
  enabled: true

# Image pull secret for private registry
imagePullSecrets:
  - name: regcred
```

### Security Audit Commands

Run these regularly (and before releasing to production) to catch misconfigurations:

```bash
# ── RBAC Audit ────────────────────────────────────────────────────────────────

# Find service accounts with wildcard permissions (the most dangerous pattern)
kubectl get clusterroles -o json | \
  jq '.items[] | select(.rules[].verbs[] == "*") | .metadata.name'

# Find pods using the default service account (should have custom SAs)
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{" "}{.spec.serviceAccountName}{"\n"}{end}' \
  | grep " default$"

# ── Pod Security Audit ────────────────────────────────────────────────────────

# Find containers running as root (missing runAsNonRoot)
kubectl get pods -A -o json | \
  jq '.items[] | select(
    .spec.containers[].securityContext.runAsNonRoot != true and
    .spec.securityContext.runAsNonRoot != true
  ) | .metadata.namespace + "/" + .metadata.name'

# Find privileged containers
kubectl get pods -A -o json | \
  jq '.items[] | select(
    .spec.containers[].securityContext.privileged == true
  ) | .metadata.namespace + "/" + .metadata.name'

# Find pods with automounted service account tokens
kubectl get pods -A -o json | \
  jq '.items[] | select(
    .spec.automountServiceAccountToken != false
  ) | .metadata.namespace + "/" + .metadata.name' | head -20

# ── Secrets Audit ─────────────────────────────────────────────────────────────

# Find secrets exposed as environment variables (mounted files are safer)
kubectl get pods -A -o json | \
  jq '.items[] | select(
    .spec.containers[].env[].valueFrom.secretKeyRef != null
  ) | .metadata.namespace + "/" + .metadata.name'

# ── Image Audit ───────────────────────────────────────────────────────────────

# Find containers using :latest tag
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | grep ":latest"

# Find containers from untrusted registries (not from your company registry)
kubectl get pods -A \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u | grep -v "^myregistry.com/"

# ── Namespace Security ────────────────────────────────────────────────────────

# Check which namespaces have Pod Security Standards labels
kubectl get ns -o json | \
  jq '.items[] | {
    name: .metadata.name,
    enforce: .metadata.labels["pod-security.kubernetes.io/enforce"] // "none"
  }'
```

### The Security Checklist

**RBAC:**
- [ ] No wildcard permissions (`verbs: ["*"]`) in any Role or ClusterRole used by applications
- [ ] Every pod has a dedicated ServiceAccount, not `default`
- [ ] `automountServiceAccountToken: false` on all pods that don't call the Kubernetes API
- [ ] CI/CD service account cannot delete resources
- [ ] `kubectl auth can-i` tested for all service accounts

**Pod Security:**
- [ ] `runAsNonRoot: true` on all pods
- [ ] `readOnlyRootFilesystem: true` with `emptyDir` volumes for temp directories
- [ ] `allowPrivilegeEscalation: false` on all containers
- [ ] `capabilities: drop: ["ALL"]` on all containers
- [ ] `seccompProfile: type: RuntimeDefault` on all pods
- [ ] Pod Security Standards `enforce: restricted` on production namespace

**Network:**
- [ ] Default-deny network policy in all production namespaces
- [ ] DNS allow policy present alongside every default-deny
- [ ] Each service has explicit allow rules — nothing implicit
- [ ] Inter-namespace access blocked unless explicitly needed

**Secrets:**
- [ ] No secrets in environment variables (use mounted files)
- [ ] No secret YAML in Git (use External Secrets, Sealed Secrets, or Vault)
- [ ] Vault or External Secrets Operator in use for production
- [ ] Secret access audited regularly

**Images:**
- [ ] All images signed with Cosign
- [ ] Image signature enforcement via Kyverno or Gatekeeper
- [ ] Trivy scanning in CI/CD pipeline (blocks on CRITICAL)
- [ ] Weekly automated scan of running images
- [ ] No `latest` tags in production
- [ ] Images only from trusted registries (enforced by Gatekeeper)

**Runtime:**
- [ ] Falco installed and connected to alerting
- [ ] kube-bench run and critical findings addressed
- [ ] Gatekeeper installed with required labels, registry, and privilege constraints
- [ ] Audit mode enabled for all new constraints before switching to deny

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `Forbidden` when pod calls Kubernetes API | Service account lacks RBAC permission | `kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa>` | Add the required verb/resource to the SA's Role |
| Pod rejected by Pod Security Standards | Pod spec violates the namespace's enforcement level | `kubectl describe pod <n>` — error message lists the violation | Fix the security context: add `runAsNonRoot`, `readOnlyRootFilesystem`, etc. |
| App crashes with `read-only file system` | `readOnlyRootFilesystem: true` and app writes to a path without `emptyDir` | `kubectl logs <pod>` | Mount an `emptyDir` volume at the path the app writes to |
| Vault Agent sidecar `CrashLoopBackOff` | Wrong role name, wrong SA, or policy doesn't grant access to the path | `kubectl logs <pod> -c vault-agent-init` | Check role binding: `vault write auth/kubernetes/role/myapp-role ...`; verify policy allows the secret path |
| `Error from server (Forbidden): pods is forbidden: violates PodSecurity` | PSA `enforce` level active on namespace | `kubectl describe ns <n>` | Fix pod security context to comply with the level, or temporarily use `warn` during migration |
| Gatekeeper blocks compliant pod | Policy has a bug or the pod is actually non-compliant | `kubectl describe constraint <n>` — shows violations with details | Test with `enforcementAction: warn` first; check Rego logic with OPA playground |
| IRSA not working (no AWS credentials) | SA annotation missing, OIDC provider not registered, trust policy wrong | `kubectl exec <pod> -- env \| grep AWS` | Verify SA annotation; check `aws iam get-role --role-name <n>` trust policy matches cluster OIDC URL |
| Falco not detecting events | Wrong driver kind for kernel version, or eBPF not supported | `kubectl logs -n falco <falco-pod>` | Try `driver.kind=module` instead of `ebpf` (or vice versa); check kernel version compatibility |
| `cosign verify` fails | Image not signed, wrong public key, or signature expired | `cosign verify --key cosign.pub <image>` — shows error detail | Re-sign the image in CI/CD; check the public key matches the private key used to sign |
| `kubectl auth can-i` returns unexpected result | RoleBinding in wrong namespace, or ClusterRoleBinding instead of RoleBinding | `kubectl get rolebinding,clusterrolebinding -A \| grep <sa>` | Check binding scope: namespace-level permissions need RoleBinding in the right namespace |

---

## Practice Exercises

**Exercise 1 — RBAC least privilege:**
Create a service account `ci-sa` in the `staging` namespace. Write a Role that allows it to update Deployments and read Pods and Secrets, but NOT list or delete Secrets and NOT create new Deployments. Test every permission with `kubectl auth can-i --as=system:serviceaccount:staging:ci-sa`. Verify you cannot do what's not permitted and can do what is.

**Exercise 2 — Pod security hardening:**
Take your `hello-app` Deployment from Part 1. Add the full security context: `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, `capabilities: drop: ALL`, `seccompProfile: RuntimeDefault`. Add `emptyDir` volumes for `/tmp`. Apply Pod Security Standards `enforce: restricted` to the namespace. Deploy and verify the pod starts. Then try creating a privileged pod in the same namespace — observe the rejection.

**Exercise 3 — OPA Gatekeeper required labels:**
Install Gatekeeper. Deploy the `K8sRequiredLabels` ConstraintTemplate and Constraint from Chapter 6. Start with `enforcementAction: warn`. Check `kubectl get constraints` to see existing violations. Fix your existing Deployments to have the `team` and `environment` labels. Switch to `enforcementAction: deny`. Verify that a new Deployment without the labels is rejected with a clear error message.

**Exercise 4 — Vault secrets injection:**
Install Vault in dev mode. Create a secret at `secret/production/myapp`. Configure the Kubernetes auth method bound to a service account in your namespace. Annotate the `hello-app` Deployment with the Vault injection annotations. Redeploy and verify: (1) the pod has the `vault-agent-init` and `vault-agent` containers, (2) the secret file exists at `/vault/secrets/`, (3) your application can read it.

**Exercise 5 — End-to-end security audit:**
Run the security audit commands from Chapter 9 against your cluster. For each finding: identify what the risk is, and fix it. Target: zero pods running as root, zero pods using the `default` service account, zero `latest` tags in production, no secrets exposed as environment variables. Run the audit commands again after fixing. Treat any remaining finding as a homework item with a written explanation of why it's acceptable or how you'll address it.
