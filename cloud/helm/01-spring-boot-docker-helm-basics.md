# Part 1: Spring Boot + Docker + Helm Basics

## Prerequisites
- Java 17+, Docker, Minikube, Maven
- Comfort with terminal commands

## What This Covers
- Build a Spring Boot REST API
- Containerize it with Docker (properly)
- Understand Kubernetes primitives: Pod, Deployment, Service
- Package with Helm and deploy to Kubernetes

---

## The Mental Model

Before touching any code, understand what each layer does and why it exists:

```
Your JAR file
    └── runs inside a Docker Container   (consistent environment)
            └── managed by a Kubernetes Pod   (lifecycle management)
                    └── scaled by a Deployment   (desired state)
                            └── exposed by a Service   (stable networking)
                                    └── packaged by Helm   (templated, repeatable)
```

**Why this stack?**

| Problem | Solution |
|---------|----------|
| "Works on my machine" | Docker — same environment everywhere |
| Manual restarts when app crashes | Kubernetes — self-healing |
| Deploying to dev vs prod differently | Helm — one chart, different values |
| Scaling manually under load | Kubernetes HPA — automatic |

---

## Chapter 1: Spring Boot Application

### Create the Project

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator \
  -d name=hello-app \
  -d groupId=com.example \
  -d artifactId=hello-app \
  -d javaVersion=17 \
  -o hello-app.zip

unzip hello-app.zip -d hello-app && cd hello-app
```

### REST Controller

**`src/main/java/com/example/helloapp/HelloController.java`**

```java
package com.example.helloapp;

import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Value;
import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class HelloController {

    // These values come from application.yml (or env vars in Kubernetes)
    @Value("${app.name:Hello App}")
    private String appName;

    @Value("${app.version:1.0.0}")
    private String appVersion;

    @GetMapping("/hello")
    public String sayHello() {
        return "Hello, World from Spring Boot!";
    }

    @GetMapping("/info")
    public Map<String, String> getInfo() {
        return Map.of(
            "appName", appName,
            "version", appVersion,
            "timestamp", LocalDateTime.now().toString(),
            "status", "healthy"
        );
    }

    @GetMapping("/greet")
    public String greet(
        @RequestParam(defaultValue = "World") String name
    ) {
        return String.format("Hello, %s!", name);
    }
}
```

### application.yml

```yaml
spring:
  application:
    name: hello-app
  shutdown: graceful          # Wait for in-flight requests before stopping

server:
  port: 8080

app:
  name: Hello Kubernetes App
  version: 1.0.0

# Actuator exposes /actuator/health, /actuator/metrics etc.
# Kubernetes uses /actuator/health/liveness and /actuator/health/readiness
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true         # Enables /actuator/health/liveness and /readiness
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### Test Locally

```bash
./mvnw spring-boot:run

curl http://localhost:8080/api/hello
# Hello, World from Spring Boot!

curl http://localhost:8080/actuator/health
# {"status":"UP"}

curl http://localhost:8080/actuator/health/readiness
# {"status":"UP"}
```

---

## Chapter 2: Docker

### Key Concepts

| Term | Analogy | What it is |
|------|---------|------------|
| Dockerfile | Recipe | Instructions to build an image |
| Image | Frozen pizza | Built artifact, ready to run |
| Container | Baked pizza | Running instance of an image |
| Registry | Grocery store | Stores and distributes images |

**Why multi-stage builds?** The JDK (needed to compile) is ~400MB. The JRE (needed to run) is ~200MB. With multi-stage builds, you compile in a large image and copy only the JAR to a small runtime image. The final image ships without Maven, source code, or build tools.

### Dockerfile

```dockerfile
# Stage 1: Build
# Full JDK + Maven to compile the application
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline          # Cache dependencies separately (Docker layer cache)
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime
# Only the JRE — no build tools, smaller attack surface
FROM eclipse-temurin:17-jre-alpine

# Security: never run as root inside a container
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080

# Health check (separate from Kubernetes probes, used by Docker itself)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", "/app/app.jar"]
```

> **JVM note:** `-XX:+UseContainerSupport` tells the JVM to respect cgroup memory limits (the container's memory limit) rather than the host machine's total RAM. Without this, the JVM may try to allocate far more heap than the container allows, causing OOMKill. This flag is on by default in Java 11+ but explicit is better.

### .dockerignore

```
.git/
target/
*.md
.idea/
*.iml
.env
```

### Build and Test

```bash
./mvnw clean package

docker build -t hello-app:1.0.0 .

docker run -d --name hello-test -p 8080:8080 hello-app:1.0.0

curl http://localhost:8080/api/hello
docker logs hello-test
docker ps                              # Check health status

docker stop hello-test && docker rm hello-test
```

---

## Chapter 3: Kubernetes Basics

### Core Concepts

| Resource | Analogy | Purpose |
|----------|---------|---------|
| **Pod** | One apartment | Runs one or more containers |
| **Deployment** | Building manager | Ensures N pods are always running |
| **Service** | Building address | Stable way to reach pods (their IPs change) |
| **Namespace** | Floor of the building | Logical grouping and isolation |
| **Node** | The building | Physical/virtual machine |
| **Cluster** | Apartment complex | Collection of nodes |

**Why Services exist:** Pods are ephemeral. When a pod restarts, it gets a new IP address. A Service provides a stable IP and DNS name that always points to healthy pods, regardless of restarts or scaling.

### Start Kubernetes Locally

```bash
minikube start --cpus=4 --memory=8192 --driver=docker
minikube addons enable ingress
minikube addons enable metrics-server

kubectl cluster-info
kubectl get nodes
```

### Deployment Manifest

**`k8s/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-deployment
  labels:
    app: hello-app
spec:
  replicas: 3

  # selector tells the Deployment which pods it owns
  selector:
    matchLabels:
      app: hello-app

  template:
    metadata:
      labels:
        app: hello-app          # Must match selector above
    spec:
      containers:
      - name: hello-app
        image: hello-app:1.0.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080

        # Resource requests: what the pod is guaranteed
        # Resource limits: the maximum it can use
        # Without these, pods compete unpredictably (noisy neighbor problem)
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"         # 250m = 0.25 of one CPU core
          limits:
            memory: "512Mi"
            cpu: "500m"

        # readinessProbe: "Is the app ready to receive traffic?"
        # Kubernetes stops sending traffic if this fails — no 503s
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3

        # livenessProbe: "Is the app still alive?"
        # Kubernetes restarts the container if this fails
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        # preStop: Give in-flight requests time to finish before the pod dies
        # Works together with spring.shutdown: graceful
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
```

### Service Manifest

**`k8s/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
spec:
  # ClusterIP: reachable only inside the cluster (default, cheapest)
  # NodePort: reachable via <node-ip>:<port> (for local testing)
  # LoadBalancer: cloud load balancer (production, costs money per service)
  type: ClusterIP

  selector:
    app: hello-app              # Routes to pods with this label

  ports:
  - port: 8080                  # Port the service listens on
    targetPort: 8080            # Port on the pod to forward to
    protocol: TCP
```

### Deploy and Test

```bash
kubectl create namespace hello-app

kubectl apply -f k8s/deployment.yaml -n hello-app
kubectl apply -f k8s/service.yaml -n hello-app

# Watch pods start up
kubectl get pods -n hello-app -w

# Verify deployment
kubectl get deployments -n hello-app

# Test from inside the cluster
kubectl run test-pod --image=curlimages/curl -it --rm --restart=Never -n hello-app -- \
  curl http://hello-app-service:8080/api/hello

# Test from your machine via port-forward
kubectl port-forward service/hello-app-service 8080:8080 -n hello-app
curl http://localhost:8080/api/hello

# Useful debugging commands
kubectl logs -f deployment/hello-app-deployment -n hello-app
kubectl describe pod <pod-name> -n hello-app
kubectl get all -n hello-app
```

---

## Chapter 4: Helm

### Why Helm Exists

Without Helm you maintain separate YAML files per environment:

```
k8s/dev/deployment.yaml      replicas: 1, image: dev-latest
k8s/staging/deployment.yaml  replicas: 2, image: staging-latest
k8s/prod/deployment.yaml     replicas: 5, image: v1.2.3
```

95% of each file is identical. Any shared change (add a label, change a probe) must be applied to all three. Helm solves this with templates + values.

### Create the Chart

```bash
helm create hello-app-chart
```

This generates:
```
hello-app-chart/
├── Chart.yaml          # Chart metadata (name, version)
├── values.yaml         # Default values — override per environment
├── templates/
│   ├── _helpers.tpl    # Named templates (shared snippets)
│   ├── deployment.yaml # Template files, not plain YAML
│   ├── service.yaml
│   └── ingress.yaml
└── charts/             # Subchart dependencies
```

### How Templates Work

Helm uses Go templates. The `{{ }}` syntax evaluates expressions at deploy time:

```yaml
# templates/deployment.yaml
metadata:
  name: {{ include "hello-app.fullname" . }}   # Calls a named template
  labels:
    {{- include "hello-app.labels" . | nindent 4 }}  # nindent adds 4-space indent to each line

spec:
  replicas: {{ .Values.replicaCount }}         # .Values = your values.yaml
  # .Release.Name = the name you give at helm install
  # .Chart.Name   = name from Chart.yaml
```

**The `include` vs inline choice:** Put anything used in more than one template into `_helpers.tpl` as a named template. This is the Helm equivalent of a function.

### `_helpers.tpl` — the important bits

```go
{{/*
Full name: <release-name>-<chart-name>, truncated to 63 chars (Kubernetes limit)
*/}}
{{- define "hello-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end -}}

{{/*
Common labels — applied to every resource so they're queryable
*/}}
{{- define "hello-app.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
Selector labels — used in Deployment.selector and Service.selector
Must be stable (don't change after creation) — different from common labels
*/}}
{{- define "hello-app.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

> **Why two label templates?** `selectorLabels` are used in `matchLabels` and `selector` fields. Kubernetes requires these to never change after a resource is created. `labels` (common labels) include version info that changes on every release — fine on metadata, not fine on selectors.

### values.yaml

```yaml
replicaCount: 2

image:
  repository: hello-app
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 10
    periodSeconds: 5

env:
  SPRING_PROFILES_ACTIVE: kubernetes
  APP_NAME: Hello Helm App
```

### Environment-Specific Values

```yaml
# values-dev.yaml — override only what differs
replicaCount: 1
image:
  tag: dev-latest
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi
env:
  SPRING_PROFILES_ACTIVE: dev
  APP_NAME: Hello App - Dev
```

```yaml
# values-prod.yaml
replicaCount: 5
image:
  tag: v1.2.3             # Always pin to a specific version in prod
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.hello-app.example.com
      paths:
        - path: /
          pathType: Prefix
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi
env:
  SPRING_PROFILES_ACTIVE: prod
  APP_NAME: Hello App - Production
```

### Helm Commands You'll Use Constantly

```bash
# Validate syntax
helm lint hello-app-chart/

# Preview what will be deployed (doesn't actually deploy)
helm template my-release ./hello-app-chart -f values-dev.yaml

# Dry run (connects to cluster, validates against API, doesn't deploy)
helm install my-release ./hello-app-chart --dry-run --debug

# Deploy to dev
helm install hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace dev \
  --create-namespace

# See what's deployed
helm list -n dev
helm get values hello-app-dev -n dev
helm get manifest hello-app-dev -n dev

# Update (deploy new version or changed values)
helm upgrade hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace dev

# Roll back to previous release
helm rollback hello-app-dev 1 -n dev

# Deploy to prod
helm upgrade --install hello-app-prod ./hello-app-chart \
  -f values-prod.yaml \
  --namespace prod \
  --create-namespace \
  --wait \
  --timeout 5m

# Clean up
helm uninstall hello-app-dev -n dev

# Package chart for distribution
helm package hello-app-chart/
# Produces: hello-app-1.0.0.tgz
```

> **`--wait` flag:** Helm blocks until all pods are ready, or times out. Use in CI/CD so your pipeline fails fast if deployment doesn't succeed.

---

## Chapter 5: Container Registry

Your Kubernetes cluster cannot pull images from your laptop. Images must live in a registry.

### Push to Docker Hub

```bash
docker login

docker tag hello-app:1.0.0 yourusername/hello-app:1.0.0
docker tag hello-app:1.0.0 yourusername/hello-app:latest

docker push yourusername/hello-app:1.0.0
docker push yourusername/hello-app:latest
```

Update your values file to use the registry image:

```yaml
# values-prod.yaml
image:
  repository: yourusername/hello-app
  tag: 1.0.0
  pullPolicy: Always              # Always re-pull on pod restart
```

### Image Pull Secret (private registries)

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=yourpassword \
  --docker-email=your@email.com \
  -n production
```

Reference in values:

```yaml
imagePullSecrets:
  - name: regcred
```

---

## Quick Reference

### Probe Decision Guide

| Probe | Question | Failure action |
|-------|----------|---------------|
| `startupProbe` | Still initializing? | Disables liveness until passes |
| `readinessProbe` | Ready for traffic? | Removed from Service (no traffic) |
| `livenessProbe` | Still alive? | Container restarted |

Use `startupProbe` for Spring Boot — it takes 20-60s to start, and without a startup probe, the liveness probe may kill it before it finishes booting.

```yaml
startupProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  failureThreshold: 30    # 30 * 5s = 150 seconds max startup time
  periodSeconds: 5
```

### Resource Sizing Guide (Spring Boot)

| Environment | CPU Request | CPU Limit | Memory Request | Memory Limit |
|-------------|-------------|-----------|----------------|--------------|
| Dev | 100m | 200m | 128Mi | 256Mi |
| Staging | 250m | 500m | 256Mi | 512Mi |
| Prod | 500m | 2000m | 512Mi | 1Gi |

Memory limit should be at least 2x request to give the JVM room to breathe. The JVM with `-XX:MaxRAMPercentage=75.0` will use 75% of the limit as max heap.

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `ImagePullBackOff` | Cluster can't pull image | Push to registry, check image name/tag |
| `CrashLoopBackOff` | App crashes on start | `kubectl logs <pod>` to see why |
| `Pending` | No node has room | Check resource requests, `kubectl describe pod` |
| `OOMKilled` | Exceeded memory limit | Increase limit or fix memory leak |
| `Connection refused` | Service selector mismatch | Check labels match between Service and pods |
| Helm template error | Go template syntax | Run `helm template --debug` to see rendered output |
