Excellent! Let me create **Part 7: Security Deep Dive** - this will cover securing your Kubernetes cluster from the ground up.

## Part 7: Security Deep Dive - Zero-Trust Kubernetes

### Prerequisites
- Completed Parts 1-6 (or equivalent experience)
- Kubernetes cluster (EKS, GKE, AKS, or Minikube)
- `kubectl` and `helm` installed
- Basic understanding of authentication/authorization

### What You'll Learn
- ✅ RBAC (Role-Based Access Control) - Who can do what
- ✅ Service Accounts - Identity for pods
- ✅ Pod Security Standards - Securing workloads
- ✅ Network Policies - Zero-trust networking (detailed)
- ✅ OPA (Open Policy Agent) - Fine-grained policies
- ✅ Secrets management with HashiCorp Vault
- ✅ Image signing and verification with Cosign
- ✅ Security context and PodSecurityContext

---

## Chapter 1: Security Layers in Kubernetes

### The Defense-in-Depth Model

```
┌─────────────────────────────────────────────────────────────┐
│                    Layer 8: Policies & Governance           │
│                    (OPA, Kyverno, Admission Controllers)     │
├─────────────────────────────────────────────────────────────┤
│                    Layer 7: Workload Security               │
│                    (Pod Security, Service Accounts)         │
├─────────────────────────────────────────────────────────────┤
│                    Layer 6: Network Security                │
│                    (Network Policies, mTLS)                 │
├─────────────────────────────────────────────────────────────┤
│                    Layer 5: Authentication & Authorization  │
│                    (RBAC, Certificates, OIDC)               │
├─────────────────────────────────────────────────────────────┤
│                    Layer 4: Container Security              │
│                    (Image scanning, non-root)               │
├─────────────────────────────────────────────────────────────┤
│                    Layer 3: Node Security                   │
│                    (OS hardening, updates)                  │
├─────────────────────────────────────────────────────────────┤
│                    Layer 2: Cluster Security                │
│                    (API server, etcd encryption)            │
├─────────────────────────────────────────────────────────────┤
│                    Layer 1: Physical Security               │
│                    (Cloud provider, datacenter)             │
└─────────────────────────────────────────────────────────────┘
```

### Common Security Mistakes

| Mistake | Impact | Solution |
|---------|--------|----------|
| **Running as root** | Container escape | Use non-root user |
| **Default service account** | Over-privileged pods | Create least-privilege SA |
| **No network policies** | East-west traffic unsecured | Default deny all |
| **Secrets in env vars** | Exposure in logs | Mount as volumes |
| **No image scanning** | Vulnerable images | Integrate Trivy/Clair |
| **Weak RBAC** | Privilege escalation | Least privilege principle |

---

## Chapter 2: RBAC - Who Can Do What

### RBAC Components

**Analogy - Office Building Security:**
- **Subject** = Person (User or Service Account)
- **Role** = Job title (Manager, Employee, Guest)
- **RoleBinding** = Assigning person to job title
- **ClusterRole** = Building-wide permissions

### Core RBAC Objects

```yaml
# 1. Role (Namespace-scoped permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: development
rules:
- apiGroups: [""]  # Core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# 2. RoleBinding (Bind Role to users)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
- kind: User
  name: alice@example.com
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: myapp-sa
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# 3. ClusterRole (Cluster-wide permissions)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]
  # Can't access secrets in kube-system!
  resourceNames: ["myapp-secret"]  # Specific secrets only

---
# 4. ClusterRoleBinding (Bind at cluster level)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: security-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

### RBAC Best Practices

**Least Privilege Examples:**

```yaml
# ✅ GOOD: Read-only access for monitoring
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]

---
# ✅ GOOD: CI/CD deployment (limited to specific namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cicd-deployer
  namespace: production
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets"]
  verbs: ["get", "update", "patch"]
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "create", "update", "patch"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "update", "patch"]
# ❌ Cannot delete resources!

---
# ❌ BAD: Wildcard permissions (security nightmare!)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: admin
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]  # Too permissive!
```

### Testing RBAC

```bash
# Check if user can list pods
kubectl auth can-i list pods --as=alice@example.com

# Check if user can create deployments in production
kubectl auth can-i create deployments --as=alice@example.com -n production

# Check all permissions for a user
kubectl auth can-i --list --as=alice@example.com

# Simulate with dry-run
kubectl create deployment test --image=nginx --dry-run=client -o yaml | \
  kubectl auth can-i --as=alice@example.com -f -

# Check service account permissions
kubectl auth can-i list pods --as=system:serviceaccount:default:myapp-sa
```

### Creating Users (Without External Auth)

```bash
# For testing/local clusters (not production!)
# Create user certificate
openssl genrsa -out alice.key 2048
openssl req -new -key alice.key -out alice.csr -subj "/CN=alice/O=developers"
openssl x509 -req -in alice.csr -CA ~/.minikube/ca.crt -CAkey ~/.minikube/ca.key -CAcreateserial -out alice.crt -days 365

# Set user context
kubectl config set-credentials alice --client-certificate=alice.crt --client-key=alice.key
kubectl config set-context alice-context --cluster=minikube --user=alice
kubectl config use-context alice-context

# Now test
kubectl get pods  # Should be denied until RBAC applied
```

---

## Chapter 3: Service Accounts - Pod Identity

### Service Account Types

| Type | Scope | Use Case | Example |
|------|-------|----------|---------|
| **Default** | Every namespace | Basic API access | Minimal permissions |
| **Custom** | Per namespace | Application identity | MyApp service account |
| **External** | Cluster-wide | Cloud provider access | AWS IAM role |

### Creating and Using Service Accounts

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-role  # AWS IAM
    iam.gke.io/gcp-service-account: myapp@project.iam.gserviceaccount.com  # GCP
---
apiVersion: v1
kind: Secret
metadata:
  name: myapp-sa-token
  annotations:
    kubernetes.io/service-account.name: myapp-sa
type: kubernetes.io/service-account-token
---
# Bind role to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-sa-binding
  namespace: production
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: myapp-role
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
```

**Using Service Account in Pod:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: myapp-sa  # Use custom SA
      containers:
      - name: app
        image: myapp:latest
        # Pod can now access K8s API with SA permissions
```

### AWS IAM Roles for Service Accounts (IRSA)

```bash
# 1. Create IAM role with trust policy
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789:oidc-provider/oidc.eks.region.amazonaws.com/id/EXAMPLE"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.region.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:production:myapp-sa"
        }
      }
    }
  ]
}
EOF

# 2. Create role
aws iam create-role --role-name myapp-s3-role --assume-role-policy-document file://trust-policy.json

# 3. Attach policy
aws iam attach-role-policy --role-name myapp-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 4. Annotate service account
kubectl annotate serviceaccount myapp-sa -n production \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789:role/myapp-s3-role

# Pod now has AWS credentials automatically!
```

### Pod Identity with Azure

```yaml
# Azure AD Pod Identity
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: myapp-identity
spec:
  type: 0  # Managed Service Identity
  resourceID: /subscriptions/xxx/resourcegroups/mygroup/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myapp-id
  clientID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
---
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: myapp-binding
spec:
  azureIdentity: myapp-identity
  selector: myapp-selector
```

---

## Chapter 4: Pod Security Standards

### Pod Security Levels

| Level | Description | Use Case |
|-------|-------------|----------|
| **Privileged** | No restrictions | System components, CI/CD |
| **Baseline** | Minimal restrictions | Most workloads |
| **Restricted** | Hardened security | Production, compliance |

### Pod Security Context

```yaml
# security-context.yaml - Pod-level settings
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true        # Don't run as root
    runAsUser: 1000           # Specific UID
    runAsGroup: 3000          # Specific GID
    fsGroup: 2000             # File system group
    seccompProfile:
      type: RuntimeDefault    # Use default seccomp
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false  # No setuid binaries
      capabilities:
        drop:
        - ALL                    # Drop all capabilities
        add:
        - NET_BIND_SERVICE       # Only allow port binding
      readOnlyRootFilesystem: true  # Read-only root FS
      privileged: false
```

### Pod Security Admission (PSA) - Native Solution

**Enforce at namespace level:**

```yaml
# namespace-with-psa.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

**Testing PSA:**

```bash
# Create namespace with restricted policy
kubectl create ns restricted-demo
kubectl label ns restricted-demo pod-security.kubernetes.io/enforce=restricted

# Try to run privileged pod (will be denied)
kubectl run privileged-pod --image=nginx --privileged -n restricted-demo
# Error: pods "privileged-pod" is forbidden: violates PodSecurity "restricted:latest"

# Run compliant pod (succeeds)
kubectl run secure-pod -n restricted-demo --image=nginx --overrides='
{
  "spec": {
    "securityContext": {
      "runAsNonRoot": true,
      "runAsUser": 1000
    },
    "containers": [{
      "name": "nginx",
      "image": "nginx",
      "securityContext": {
        "allowPrivilegeEscalation": false,
        "capabilities": {"drop": ["ALL"]}
      }
    }]
  }
}'
```

### Pod Security Standards in Helm

```yaml
# templates/pod-security.yaml in your Helm chart
{{- if .Values.podSecurity.enabled }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Release.Namespace }}
  labels:
    pod-security.kubernetes.io/enforce: {{ .Values.podSecurity.level }}
    pod-security.kubernetes.io/audit: {{ .Values.podSecurity.level }}
    pod-security.kubernetes.io/warn: {{ .Values.podSecurity.level }}
{{- end }}

# values.yaml
podSecurity:
  enabled: true
  level: restricted  # or baseline, privileged
```

---

## Chapter 5: Network Policies - Zero-Trust Networking (Deep Dive)

### Network Policy Components

```yaml
# Complete network policy example
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
      tier: backend
  
  policyTypes:
  - Ingress
  - Egress
  
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
          tier: frontend
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.0.1.0/24
    ports:
    - protocol: TCP
      port: 8080
  
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Multi-Tier Application Policies

```yaml
# 1. Default deny all traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# 2. Allow web to API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    ports:
    - port: 8080
---
# 3. Allow API to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - port: 5432
---
# 4. Allow egress to internet (only HTTPS)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-internet
spec:
  podSelector:
    matchLabels:
      app: api
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 10.0.0.0/8
        - 172.16.0.0/12
        - 192.168.0.0/16
    ports:
    - protocol: TCP
      port: 443
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
```

### Advanced Network Policy Patterns

**Isolate Namespace Completely:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []  # No ingress
  egress: []   # No egress
```

**Allow Monitoring Only:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9090   # Prometheus
    - port: 9102   # Metrics
```

**Allow GitLab CI/CD:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gitlab-runner
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: gitlab-runner
    ports:
    - port: 8080
    - port: 5005   # Debug port
```

### Testing Network Policies

```bash
# 1. Create test pods
kubectl run test-web --image=nginx --labels="app=web"
kubectl run test-api --image=nginx --labels="app=api"
kubectl run test-db --image=postgres --labels="app=database"

# 2. Test connectivity
kubectl exec test-web -- curl -v test-api:8080  # Should work
kubectl exec test-web -- curl -v test-db:5432   # Should fail

# 3. Check policy status
kubectl get networkpolicies
kubectl describe networkpolicy allow-web-to-api

# 4. Debug with temporary policy
kubectl run debug-pod --image=nicolaka/netshoot --rm -it -- bash
# Inside pod:
curl test-api:8080
nslookup google.com
```

---

## Chapter 6: OPA (Open Policy Agent) - Fine-Grained Policies

### Why OPA?

**Beyond RBAC and Network Policies:**

| Requirement | RBAC | NetworkPolicy | OPA |
|-------------|------|---------------|-----|
| "Only allow images from trusted registry" | ❌ | ❌ | ✅ |
| "Require specific labels on all resources" | ❌ | ❌ | ✅ |
| "No privileged containers in production" | ❌ | ❌ | ✅ |
| "Only allow deployments during business hours" | ❌ | ❌ | ✅ |
| "Require approval for large resource requests" | ❌ | ❌ | ✅ |

### Installing OPA Gatekeeper

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.12/deploy/gatekeeper.yaml

# Verify installation
kubectl get pods -n gatekeeper-system
# gatekeeper-controller-manager-xxx   1/1     Running
# gatekeeper-audit-xxx                1/1     Running
```

### OPA Constraint Templates

**Template 1: Require Labels**

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
        kind: K8sRequiredLabels
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
        
        violation[{"msg": msg, "details": {"missing_labels": missing}}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_]}
          missing := required - provided
          count(missing) > 0
          msg := sprintf("You must provide labels: %v", [missing])
        }
```

**Constraint (Apply the policy):**

```yaml
# constraint-required-labels.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "production"
      - "staging"
  parameters:
    labels:
      - "team"
      - "environment"
```

**Test the constraint:**

```bash
# Create pod without required labels (should be denied)
kubectl run test-pod --image=nginx -n production
# Error: admission webhook "validation.gatekeeper.sh" denied request: 
# You must provide labels: {"environment", "team"}

# Create pod with labels (succeeds)
kubectl run test-pod --image=nginx -n production \
  --labels="team=platform,environment=production"
# Pod created successfully
```

### Template 2: No Privileged Containers

```yaml
# constrainttemplate-no-privileged.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblockprivileged
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          container.securityContext.privileged == true
          msg := sprintf("Privileged container is not allowed: %v", [container.name])
        }
        
        violation[{"msg": msg}] {
          input.review.object.spec.securityContext.privileged == true
          msg := "Privileged pod is not allowed"
        }

---
# constraint-no-privileged.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - "production"
```

### Template 3: Registry Restriction

```yaml
# constrainttemplate-allowed-registry.yaml
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
          image := container.image
          not startswith(image, input.parameters.repos[_])
          msg := sprintf("Image %v not allowed from registry. Allowed repos: %v", [image, input.parameters.repos])
        }

---
# constraint-allowed-repos.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allow-trusted-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    repos:
      - "myregistry.com/"
      - "docker.io/trusted/"
```

### Template 4: Business Hours Only

```yaml
# constrainttemplate-business-hours.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sdeploymentwindow
spec:
  crd:
    spec:
      names:
        kind: K8sDeploymentWindow
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdeploymentwindow
        
        violation[{"msg": msg}] {
          # Get current hour (UTC)
          time := time.now_ns()
          hour := time / 3600000000000 % 24
          
          # Block deployments outside 9 AM - 5 PM
          not hour >= 9
          not hour <= 17
          
          msg := sprintf("Deployments only allowed between 9 AM and 5 PM UTC. Current hour: %v", [hour])
        }

---
# constraint-deployment-window.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDeploymentWindow
metadata:
  name: deployment-window
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
```

### OPA Audit and Reporting

```bash
# Check all constraints
kubectl get constraints

# Check violations
kubectl get k8srequiredlabels -o yaml
# status:
#   auditTimestamp: "2024-01-15T10:30:00Z"
#   totalViolations: 3
#   violations:
#   - enforcementAction: deny
#     kind: Pod
#     message: You must provide labels: team
#     name: test-pod
#     namespace: production

# Run manual audit
kubectl label ns production audit=true

# View OPA metrics
kubectl get services -n gatekeeper-system
kubectl port-forward svc/gatekeeper-webhook-service 8888:443 -n gatekeeper-system
# Metrics available at: https://localhost:8888/metrics
```

---

## Chapter 7: Advanced Security Patterns

### Pattern 1: Image Signing with Cosign

```bash
# 1. Generate key pair
cosign generate-key-pair

# 2. Sign your image
cosign sign --key cosign.key myregistry.com/myapp:v1.2.3

# 3. Verify signature
cosign verify --key cosign.pub myregistry.com/myapp:v1.2.3

# 4. In Kubernetes, use policy controller
# Install cosign policy controller
kubectl apply -f https://raw.githubusercontent.com/sigstore/cosign/main/third_party/connaisseur/deploy/connaisseur.yaml

# Require signed images
apiVersion: v1
kind: Namespace
metadata:
  name: production
  annotations:
    securesign.kubernetes.io/signed: "true"
```

### Pattern 2: Secrets Management with Vault

```bash
# Install Vault
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set "server.dev.enabled=true"  # Dev mode (not for prod!)

# Configure Vault
kubectl exec -it vault-0 -n vault -- vault secrets enable -path=secret kv-v2
kubectl exec -it vault-0 -n vault -- vault kv put secret/myapp DB_PASSWORD="S3cr3tP@ss"

# Enable Kubernetes auth
kubectl exec -it vault-0 -n vault -- vault auth enable kubernetes
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"

# Create Vault policy
kubectl exec -it vault-0 -n vault -- vault policy write myapp - <<EOF
path "secret/data/myapp" {
  capabilities = ["read"]
}
EOF

# Create service account and role
kubectl create sa myapp-sa -n production
kubectl exec -it vault-0 -n vault -- vault write auth/kubernetes/role/myapp \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=production \
  policies=myapp \
  ttl=24h
```

**Pod with Vault Agent:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-database: "secret/data/myapp"
        vault.hashicorp.com/agent-inject-template-database: |
          {{- with secret "secret/data/myapp" -}}
          export DB_PASSWORD="{{ .Data.data.DB_PASSWORD }}"
          {{- end }}
      containers:
      - name: app
        image: myapp:latest
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: vault-database
              key: DB_PASSWORD
```

### Pattern 3: Pod Identity with SPIFFE/SPIRE

```bash
# Install SPIRE
helm repo add spire https://spiffe.io/helm-charts
helm install spire spire/spire \
  --namespace spire \
  --create-namespace

# Register workload
kubectl exec -n spire spire-server-0 -- \
  /opt/spire/bin/spire-server entry create \
  -spiffeID spiffe://example.org/myapp \
  -parentID spiffe://example.org/agent \
  -selector k8s:sa:myapp-sa \
  -selector k8s:ns:production

# Pod gets SPIFFE identity
# Use for mTLS between services
```

---

## Chapter 8: Security Audit and Compliance

### CIS Benchmark Checking with kube-bench

```bash
# Run kube-bench (security scan)
docker run --pid=host -v /etc:/etc:ro -v /var:/var:ro \
  aquasec/kube-bench:latest

# Install as job in cluster
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Check results
kubectl logs job.batch/kube-bbench-master
```

### Vulnerability Scanning with Trivy

```yaml
# cronjob-trivy-scan.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: trivy-scan
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: scanner
            image: aquasec/trivy:latest
            command:
            - /bin/sh
            - -c
            - |
              # Scan all running images
              kubectl get pods -A -o jsonpath='{.items[*].spec.containers[*].image}' | \
                tr ' ' '\n' | sort -u | \
                while read image; do
                  trivy image --severity HIGH,CRITICAL --exit-code 0 $image
                done
          serviceAccountName: image-scanner
          restartPolicy: Never
```

### Runtime Security with Falco

```bash
# Install Falco
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set ebpf.enabled=true

# Falco detects suspicious behavior:
# - Shell spawned in container
# - Read sensitive files (/etc/shadow)
# - Outbound network connections
# - Package management commands

# View alerts
kubectl logs -n falco -l app.kubernetes.io/name=falco
```

---

## Chapter 9: Complete Security Hardened Deployment

### Production-Ready Security Configuration

**`values-prod-secure.yaml`:**
```yaml
# Complete security-hardened values
global:
  environment: production
  
# Pod Security
podSecurity:
  enabled: true
  level: restricted
  audit: true

# Service Account
serviceAccount:
  create: true
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-role

# Security Context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

# Container Security
containerSecurityContext:
  allowPrivilegeEscalation: false
  privileged: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE

# Network Policies
networkPolicies:
  enabled: true
  defaultDeny: true
  allow:
    ingress:
      - fromIngressController: true
      - fromMonitoring: true
    egress:
      - toDNS: true
      - toAPI: true

# OPA Constraints
opa:
  enabled: true
  constraints:
    - requireLabels: ["team", "environment"]
    - blockPrivileged: true
    - allowedRegistries: ["myregistry.com/"]
    - deploymentWindow:
        startHour: 9
        endHour: 17
        timezone: UTC

# Image Security
image:
  repository: myregistry.com/myapp
  tag: v1.2.3
  pullPolicy: Always
  pullSecret: regcred
  signature:
    enabled: true
    publicKey: cosign.pub

# Secrets
secrets:
  manager: vault
  vault:
    enabled: true
    role: myapp
    path: secret/data/myapp

# Runtime Security
runtimeSecurity:
  falco:
    enabled: true
    rules:
      - shell_in_container
      - read_sensitive_file
      - outbound_connection
```

**Deploy Security Hardened App:**

```bash
# 1. Apply baseline security
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# 2. Install OPA Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.12/deploy/gatekeeper.yaml

# 3. Apply constraints
kubectl apply -f constraint-required-labels.yaml
kubectl apply -f constraint-no-privileged.yaml

# 4. Install Falco
helm install falco falcosecurity/falco --set ebpf.enabled=true

# 5. Deploy app with security context
helm upgrade --install myapp ./helm-chart -f values-prod-secure.yaml

# 6. Verify security
kubectl auth can-i --list --as=system:serviceaccount:production:myapp-sa
kubectl get networkpolicies
kubectl get constraints
kubectl get pods -n falco
```

---

## Summary: Security Checklist

| Security Layer | Tool/Feature | Status |
|----------------|--------------|--------|
| **Authentication** | RBAC, OIDC | ✅ |
| **Authorization** | RBAC, ABAC | ✅ |
| **Pod Identity** | Service Accounts | ✅ |
| **Pod Security** | PSA, SecurityContext | ✅ |
| **Network Security** | Network Policies | ✅ |
| **Policy Enforcement** | OPA Gatekeeper | ✅ |
| **Image Security** | Cosign signatures | ✅ |
| **Secret Management** | Vault | ✅ |
| **Runtime Security** | Falco | ✅ |
| **Compliance** | kube-bench | ✅ |

## Security Incident Response

```bash
# 1. Audit recent changes
kubectl get events -A --sort-by='.lastTimestamp'

# 2. Check for privilege escalation
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged==true)'

# 3. Investigate pod access
kubectl logs suspicious-pod --previous
kubectl exec suspicious-pod -- cat /proc/self/status | grep -E "Uid|Gid"

# 4. Isolate namespace
kubectl label ns production pod-security.kubernetes.io/enforce=restricted
kubectl delete networkpolicy --all -n production
kubectl apply -f isolate-namespace.yaml

# 5. Force rotation of secrets
kubectl delete secret myapp-sa-token
kubectl rollout restart deployment/myapp
```

## Practice Exercises

### Exercise 1: Implement RBAC
Create Role and RoleBinding for CI/CD pipeline that can only update deployments in staging namespace.

### Exercise 2: Pod Security Standards
Create namespace with restricted PSA and try to run privileged pod.

### Exercise 3: Network Policies
Implement zero-trust network policy: default deny, allow only necessary traffic.

### Exercise 4: OPA Constraint
Create OPA policy that requires all images to have "security-scan-passed" label.

### Exercise 5: Vault Integration
Set up Vault and inject database password into your Spring Boot app.

## Next Steps

After mastering security, you're ready for:
- **Part 8: Observability** - Prometheus, Grafana, distributed tracing
- **Part 9: Scaling & Resilience** - HPA, VPA, chaos engineering
- **Part 10: Production Playbook** - Complete deployment guide

---

**Ready for Part 8?** Let me know and I'll create **Observability Deep Dive** covering:
- Prometheus metrics collection
- Grafana dashboards
- Loki for log aggregation
- Tempo/Jaeger for distributed tracing
- Alerting rules and Alertmanager
- SLI/SLO implementation

Would you also like me to:
1. Create a security hardening script for EKS/GKE/AKS?
2. Provide compliance mapping (SOC2, HIPAA, GDPR)?
3. Create incident response playbook?
4. Add penetration testing guidelines?