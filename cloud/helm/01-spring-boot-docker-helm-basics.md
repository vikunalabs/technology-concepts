## Part 1: Spring Boot + Docker + Helm Basics - Building Your First Cloud-Ready App

### Prerequisites
- Java 17 or later installed
- Docker installed and running
- Minikube (for local Kubernetes) or cloud account
- Basic knowledge of Java/Spring Boot
- Terminal/command line comfort

### What You'll Learn
- ✅ Create a production-ready Spring Boot REST API
- ✅ Containerize your app with Docker best practices
- ✅ Understand Kubernetes basics (Pods, Deployments, Services)
- ✅ Package your app with Helm charts
- ✅ Deploy to any Kubernetes cluster
- ✅ Connect all pieces together

---

## Chapter 1: The Big Picture - Why This Stack?

### What We're Building

```
┌─────────────────────────────────────────────────────────────┐
│                    User Request                             │
│              GET /hello → "Hello, World!"                   │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Ingress (Part 5)                         │
│                 api.myapp.com/hello                         │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Service                       │
│              Load balances to healthy pods                  │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Spring Boot Pod (Your App)                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Docker Container with Java 17 + Your JAR           │    │
│  │  - Listens on port 8080                             │    │
│  │  - Returns "Hello, World!"                          │    │
│  │  - Health checks at /actuator/health                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### The Complete Flow

**Analogy - Restaurant Chain:**
- **Spring Boot App** = Your secret recipe (the food)
- **Docker** = Kitchen + equipment (standardized environment)
- **Kubernetes** = Restaurant chain manager (scales, manages locations)
- **Helm** = Standard operating procedures (SOPs for each location)
- **Cloud** = Physical restaurant buildings (AWS, GCP, Azure)

---

## Chapter 2: Spring Boot Application - The Foundation

### Step 1: Create the Project

**Method 1: Using Spring Initializr (Web UI)**
1. Go to https://start.spring.io
2. Select:
   - Project: Maven
   - Language: Java
   - Spring Boot: 3.2.x
   - Group: com.example
   - Artifact: hello-app
   - Name: hello-app
   - Package name: com.example.helloapp
   - Packaging: Jar
   - Java: 17
3. Add dependencies:
   - Spring Web
   - Spring Boot Actuator (for health checks)
4. Click "Generate" → Download ZIP
5. Extract to `hello-app/`

**Method 2: Using curl (Command Line)**
```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator \
  -d name=hello-app \
  -d groupId=com.example \
  -d artifactId=hello-app \
  -d javaVersion=17 \
  -o hello-app.zip

unzip hello-app.zip -d hello-app
cd hello-app
```

### Step 2: Create REST Controller

**Create `src/main/java/com/example/helloapp/HelloController.java`:**
```java
package com.example.helloapp;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.beans.factory.annotation.Value;
import java.time.LocalDateTime;
import java.util.Map;

/**
 * Simple REST controller for our hello world application.
 * 
 * WHAT: Handles HTTP requests and returns responses
 * WHY: Spring Boot maps HTTP requests to Java methods
 * HOW: @RestController tells Spring this class handles web requests
 * WHO: Developers write this once, users consume the API
 */
@RestController
@RequestMapping("/api")  // Base path for all endpoints
public class HelloController {
    
    @Value("${app.name:Hello App}")
    private String appName;
    
    @Value("${app.version:1.0.0}")
    private String appVersion;
    
    /**
     * Basic hello world endpoint
     * GET /api/hello → "Hello, World from Spring Boot!"
     */
    @GetMapping("/hello")
    public String sayHello() {
        return "Hello, World from Spring Boot!";
    }
    
    /**
     * JSON response with metadata
     * GET /api/info → {"appName":"Hello App","version":"1.0.0","timestamp":"2024-01-15T10:30:00"}
     */
    @GetMapping("/info")
    public Map<String, String> getInfo() {
        return Map.of(
            "appName", appName,
            "version", appVersion,
            "timestamp", LocalDateTime.now().toString(),
            "status", "healthy"
        );
    }
    
    /**
     * Personalized greeting with parameter
     * GET /api/greet?name=John → "Hello, John!"
     */
    @GetMapping("/greet")
    public String greet(@org.springframework.web.bind.annotation.RequestParam(defaultValue = "World") String name) {
        return String.format("Hello, %s!", name);
    }
}
```

### Step 3: Add Configuration

**Update `src/main/resources/application.yml`:**
```yaml
# Application configuration
# WHAT: Central configuration file for Spring Boot
# WHY: Externalize configuration for different environments
# HOW: YAML format with hierarchical structure

spring:
  application:
    name: hello-app
  
server:
  port: 8080
  # Graceful shutdown for Kubernetes
  shutdown: graceful

# Application custom properties
app:
  name: Hello Kubernetes App
  version: 1.0.0

# Actuator configuration (for health checks)
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
      base-path: /actuator
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true  # Enable liveness/readiness endpoints
  info:
    env:
      enabled: true

# Logging configuration
logging:
  level:
    com.example.helloapp: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

### Step 4: Test Locally

```bash
# Run the application
./mvnw spring-boot:run

# In another terminal, test the endpoints
curl http://localhost:8080/api/hello
# Output: Hello, World from Spring Boot!

curl http://localhost:8080/api/info
# Output: {"appName":"Hello Kubernetes App","version":"1.0.0","timestamp":"2024-01-15T10:30:00","status":"healthy"}

curl http://localhost:8080/api/greet?name=Developer
# Output: Hello, Developer!

# Health check endpoint (used by Kubernetes)
curl http://localhost:8080/actuator/health
# Output: {"status":"UP"}

# Stop the app (Ctrl+C)
```

---

## Chapter 3: Docker - Containerization

### Understanding Docker

**Why Docker?** 
Without Docker: "It works on my machine!" → Deployment nightmare
With Docker: Same environment everywhere (dev, test, prod)

**Analogy:**
- **Docker image** = Frozen pizza (recipe + ingredients)
- **Docker container** = Baked pizza (running instance)
- **Docker registry** = Pizza delivery service (stores images)

### Step 1: Create Dockerfile

**Create `Dockerfile` in project root:**

```dockerfile
# Multi-stage Dockerfile for optimal production builds
# WHY: Separate build environment from runtime (smaller image, more secure)

# Stage 1: Build (uses Maven to compile code)
# WHAT: Temporary container with JDK and Maven to build the JAR
# WHY: Build tools not needed in final image
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app

# Copy pom.xml first (leverage Docker cache)
COPY pom.xml .
RUN mvn dependency:go-offline

# Copy source code and build
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime (only JRE, no build tools)
# WHAT: Final lightweight image with only the JAR
# WHY: Smaller image = faster pulls, less vulnerability surface
FROM eclipse-temurin:17-jre-alpine

# Create non-root user (security best practice)
# WHY: Running as root is dangerous - container escape risk
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Create app directory
WORKDIR /app

# Copy JAR from build stage
COPY --from=build /app/target/*.jar app.jar

# Copy startup script
COPY docker-entrypoint.sh /docker-entrypoint.sh
RUN chmod +x /docker-entrypoint.sh

# Change ownership to non-root user
RUN chown -R appuser:appgroup /app
USER appuser

# Expose port (documentation, doesn't actually publish)
EXPOSE 8080

# Health check (Kubernetes will use separate probes)
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# Entry point with graceful shutdown support
ENTRYPOINT ["/docker-entrypoint.sh"]
```

### Step 2: Create Docker Entrypoint Script

**Create `docker-entrypoint.sh`:**
```bash
#!/bin/sh
# WHAT: Startup script for the container
# WHY: Allows graceful shutdown and JVM tuning

set -e

# Default JVM options for containers
# WHY: Container-aware JVM respects cgroup limits
JAVA_OPTS="${JAVA_OPTS:--XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0}"

# Enable graceful shutdown
JAVA_OPTS="$JAVA_OPTS -Dspring.lifecycle.timeout-per-shutdown-phase=30s"

echo "Starting Hello App with JVM options: $JAVA_OPTS"

# Execute the JAR
exec java $JAVA_OPTS -jar /app/app.jar
```

### Step 3: Create .dockerignore

**Create `.dockerignore`:**
```
# WHAT: Files to exclude from Docker build context
# WHY: Smaller build context, faster builds, no secrets in image

.git/
.gitignore
.mvn/
mvnw
mvnw.cmd
*.md
*.log
target/
.dockerignore
Dockerfile
docker-compose.yml
.idea/
*.iml
.env
```

### Step 4: Build and Test Docker Image

```bash
# Build the JAR first
./mvnw clean package

# Build Docker image
docker build -t hello-app:1.0.0 .

# List images to verify
docker images | grep hello-app
# Output: hello-app   1.0.0    abc123def456   2 minutes ago   250MB

# Test running locally
docker run -d --name hello-app-test -p 8080:8080 hello-app:1.0.0

# Test endpoints
curl http://localhost:8080/api/hello
# Output: Hello, World from Spring Boot!

# Check container logs
docker logs hello-app-test

# Check container health
docker ps
# Output: CONTAINER ID   STATUS                    PORTS
# abc123def456         Up 2 minutes (healthy)     0.0.0.0:8080->8080/tcp

# Stop and remove test container
docker stop hello-app-test
docker rm hello-app-test

# Check image size (important for production!)
docker images hello-app:1.0.0
# Size: ~250MB (much smaller than 1GB+ typical images)
```

---

## Chapter 4: Kubernetes Basics - Where Apps Live

### Core Kubernetes Concepts

**Analogy - Apartment Building:**
- **Pod** = An apartment (one or more rooms/containers)
- **Deployment** = Building manager (ensures correct number of apartments)
- **Service** = Building address (stable way to find apartments)
- **Namespace** = Floor number (organization)
- **Node** = The building itself (physical/virtual machine)
- **Cluster** = Apartment complex (multiple buildings)

### Step 1: Start Kubernetes Locally

```bash
# Start Minikube (local Kubernetes)
minikube start --cpus=4 --memory=8192 --driver=docker

# Verify cluster is running
kubectl cluster-info
# Output: Kubernetes control plane is running at https://127.0.0.1:8443

# Check nodes
kubectl get nodes
# Output: NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   1m    v1.28.0

# Enable addons
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Get Minikube IP (for accessing services)
minikube ip
# Output: 192.168.49.2
```

### Step 2: Manual Kubernetes Manifests (Before Helm)

**Create `k8s/deployment.yaml`:**
```yaml
# WHAT: Deployment defines desired state for our pods
# WHY: Kubernetes maintains this state automatically
# HOW: Declarative YAML - say WHAT you want, not HOW

apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app-deployment
  labels:
    app: hello-app
    version: v1
spec:
  # HOW MANY: Number of pod replicas to run
  replicas: 3
  
  # SELECTOR: How to find pods managed by this deployment
  selector:
    matchLabels:
      app: hello-app
  
  # TEMPLATE: Definition of each pod
  template:
    metadata:
      labels:
        app: hello-app
        version: v1
    spec:
      # CONTAINERS: What runs in the pod
      containers:
      - name: hello-app
        image: hello-app:1.0.0  # Must be in registry for cloud
        imagePullPolicy: IfNotPresent
        
        # PORTS: Container listens on port 8080
        ports:
        - containerPort: 8080
          name: http
        
        # ENVIRONMENT: Configuration variables
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: APP_NAME
          value: "Hello Kubernetes"
        
        # RESOURCES: CPU/Memory limits (critical!)
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # READINESS PROBE: When can pod receive traffic?
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        
        # LIVENESS PROBE: Is pod still healthy?
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
```

**Create `k8s/service.yaml`:**
```yaml
# WHAT: Service provides stable network endpoint
# WHY: Pods come and go (IPs change), Service gives fixed IP/DNS

apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
  labels:
    app: hello-app
spec:
  # SERVICE TYPE: How to expose
  # ClusterIP: Internal only (default)
  # NodePort: Access via node IP:port
  # LoadBalancer: Cloud load balancer (production)
  type: ClusterIP
  
  # SELECTOR: Which pods to route traffic to
  selector:
    app: hello-app
  
  # PORTS: Map service port to container port
  ports:
  - port: 8080
    targetPort: 8080
    protocol: TCP
    name: http
```

**Create `k8s/ingress.yaml` (optional, for production):**
```yaml
# WHAT: Ingress routes external traffic to services
# WHY: Single entry point for multiple services

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - host: hello-app.local  # For local testing
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-app-service
            port:
              number: 8080
```

### Step 3: Deploy to Kubernetes

```bash
# 1. Create namespace (organization)
kubectl create namespace hello-app

# 2. Deploy the application
kubectl apply -f k8s/deployment.yaml -n hello-app
kubectl apply -f k8s/service.yaml -n hello-app
kubectl apply -f k8s/ingress.yaml -n hello-app

# 3. Watch pods come up
kubectl get pods -n hello-app -w
# Output:
# NAME                                    READY   STATUS    RESTARTS   AGE
# hello-app-deployment-7d8f9c5b6-abc12   1/1     Running   0          10s
# hello-app-deployment-7d8f9c5b6-def34   1/1     Running   0          10s
# hello-app-deployment-7d8f9c5b6-ghi56   1/1     Running   0          10s

# 4. Check deployment status
kubectl get deployments -n hello-app
# NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
# hello-app-deployment    3/3     3            3           30s

# 5. Check service
kubectl get services -n hello-app
# NAME                 TYPE        CLUSTER-IP      PORT(S)    AGE
# hello-app-service    ClusterIP   10.96.123.45    8080/TCP   30s

# 6. Test internal access (from within cluster)
kubectl run test-pod --image=curlimages/curl -it --rm --restart=Never -n hello-app -- \
  curl http://hello-app-service:8080/api/hello
# Output: Hello, World from Spring Boot!

# 7. Port forward for local testing
kubectl port-forward service/hello-app-service 8080:8080 -n hello-app

# In another terminal:
curl http://localhost:8080/api/hello
# Output: Hello, World from Spring Boot!

# 8. Check pod logs
kubectl logs -f deployment/hello-app-deployment -n hello-app

# 9. See what's running
kubectl get all -n hello-app
```

---

## Chapter 5: Helm - The Package Manager

### Why Helm?

**Problem with raw Kubernetes YAML:**
```bash
# For 3 environments, you need 3 sets of files:
k8s/dev/deployment.yaml  (replicas: 1, image: dev)
k8s/staging/deployment.yaml (replicas: 2, image: staging)
k8s/prod/deployment.yaml (replicas: 5, image: prod)

# Lots of duplication! 
```

**Helm Solution:** Templates + values = One chart, many environments

### Step 1: Create Helm Chart

```bash
# Create chart structure
helm create hello-app-chart

# Explore the structure
tree hello-app-chart/
# hello-app-chart/
# ├── Chart.yaml          # Chart metadata
# ├── values.yaml         # Default configuration values
# ├── templates/          # Kubernetes YAML templates
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   ├── _helpers.tpl    # Helper functions
# │   └── tests/
# │       └── test-connection.yaml
# └── charts/             # Subchart dependencies
```

### Step 2: Customize Chart for Our App

**Update `hello-app-chart/Chart.yaml`:**
```yaml
apiVersion: v2
name: hello-app
description: A production-ready Hello World Spring Boot application
type: application

# Chart version (changes when packaging changes)
version: 1.0.0

# App version (your application version)
appVersion: "1.0.0"

# Metadata
home: https://github.com/yourusername/hello-app
sources:
  - https://github.com/yourusername/hello-app

maintainers:
  - name: Platform Team
    email: platform@example.com

# Keywords for Helm Hub
keywords:
  - hello-world
  - spring-boot
  - web
  - rest-api
```

**Update `hello-app-chart/values.yaml`:**
```yaml
# Default values for hello-app
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Global configuration (applies to all subcharts)
global:
  environment: development
  imageRegistry: docker.io

# Application configuration
replicaCount: 2

# Image configuration
image:
  repository: hello-app
  tag: latest
  pullPolicy: IfNotPresent
  # Overrides the image tag whose default is the chart appVersion.
  digest: ""

# Image pull secrets for private registries
imagePullSecrets: []

# Name override
nameOverride: ""
fullnameOverride: ""

# Service account
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod security context
podSecurityContext:
  fsGroup: 2000

# Container security context
securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

# Service configuration
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
  annotations: {}

# Ingress configuration
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Resource limits
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Autoscaling (Part 9)
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

# Node selector
nodeSelector: {}

# Tolerations
tolerations: []

# Affinity
affinity: {}

# Probes
probes:
  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 10
    periodSeconds: 5
  startup:
    enabled: true
    failureThreshold: 30
    periodSeconds: 10

# Environment variables
env:
  SPRING_PROFILES_ACTIVE: kubernetes
  APP_NAME: Hello Helm App

# Environment from config map
envFrom: []

# Extra volumes
volumes: []
volumeMounts: []

# Lifecycle hooks
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]
```

**Update `hello-app-chart/templates/deployment.yaml`:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "hello-app.fullname" . }}
  labels:
    {{- include "hello-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "hello-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "hello-app.selectorLabels" . | nindent 8 }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "hello-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if .Values.probes.liveness.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          {{- end }}
          {{- if .Values.probes.readiness.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          {{- end }}
          {{- if .Values.probes.startup.enabled }}
          startupProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- with .Values.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.envFrom }}
          envFrom:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.lifecycle }}
          lifecycle:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with .Values.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

**Update `hello-app-chart/templates/_helpers.tpl`:**
```go
{{/*
Expand the name of the chart.
*/}}
{{- define "hello-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
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
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "hello-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "hello-app.labels" -}}
helm.sh/chart: {{ include "hello-app.chart" . }}
{{ include "hello-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "hello-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "hello-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "hello-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "hello-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Step 3: Create Environment-Specific Values

**Create `values-dev.yaml`:**
```yaml
# Development environment values
replicaCount: 1

image:
  tag: dev-latest

ingress:
  enabled: false

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 256Mi

env:
  SPRING_PROFILES_ACTIVE: dev
  APP_NAME: Hello App - Development
```

**Create `values-staging.yaml`:**
```yaml
# Staging environment values
replicaCount: 2

image:
  tag: staging-latest

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: staging.hello-app.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  SPRING_PROFILES_ACTIVE: staging
  APP_NAME: Hello App - Staging
```

**Create `values-prod.yaml`:**
```yaml
# Production environment values
replicaCount: 3

image:
  tag: latest

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100r/s"
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
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 70

podDisruptionBudget:
  enabled: true
  minAvailable: 2

env:
  SPRING_PROFILES_ACTIVE: prod
  APP_NAME: Hello App - Production
  LOG_LEVEL: WARN

nodeSelector:
  node-type: production

tolerations:
  - key: "production"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

### Step 4: Test and Deploy with Helm

```bash
# 1. Lint the chart (syntax checking)
helm lint hello-app-chart/

# 2. Template rendering test (see what will be deployed)
helm template test-release ./hello-app-chart --debug

# 3. Dry run (simulate installation)
helm install test-release ./hello-app-chart --dry-run --debug

# 4. Install to development
helm install hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace dev \
  --create-namespace

# 5. Check release status
helm list -n dev
# NAME            NAMESPACE       REVISION        STATUS          CHART
# hello-app-dev   dev             1               deployed        hello-app-1.0.0

# 6. Get deployed values
helm get values hello-app-dev -n dev

# 7. Get all manifests
helm get manifest hello-app-dev -n dev

# 8. Upgrade to staging
helm upgrade hello-app-dev ./hello-app-chart \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace

# 9. Check rollout status
kubectl rollout status deployment/hello-app-dev -n staging

# 10. Test the deployment
kubectl port-forward service/hello-app-dev 8080:8080 -n staging
curl http://localhost:8080/api/hello

# 11. Uninstall
helm uninstall hello-app-dev -n dev

# 12. Package chart for distribution
helm package hello-app-chart/
# Creates: hello-app-1.0.0.tgz
```

---

## Chapter 6: Container Registry Integration

### Why Registry is Critical

**Problem:** Your Kubernetes cluster can't see images on your laptop!

**Solution:** Push to container registry (Docker Hub, AWS ECR, GCR, ACR)

### Step 1: Push to Docker Hub (Free)

```bash
# 1. Create account at hub.docker.com

# 2. Login
docker login
# Enter username and password

# 3. Tag your image with your Docker Hub username
docker tag hello-app:1.0.0 yourusername/hello-app:1.0.0
docker tag hello-app:1.0.0 yourusername/hello-app:latest

# 4. Push to Docker Hub
docker push yourusername/hello-app:1.0.0
docker push yourusername/hello-app:latest

# 5. Update Helm values to use your registry
cat > values-registry.yaml << EOF
image:
  repository: yourusername/hello-app
  tag: 1.0.0
  pullPolicy: Always
EOF

# 6. Deploy using registry image
helm upgrade --install hello-app ./hello-app-chart \
  -f values-registry.yaml \
  --namespace production

# Now your cluster can pull the image!
```

### Step 2: Create Image Pull Secret (For Private Registry)

```yaml
# image-pull-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: regcred
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-docker-config>
---
# Or create via command line
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=yourpassword \
  --docker-email=your@email.com \
  -n production
```

---

## Chapter 7: Complete Working Example

### Putting It All Together

```bash
#!/bin/bash
# complete-deployment.sh
# One script to build, package, and deploy everything

set -e

echo "🚀 Starting complete deployment pipeline"

# 1. Build Spring Boot app
echo "📦 Building Spring Boot application..."
./mvnw clean package

# 2. Build Docker image
echo "🐳 Building Docker image..."
docker build -t hello-app:1.0.0 .

# 3. Tag for registry
echo "🏷️  Tagging image for registry..."
docker tag hello-app:1.0.0 yourusername/hello-app:1.0.0

# 4. Push to registry
echo "📤 Pushing to container registry..."
docker push yourusername/hello-app:1.0.0

# 5. Create namespace
echo "🏗️  Creating namespace..."
kubectl create namespace hello-app-prod --dry-run=client -o yaml | kubectl apply -f -

# 6. Deploy with Helm
echo "🎯 Deploying to Kubernetes with Helm..."
helm upgrade --install hello-app ./hello-app-chart \
  -f values-prod.yaml \
  --set image.repository=yourusername/hello-app \
  --set image.tag=1.0.0 \
  --namespace hello-app-prod \
  --wait \
  --timeout 5m

# 7. Get deployment status
echo "📊 Deployment status:"
kubectl get pods,svc,ingress -n hello-app-prod

# 8. Get service URL
if command -v minikube &> /dev/null; then
  echo "🌐 Access the app at:"
  minikube service list | grep hello-app
else
  echo "🌐 Get external IP with: kubectl get svc -n hello-app-prod"
fi

echo "✅ Deployment complete!"
```

### Verify Everything Works

```bash
# Check all resources
kubectl get all -n hello-app-prod

# Check Helm releases
helm list -n hello-app-prod

# Test the API
kubectl run test --image=curlimages/curl -it --rm --restart=Never -n hello-app-prod -- \
  curl http://hello-app:8080/api/hello

# Check logs
kubectl logs -f deployment/hello-app -n hello-app-prod

# Scale manually (if HPA not enabled)
kubectl scale deployment hello-app --replicas=5 -n hello-app-prod

# Check resource usage
kubectl top pods -n hello-app-prod
```

---

## Summary: Part 1 Checklist

| Component | Status | Verification |
|-----------|--------|--------------|
| Spring Boot App | ✅ | `curl localhost:8080/api/hello` works |
| Docker Image | ✅ | `docker run hello-app:1.0.0` works |
| Container Registry | ✅ | Image pushed to Docker Hub/ECR |
| Kubernetes Manifests | ✅ | `kubectl apply -f k8s/` works |
| Helm Chart | ✅ | `helm install` works |
| Multi-environment | ✅ | Different values for dev/staging/prod |
| Health Checks | ✅ | `/actuator/health` returns UP |
| Resource Limits | ✅ | Pods have CPU/memory limits |
| Graceful Shutdown | ✅ | Pods wait 15s before termination |

## Common Issues and Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| **ImagePullBackOff** | Pod stuck in ImagePullBackOff | Push image to registry, check image name |
| **CrashLoopBackOff** | Pod keeps restarting | Check logs: `kubectl logs pod-name` |
| **Pending** | Pod not scheduling | Check resources: `kubectl describe pod` |
| **Connection refused** | Can't reach service | Check service selector, port numbers |
| **Helm template errors** | Failed to render | Run `helm template --debug` |
| **Registry auth failed** | Can't pull image | Create image pull secret |

## Next Steps

After mastering Part 1, you're ready for:
- **Part 2: Kubernetes Core Concepts** - Namespaces, ConfigMaps, Secrets
- **Part 3: Helm Deep Dive** - Custom templates, hooks, dependencies
- **Part 4: CI/CD Pipeline** - Automate everything

---

## Practice Exercises

### Exercise 1: Modify the App
Add a new endpoint `/api/version` that returns the app version. Build and redeploy.

### Exercise 2: Scale Testing
Deploy with 5 replicas and watch how Kubernetes distributes them.

### Exercise 3: Failure Testing
Delete a pod manually and watch Kubernetes recreate it automatically.

### Exercise 4: Configuration
Use Helm to deploy the same chart to dev and prod with different replica counts.

### Exercise 5: Registry
Push your image to Docker Hub and deploy to Minikube pulling from registry.

---
