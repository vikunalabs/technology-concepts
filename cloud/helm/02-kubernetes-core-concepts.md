## Part 2: Kubernetes Core Concepts

## Kubernetes Core Concepts: Beyond Hello World

### Prerequisites
- Completed Part 1 (Spring Boot + Docker + Helm basics)
- Docker installed and running
- Minikube or a cloud Kubernetes cluster (EKS/GKE/AKS)
- `kubectl` installed

### What You'll Learn
- ✅ Organize apps with **Namespaces**
- ✅ Manage configuration with **ConfigMaps**
- ✅ Handle secrets securely
- ✅ Set **resource limits** (no more noisy neighbors!)
- ✅ Add **health checks** for self-healing apps
- ✅ Push images to **container registries** (real cloud deployment)

---

## Chapter 1: Container Registry - The Missing Link from Part 1

### The Problem You Discovered

Remember Part 1 where we did:
```bash
docker build -t my-hello-app:1.0 .
helm install my-release ./hello-helm-app
```

**Why this fails in the cloud:** Your cloud cluster can't see the Docker image on your laptop. It's like telling a restaurant in New York to use ingredients from your personal fridge in Tokyo.

### What is a Container Registry?

**Analogy:** Think of Docker Hub as **GitHub for Docker images**:
- **GitHub** stores your code
- **Docker Hub** stores your built images
- **Your cluster** downloads images from registry on demand

### How to Use Container Registries

**Step 1: Choose a Registry**

| Registry | Free Tier | Best For |
|----------|-----------|----------|
| Docker Hub | 1 private repo, unlimited public | Learning, open source |
| AWS ECR | Pay for storage (very cheap) | AWS users |
| Google GCR | 5GB/month free | GCP users |
| Azure ACR | Basic tier cheap | Azure users |
| GitHub Container Registry | 500MB free | GitHub users |

**Step 2: Push Your Image**

```bash
# Option A: Docker Hub (easiest for learning)
docker tag my-hello-app:1.0 yourusername/my-hello-app:1.0
docker push yourusername/my-hello-app:1.0

# Option B: AWS ECR (production)
aws ecr create-repository --repository-name my-hello-app
aws ecr get-login-password | docker login --username AWS --password-stdin YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
docker tag my-hello-app:1.0 YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-hello-app:1.0
docker push YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-hello-app:1.0
```

**Step 3: Update Helm Values**

```yaml
# values.yaml - Now using real registry!
image:
  repository: yourusername/my-hello-app  # or full ECR URL
  tag: 1.0
  pullPolicy: Always  # Always pull fresh image
```

**Step 4: Create Image Pull Secret (for private registries)**

```yaml
# Create secret for Docker Hub
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=yourpassword \
  --docker-email=your@email.com

# Reference in Helm values
imagePullSecrets:
  - name: regcred
```

**Why this matters now:** Your cloud cluster can now pull your image from a registry, not your laptop!

---

## Chapter 2: Namespaces - Organizing Your Cluster

### The Problem

Without namespaces, your cluster becomes a messy shared apartment:
```
kubectl get pods
NAME                          STATUS
alice-dev-app-123            Running
bob-test-database-456        Running
production-api-789           Running
staging-api-101              Running
alice-test-app-112           Running
... (50 more confusing pods)
```

### What are Namespaces?

**Analogy:** 
- **Cluster** = Entire office building
- **Namespace** = Individual team's floor
- **Resources** = Desks, computers on that floor

**Why namespaces?**
- **Isolation** - Dev doesn't see prod resources
- **Organization** - Group related apps
- **Resource limits** - Limit CPU/memory per team
- **Access control** - Devs can't delete prod

### Common Namespace Patterns

```yaml
# Typical namespace structure for any company
├── dev           # Development (unstable, frequent changes)
├── staging       # Testing (stable for QA)
├── prod          # Production (customer-facing, critical)
├── monitoring    # Prometheus, Grafana (observability tools)
└── ingress-nginx # Ingress controllers (networking)
```

### Working with Namespaces

```bash
# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Deploy to specific namespace
helm install my-app ./chart --namespace dev --create-namespace

# List resources in namespace
kubectl get pods -n dev
kubectl get all -n staging

# Switch default namespace (saves typing)
kubectl config set-context --current --namespace=dev

# Delete everything in a namespace (careful!)
kubectl delete namespace dev  # Deletes ALL resources in dev
```

### Namespace YAML Example

```yaml
# namespace-dev.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
    team: backend
---
# ResourceQuota - Limit total resources in namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"
    pods: "20"
```

**Helm with Namespaces:**

```bash
# Chart.yaml - Make namespace configurable
# values-dev.yaml
namespace: dev
replicaCount: 1

# Deploy with different values per namespace
helm install myapp-dev ./chart -f values-dev.yaml -n dev
helm install myapp-staging ./chart -f values-staging.yaml -n staging
helm install myapp-prod ./chart -f values-prod.yaml -n prod
```

---

## Chapter 3: ConfigMaps - Configuration Without Rebuilding

### The Problem

Without ConfigMaps:
1. Change a database URL
2. Rebuild Docker image (`docker build`)
3. Push to registry (`docker push`)
4. Restart pods (`kubectl rollout restart`)
5. Wait 5 minutes... **This is insane!**

### What are ConfigMaps?

**Analogy:** 
- **ConfigMap** = Whiteboard where you write settings
- **Application** = Employee who reads the whiteboard
- **Changing settings** = Erase and rewrite on whiteboard (app sees new values instantly or after restart)

**Why ConfigMaps?**
- **Separation** - Code vs configuration
- **Reusability** - Same image, different configs (dev/staging/prod)
- **Agility** - Change config without rebuild

### Creating ConfigMaps

**Method 1: From literal values**
```bash
# Simple key-value pairs
kubectl create configmap app-config \
  --from-literal=app.name=hello-app \
  --from-literal=log.level=DEBUG \
  --from-literal=feature.new-ui=enabled

# Resulting ConfigMap:
# app.name=hello-app
# log.level=DEBUG
# feature.new-ui=enabled
```

**Method 2: From file**
```bash
# application.properties
echo "spring.datasource.url=jdbc:postgresql://localhost:5432/mydb" > app.properties
echo "spring.datasource.username=user" >> app.properties

kubectl create configmap app-properties --from-file=app.properties
```

**Method 3: From YAML (declarative)**
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config
  namespace: dev
data:
  # Key-value pairs
  player.initial.lives: "3"
  ui.properties: |
    color.good=green
    color.bad=red
    # Multi-line string using pipe (|)
  
  # File-like config
  database.conf: |
    host=postgres-svc
    port=5432
    dbname=mydb
```

### Using ConfigMaps in Your Spring Boot App

**Option 1: Environment Variables (Easiest)**
```yaml
# deployment.yaml snippet
spec:
  containers:
  - name: app
    env:
    # Single value from ConfigMap
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log.level
    
    # All values as environment variables
    envFrom:
    - configMapRef:
        name: app-config
```

**Option 2: Mount as File (For config files)**
```yaml
# deployment.yaml snippet
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config-volume
      mountPath: /app/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config  # Each key becomes a file
      # app.properties file will appear at /app/config/app.properties
```

**Spring Boot Example:**
```java
@SpringBootApplication
public class HelloApplication {
    
    @Value("${app.name:default}")  // Reads from environment or config
    private String appName;
    
    @Value("${log.level:INFO}")
    private String logLevel;
    
    @GetMapping("/config")
    public String showConfig() {
        return "App: " + appName + ", Log Level: " + logLevel;
    }
}
```

### Helm + ConfigMaps = Powerful!

```yaml
# values-dev.yaml
config:
  app:
    name: hello-dev
    logLevel: DEBUG
  database:
    host: postgres-dev-svc
    port: 5432

# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "app.fullname" . }}
data:
  application.yml: |
    app:
      name: {{ .Values.config.app.name }}
      log:
        level: {{ .Values.config.app.logLevel }}
    
    spring:
      datasource:
        url: jdbc:postgresql://{{ .Values.config.database.host }}:{{ .Values.config.database.port }}/mydb

# Update config without rebuild
helm upgrade myapp ./chart -f values-dev.yaml
# Pods need restart to see changes (kubectl rollout restart)
```

---

## Chapter 4: Secrets - Handling Sensitive Data

### The Problem

ConfigMaps store data in **plain text**. Anyone with access can see:
- Database passwords
- API keys
- TLS certificates
- Third-party tokens

### What are Secrets?

**Analogy:** 
- **ConfigMap** = Post-it note on monitor (everyone sees)
- **Secret** = Locked safe (encrypted, access-controlled)

**Why Secrets?**
- **Encryption** - Data is encrypted at rest (in etcd)
- **Access control** - RBAC to restrict who sees secrets
- **Safety** - Not accidentally printed in logs

### Creating Secrets

**Method 1: From literal**
```bash
kubectl create secret generic db-secret \
  --from-literal=username=produser \
  --from-literal=password=S3cr3tP@ssw0rd \
  --from-literal=api-key=abc123xyz
```

**Method 2: From files (safer for complex data)**
```bash
# Store password in file (don't commit this!)
echo -n "S3cr3tP@ssw0rd" > ./password.txt
kubectl create secret generic db-secret --from-file=./password.txt

# Multiple files
kubectl create secret generic app-secrets \
  --from-file=./secrets/db-password \
  --from-file=./secrets/api-key \
  --from-file=./secrets/jwt-secret
```

**Method 3: Declarative YAML (with encoding)**
```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque  # Generic secret type
data:
  # Values must be base64 encoded!
  username: cHJvZHVzZXI=  # echo -n "produser" | base64
  password: UzNjcjN0UEBzc3cwcmQ=  # echo -n "S3cr3tP@ssw0rd" | base64
  
# For readability, use stringData (plain text, auto-encoded)
stringData:
  api-key: "abc123xyz-secret-key"
  jwt-secret: "my-super-secret-jwt-key"
```

**⚠️ WARNING:** Never commit unencoded secrets to Git! Use tools like:
- **Sealed Secrets** (encrypt secrets for Git)
- **External Secrets Operator** (sync from AWS Secrets Manager)
- **Hashicorp Vault** (enterprise solution)

### Using Secrets in Pods

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
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
        
        # As mounted files (more secure, won't appear in env)
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: secret-volume
        secret:
          secretName: db-secret
          # Each key becomes a file: /etc/secrets/username, /etc/secrets/password
```

**Spring Boot with Secrets:**
```java
@ConfigurationProperties(prefix = "db")
@Component
public class DatabaseConfig {
    private String username;  // From env: DB_USERNAME
    private String password;  // From env: DB_PASSWORD
    
    // Or read from mounted file
    @Value("${db.password.file}")
    private String passwordFile;  // /etc/secrets/password
}
```

### Best Practice: Don't Use kubectl create secret!

Use **Helm with external secrets**:

```yaml
# values.yaml - Reference external secret (don't store values!)
secrets:
  dbPasswordRef: "arn:aws:secretsmanager:us-east-1:123:secret:db-password"
  apiKeyRef: "secret://api-key"

# Install external-secrets operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets

# Let K8s sync from cloud secret manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  secretStoreRef:
    name: aws-secretsmanager
  data:
  - secretKey: db-password
    remoteRef:
      key: production/db/password
```

---

## Chapter 5: Resource Management - Stop Noisy Neighbors

### The Problem

Without resource limits:
- One bad app consumes all CPU → Other apps starve
- Memory leak crashes node → Entire node goes down
- No way to schedule pods (K8s doesn't know resource needs)

### Resource Requests vs Limits

**Analogy - Renting an apartment:**
- **Request** = Minimum guaranteed space (you always get this)
- **Limit** = Maximum allowed space (can't exceed this)
- **K8s** = Building manager (schedules based on requests)

```yaml
# Example: A realistic Spring Boot app
resources:
  requests:
    memory: "256Mi"   # Guaranteed minimum (always available)
    cpu: "250m"       # 1/4 of a CPU core guaranteed
  limits:
    memory: "512Mi"   # Can use up to 512MB, then OOM killed
    cpu: "500m"       # Can burst to 1/2 core, then throttled
```

### Why Both Are Important

| Scenario | Without Requests | Without Limits |
|----------|-----------------|----------------|
| **Scheduling** | K8s guesses wrong → overcommits nodes | Fine |
| **Resource contention** | Some pods starve | Some pods get throttled |
| **OOM kills** | Random kills | Controlled kills |
| **Cost** | Can't optimize | Can't limit spend |

### Finding the Right Values

**Step 1: Measure locally**
```bash
# Test your Spring Boot app locally
docker run -d --name test my-app:latest

# Monitor memory usage
docker stats test
# CONTAINER ID   CPU %   MEM USAGE / LIMIT
# abc123         2.5%    328MiB / 7.7GiB

# Load test (use hey, ab, or wrk)
hey -n 10000 -c 100 http://localhost:8080/hello
```

**Step 2: Add to your Helm chart**

```yaml
# values.yaml (reasonable defaults for Spring Boot)
resources:
  requests:
    cpu: 200m      # 0.2 core
    memory: 256Mi  # 256 megabytes
  limits:
    cpu: 500m      # 0.5 core (can spike to 2.5x request)
    memory: 512Mi  # 512MB (2x request, safe for JVM)
```

**Step 3: Use Vertical Pod Autoscaler (advanced)**
```bash
# Let K8s auto-recommend values
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/download/vertical-pod-autoscaler-0.13.0/vpa-v1.0.yaml

# Create VPA
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app-deployment
  updatePolicy:
    updateMode: "Auto"  # Automatically adjust
```

### JVM-Specific Considerations

**⚠️ Critical for Java apps:** The JVM's memory usage is tricky!

```yaml
# BAD - This WILL crash your app!
resources:
  limits:
    memory: 512Mi
# JVM sees 512Mi and reserves 1/4 for heap (128Mi) → Too small!

# GOOD - Use container-aware JVM (Java 10+)
# Add to Dockerfile:
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# In deployment:
resources:
  limits:
    memory: 512Mi
# JVM uses 75% of 512Mi = 384Mi heap (good!)

# For Java 8, use explicit flags:
ENV JAVA_OPTS="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2"
```

---

## Chapter 6: Health Probes - Self-Healing Applications

### The Problem

Kubernetes needs to know:
- Is your app **running** (process alive)?
- Is your app **ready** (can handle traffic)?
- Is your app **starting up** (needs extra time)?

Without probes, K8s sends traffic to broken apps or kills starting apps.

### The Three Probes

| Probe | Purpose | What Happens on Failure |
|-------|---------|------------------------|
| **livenessProbe** | Is app still alive? | Restart container |
| **readinessProbe** | Can app handle traffic? | Remove from service (no traffic) |
| **startupProbe** | Is app still starting? | Disable liveness during startup |

### Real-World Example: Spring Boot Startup

```yaml
# deployment.yaml with all three probes
spec:
  containers:
  - name: spring-boot-app
    image: myapp:latest
    
    # READINESS: Is app ready for traffic?
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      initialDelaySeconds: 10  # Wait 10s before first check
      periodSeconds: 5         # Check every 5 seconds
      failureThreshold: 3      # Fail after 3 failures (15s)
    
    # LIVENESS: Is app still alive?
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 60  # Give app time to start
      periodSeconds: 10
      failureThreshold: 3
    
    # STARTUP: Give slow-starting apps time
    startupProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
      failureThreshold: 30      # Allow 30 failures
      periodSeconds: 5          # Check every 5 seconds
      # Total startup time allowed: 30 * 5 = 150 seconds
```

### Adding Health Endpoints to Spring Boot

**Add dependency to `pom.xml`:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

**Configure in `application.yml`:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # Enable liveness/readiness endpoints
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

**Now your app has:**
- `GET /actuator/health/liveness` → Always returns 200 (app is running)
- `GET /actuator/health/readiness` → 200 only if DB, caches, etc. are ready
- `GET /actuator/health` → Combined status

### Testing Probes Locally

```bash
# Start your app
./mvnw spring-boot:run

# Check health endpoints
curl http://localhost:8080/actuator/health/liveness
# {"status":"UP"}

curl http://localhost:8080/actuator/health/readiness
# {"status":"UP"}

# Simulate broken state (add this controller)
@GetMapping("/break")
public String breakApp() {
    // This will make readiness probe fail
    return "Broken!";
}
```

### Common Probe Patterns

```yaml
# Pattern 1: Fast, simple web app
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10

# Pattern 2: App with slow initialization (database migrations)
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 60
  periodSeconds: 5
# No liveness probe needed until startup succeeds

# Pattern 3: Non-HTTP app (gRPC, queue consumer)
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy  # App touches this file when healthy
  initialDelaySeconds: 5
  periodSeconds: 5

# Pattern 4: TCP-based probe
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10
```

### Why Probes Matter: A Real Scenario

**Without probes:**
```
1. App starts (takes 45 seconds to load Spring context)
2. K8s sends traffic immediately → Connection refused
3. Users see 500 errors
4. App finally starts, but some users are already gone
```

**With probes:**
```
1. App starts
2. Readiness probe fails (returns 503)
3. K8s keeps app out of service
4. After 45 seconds, readiness probe succeeds
5. K8s adds app to service
6. Zero downtime! ✅
```

---

## Chapter 7: Putting It All Together

### Complete Production-Ready Deployment

Here's everything combined into one example:

**`values-prod.yaml`:**
```yaml
# Environment
namespace: prod

# Application config
replicaCount: 3

image:
  repository: your-registry/my-hello-app
  tag: v1.2.3
  pullPolicy: Always

# Resource limits (based on load testing)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# ConfigMap data
config:
  app:
    name: hello-prod
    logLevel: INFO
  database:
    host: postgres-prod.internal
    port: 5432

# References to secrets (managed externally)
secrets:
  dbPasswordSecretName: prod-db-password
  apiKeySecretName: prod-api-key

# Probe configuration
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 60
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 10
    periodSeconds: 5
  startup:
    enabled: true
    failureThreshold: 30
    periodSeconds: 5

# Image pull secret for private registry
imagePullSecrets:
  - name: regcred-prod
```

**`templates/deployment.yaml` (complete version):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  namespace: {{ .Values.namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "app.name" . }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        
        ports:
        - containerPort: 8080
          name: http
        
        # Environment variables from ConfigMap
        env:
        - name: SPRING_APPLICATION_NAME
          valueFrom:
            configMapKeyRef:
              name: {{ include "app.fullname" . }}-config
              key: app.name
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: {{ include "app.fullname" . }}-config
              key: log.level
        
        # Secrets as environment
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Values.secrets.dbPasswordSecretName }}
              key: password
        
        # Resource limits
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        
        # Probes
        livenessProbe:
          httpGet:
            path: {{ .Values.probes.liveness.path }}
            port: http
          initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: {{ .Values.probes.readiness.path }}
            port: http
          initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
          periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          failureThreshold: 3
        
        {{- if .Values.probes.startup.enabled }}
        startupProbe:
          httpGet:
            path: {{ .Values.probes.readiness.path }}
            port: http
          failureThreshold: {{ .Values.probes.startup.failureThreshold }}
          periodSeconds: {{ .Values.probes.startup.periodSeconds }}
        {{- end }}
        
        volumeMounts:
        - name: config
          mountPath: /app/config
          readOnly: true
      
      volumes:
      - name: config
        configMap:
          name: {{ include "app.fullname" . }}-config
```

### Deploy to Production

```bash
# 1. Push image to registry
docker build -t your-registry/my-hello-app:v1.2.3 .
docker push your-registry/my-hello-app:v1.2.3

# 2. Create namespace
kubectl create namespace prod

# 3. Create secrets (from your secret manager)
kubectl create secret generic prod-db-password \
  --from-literal=password=$(aws secretsmanager get-secret-value --secret-id prod/db/password --query SecretString --output text) \
  -n prod

# 4. Deploy with Helm
helm install hello-prod ./helm-chart \
  -f values-prod.yaml \
  -n prod

# 5. Verify deployment
kubectl get pods -n prod -w
kubectl get svc -n prod
kubectl describe pod -n prod <pod-name>

# 6. Test health endpoints
kubectl port-forward pod/<pod-name> 8080:8080 -n prod
curl http://localhost:8080/actuator/health

# 7. Monitor rollout
kubectl rollout status deployment/hello-prod -n prod
```

---

## Summary: What You've Learned

| Concept | Why It Matters | How to Use |
|---------|----------------|------------|
| **Container Registry** | Cloud needs to access your image | Push to Docker Hub/ECR/GCR |
| **Namespaces** | Organize and isolate environments | `kubectl create namespace` |
| **ConfigMaps** | Change config without rebuild | `kubectl create configmap` |
| **Secrets** | Secure sensitive data | Use external secrets operator |
| **Resource Limits** | Prevent noisy neighbors | Set requests/limits in deployment |
| **Health Probes** | Self-healing, zero downtime | Add actuator, configure probes |

## Hands-On Exercises

### Exercise 1: Set Up Registry and Deploy to Cloud
1. Create Docker Hub account (free)
2. Push your image from Part 1 to Docker Hub
3. Update Helm chart to use Docker Hub image
4. Deploy to Minikube or cloud cluster

### Exercise 2: Multi-Namespace Deployment
1. Create dev, staging, prod namespaces
2. Deploy same Helm chart with different values to each
3. Verify isolation (can't see pods across namespaces)

### Exercise 3: ConfigMap Rollout
1. Create ConfigMap with log.level=DEBUG
2. Deploy app and verify DEBUG logs
3. Update ConfigMap to log.level=INFO
4. Restart pods and verify new log level

### Exercise 4: Resource Limits
1. Add resource limits to your deployment
2. Deploy and verify with `kubectl describe pod`
3. Try to exceed limits (add memory leak) and watch OOM kill

### Exercise 5: Health Probes
1. Add Spring Boot Actuator to your app
2. Configure liveness/readiness probes
3. Simulate failure (create /break endpoint that fails readiness)
4. Watch K8s stop sending traffic

## Next Steps

After mastering these core concepts, you're ready for:

- **Part 3: Helm Deep Dive** - Custom templates, subcharts, hooks
- **Part 4: CI/CD Pipeline** - GitHub Actions → Build → Push → Deploy
- **Part 5: Networking & Ingress** - TLS, DNS, load balancing
- **Part 6: Storage** - Databases in K8s (StatefulSets, PV/PVC)

## Troubleshooting Common Issues

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| `ImagePullBackOff` | Can't pull image | Check registry credentials, image name |
| `CrashLoopBackOff` | App crashes on start | Check logs, increase memory limits |
| `Pod stuck in Pending` | Insufficient resources | Add nodes, reduce requests |
| `Connection refused` | Readiness probe failing | Check app startup time, increase initialDelaySeconds |
| `OOMKilled` | Exceeds memory limit | Increase memory limit, fix memory leak |
| `ConfigMap not found` | Wrong namespace | Check namespace of ConfigMap |

---