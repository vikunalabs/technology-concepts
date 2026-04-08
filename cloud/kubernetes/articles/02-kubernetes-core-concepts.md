# Part 2: Kubernetes Core Concepts — Beyond Hello World

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Beginner–Intermediate
> **Prerequisites:** Completed Part 1 (Spring Boot app with Actuator endpoints, Docker image built and pushed to a registry, Helm chart available)
> **Time to complete:** 4–5 hours
> **What you'll learn:** Container registries, namespaces, ConfigMaps, Secrets, resource management, and health probes — the building blocks every real deployment depends on

---

## What This Part Covers

In Part 1, you got something running. A Spring Boot app, containerized, deployed to Kubernetes with Helm. It worked. But the deployment had hard-coded values, no real configuration management, and would fall apart the moment you tried to deploy it to a second environment.

This part fills those gaps. Each chapter introduces one core concept — what it is, why it exists, and how to use it properly. By the end, you'll have a deployment that handles real-world concerns: environment-specific configuration, sensitive credentials, resource boundaries, and a self-healing application that tells Kubernetes exactly what it needs to make good decisions.

---

## Chapter 1: Container Registries — The Missing Link

### Why the Cluster Cannot Use Your Laptop's Images

In Part 1 we used a trick: we pointed Docker at Minikube's internal daemon (`eval $(minikube docker-env)`) so the image was built directly inside the cluster. This works for a single-node local setup. It breaks everywhere else.

A real Kubernetes cluster has multiple nodes — often dozens or hundreds. Each node is a separate machine. When Kubernetes schedules a pod onto a node, that node's `kubelet` process needs to pull the container image. It cannot reach your laptop. It needs a central, always-available location to pull from. That location is a **container registry**.

```
Your laptop                Registry                  Cluster
───────────                ────────                  ───────
docker build          →    docker push          →    kubelet pulls image
hello-app:1.0.0            hello-app:1.0.0           on Node 3 when pod
(local only)               (stored, served)          is scheduled there
```

Every image your cluster runs must live in a registry. Full stop.

### Choosing a Registry

| Registry | Free tier | Best for | Authentication |
|----------|-----------|----------|---------------|
| **Docker Hub** | 1 private repo, unlimited public | Learning, open source | Username + access token |
| **GitHub Container Registry (ghcr.io)** | Free for public repos, included in GitHub Actions minutes | Teams already on GitHub | GitHub PAT or OIDC |
| **AWS ECR** | 500MB free, then pay-per-GB | Apps running on EKS | IAM role or access key |
| **Google Artifact Registry** | 500MB free, then pay-per-GB | Apps running on GKE | Service account key or Workload Identity |
| **Azure Container Registry** | No free tier (Basic ~$5/mo) | Apps running on AKS | Service principal or Managed Identity |
| **GitLab Registry** | Included with GitLab | Teams already on GitLab | GitLab CI token or personal token |

For learning: **Docker Hub** — it's universal, free for public images, and requires no cloud account.
For production on a cloud provider: use that cloud's native registry. The authentication integrates directly with the cluster's IAM system, which is more secure and avoids storing static credentials.

### Setting Up Each Registry

**Docker Hub:**
```bash
# Log in once — credentials saved to ~/.docker/config.json
docker login

# Tag your image with your Docker Hub username prefix
docker tag hello-app:1.0.0 yourusername/hello-app:1.0.0

# Push
docker push yourusername/hello-app:1.0.0

# Verify: visit https://hub.docker.com/r/yourusername/hello-app
```

**GitHub Container Registry (ghcr.io):**
```bash
# Create a Personal Access Token at GitHub → Settings → Developer Settings
# Grant: write:packages, read:packages, delete:packages

export CR_PAT=your_github_token
echo $CR_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin

docker tag hello-app:1.0.0 ghcr.io/yourusername/hello-app:1.0.0
docker push ghcr.io/yourusername/hello-app:1.0.0
```

**AWS ECR:**
```bash
# Create the repository (one-time)
aws ecr create-repository \
  --repository-name hello-app \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# Login — this token expires every 12 hours, so your CI pipeline must re-authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag hello-app:1.0.0 \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/hello-app:1.0.0

docker push \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/hello-app:1.0.0
```

**Google Artifact Registry:**
```bash
# Create repository
gcloud artifacts repositories create hello-app \
  --repository-format=docker \
  --location=us-central1

# Configure Docker to use gcloud as credential helper
gcloud auth configure-docker us-central1-docker.pkg.dev

# Tag and push
docker tag hello-app:1.0.0 \
  us-central1-docker.pkg.dev/YOUR_PROJECT/hello-app/hello-app:1.0.0

docker push \
  us-central1-docker.pkg.dev/YOUR_PROJECT/hello-app/hello-app:1.0.0
```

**Azure Container Registry:**
```bash
# Create registry
az acr create --name myregistry --resource-group mygroup --sku Basic

# Login
az acr login --name myregistry

# Tag and push
docker tag hello-app:1.0.0 myregistry.azurecr.io/hello-app:1.0.0
docker push myregistry.azurecr.io/hello-app:1.0.0
```

### Image Pull Secrets — Giving Kubernetes Registry Credentials

For private registries, Kubernetes needs credentials to pull images. You provide these as a Secret of type `docker-registry`, then reference it in your pod spec.

```bash
# Create the pull secret in the namespace where your pods run
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \  # Docker Hub
  --docker-username=yourusername \
  --docker-password=youraccesstoken \            # Use an access token, not your password
  --docker-email=you@example.com \
  --namespace hello-app
```

For AWS ECR, the server is your account's ECR endpoint:
```bash
kubectl create secret docker-registry ecr-cred \
  --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region us-east-1) \
  --namespace hello-app
```

Reference the secret in your Helm `values.yaml`:
```yaml
imagePullSecrets:
  - name: regcred
```

Helm renders this into the pod spec:
```yaml
spec:
  imagePullSecrets:
    - name: regcred
  containers:
  - name: hello-app
    image: yourusername/hello-app:1.0.0
```

**One practical issue with ECR:** The login token expires every 12 hours. In production you have two good options: use the [ECR Credential Helper](https://github.com/awslabs/amazon-ecr-credential-helper) which refreshes automatically, or use IAM Roles for Service Accounts (IRSA, covered in Part 7) so the node itself has pull permission without a stored credential.

### Image Tagging Strategy — Why `latest` Will Burn You

`latest` seems convenient. It's the default tag if you don't specify one. But in production, it creates a class of bugs that are genuinely difficult to debug:

**Problem 1: No traceability.** When you see `image: myapp:latest` in a running pod, you cannot tell which version it is without inspecting the image digest. Did the node pull this before or after your last push?

**Problem 2: Inconsistent rollouts.** If you push a new image with the `latest` tag and then trigger a rollout, some nodes may have cached the old `latest` and others pull the new one. You have two versions running simultaneously with no way to tell which is which.

**Problem 3: Rollback is broken.** If `v1.2.3` had a bug and you roll back, you push the old code with the `latest` tag again. But Kubernetes uses `imagePullPolicy: IfNotPresent` by default — if the node already has `latest` cached (the bad version), it won't pull the fixed one.

The solution is straightforward:

```yaml
# ❌ Never in production — untraceable and un-rollbackable
image: yourusername/hello-app:latest

# ✅ Semantic version — human-readable, pinned, rollbackable
image: yourusername/hello-app:v1.2.3

# ✅ Git SHA — absolute traceability from running pod back to commit
image: yourusername/hello-app:git-a1b2c3d

# ✅ Both together — CI/CD pipeline creates the semver tag,
#    SHA tag provides the audit trail
image: yourusername/hello-app:v1.2.3
# (and separately: yourusername/hello-app:git-a1b2c3d pointing to the same digest)
```

In CI/CD pipelines (Part 4), we'll automate this: every build tags the image with both the git SHA (for traceability) and the semantic version (for humans).

---

## Chapter 2: Namespaces — Organizing Your Cluster

### The "50 Pods" Problem

Imagine deploying five services — API, frontend, database, cache, and worker. Each has dev, staging, and prod environments. That's already 15 deployments, 15 services, 15 ConfigMaps, plus any secrets, HPA configs, and ingress rules. All in one cluster.

Without namespaces, everything lands in the `default` namespace. `kubectl get pods` returns 50+ rows. Names collide — you can't have two Deployments called `api` in the same namespace. Teams accidentally modify each other's resources. A mistake in dev can nuke prod.

Namespaces solve this by dividing the cluster into isolated virtual compartments. Resources in different namespaces can have the same name. RBAC can be scoped to a namespace so teams only have access to their own. Resource quotas can be applied per namespace so one team can't exhaust cluster capacity.

### What Namespaces Do and Don't Do

Namespaces provide:
- **Name isolation:** Two Deployments named `api` can coexist — one in `dev`, one in `prod`
- **Resource boundaries:** ResourceQuotas cap total CPU, memory, and pod count per namespace
- **Access control scope:** RBAC roles can be limited to a specific namespace (Part 7)
- **Organizational clarity:** `kubectl get pods -n production` shows only prod pods

Namespaces do NOT provide:
- **Network isolation by default:** A pod in `dev` can still reach a pod in `production` unless you add Network Policies (Part 5)
- **Hard security boundaries:** A cluster admin can see and modify everything regardless of namespace

### Standard Namespace Patterns

Most teams settle on one of two patterns:

**Pattern 1: Environment-per-namespace (most common)**
```
dev           ← development environment
staging       ← pre-production testing
production    ← live traffic
```
Simple and easy to manage. Each environment gets its own namespace, with RBAC preventing developers from accidentally touching production.

**Pattern 2: Team-per-namespace with environment suffixes**
```
team-a-dev
team-a-prod
team-b-dev
team-b-prod
```
Better for large organisations where teams need strong isolation from each other.

**System namespaces (always present, don't touch without reason):**
```
kube-system        ← Kubernetes system components (scheduler, API server, etc.)
kube-public        ← Publicly readable data (cluster info)
kube-node-lease    ← Node heartbeat objects
```

**Common additional namespaces you'll create:**
```
monitoring         ← Prometheus, Grafana, Alertmanager (Part 8)
ingress-nginx      ← Ingress controller (Part 5)
cert-manager       ← TLS certificate automation (Part 5)
```

### Creating and Managing Namespaces

**Declarative (preferred — version-controllable):**
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
    team: platform
```

```bash
kubectl apply -f namespace.yaml
```

**Imperative (quick, local testing):**
```bash
kubectl create namespace staging
```

**Working with namespaces:**
```bash
# List all namespaces
kubectl get namespaces

# Everything in a namespace
kubectl get all -n production

# Set a default namespace so you don't type -n on every command
kubectl config set-context --current --namespace=production

# Reset to no default
kubectl config set-context --current --namespace=default

# Delete a namespace — THIS DELETES EVERYTHING INSIDE IT
kubectl delete namespace staging
```

The delete command is worth emphasising. When you delete a namespace, Kubernetes deletes every resource inside it — deployments, pods, services, configmaps, secrets, persistent volume claims — in one command. There is no confirmation prompt. Be deliberate with this.

### ResourceQuota — Preventing One Team from Starving Others

Without quotas, a misconfigured deployment requesting huge CPU or memory can consume all available cluster resources, preventing other teams' pods from scheduling. A ResourceQuota caps a namespace's total resource consumption.

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Total resource requests across all pods in this namespace
    requests.cpu: "10"           # 10 CPU cores total
    requests.memory: "20Gi"

    # Total resource limits across all pods
    limits.cpu: "20"
    limits.memory: "40Gi"

    # Maximum number of pods
    pods: "50"

    # Maximum number of each resource type
    count/deployments.apps: "20"
    count/services: "20"
    persistentvolumeclaims: "10"
```

```bash
kubectl apply -f resourcequota.yaml

# Check current usage against quota
kubectl describe resourcequota production-quota -n production
# Name:                   production-quota
# Namespace:              production
# Resource                Used    Hard
# --------                ----    ----
# limits.cpu              2500m   20
# limits.memory           2Gi     40Gi
# pods                    8       50
# requests.cpu            1250m   10
```

When a pod creation would exceed the quota, Kubernetes rejects it with a clear error:
```
Error: pods "my-app-xyz" is forbidden: exceeded quota: production-quota,
requested: requests.cpu=500m, used: requests.cpu=9750m, limited: requests.cpu=10
```

### Helm and Namespaces

Helm has excellent namespace support built in:

```bash
# Create namespace and deploy in one command
helm upgrade --install my-app ./chart \
  --namespace production \
  --create-namespace  # Creates the namespace if it doesn't exist

# Upgrade only affects resources in that namespace
helm upgrade my-app ./chart --namespace production

# List releases per namespace
helm list --namespace production
helm list --all-namespaces  # See everything
```

A Helm release is scoped to a namespace. Two releases named `my-app` can coexist in different namespaces — they're completely independent, with independent history and independent rollback.

---

## Chapter 3: ConfigMaps — Configuration Without Rebuilding

### The Five-Step Rebuild Problem

Picture this scenario: your app connects to a PostgreSQL database. The database URL is hardcoded in `application.yml`, which is compiled into the JAR, which is baked into the Docker image.

Your database hostname changes. To update it, you must:

1. Change the URL in `application.yml`
2. Run `mvn clean package` to recompile
3. Run `docker build` to rebuild the image
4. Run `docker push` to upload the new image
5. Run `kubectl rollout restart` to deploy the new image

Five steps — for changing a URL. And the new image is functionally identical to the old one except for one string. You've burned 10 minutes and created a new artefact for no good reason.

**ConfigMaps** exist to break this cycle. A ConfigMap holds configuration data separately from your application image. The image stays the same; you update the ConfigMap and restart the pod. No rebuild, no new image, no push.

**Quick start — apply a ConfigMap:**
```bash
kubectl apply -f configmap.yaml -n hello-app
```

### The Whiteboard Analogy

Think of a ConfigMap as a whiteboard in a shared office. Your application walks in, reads what's on the whiteboard (the ConfigMap), and uses those values. When you need to change the database URL, you erase the old value and write the new one. The next time the application starts, it reads the new value. The application itself (the Docker image) never changed — only what was written on the whiteboard.

### Three Ways to Create a ConfigMap

**Method 1: From literal values (quick, for small configs)**
```bash
kubectl create configmap app-config \
  --from-literal=database.url=jdbc:postgresql://db-svc:5432/mydb \
  --from-literal=cache.ttl=300 \
  --from-literal=log.level=INFO \
  --namespace hello-app
```

**Method 2: From a file (for entire config files)**
```bash
# Create from an application.properties file
kubectl create configmap app-properties \
  --from-file=application.properties \
  --namespace hello-app
# The key is the filename: "application.properties"
# The value is the file's entire contents
```

**Method 3: Declarative YAML (preferred — version-controllable, reviewable)**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: hello-app
data:
  # Simple key-value pairs
  LOG_LEVEL: "INFO"
  CACHE_TTL: "300"
  APP_NAME: "Hello App"

  # A complete config file stored as a single multi-line value.
  # The | character means "literal block scalar" — preserves newlines.
  # Spring Boot can read this if you mount it as a file.
  application.properties: |
    spring.datasource.url=jdbc:postgresql://db-svc:5432/mydb
    spring.datasource.username=appuser
    spring.jpa.show-sql=false
    spring.cache.type=redis
    spring.data.redis.host=redis-svc
    spring.data.redis.port=6379
```

### Using ConfigMaps in Pods

There are two ways to expose ConfigMap data to your application: environment variables and mounted files. Each has different characteristics and appropriate use cases.

**Option A: Environment Variables**

Individual keys:
```yaml
spec:
  containers:
  - name: hello-app
    env:
    - name: LOG_LEVEL              # Name of the env var inside the container
      valueFrom:
        configMapKeyRef:
          name: app-config         # ConfigMap name
          key: LOG_LEVEL           # Key inside the ConfigMap
    - name: APP_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_NAME
```

All keys at once (simpler, but less explicit):
```yaml
spec:
  containers:
  - name: hello-app
    envFrom:
    - configMapRef:
        name: app-config  # Every key in app-config becomes an environment variable
```

Spring Boot automatically picks up environment variables as property sources — no code changes needed. `LOG_LEVEL=DEBUG` in the environment overrides `logging.level.root=INFO` in `application.yml`.

**Option B: Mounted Files**

```yaml
spec:
  containers:
  - name: hello-app
    volumeMounts:
    - name: config-volume
      mountPath: /app/config       # Each key in the ConfigMap becomes a file here
      readOnly: true

  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Optional: mount only specific keys as files
      items:
      - key: application.properties
        path: application.properties   # Filename in the mount directory
```

With this setup, `/app/config/application.properties` inside the container contains the full properties content. Point Spring Boot at this directory:

```yaml
# application.yml
spring:
  config:
    import: "optional:file:/app/config/application.properties"
```

**When to use which:**

| | Environment Variables | Mounted Files |
|-|----------------------|--------------|
| **Use for** | Simple key-value config, feature flags | Full config files, Spring `application.properties` |
| **Visibility in `kubectl describe pod`** | Values shown in pod description | Only mount path shown, not contents |
| **App reads values** | On startup, not updated until restart | File is updated by kubelet (eventually consistent) |
| **Spring Boot hot-reload** | Not possible without restart | Possible with additional setup (see below) |

### ConfigMap vs Secret — When to Use Which

| | ConfigMap | Secret |
|-|-----------|--------|
| **Purpose** | Non-sensitive configuration (URLs, feature flags, log levels) | Sensitive data (passwords, API keys, TLS certificates) |
| **Storage** | base64 encoding (for transport, not security) | base64 encoding + optional encryption at rest in etcd |
| **RBAC** | Standard access control | Often restricted — limit who can `get`/`list` secrets |
| **Visibility in `describe pod`** | Values shown when used as env vars | Hidden when mounted as files; secret names shown for env vars |
| **Audit trail** | Logged as normal resources | Often excluded from logs by audit policies |

**Rule of thumb:** If you wouldn't want it in your Slack history, put it in a Secret.

### ConfigMap Hot-Reload — Three Options

When you update a ConfigMap, what happens to the running app? The answer depends on how the ConfigMap is consumed and which reload mechanism you use.

**For environment variables:** Nothing happens automatically. Environment variables are set when the process starts. Changing the ConfigMap has no effect on running pods. You must restart.

**For mounted files:** The kubelet periodically syncs ConfigMap contents to the mounted files (typically within 60 seconds). The file on disk changes — but your Spring Boot app has already read it at startup and stored the values in memory. The file changing on disk doesn't automatically cause Spring Boot to re-read it.

To actually pick up changes without a full pod restart, you have three options:

**Option 1: Pod restart with `kubectl rollout restart` (simplest, always works)**
```bash
kubectl rollout restart deployment/hello-app -n hello-app
```
This does a rolling restart — new pods start with the new ConfigMap values, old pods terminate as new ones become ready. Brief disruption (a few seconds), zero-downtime if you have multiple replicas. This is the right choice for most situations.

**Option 2: Spring Cloud Kubernetes reload (no restart, automatic)**

Add the dependency:
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-client-config</artifactId>
</dependency>
```

Enable reload in `application.yml`:
```yaml
spring:
  cloud:
    kubernetes:
      config:
        enabled: true
      reload:
        enabled: true
        mode: polling          # polling checks periodically; event watches for changes
        period: 15000          # Check every 15 seconds (polling mode)
```

Spring Cloud Kubernetes watches the ConfigMap and reloads `@ConfigurationProperties` beans when changes are detected — no restart needed. The tradeoff: you're adding a framework dependency specifically to handle configuration. It adds complexity, requires RBAC for the pod to watch ConfigMaps, and the reload isn't instantaneous.

**Option 3: `@RefreshScope` with `/actuator/refresh` (targeted, manual trigger)**

```java
@RestController
@RefreshScope  // This bean is recreated when /actuator/refresh is called
public class HelloController {

    @Value("${app.name}")
    private String appName;

    // ...
}
```

```yaml
# Enable the refresh endpoint
management:
  endpoints:
    web:
      exposure:
        include: health, refresh
```

```bash
# Trigger reload manually after updating ConfigMap
kubectl exec -it <pod-name> -n hello-app -- \
  curl -X POST http://localhost:8080/actuator/refresh
```

`@RefreshScope` only recreates beans annotated with it. Beans without the annotation keep their old values. Useful when you have a specific component that needs live config updates, but overkill for most cases. Also requires adding `spring-cloud-context` as a dependency.

**The practical recommendation:** Use `kubectl rollout restart` for most cases. It's one command, always works, and the brief rolling restart is not noticeable with multiple replicas. Reach for Spring Cloud Kubernetes reload only when you genuinely cannot tolerate pod restarts for config changes.

### Helm + ConfigMaps

Helm can generate ConfigMaps from values, giving you environment-specific config without separate ConfigMap files per environment:

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "hello-app-chart.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hello-app-chart.labels" . | nindent 4 }}
data:
  LOG_LEVEL: {{ .Values.config.logLevel | quote }}
  APP_NAME: {{ .Values.config.appName | quote }}
  SPRING_PROFILES_ACTIVE: {{ .Values.config.profile | quote }}

  application.properties: |
    spring.datasource.url={{ .Values.database.url }}
    spring.datasource.username={{ .Values.database.username }}
    spring.cache.type={{ .Values.config.cacheType }}
```

```yaml
# values-dev.yaml
config:
  logLevel: DEBUG
  appName: Hello App (Dev)
  profile: dev
  cacheType: simple
database:
  url: jdbc:postgresql://postgres-dev:5432/devdb
  username: devuser

# values-prod.yaml
config:
  logLevel: WARN
  appName: Hello App
  profile: prod
  cacheType: redis
database:
  url: jdbc:postgresql://postgres-prod:5432/proddb
  username: produser
```

One chart template, two completely different ConfigMaps produced — with no duplication.

When you run `helm upgrade` with new values, the ConfigMap is updated. The pods still need a restart to pick up the new values. You can automate this with the `rollme` annotation trick:

```yaml
# templates/deployment.yaml
spec:
  template:
    metadata:
      annotations:
        # This annotation changes whenever the ConfigMap contents change.
        # A changed annotation triggers a rolling restart automatically.
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

When the ConfigMap changes, its checksum changes, the annotation changes, and Kubernetes sees the pod template as modified — triggering a rolling restart automatically as part of `helm upgrade`.

---

## Chapter 4: Secrets — Handling Sensitive Data

### Why Secrets Exist (and What They Actually Are)

Your application needs a database password. You could put it in a ConfigMap — but then it's stored in plaintext, visible to anyone who can run `kubectl get configmap`. You could hardcode it in the image — even worse, now it's in your git history forever. You could pass it as a command-line argument — it appears in `ps aux` output on the node.

Kubernetes Secrets exist as a dedicated resource type for sensitive data. They're handled differently from ConfigMaps:
- Access can be restricted independently via RBAC
- They can be encrypted at rest in etcd (if the cluster is configured for it)
- They don't appear in `kubectl describe pod` output when mounted as volumes
- Tools and auditing systems treat them differently from regular config

Here is the critical thing to understand: **Kubernetes Secrets are not encrypted by default.** When you create a Secret, Kubernetes stores the values as base64-encoded strings in etcd. Base64 is an encoding scheme — it's how binary data is represented as text. It is not encryption. Anyone who can run `kubectl get secret` and knows how to run `base64 --decode` can read your secret values in plaintext.

**Quick start — apply a Secret:**
```bash
kubectl apply -f secret.yaml -n hello-app
```

You can verify this yourself right now:

```bash
# Create a secret
kubectl create secret generic test-secret \
  --from-literal=password=SuperSecret123 \
  --namespace hello-app

# Get it back as YAML — notice the base64 value
kubectl get secret test-secret -n hello-app -o yaml
# data:
#   password: U3VwZXJTZWNyZXQxMjM=   ← this is just base64

# Decode it — trivial, anyone can do this
echo "U3VwZXJTZWNyZXQxMjM=" | base64 -d
# SuperSecret123
```

So what actually makes Secrets secure? A combination of things that you must configure:

1. **RBAC:** Restrict who can `get`, `list`, or `watch` secrets. A developer shouldn't be able to run `kubectl get secrets -n production` unless they explicitly need it (Part 7).
2. **Encryption at rest:** Configure etcd to encrypt Secret values using a KMS provider (AWS KMS, GCP KMS, etc.). This means the base64 blob stored in etcd is itself encrypted at the storage layer.
3. **External secrets managers:** The most robust approach — don't store sensitive values in Kubernetes Secrets at all. Sync them from AWS Secrets Manager, GCP Secret Manager, or HashiCorp Vault (covered later in this chapter and in Part 7).

For learning and non-production use, Kubernetes Secrets with RBAC are fine. For production, use external secrets.

### Creating Secrets

**`stringData` vs `data` — use `stringData`**

Secrets have two fields: `data` (base64-encoded values) and `stringData` (plain text values, automatically base64-encoded by Kubernetes when stored). Always use `stringData` in YAML — it's more readable and avoids encoding mistakes.

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: hello-app
type: Opaque   # Generic secret — use this for most secrets
stringData:    # Plain text here — Kubernetes encodes it when stored
  DB_PASSWORD: "SuperSecret123"
  API_KEY: "sk-abc123xyz456"
  JWT_SECRET: "my-super-long-random-jwt-signing-key"
```

**⚠️ Never commit this file to Git.** Even if it's `stringData`, the values are visible to anyone who can read the file. Use `.gitignore`, external secrets, or Sealed Secrets (covered below) instead.

**From the command line:**
```bash
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD=SuperSecret123 \
  --from-literal=API_KEY=sk-abc123xyz456 \
  --namespace hello-app
```

**From files (useful for TLS certificates or SSH keys):**
```bash
kubectl create secret generic tls-certs \
  --from-file=tls.crt=server.crt \
  --from-file=tls.key=server.key \
  --namespace hello-app
```

### Using Secrets in Pods

The same two options as ConfigMaps — environment variables or mounted files — but mounted files are significantly more secure for secrets.

**Option A: Environment Variables**
```yaml
spec:
  containers:
  - name: hello-app
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DB_PASSWORD
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: API_KEY
```

The downside: environment variables appear in `kubectl describe pod` output:
```
Environment:
  DB_PASSWORD:  <set to the key 'DB_PASSWORD' in secret 'app-secrets'>  Optional: false
```

While the value itself isn't printed, the existence of the variable is visible. More importantly, environment variables are sometimes captured in log output, crash reports, and debugging tools without you realising.

**Option B: Mounted Files (more secure)**
```yaml
spec:
  containers:
  - name: hello-app
    volumeMounts:
    - name: secrets-volume
      mountPath: /app/secrets    # Each key becomes a file here
      readOnly: true             # Pod can read but not modify

  volumes:
  - name: secrets-volume
    secret:
      secretName: app-secrets
      # Result:
      # /app/secrets/DB_PASSWORD  ← contains "SuperSecret123"
      # /app/secrets/API_KEY      ← contains "sk-abc123xyz456"
```

Read in Spring Boot:
```java
// Option 1: Read the file directly
@Value("${DB_PASSWORD:#{null}}")
private String dbPassword;  // Spring auto-reads env vars named DB_PASSWORD

// Option 2: If using mounted file path
@Value("#{T(java.nio.file.Files).readString(T(java.nio.file.Path).of('/app/secrets/DB_PASSWORD')).strip()}")
private String dbPassword;

// Option 3: Use @ConfigurationProperties with a custom source
// (cleaner for many secrets — covered in Part 7 with Vault integration)
```

With mounted files, `kubectl describe pod` shows only the mount path, not any indication of secret names or keys. The values never appear in any Kubernetes API output.

### Production Approach: External Secrets Operator

The most robust approach for production: don't store sensitive values in Kubernetes Secrets at all. Store them in a dedicated secrets manager — AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, or Azure Key Vault — and use the **External Secrets Operator** to sync them into Kubernetes Secrets automatically.

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

**Supported providers:** AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, IBM Secrets Manager, Akeyless, Webhook (generic), and more. The configuration varies by provider — we show AWS below as the most common, but the pattern is similar for all.

Configure a connection to AWS Secrets Manager:
```yaml
# clusterSecretStore.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secretsmanager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

Declare which secrets to sync:
```yaml
# externalsecret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
  namespace: hello-app
spec:
  refreshInterval: 1h             # Re-sync from AWS every hour
                                  # WARNING: Too low (< 5m) = API rate limits + increased costs.
                                  # Too high (> 1h) = stale secrets. 15-60min is typical.

  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore

  target:
    name: app-secrets             # Creates a regular Kubernetes Secret with this name
    creationPolicy: Owner         # ExternalSecret owns the created Secret

  data:
  - secretKey: DB_PASSWORD        # Key in the created Kubernetes Secret
    remoteRef:
      key: production/hello-app   # AWS Secrets Manager secret name
      property: db_password       # Property within that secret (if it's a JSON object)

  - secretKey: API_KEY
    remoteRef:
      key: production/hello-app
      property: api_key
```

The ExternalSecret controller syncs values from AWS Secrets Manager into a standard Kubernetes Secret, which your pod uses normally. If someone updates the value in AWS Secrets Manager, it propagates to Kubernetes within `refreshInterval`. The actual sensitive values never live in Git or in Kubernetes — only in the secrets manager.

### Sealed Secrets — Safe Git Storage

If you use GitOps (everything in Git, Part 4) and need secrets in Git too, Sealed Secrets encrypts them so only your cluster can decrypt them.

```bash
# Install the controller in your cluster
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install the kubeseal CLI tool
brew install kubeseal  # macOS
# Or download from: https://github.com/bitnami-labs/sealed-secrets/releases

# Fetch the cluster's public key
kubeseal --fetch-cert \
  --controller-name=sealed-secrets \
  --controller-namespace=kube-system \
  > sealed-secrets-cert.pem

# Create a regular Secret (don't apply it)
kubectl create secret generic app-secrets \
  --from-literal=DB_PASSWORD=SuperSecret123 \
  --dry-run=client \
  -o yaml > app-secrets.yaml

# Encrypt it with the cluster's public key → safe to commit
kubeseal --cert sealed-secrets-cert.pem \
  --format yaml \
  < app-secrets.yaml \
  > sealed-app-secrets.yaml

# This file is encrypted — commit it to Git safely
git add sealed-app-secrets.yaml
git commit -m "Add sealed app secrets"

# Apply it — only the cluster with the matching private key can decrypt it
kubectl apply -f sealed-app-secrets.yaml -n hello-app
```

The sealed file looks like this — unreadable without the cluster's private key:
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: app-secrets
spec:
  encryptedData:
    DB_PASSWORD: AgBx3F9T1XJ...  # Encrypted — safe for Git
    API_KEY: AgCy4G0U2YK...
```

---

## Chapter 5: Resource Management — Preventing the Noisy Neighbor

### The Shared Resource Problem

A Kubernetes cluster is a pool of shared CPU and memory across multiple nodes. Multiple pods from multiple teams run on the same nodes. Without resource management, one poorly-configured pod can exhaust all CPU on a node — causing every other pod on that node to slow to a crawl or fail. This is the "noisy neighbor" problem.

Resource management in Kubernetes has two layers: **requests and limits** at the pod level, and **ResourceQuota and LimitRange** at the namespace level.

### Requests vs Limits — A Critical Distinction

Every container in a pod can declare two resource figures:

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

These are not the same thing, and confusing them leads to real production problems.

**Requests** are a *promise to the scheduler*. When Kubernetes schedules a pod, it finds a node with enough free capacity to satisfy the requests. If a node has 1000m CPU available and a pod requests 250m, Kubernetes places the pod there and marks 250m of that node's capacity as reserved. Even if the pod only uses 10m in practice, those 250m are "spoken for" as far as the scheduler is concerned.

**Limits** are a *ceiling enforced by the kernel*. After the pod is running, the container runtime uses Linux cgroups to enforce limits. What happens when a container hits its limit differs by resource type:

| Resource | At the request | Over the limit |
|----------|---------------|----------------|
| **CPU** | Scheduler uses for placement | Container is *throttled* — it runs slower, but keeps running |
| **Memory** | Scheduler uses for placement | Container is *OOMKilled* — the kernel kills it immediately |

The asymmetry matters. CPU throttling is survivable — the app slows down but keeps serving requests. Memory OOMKill is an immediate, hard crash. This is why memory limits deserve more care: set them too low and your app crashes unpredictably under load.

### How OOMKill Happens in Practice

Your Spring Boot app has a 512Mi memory limit. The JVM starts up and — without `UseContainerSupport` — reads the host machine's 32GB of total RAM. It allocates 8GB as max heap. Kubernetes kills the container before it finishes starting. You see `OOMKilled` in `kubectl describe pod`.

Even with `UseContainerSupport` and `MaxRAMPercentage=75.0`, the JVM allocates ~384Mi of heap. But memory usage is not just heap. A JVM process uses:
- **Heap:** ~75% of limit (384Mi with a 512Mi limit)
- **Metaspace:** ~100–300MB (class metadata, JIT-compiled code)
- **Thread stacks:** ~1MB per thread × number of threads
- **JVM overhead:** GC bookkeeping, JNI, etc.

A 512Mi limit is genuinely tight for a Spring Boot app. The minimum realistic limit for a typical Spring Boot service is 400–512Mi for development, 512Mi–1Gi for production under real load.

### Finding the Right Values

Don't guess. Measure.

**Step 1: Run locally and observe:**
```bash
# Start your container with no memory limit
docker run -d --name hello-test hello-app:1.0.0

# Watch memory and CPU usage live
docker stats hello-test

# CONTAINER  CPU %   MEM USAGE / LIMIT     MEM %   ...
# hello-test  0.1%   320MiB / 31.4GiB      1.02%   ...
#              ↑ idle    ↑ actual usage
```

**Step 2: Apply realistic load and observe peak:**
```bash
# Use a simple load generator
kubectl run load-generator --image=busybox --rm -it --restart=Never -- \
  /bin/sh -c 'while true; do wget -q -O- http://hello-app-service:8080/api/hello; done'

# Watch peak memory during load
docker stats hello-test
```

**Step 3: Set values based on observation:**
- `memory.requests` = idle memory usage (what the app uses when doing nothing)
- `memory.limits` = peak memory × 1.5 (headroom for traffic spikes)
- `cpu.requests` = average CPU under normal load
- `cpu.limits` = peak CPU × 2 (allow bursting, but cap runaway usage)

**Step 4: Deploy and watch real usage:**
```bash
# Requires metrics-server to be running
kubectl top pods -n hello-app
# NAME                    CPU(cores)   MEMORY(bytes)
# hello-app-7d8b-2xmpq    15m          287Mi
```

Adjust over time as you observe real traffic patterns. The VPA (Vertical Pod Autoscaler, Part 9) can automate this by watching actual usage and recommending better values.

### LimitRange — Namespace-Level Defaults

If a pod doesn't specify resource requests or limits, it gets none — which breaks HPA (no requests = no utilisation metric) and risks the noisy neighbor problem. A `LimitRange` sets defaults for any pod that doesn't specify its own.

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: hello-app
spec:
  limits:
  - type: Container
    # Applied to every container that doesn't specify its own resources
    # NOTE: These must be realistic for your workload. A Spring Boot app
    # needs ~512Mi minimum (see line 967). Too-low defaults cause OOMKilled.
    default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    # Absolute maximum any single container can request
    max:
      cpu: "4"
      memory: "4Gi"
    # Absolute minimum — prevents setting requests too low to be meaningful
    min:
      cpu: "50m"
      memory: "64Mi"
```

With this LimitRange in place, a pod that omits `resources` entirely will receive `requests: {cpu: 100m, memory: 128Mi}` and `limits: {cpu: 200m, memory: 256Mi}` automatically. No pod in this namespace can request more than `{cpu: 4, memory: 4Gi}`.

### VPA Preview — Automatic Right-Sizing

Getting resource values right manually is tedious, and they need regular tuning as traffic patterns change. The Vertical Pod Autoscaler (VPA) observes actual CPU and memory usage over time and either recommends better values or applies them automatically.

We cover VPA in depth in Part 9. For now, knowing it exists is enough. Once your app is running in production for a week, VPA recommendations are far more reliable than any manual estimate.

```bash
# After VPA is installed (Part 9), create a VPA object in Off mode
# It will observe and recommend without changing anything
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hello-app-vpa
  namespace: hello-app
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hello-app
  updatePolicy:
    updateMode: "Off"   # Recommendations only — no automatic changes

# Check recommendations after a few hours:
# kubectl get vpa hello-app-vpa -n hello-app -o yaml
# Look for status.recommendation.containerRecommendations
```

---

## Chapter 6: Health Probes — Self-Healing Applications

### Why Kubernetes Needs to Know Your App's State

Kubernetes makes decisions constantly: which pods should receive traffic? Which pods need to be restarted? Is a rolling update safe to continue? All of these decisions require knowing whether your app is healthy and ready.

Without probes, Kubernetes treats a pod as healthy the moment its container process starts running — regardless of whether Spring Boot has finished initialising, whether the database connection pool is ready, or whether the app has hit a deadlock. The result is traffic sent to pods that can't handle it, and crashed pods that aren't restarted.

Health probes are how you tell Kubernetes exactly what "healthy" means for your app.

### The Three Probes and Their Distinct Jobs

Kubernetes has three probe types, and each has a specific, non-overlapping job:

**`startupProbe` — "The app is still initialising, leave it alone"**

This probe runs only during startup. While it's failing, the liveness probe is completely disabled — it won't fire at all. Once the startup probe passes for the first time, it's done. Kubernetes moves on to running liveness and readiness probes.

**`readinessProbe` — "Is the app ready to receive traffic right now?"**

This probe runs continuously throughout the pod's life. When it fails, Kubernetes removes the pod from the Service's list of endpoints — no new requests are sent to it. The pod keeps running; it just receives no traffic. When the probe passes again, traffic resumes. The pod is never restarted due to readiness failures.

Use case: database connection drops temporarily. Readiness probe fails → traffic stops going to this pod → no errors returned to users. When the database comes back and the connection pool reconnects, readiness passes → traffic resumes. Restarting the pod wouldn't help, so liveness probe stays UP.

**`livenessProbe` — "Is the app still alive, or is it stuck?"**

This probe runs continuously. When it fails the configured number of times, Kubernetes restarts the container. Use this for truly unrecoverable states: a deadlock where all threads are stuck, a corrupted in-memory state, or a JVM that's alive but not processing any requests.

**The critical Spring Boot scenario — what happens without `startupProbe`:**

Spring Boot takes 20–60 seconds to start. During that time, the app is booting:
- Loading the Spring application context
- Connecting to databases and running Flyway migrations
- Warming up caches
- Starting embedded Tomcat

The `/actuator/health/liveness` endpoint is not available yet — the app isn't listening on port 8080.

Without a `startupProbe`, Kubernetes starts running the `livenessProbe` immediately. It fails. After 3 failures (at `periodSeconds: 10`, that's 30 seconds), Kubernetes restarts the container. The app starts booting again. The liveness probe fails again. Another restart. This is `CrashLoopBackOff` — a loop your app can never escape because it keeps being killed before it finishes starting.

The `startupProbe` exists specifically to break this loop. While it's running, liveness is silent. The app gets its full startup time.

### Complete Probe Configuration — Every Field Explained

```yaml
containers:
- name: hello-app
  image: hello-app:1.0.0

  # ─────────────────────────────────────────────────────────────────────────
  # startupProbe
  # Runs ONLY during startup. Disables livenessProbe until it passes once.
  #
  # Total startup window = failureThreshold × periodSeconds
  # = 30 × 10s = 300 seconds (5 minutes)
  # Spring Boot rarely needs more than 2 minutes even in slow environments.
  # Set this generously — you'd rather the app be slow to start than killed.
  # ─────────────────────────────────────────────────────────────────────────
  startupProbe:
    httpGet:
      path: /actuator/health/readiness   # Reuse readiness path — if it's ready, it started
      port: 8080
    failureThreshold: 30    # Allow 30 failures before giving up
    periodSeconds: 10       # Check every 10 seconds
    # No initialDelaySeconds needed — the probe starts immediately and just
    # keeps trying until failureThreshold is reached or it passes.

  # ─────────────────────────────────────────────────────────────────────────
  # readinessProbe
  # Runs continuously. Controls whether traffic is sent to this pod.
  # Failing = pod removed from Service endpoints (no traffic).
  # Passing = pod added back to Service endpoints (traffic resumes).
  #
  # Tie this to: database connectivity, cache connectivity, external APIs
  # the app CANNOT function without. If those fail, the pod shouldn't get traffic.
  #
  # Do NOT tie this to: things that are optional or degraded-mode-capable.
  # If the probe is too strict, a single flaky dependency takes your whole service down.
  # ─────────────────────────────────────────────────────────────────────────
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    periodSeconds: 5         # Check every 5 seconds
    failureThreshold: 3      # Remove from Service after 3 consecutive failures (15s)
    successThreshold: 1      # One success brings it back (default, usually fine)
    timeoutSeconds: 3        # Probe times out after 3s — don't let slow probes hang

  # ─────────────────────────────────────────────────────────────────────────
  # livenessProbe
  # Runs continuously. Controls whether the container is restarted.
  #
  # Tie this to: internal JVM health only — NOT external dependencies.
  # WHY: if your database is down and you tie liveness to DB connectivity,
  # all pods restart in a loop. A restart doesn't fix the database.
  # It just makes recovery slower (restarting pods takes time, and they all
  # restart at once if the DB comes back during a crash loop).
  #
  # Spring Boot's liveness endpoint (/actuator/health/liveness) checks only
  # internal state: is the JVM in a healthy state? It doesn't check databases.
  # This is exactly what you want for liveness.
  # ─────────────────────────────────────────────────────────────────────────
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    periodSeconds: 10        # Check every 10 seconds (less frequent than readiness)
    failureThreshold: 3      # Restart after 3 consecutive failures (30s)
    timeoutSeconds: 5        # Give it 5 seconds to respond
```

### Spring Boot Actuator: Liveness vs Readiness Endpoints

Spring Boot's Actuator gives you two separate endpoints with different semantics:

`/actuator/health/liveness` — checks internal application state:
- Is the Spring application context running?
- Are there any fatal internal errors?
- By default: returns `UP` unless the application is in a broken state it can't recover from

`/actuator/health/readiness` — checks whether the app can serve traffic:
- Are database connections available?
- Are required caches initialised?
- Are external APIs reachable?
- By default: returns `UP` once the app is fully started, `OUT_OF_SERVICE` during startup

Configure in `application.yml`:
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true    # Creates /liveness and /readiness endpoints
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### Custom Health Indicators

The default readiness check is basic — it only knows the app has started. For a database-dependent service, you want readiness to fail if the database connection is unavailable. Implement a custom `HealthIndicator`:

```java
package com.example.helloapp;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Component;

/**
 * Custom health indicator that checks database connectivity.
 *
 * WHY: The default readiness check only knows the app has started.
 * This makes readiness fail if the database is unreachable,
 * causing Kubernetes to stop sending traffic until connectivity is restored.
 * This is exactly the right behaviour — the app can't serve requests
 * without a database anyway.
 *
 * Spring Boot automatically includes HealthIndicators in the /readiness endpoint.
 */
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final JdbcTemplate jdbcTemplate;

    public DatabaseHealthIndicator(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    @Override
    public Health health() {
        try {
            jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            return Health.up()
                .withDetail("database", "reachable")
                .build();
        } catch (Exception e) {
            // Return DOWN — this causes /readiness to return OUT_OF_SERVICE
            // which causes Kubernetes to stop sending traffic to this pod
            return Health.down()
                .withDetail("database", "unreachable")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

When the database goes down:
1. `DatabaseHealthIndicator.health()` returns `DOWN`
2. `/actuator/health/readiness` returns `{"status":"OUT_OF_SERVICE"}`
3. Readiness probe fails
4. Kubernetes removes pod from Service endpoints
5. No traffic reaches this pod
6. Database comes back → health check passes → pod receives traffic again

All without a single pod restart.

### Graceful Shutdown — Completing the Picture

Part 1 introduced graceful shutdown briefly. Here's the complete picture of what happens when Kubernetes terminates a pod:

```
Time 0:   Pod marked as Terminating
          → Removed from Service endpoints immediately
          → No new requests routed to this pod

Time 0+:  preStop hook executes
          → We sleep for 15 seconds
          WHY: The load balancer (Ingress controller) may take a few seconds
          to propagate the endpoint removal. Without the sleep, requests
          already in flight from the load balancer arrive after SIGTERM.

Time 15s: SIGTERM sent to the container process
          → Spring Boot's graceful shutdown begins
          → Spring stops accepting new HTTP connections
          → In-flight requests continue processing

Time 15–45s: Active requests complete (or timeout-per-shutdown-phase reached)

Time 60s: terminationGracePeriodSeconds reached
          → SIGKILL sent — forced exit regardless of in-flight state
```

```yaml
# deployment.yaml
spec:
  template:
    spec:
      # Total time Kubernetes waits from SIGTERM to SIGKILL
      # Must be > preStop sleep + Spring Boot shutdown timeout
      # 60s > 15s (preStop) + 30s (Spring shutdown) = 45s minimum
      terminationGracePeriodSeconds: 60

      containers:
      - name: hello-app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

```yaml
# application.yml
spring:
  shutdown: graceful
  lifecycle:
    timeout-per-shutdown-phase: 30s   # Max time to wait for in-flight requests
```

### Testing Probes

To verify your probes behave correctly, simulate failures:

**Method 1: Kill the database connection (tests readiness probe)**

If you have a custom `DatabaseHealthIndicator`, stop the database or block network access:
```bash
# Watch the pod's readiness status
kubectl get pods -n hello-app -w

# Watch Service endpoints — pod should disappear when DB is unreachable
kubectl get endpoints hello-app-service -n hello-app -w

# Restore the database and watch the pod recover (no restart needed)
```

**Method 2: Use Spring Boot Actuator's state endpoint (requires actuator extras)**

Add the `spring-boot-starter-actuator` with full health management:
```java
// Temporarily force readiness to fail by setting a "degraded" flag
// This requires a custom HealthIndicator that reads a toggle flag
```

**Method 3: Break the probe endpoint directly**

Temporarily modify the Deployment to point the probe at a wrong path:
```bash
kubectl edit deployment/hello-app -n hello-app
# Change readiness probe path from /actuator/health/readiness to /nonexistent
# Save and watch the pod transition to 0/1 (not ready)
# Then revert and watch it recover
```

**Observe the results:**
```bash
# Pod should show 0/1 READY when probe fails
kubectl get pods -n hello-app

# Events section shows probe failures
kubectl describe pod <pod-name> -n hello-app
# Look for: "Readiness probe failed" in Events

# Pod disappears from Service endpoints when not ready
kubectl get endpoints hello-app-service -n hello-app -w
```

---

## Chapter 7: Complete Production-Ready Deployment

### Combining Everything

Here's a complete `values-prod.yaml` incorporating every concept from this part: registry, namespace, ConfigMap-driven config, externally managed secrets, resource limits, and fully configured probes.

```yaml
# values-prod.yaml

replicaCount: 3

image:
  repository: yourusername/hello-app    # Full registry path
  tag: "v1.2.3"                          # Always pinned — never "latest" in prod
  pullPolicy: Always                     # Always pull — ensures nodes get the right tag

imagePullSecrets:
  - name: regcred                        # Created separately, not in this file

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # Auto TLS (Part 5)
  hosts:
    - host: api.hello-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: hello-app-tls
      hosts:
        - api.hello-app.example.com

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

# Probe settings — generous startup window, frequent readiness, less frequent liveness
probes:
  startup:
    path: /actuator/health/readiness
    failureThreshold: 30
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    periodSeconds: 5
    failureThreshold: 3
    timeoutSeconds: 3
  liveness:
    path: /actuator/health/liveness
    periodSeconds: 10
    failureThreshold: 3
    timeoutSeconds: 5

# Graceful shutdown
terminationGracePeriodSeconds: 60

# Config injected via ConfigMap (no sensitive values here)
config:
  logLevel: "WARN"
  appName: "Hello App"
  profile: "prod"
  cacheType: "redis"

# Database connection (non-secret parts)
database:
  url: "jdbc:postgresql://postgres-prod-svc:5432/proddb"
  username: "produser"

# Autoscaling — covered in Part 9
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60
```

### Deploying to Production

```bash
# First: make sure the namespace and pull secret exist
kubectl create namespace production || true
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=$DOCKER_USERNAME \
  --docker-password=$DOCKER_PASSWORD \
  --namespace production

# Validate before deploying
helm lint hello-app-chart/
helm template prod-release ./hello-app-chart -f values-prod.yaml | kubectl apply --dry-run=client -f -

# Deploy
helm upgrade --install hello-app-prod ./hello-app-chart \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace production \
  --create-namespace \
  --wait \
  --timeout 10m \
  --atomic   # Auto-rollback if pods don't become healthy

# Verify
kubectl get pods -n production
kubectl top pods -n production
helm history hello-app-prod -n production
```

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `ImagePullBackOff` | Wrong registry URL, wrong tag, missing pull secret, or expired ECR token | `kubectl describe pod <n> -n <ns>` → look at Events section | Verify image exists in registry; check pull secret is in correct namespace; re-create ECR token |
| Pod stuck at `0/1 Running` forever | Startup probe window too short — pod killed before Spring Boot finishes booting | `kubectl describe pod <n>` → look for "Startup probe failed" in Events | Increase `startupProbe.failureThreshold` or `periodSeconds` |
| Pod in `CrashLoopBackOff` | App crashing on startup (exception, missing env var, bad config) | `kubectl logs <n> --previous` | Read the stack trace; common causes: missing required env var, can't connect to DB, bad YAML config |
| Readiness never passes | App started but health check fails (DB unreachable, missing config) | `kubectl exec -it <n> -- curl localhost:8080/actuator/health/readiness` | Check what the health endpoint actually returns; fix the underlying dependency |
| Pod `OOMKilled` | Memory limit too low or JVM not configured for container limits | `kubectl describe pod <n>` → Last State: OOMKilled | Add `-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0` to JVM args; increase memory limit if needed |
| Secret not found | Secret created in wrong namespace, or wrong name in pod spec | `kubectl get secrets -n <ns>` | Verify secret exists in the same namespace as the pod |
| ConfigMap changes not picked up | Env vars require pod restart; mounted files need app to re-read | N/A | Run `kubectl rollout restart deployment/<n>` to rolling-restart pods |
| Quota exceeded — pod won't schedule | Namespace ResourceQuota reached | `kubectl describe resourcequota -n <ns>` | Reduce requests, or ask cluster admin to increase quota |
| `base64: invalid input` when decoding a secret | Extra newline from `base64` without `-w 0` on Linux | N/A | Use `base64 -w 0` when encoding; use `base64 -d` (not `--decode`) on macOS |
| ExternalSecret stuck in `SecretSyncedError` | Wrong IAM permissions or wrong secret path in secrets manager | `kubectl describe externalsecret -n <ns>` | Check IAM policy allows `secretsmanager:GetSecretValue` for the correct ARN |
| `helm upgrade` fails with "release not found" | Using `upgrade` before the release exists | `helm list -n <ns>` | Use `helm upgrade --install` which does both in one command |
| ConfigMap key not found in pod | Pod references a key that doesn't exist in the ConfigMap | `kubectl get configmap <name> -n <ns> -o yaml` | Verify the key name matches exactly; check for typos |
| Service has no endpoints (`<none>`) | Pod labels don't match Service selector | `kubectl get pods --show-labels -n <ns>` | Make sure pod labels match `spec.selector` in Service |

---

## Practice Exercises

**Exercise 1 — Registry round-trip:**
Push your `hello-app:1.0.0` image to Docker Hub (or another registry of your choice). Update `values.yaml` to use the full registry path. Delete the local image (`docker rmi hello-app:1.0.0`), then deploy via Helm. Confirm the image was pulled from the registry by checking `kubectl describe pod <n>` — the Events section shows the pull.

**Exercise 2 — ConfigMap-driven configuration:**
Add a new endpoint `GET /api/feature` that returns the value of a `FEATURE_FLAGS` environment variable. Create a ConfigMap with `FEATURE_FLAGS=new-ui:false,dark-mode:true`. Inject it as an environment variable. Verify the endpoint returns the correct value. Then update the ConfigMap value, run `kubectl rollout restart`, and confirm the new value is returned without rebuilding the image.

**Exercise 3 — Resource limits experiment:**
Deploy with `memory.limits: 128Mi`. Generate load using the load generator from Chapter 5:

```bash
kubectl run load-generator --image=busybox --rm -it --restart=Never -- \
  /bin/sh -c 'while true; do wget -q -O- http://hello-app-service:8080/api/hello; done'
```

Watch `kubectl top pods -n hello-app` as memory climbs. When the pod is OOMKilled, observe the `RESTARTS` counter in `kubectl get pods` increment. Now set a reasonable limit based on what you observed in `docker stats` (typically 512Mi–1Gi for Spring Boot), redeploy, and confirm stability under load.

**Exercise 4 — Secret rotation:**
Create a Kubernetes Secret with a test value (e.g., `API_TOKEN=abc123`). Reference it in the Deployment as a mounted file at `/app/secrets/api_token`. Add an endpoint `GET /api/token-info` that returns the **file's SHA256 hash** (not the actual value — never expose secrets via HTTP). 

```java
@GetMapping("/token-info")
public Map<String, String> getTokenInfo() throws Exception {
    String content = Files.readString(Path.of("/app/secrets/api_token")).trim();
    String hash = DigestUtils.sha256Hex(content);  // Apache Commons Codec
    return Map.of("hash", hash, "length", String.valueOf(content.length()));
}
```

Update the Secret with a new value:
```bash
kubectl create secret generic app-secrets \
  --from-literal=API_TOKEN=xyz789 \
  --dry-run=client -o yaml | kubectl apply -f -
```

Run `kubectl rollout restart deployment/hello-app -n hello-app` and verify the hash changes. Now try the same experiment with the secret as an environment variable — notice that `kubectl describe pod` shows the variable exists (but not its value), whereas mounted files show no indication of the secret's contents.

**Exercise 5 — Probe failure simulation:**
Configure a custom readiness indicator that checks for an environment variable `READY=true`. When `READY` is absent or false, the indicator returns `DOWN`. Deploy with `READY=true`. Then update the ConfigMap to set `READY=false` and rollout restart. Watch `kubectl get endpoints` in one terminal and `kubectl get pods -w` in another. Observe the pod transition to `0/1` (readiness failing) and disappear from Service endpoints. Restore `READY=true` and watch it recover — without a restart, just by the probe passing again.

---

## What's Next?

You now have the core building blocks for production Kubernetes deployments: configuration externalized with ConfigMaps, secrets managed securely, resource boundaries enforced, and self-healing applications with proper health probes.

**Part 3: Helm Deep Dive** takes your Helm skills beyond basic templating:
- **Go template engine** — pipelines, conditionals, loops, and the `nindent` function that trips everyone up
- **Named templates (`_helpers.tpl`)** — DRY patterns for labels, selectors, and shared snippets
- **Subcharts** — deploy postgresql, redis, or rabbitmq alongside your app as dependencies
- **Hooks** — run database migrations before new pods start serving traffic
- **Advanced patterns** — feature flags, conditional resources, and generating random passwords

By the end of Part 3, you'll write production-grade Helm charts that handle complex deployments with multiple services, dependencies, and lifecycle hooks.
