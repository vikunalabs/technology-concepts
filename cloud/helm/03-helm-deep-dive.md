## Part 3: Helm Deep Dive - Beyond Basic Templating

### Prerequisites
- Completed Parts 1 & 2 (or equivalent experience)
- Helm 3.x installed (`helm version`)
- Kubernetes cluster (Minikube works fine)
- Basic understanding of Go templates

### What You'll Learn
- ✅ Advanced template functions and pipelines
- ✅ Custom template definitions (Named templates)
- ✅ Managing dependencies with subcharts
- ✅ Hooks for database migrations and initialization
- ✅ Testing Helm charts
- ✅ Chart validation and best practices

---

## Chapter 1: Understanding Helm's Templating Engine

### Why Go Templates?

**The Problem Helm Solves:** Kubernetes YAML has no variables, loops, or conditionals. You'd repeat yourself constantly:

```yaml
# Without Helm - 3 environments = 3 almost-identical files
# dev-deployment.yaml
replicas: 1
image: myapp:dev
memory: 256Mi

# staging-deployment.yaml (95% same!)
replicas: 2
image: myapp:staging
memory: 512Mi

# prod-deployment.yaml (95% same!)
replicas: 5
image: myapp:prod
memory: 1Gi
```

**With Helm templates:** One file, variables for what changes.

### Helm Template Syntax Basics

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.appName }}-deployment
  # {{ ... }} - Action (evaluate expression)
  # .Values - Root object containing values.yaml
  # .Release - Metadata about this release
spec:
  replicas: {{ .Values.replicaCount }}
  
  # Pipeline (pass output to next function)
  image: {{ .Values.image.repository | quote }}
  
  # Conditionals
  {{- if .Values.ingress.enabled }}
  # This block only included if enabled
  {{- end }}
  
  # Loops
  env:
  {{- range .Values.envVars }}
  - name: {{ .name }}
    value: {{ .value }}
  {{- end }}
```

### The Four Most Important Objects

| Object | What It Contains | Example |
|--------|-----------------|---------|
| `.Values` | Your `values.yaml` content | `{{ .Values.replicaCount }}` |
| `.Release` | Helm release metadata | `{{ .Release.Name }}`, `{{ .Release.Namespace }}` |
| `.Chart` | `Chart.yaml` metadata | `{{ .Chart.Name }}`, `{{ .Chart.Version }}` |
| `.Files` | Access chart files | `{{ .Files.Get "config/app.conf" }}` |

### Template Functions and Pipelines

```yaml
# Basic function usage
image: {{ .Values.image.repository | default "nginx" | quote }}
# Pipeline: repository → default to "nginx" → add quotes

# Common functions
name: {{ .Values.appName | upper }}           # UPPERCASE
name: {{ .Values.appName | lower }}           # lowercase
name: {{ .Values.appName | trim }}            # remove whitespace
name: {{ .Values.appName | replace " " "-" }} # spaces to dashes
name: {{ .Values.appName | trunc 63 }}        # limit length

# Indentation (critical for YAML)
config: |
  {{- .Values.config | nindent 4 }}
  # Adds 4 spaces to every line

# Required values (fail if missing)
database: {{ required "A valid database host is required!" .Values.dbHost }}
```

### Real Example: Conditional Logic

```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  
  {{- if eq .Values.service.type "LoadBalancer" }}
  # Only for LoadBalancer services
  loadBalancerIP: {{ .Values.service.loadBalancerIP | default "" }}
  loadBalancerSourceRanges:
    {{- toYaml .Values.service.sourceRanges | nindent 4 }}
  {{- end }}
  
  ports:
  - port: {{ .Values.service.port }}
    targetPort: {{ .Values.service.targetPort }}
    {{- if and (eq .Values.service.type "NodePort") (.Values.service.nodePort) }}
    nodePort: {{ .Values.service.nodePort }}
    {{- end }}
```

---

## Chapter 2: Named Templates (Partial/Include)

### The DRY Problem

Without named templates, you repeat common patterns:

```yaml
# deployment.yaml - repeated labels everywhere!
metadata:
  labels:
    app: myapp
    version: v1
    managed-by: helm
spec:
  template:
    metadata:
      labels:  # Same labels again!
        app: myapp
        version: v1
        managed-by: helm
```

### Creating Named Templates

**Define in `templates/_helpers.tpl`** (underscore = won't render directly):

```go
{{/*
Define common labels
Usage: {{ include "app.labels" . | indent 4 }}
*/}}
{{- define "app.labels" -}}
app.kubernetes.io/name: {{ include "app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
helm.sh/chart: {{ include "app.chart" . }}
{{- end -}}

{{/*
Generate full name
*/}}
{{- define "app.fullname" -}}
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
Generate selector labels (for service to find pods)
*/}}
{{- define "app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

### Using Named Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    {{- include "app.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### Advanced Named Template Patterns

```go
{{/*
Generate image pull secret names (handle list or single)
*/}}
{{- define "app.imagePullSecrets" -}}
{{- if .Values.imagePullSecrets }}
imagePullSecrets:
{{- range .Values.imagePullSecrets }}
- name: {{ . }}
{{- end }}
{{- else if .Values.global.imagePullSecrets }}
imagePullSecrets:
{{- range .Values.global.imagePullSecrets }}
- name: {{ . }}
{{- end }}
{{- end }}
{{- end -}}

{{/*
Merge environment variables from multiple sources
*/}}
{{- define "app.envVars" -}}
{{- $envVars := list }}
{{- $envVars = append $envVars .Values.baseEnv }}
{{- if .Values.extraEnv }}
{{- $envVars = append $envVars .Values.extraEnv }}
{{- end }}
{{- toYaml $envVars }}
{{- end -}}
```

---

## Chapter 3: Subcharts - Managing Dependencies

### The Problem

Your app needs:
- PostgreSQL database
- Redis cache
- Prometheus monitoring
- Grafana dashboards

**Without subcharts:** You'd manually deploy each component separately.

**With subcharts:** Define dependencies, Helm installs everything together.

### Chart Dependency Structure

```
myapp-chart/              # Parent chart
├── Chart.yaml            # Lists dependencies
├── values.yaml           # Override child values
├── charts/               # Subcharts directory
│   ├── postgresql/       # PostgreSQL subchart
│   ├── redis/            # Redis subchart
│   └── prometheus/       # Prometheus subchart
└── templates/            # Parent templates
```

### Defining Dependencies in Chart.yaml

**Method 1: Manual (for local charts)**
```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0

dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled  # Can disable if false
    tags:
      - database
    
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
    
  - name: prometheus
    version: "19.x.x"
    repository: "https://prometheus-community.github.io/helm-charts"
    condition: monitoring.enabled
    import-values:  # Import values from subchart
      - child: prometheus
        parent: monitoring
    
  - name: local-subchart
    version: "0.1.0"
    repository: "file://./charts/local-subchart"  # Local file
```

**Method 2: Using helm dependency (recommended)**
```bash
# Create Chart.yaml with dependencies
cat > Chart.yaml << 'EOF'
apiVersion: v2
name: myapp
version: 1.0.0
dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
EOF

# Download dependencies to charts/ directory
helm dependency update

# List dependencies
helm dependency list
```

### Overriding Subchart Values

**`values.yaml` for parent chart:**
```yaml
# Parent app configuration
replicaCount: 3
image:
  repository: myapp
  tag: latest

# PostgreSQL subchart configuration
postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        username: myappuser
        password: myapppass  # In real life, use secrets!
        database: myappdb
  primary:
    persistence:
      size: 10Gi
    resources:
      requests:
        memory: 256Mi

# Redis subchart configuration
redis:
  enabled: true
  architecture: standalone
  auth:
    password: redispass
  master:
    persistence:
      size: 5Gi

# Monitoring stack
monitoring:
  enabled: false  # Disable for dev, enable for prod
```

### Accessing Subchart Services

```yaml
# templates/configmap.yaml - Connect to subchart services
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  DATABASE_URL: |-
    {{- if .Values.postgresql.enabled }}
    postgresql://{{ .Values.postgresql.global.postgresql.auth.username }}:{{ .Values.postgresql.global.postgresql.auth.password }}@{{ .Release.Name }}-postgresql:5432/{{ .Values.postgresql.global.postgresql.auth.database }}
    {{- else }}
    {{ .Values.externalDatabase.url }}
    {{- end }}
  
  REDIS_URL: |-
    {{- if .Values.redis.enabled }}
    redis://default:{{ .Values.redis.auth.password }}@{{ .Release.Name }}-redis-master:6379
    {{- else }}
    {{ .Values.externalRedis.url }}
    {{- end }}
```

### Conditionally Including Subcharts

```yaml
# Install with all dependencies
helm install myapp ./myapp-chart

# Install without database (use external one)
helm install myapp ./myapp-chart --set postgresql.enabled=false

# Install only monitoring for existing app
helm install monitoring ./myapp-chart --set postgresql.enabled=false --set redis.enabled=false --set myapp.enabled=false
```

### Global Values - Sharing Across Subcharts

```yaml
# Parent values.yaml with globals
global:
  environment: production
  imageRegistry: myregistry.com
  imagePullSecrets:
    - regcred
  storageClass: fast-ssd

# Any subchart can access these
postgresql:
  global:
    storageClass: {{ .Values.global.storageClass }}
  image:
    registry: {{ .Values.global.imageRegistry }}
```

---

## Chapter 4: Hooks - Lifecycle Management

### The Problem

Before your app starts, you need to:
1. Run database migrations
2. Initialize schemas
3. Create admin users
4. Warm up caches

After installation, you need to:
1. Send Slack notification
2. Run smoke tests
3. Backup configuration

### What Are Hooks?

**Analogy:** 
- **Helm** = Event planner
- **Hooks** = Tasks to run at specific times (before dinner, after ceremony, etc.)

### Hook Types and Timing

| Hook | When It Runs | Typical Use |
|------|--------------|--------------|
| `pre-install` | Before installing resources | Validate environment |
| `post-install` | After all resources created | Database migrations, seeding |
| `pre-delete` | Before deleting release | Backup data |
| `post-delete` | After deletion | Clean up external resources |
| `pre-upgrade` | Before upgrade | Backup, drain queues |
| `post-upgrade` | After upgrade | Smoke tests, notifications |
| `pre-rollback` | Before rollback | Create restore point |
| `post-rollback` | After rollback | Verify rollback |
| `test` | When `helm test` runs | Integration tests |

### Creating a Database Migration Job

```yaml
# templates/migrations-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-migrations
  annotations:
    # This is a hook - runs after installation
    "helm.sh/hook": post-install,post-upgrade
    # Weight controls order (lower runs first)
    "helm.sh/hook-weight": "5"
    # Delete after success (clean up)
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["node", "run-migrations.js"]
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### Complex Hook Example: Pre-Install Validation

```yaml
# templates/pre-install-validator.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "app.fullname" . }}-validator
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"  # Run before other hooks
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
  - name: validator
    image: alpine:latest
    command:
      - sh
      - -c
      - |
        echo "Validating environment..."
        
        # Check required values
        if [ -z "$DB_PASSWORD" ]; then
          echo "ERROR: DB_PASSWORD is required"
          exit 1
        fi
        
        # Check network connectivity
        if ! nc -z $DB_HOST $DB_PORT; then
          echo "ERROR: Cannot reach database at $DB_HOST:$DB_PORT"
          exit 1
        fi
        
        echo "Validation passed!"
    env:
    - name: DB_HOST
      value: {{ .Values.database.host }}
    - name: DB_PORT
      value: {{ .Values.database.port | quote }}
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

### Hook Deletion Policies

```yaml
# Different deletion strategies
annotations:
  # Option 1: Delete after success
  "helm.sh/hook-delete-policy": hook-succeeded
  
  # Option 2: Delete before new hook runs (for upgrades)
  "helm.sh/hook-delete-policy": before-hook-creation
  
  # Option 3: Keep on failure (debugging)
  "helm.sh/hook-delete-policy": hook-failed
  
  # Option 4: Multiple policies (OR)
  "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

### Post-Install Notification Hook

```yaml
# templates/post-install-notify.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "app.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "100"  # Run last
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      containers:
      - name: notifier
        image: curlimages/curl:latest
        command:
          - sh
          - -c
          - |
            MESSAGE="Deployment of {{ .Release.Name }} v{{ .Chart.Version }} completed!"
            
            # Send to Slack
            curl -X POST -H 'Content-type: application/json' \
              --data '{"text":"'"$MESSAGE"'"}' \
              {{ .Values.slackWebhook }}
            
            echo "Notification sent!"
      restartPolicy: Never
```

### Helm Test Hooks

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "app.fullname" . }}-test-connection"
  annotations:
    "helm.sh/hook": test
spec:
  containers:
  - name: wget
    image: busybox
    command: ['wget']
    args: ['{{ include "app.fullname" . }}:{{ .Values.service.port }}/health']
  restartPolicy: Never
```

```bash
# Run tests after installation
helm test myapp

# Output:
# POD: myapp-test-connection   STATUS: PASSED
```

---

## Chapter 5: Chart Development Best Practices

### Chart Structure Checklist

```
my-chart/
├── Chart.yaml          # REQUIRED: Name, version, dependencies
├── values.yaml         # REQUIRED: Default values
├── templates/          # REQUIRED: Kubernetes manifests
│   ├── _helpers.tpl    # Named templates
│   ├── NOTES.txt       # Post-install instructions
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secrets.yaml
│   ├── hpa.yaml        # Horizontal Pod Autoscaler
│   ├── ingress.yaml
│   ├── pvc.yaml        # Persistent Volume Claims
│   └── tests/          # Test pods
│       └── test-connection.yaml
├── charts/             # Dependencies (auto-downloaded)
├── crds/               # Custom Resource Definitions
├── templates/          # Core templates
└── .helmignore         # Files to exclude
```

### Versioning Strategy

```yaml
# Chart.yaml - Semantic versioning
apiVersion: v2
name: myapp
version: 1.2.3  # Chart version (changes when packaging changes)
appVersion: "2.0.0"  # App version (your application version)

# Update strategy:
# - Patch (1.2.3 → 1.2.4): Bug fixes, no feature changes
# - Minor (1.2.3 → 1.3.0): New features, backward compatible
# - Major (1.2.3 → 2.0.0): Breaking changes
```

### NOTES.txt - User Instructions

```text
# templates/NOTES.txt
Thank you for installing {{ .Chart.Name }} v{{ .Chart.Version }}!

Your application is now running.

Get the application URL:
{{- if .Values.ingress.enabled }}
  http{{ if .Values.ingress.tls }}s{{ end }}://{{ .Values.ingress.hostname }}
{{- else if contains "LoadBalancer" .Values.service.type }}
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "app.fullname" . }} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "app.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else }}
  kubectl port-forward --namespace {{ .Release.Namespace }} svc/{{ include "app.fullname" . }} {{ .Values.service.port }}:{{ .Values.service.port }}
  echo "Visit http://127.0.0.1:{{ .Values.service.port }} to use your application"
{{- end }}

Database credentials:
  Username: {{ .Values.postgresql.global.postgresql.auth.username }}
  Password: (stored in secret "{{ .Release.Name }}-postgresql")

Watch pods:
  kubectl get pods --namespace {{ .Release.Namespace }} -w

View logs:
  kubectl logs -f --namespace {{ .Release.Namesule }} -l app.kubernetes.io/instance={{ .Release.Name }}
```

### Testing Charts with `helm test` and `helm lint`

```bash
# Lint chart (syntax checking)
helm lint ./my-chart

# Template rendering test
helm template test ./my-chart --debug

# Install and test
helm install test-release ./my-chart --dry-run --debug
helm install test-release ./my-chart --wait --timeout 5m

# Run test pods
helm test test-release

# Uninstall
helm uninstall test-release
```

### Chart Testing with ct (Community Tool)

```yaml
# .github/workflows/chart-test.yaml
name: Test Helm Charts

on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Run chart-testing
        uses: helm/chart-testing-action@v2.1.0
        with:
          command: ct
          args: lint --all --validate-maintainers=false
      
      - name: Create kind cluster
        uses: helm/kind-action@v1.2.0
      
      - name: Run chart install tests
        run: ct install --all
```

---

## Chapter 6: Advanced Patterns

### Pattern 1: Feature Flag Pattern

```yaml
# values.yaml
features:
  caching:
    enabled: true
    ttl: 300
  monitoring:
    enabled: false
  newApi:
    enabled: false
    version: v2

# templates/deployment.yaml
env:
{{- if .Values.features.caching.enabled }}
- name: CACHE_ENABLED
  value: "true"
- name: CACHE_TTL
  value: {{ .Values.features.caching.ttl | quote }}
{{- end }}

{{- if .Values.features.newApi.enabled }}
- name: API_VERSION
  value: {{ .Values.features.newApi.version }}
{{- end }}
```

### Pattern 2: Multi-Environment Configuration

```yaml
# values/values.yaml (base)
replicaCount: 1
image:
  tag: latest
resources:
  requests:
    memory: 256Mi

# values/dev.yaml
environment: dev
replicaCount: 1
ingress:
  enabled: false
resources:
  requests:
    memory: 256Mi  # Keep small for dev

# values/staging.yaml
environment: staging
replicaCount: 2
ingress:
  enabled: true
  host: staging.myapp.com

# values/prod.yaml
environment: prod
replicaCount: 5
ingress:
  enabled: true
  host: myapp.com
  tls:
    - secretName: myapp-tls
resources:
  requests:
    memory: 512Mi
  limits:
    memory: 1Gi
```

```bash
# Deploy different environments
helm install myapp-dev ./chart -f values/values.yaml -f values/dev.yaml
helm install myapp-staging ./chart -f values/values.yaml -f values/staging.yaml
helm install myapp-prod ./chart -f values/values.yaml -f values/prod.yaml
```

### Pattern 3: Lookup Function (Check Existing Resources)

```yaml
# templates/service.yaml - Avoid creating if exists
{{- $existingService := lookup "v1" "Service" .Release.Namespace (include "app.fullname" .) }}
{{- if not $existingService }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "app.fullname" . }}
  # Service doesn't exist, create it
{{- else }}
# Service already exists, skip creation
{{- end }}
```

### Pattern 4: Generating Random Passwords (If Not Provided)

```yaml
# templates/secrets.yaml
{{- $dbPassword := .Values.dbPassword | default (randAlphaNum 32) }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "app.fullname" . }}-db
type: Opaque
stringData:
  password: {{ $dbPassword }}
---
# Store in a secret for persistence across upgrades
{{- if not .Values.dbPassword }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "app.fullname" . }}-generated
  annotations:
    "helm.sh/resource-policy": keep  # Don't delete on upgrade
stringData:
  db-password: {{ $dbPassword }}
{{- end }}
```

### Pattern 5: Conditional Resource Creation

```yaml
# templates/hpa.yaml - Only if metrics server exists
{{- if .Values.autoscaling.enabled }}
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
apiVersion: autoscaling/v2
{{- else if .Capabilities.APIVersions.Has "autoscaling/v1" }}
apiVersion: autoscaling/v1
{{- end }}
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "app.fullname" . }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
  targetCPUUtilizationPercentage: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
  {{- end }}
{{- end }}
```

---

## Chapter 7: Complete Real-World Example

### E-Commerce Application with All Features

**`Chart.yaml`:**
```yaml
apiVersion: v2
name: ecommerce-app
version: 1.0.0
appVersion: "2.1.0"
description: E-commerce microservices application
type: application

dependencies:
  - name: postgresql
    version: 11.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  
  - name: redis
    version: 17.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
  
  - name: rabbitmq
    version: 11.x.x
    repository: https://charts.bitnami.com/bitnami
    condition: rabbitmq.enabled
  
  - name: prometheus
    version: 19.x.x
    repository: https://prometheus-community.github.io/helm-charts
    condition: monitoring.enabled

maintainers:
  - name: Platform Team
    email: platform@example.com
```

**`values.yaml` (Complete):**
```yaml
# Global configuration
global:
  environment: production
  imageRegistry: docker.io
  storageClass: gp2

# Application configuration
replicaCount: 3

image:
  repository: mycompany/ecommerce-api
  tag: latest
  pullPolicy: IfNotPresent

# Service configuration
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

# Ingress configuration
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "10r/s"
  hosts:
    - host: api.ecommerce.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: ecommerce-tls
      hosts:
        - api.ecommerce.com

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# Resource limits
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 2Gi

# Probes
probes:
  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 60
    periodSeconds: 10
  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 30
    periodSeconds: 5

# Environment variables
env:
  - name: SPRING_PROFILES_ACTIVE
    value: production
  - name: LOG_LEVEL
    value: INFO

# Feature flags
features:
  checkout:
    enabled: true
  recommendations:
    enabled: true
  analytics:
    enabled: false

# Database configuration
postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        username: ecommerce
        database: ecommerce
  primary:
    persistence:
      size: 100Gi
    resources:
      requests:
        memory: 512Mi

# Redis configuration
redis:
  enabled: true
  architecture: replication
  auth:
    enabled: true
  replica:
    replicaCount: 2
  resources:
    requests:
      memory: 256Mi

# Message queue
rabbitmq:
  enabled: true
  replicaCount: 3
  persistence:
    enabled: true
    size: 50Gi

# Monitoring
monitoring:
  enabled: true
  prometheus:
    prometheusSpec:
      retention: 15d

# Migration job
migrations:
  enabled: true
  image:
    repository: mycompany/ecommerce-migrations
    tag: latest
```

**`templates/hooks/migrations-job.yaml`:**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "ecommerce.fullname" . }}-migrations
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "10"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migration
        image: "{{ .Values.migrations.image.repository }}:{{ .Values.migrations.image.tag }}"
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: {{ .Release.Name }}-postgresql
              key: postgresql-url
        - name: MIGRATION_DIR
          value: /migrations
        command:
          - /bin/sh
          - -c
          - |
            echo "Starting database migrations..."
            /app/migrate up
            echo "Migrations completed!"
```

**`templates/hooks/smoke-test.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "ecommerce.fullname" . }}-smoke-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  containers:
  - name: smoke-test
    image: curlimages/curl:latest
    command:
      - /bin/sh
      - -c
      - |
        echo "Testing API health..."
        curl -f http://{{ include "ecommerce.fullname" . }}:{{ .Values.service.port }}/actuator/health || exit 1
        
        echo "Testing checkout endpoint..."
        curl -f -X POST http://{{ include "ecommerce.fullname" . }}:{{ .Values.service.port }}/api/checkout/health || exit 1
        
        echo "All smoke tests passed!"
  restartPolicy: Never
```

### Deploying the Complete Example

```bash
# Development environment
helm install ecommerce-dev ./ecommerce-app \
  -f values.yaml \
  -f values-dev.yaml \
  --set global.environment=dev \
  --set monitoring.enabled=false \
  --set autoscaling.enabled=false \
  --namespace dev

# Staging (with database seeding)
helm install ecommerce-staging ./ecommerce-app \
  -f values.yaml \
  -f values-staging.yaml \
  --set postgresql.primary.persistence.size=20Gi \
  --namespace staging

# Production (full features)
helm install ecommerce-prod ./ecommerce-app \
  -f values.yaml \
  -f values-prod.yaml \
  --set replicaCount=5 \
  --set autoscaling.maxReplicas=20 \
  --set monitoring.enabled=true \
  --wait --timeout 10m \
  --namespace prod

# Run smoke tests
helm test ecommerce-prod -n prod

# Upgrade with new version
helm upgrade ecommerce-prod ./ecommerce-app \
  -f values-prod.yaml \
  --set image.tag=v2.1.0 \
  --atomic  # Rollback on failure

# View history
helm history ecommerce-prod -n prod

# Rollback if needed
helm rollback ecommerce-prod 2 -n prod
```

---

## Summary: Helm Deep Dive Checklist

| Concept | Mastery Level | Practice Exercise |
|---------|--------------|-------------------|
| Go Templates | ⬜ Basic | Create conditional deployment based on environment |
| Named Templates | ⬜ Intermediate | Create _helpers.tpl with 5+ reusable templates |
| Subcharts | ⬜ Intermediate | Add PostgreSQL and Redis as dependencies |
| Hooks | ⬜ Advanced | Create pre-upgrade backup job |
| Testing | ⬜ Intermediate | Write helm test for health endpoint |
| Production Patterns | ⬜ Advanced | Implement feature flags and multi-env config |

## Common Pitfalls and Solutions

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| **Template indentation** | YAML parse errors | Always use `nindent` or `indent` functions |
| **Hook resource leak** | Old jobs piling up | Set `hook-delete-policy` |
| **Subchart value confusion** | Values not overriding | Use `helm get values` to debug |
| **Missing required values** | Rendering errors | Use `required` function |
| **Secret regeneration** | Passwords change on upgrade | Use `lookup` or store generated secrets |

## Next Steps

After mastering Helm, you're ready for:
- **Part 4: Container Registry & CI/CD** - Automate the build → push → deploy pipeline
- **Part 5: Networking & Ingress** - TLS, load balancing, service mesh
- **Part 6: Storage & Stateful Apps** - Databases, StatefulSets, operators

## Practice Exercises

### Exercise 1: Create Multi-Tier Chart
Build a chart that deploys a web app + Redis cache + PostgreSQL database with proper dependencies.

### Exercise 2: Implement Database Migration Hook
Create a pre-upgrade hook that runs schema migrations before the new version starts.

### Exercise 3: Feature Flag System
Build a chart that conditionally includes monitoring, tracing, and analytics based on feature flags.

### Exercise 4: Chart Testing Pipeline
Create GitHub Actions workflow that lints, templates, and installs your chart on PR.

### Exercise 5: Production-Ready Chart
Take your Part 2 app and create production-ready chart with all probes, resources, autoscaling, and monitoring.

---

**Ready for Part 4?** Let me know and I'll create **Container Registry & CI/CD Pipeline** covering:
- Docker registry deep dive (ECR, GCR, ACR, Docker Hub)
- GitHub Actions workflows
- GitLab CI/CD
- Automated semantic versioning
- Security scanning
- Multi-environment promotion

Would you also like me to:
1. Create a template repository with all these patterns?
2. Add interactive examples using `helm install`?
3. Provide a cheat sheet PDF for Helm commands?