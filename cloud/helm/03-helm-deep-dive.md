# Part 3: Helm Deep Dive

## What This Covers
- Go template syntax — the engine behind Helm
- Named templates — reusable snippets
- Subcharts — managing dependencies
- Hooks — run jobs at specific lifecycle points
- Testing charts

---

## Chapter 1: Go Templates

### The Syntax

Helm templates are Kubernetes YAML files with `{{ }}` expressions embedded. Everything inside `{{ }}` is evaluated at deploy time.

```yaml
# The four objects you'll use constantly:
metadata:
  name: {{ .Release.Name }}           # Name given at helm install
  namespace: {{ .Release.Namespace }} # Namespace deployed into

spec:
  replicas: {{ .Values.replicaCount }} # From values.yaml

image: "{{ .Values.image.repository }}:{{ .Chart.AppVersion }}"
                                        # .Chart = Chart.yaml content
```

### Pipelines

The pipe `|` passes output left-to-right, like Unix pipes:

```yaml
# Read value → apply default if empty → add quotes
image: {{ .Values.image.repository | default "nginx" | quote }}

# Common functions
name: {{ .Values.appName | upper }}            # UPPERCASE
name: {{ .Values.appName | lower }}            # lowercase
name: {{ .Values.appName | trunc 63 }}         # Limit to 63 chars (K8s name limit)
name: {{ .Values.appName | replace " " "-" }} # spaces → dashes

# nindent: critical for YAML indentation
labels:
  {{- include "app.labels" . | nindent 2 }}
# The - before {{ trims the newline before it
# nindent 2 adds a newline + 2 spaces before each line of output
```

### Conditionals

```yaml
# Simple if
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}

# if/else
type: {{ if eq .Values.service.type "LoadBalancer" }}LoadBalancer{{ else }}ClusterIP{{ end }}

# Check if a value exists and is not empty
{{- if .Values.podAnnotations }}
annotations:
  {{- toYaml .Values.podAnnotations | nindent 8 }}
{{- end }}
```

### Loops

```yaml
# Loop over a list
env:
{{- range .Values.envVars }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}

# values.yaml
envVars:
  - name: SPRING_PROFILES_ACTIVE
    value: production
  - name: LOG_LEVEL
    value: INFO

# Loop over a map
labels:
{{- range $key, $value := .Values.labels }}
  {{ $key }}: {{ $value }}
{{- end }}
```

### The `required` Function

Fail loudly if a required value is missing:

```yaml
host: {{ required "database.host is required" .Values.database.host }}
```

This fails at `helm install` with a clear error rather than silently deploying a broken config.

### Whitespace Control

The `-` trims whitespace (including newlines):

```yaml
# Without -: blank lines appear in output
{{if .Values.enabled}}
value: true
{{end}}

# With -: no blank lines
{{- if .Values.enabled }}
value: true
{{- end }}
```

---

## Chapter 2: Named Templates (`_helpers.tpl`)

### What They Are

Named templates are reusable snippets defined in `_helpers.tpl`. Files starting with `_` are not rendered as Kubernetes manifests — they only define templates for use by other files.

```go
{{/*
Comment block — describe what the template does
*/}}
{{- define "myapp.fullname" -}}
{{- ... logic ... -}}
{{- end -}}
```

The `{{- define ... -}}` and `{{- end -}}` both have `-` to trim all whitespace around them — critical to avoid inserting blank lines into calling files.

### A Complete `_helpers.tpl`

```go
{{/*
Full name: prefer fullnameOverride, otherwise <release>-<chart>
Truncated to 63 chars — Kubernetes DNS label limit
*/}}
{{- define "myapp.fullname" -}}
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
Chart label: name-version, + replaced with _ (+ is invalid in label values)
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end -}}

{{/*
Common labels — applied to every resource
app.kubernetes.io/* are standard labels that tools (Helm, kubectl) understand
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}

{{/*
Selector labels — these go in matchLabels and Service selector
MUST NOT CHANGE after first deployment (Kubernetes rejects the update)
So they contain only stable identifiers, not version
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}

{{/*
Service account name
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
  {{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
  {{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end -}}
```

### Using Named Templates

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}      # include calls the template
  labels:
    {{- include "myapp.labels" . | nindent 4 }}  # nindent 4 = 4 spaces indent
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
```

> **`include` vs `template`:** Always use `include` rather than `template`. `include` is a function that returns a string (which you can then pipe to `nindent`). `template` outputs directly and can't be piped — indentation breaks.

---

## Chapter 3: Subcharts (Dependencies)

### What They Are

Your app usually needs a database and a cache. Instead of deploying them separately before your app, declare them as Helm dependencies. `helm install` deploys everything together.

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "11.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled    # Skip if postgresql.enabled=false

  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
# Download dependencies into charts/ directory
helm dependency update

# Now helm install deploys myapp + postgresql + redis
helm install myapp . -f values.yaml
```

### Configuring Subcharts from Parent values.yaml

Subchart values are nested under the subchart name:

```yaml
# values.yaml
replicaCount: 3

# PostgreSQL subchart configuration — nested under "postgresql"
postgresql:
  enabled: true
  global:
    postgresql:
      auth:
        username: myapp
        database: myappdb
        password: ""              # Will be auto-generated or use existingSecret
  primary:
    persistence:
      size: 50Gi
    resources:
      requests:
        memory: "512Mi"

# Redis subchart configuration
redis:
  enabled: true
  architecture: standalone        # standalone or replication
  auth:
    enabled: false                # Disable auth for dev
  master:
    persistence:
      size: 10Gi

# Disable dependencies for environments that use external services
# values-prod.yaml:
# postgresql:
#   enabled: false
# externalDatabase:
#   host: my-rds-instance.us-east-1.rds.amazonaws.com
```

### Connecting Your App to Subchart Services

Helm creates services for subcharts with predictable names: `<release-name>-<chart-name>`. Use this in your ConfigMap:

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}
data:
  DATABASE_URL: |-
    {{- if .Values.postgresql.enabled }}
    jdbc:postgresql://{{ .Release.Name }}-postgresql:5432/{{ .Values.postgresql.global.postgresql.auth.database }}
    {{- else }}
    {{ .Values.externalDatabase.url }}
    {{- end }}
  REDIS_URL: |-
    {{- if .Values.redis.enabled }}
    redis://{{ .Release.Name }}-redis-master:6379
    {{- else }}
    {{ .Values.externalRedis.url }}
    {{- end }}
```

### Checking Available Dependency Versions

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/postgresql --versions | head -10
helm search repo bitnami/redis --versions | head -10
```

---

## Chapter 4: Hooks

### What Hooks Do

Hooks let you run a Job at specific points in the Helm lifecycle. The most common use case: database migrations before the new app version starts.

```
helm upgrade
    │
    ▼
[pre-upgrade hook runs]     ← database migration job
    │ job completes
    ▼
[new pods deploy]           ← new app version
    │
    ▼
[post-upgrade hook runs]    ← smoke test, notification
```

### Hook Types

| Hook | When | Typical Use |
|------|------|-------------|
| `pre-install` | Before any resources created | Validate environment |
| `post-install` | After all resources created | Seed data, send notification |
| `pre-upgrade` | Before upgrade begins | **Database migrations**, backup |
| `post-upgrade` | After upgrade completes | Smoke tests, Slack notification |
| `pre-delete` | Before helm uninstall | Export data |
| `test` | When `helm test` runs | Integration tests |

### Database Migration Hook

```yaml
# templates/hooks/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migrations-{{ .Release.Revision }}
  annotations:
    # This annotation is what makes it a hook
    "helm.sh/hook": pre-upgrade,pre-install

    # Weight controls order when multiple hooks run at the same point
    # Lower numbers run first
    "helm.sh/hook-weight": "5"

    # What to do with the Job after it runs
    # hook-succeeded: delete only if successful (keep on failure for debugging)
    # before-hook-creation: delete old job before running new one
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2               # Retry twice on failure
  template:
    spec:
      restartPolicy: Never      # Required for Jobs
      containers:
      - name: migration
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        command: ["java", "-jar", "app.jar", "--spring.batch.job.enabled=true"]
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: migration
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Release.Name }}-db-secret
              key: password
```

### Post-Deploy Smoke Test

```yaml
# templates/tests/smoke-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-smoke-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
  - name: smoke-test
    image: curlimages/curl:latest
    command:
    - /bin/sh
    - -c
    - |
      echo "Testing health endpoint..."
      curl -f http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/actuator/health || exit 1
      echo "Testing API endpoint..."
      curl -f http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/api/hello || exit 1
      echo "All tests passed!"
```

```bash
# Run tests after installation
helm test myapp -n staging

# Output:
# NAME: myapp
# LAST DEPLOYED: ...
# NAMESPACE: staging
# STATUS: deployed
# TEST SUITE: myapp-smoke-test
# Last Started: ...
# Last Completed: ...
# Phase: Succeeded
```

### Notification Hook

```yaml
# templates/hooks/post-deploy-notify.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-upgrade,post-install
    "helm.sh/hook-weight": "100"              # Run last
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: notify
        image: curlimages/curl:latest
        command:
        - /bin/sh
        - -c
        - |
          curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"Deployed {{ .Chart.Name }} {{ .Chart.AppVersion }} to {{ .Release.Namespace }}"}' \
            $(SLACK_WEBHOOK)
        env:
        - name: SLACK_WEBHOOK
          valueFrom:
            secretKeyRef:
              name: slack-webhook
              key: url
```

---

## Chapter 5: Multi-Environment Pattern

### The Standard Pattern

```
myapp/
├── Chart.yaml
├── values.yaml           # Base defaults (shared by all environments)
├── values-dev.yaml       # Dev overrides
├── values-staging.yaml   # Staging overrides
└── values-prod.yaml      # Prod overrides
```

Base `values.yaml` defines all keys with safe defaults. Environment files override only what differs. Helm merges them:

```bash
# Merge order: values.yaml → values-prod.yaml (prod wins)
helm install myapp ./myapp \
  -f values.yaml \
  -f values-prod.yaml
```

### What Goes Where

```yaml
# values.yaml — DEFAULTS (all environments share these)
replicaCount: 1
image:
  pullPolicy: IfNotPresent
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 10
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
ingress:
  enabled: false
autoscaling:
  enabled: false
```

```yaml
# values-prod.yaml — OVERRIDES (only prod-specific differences)
replicaCount: 5
image:
  tag: v1.2.3
  pullPolicy: Always
ingress:
  enabled: true
  hosts:
    - host: api.myapp.com
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 1Gi
```

---

## Chapter 6: Useful Patterns

### Conditional Resource Creation

Don't create a resource if a value is false:

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  ...
{{- end }}
```

### Checking Kubernetes API Availability

Create resources only if the API version exists on this cluster:

```yaml
{{- if .Values.autoscaling.enabled }}
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
apiVersion: autoscaling/v2
{{- else }}
apiVersion: autoscaling/v1
{{- end }}
kind: HorizontalPodAutoscaler
...
{{- end }}
```

### Feature Flags

```yaml
# values.yaml
features:
  caching: true
  newCheckout: false
  analytics: false

# templates/deployment.yaml
env:
{{- if .Values.features.caching }}
- name: CACHE_ENABLED
  value: "true"
{{- end }}
{{- if .Values.features.newCheckout }}
- name: CHECKOUT_VERSION
  value: "v2"
{{- end }}
```

### NOTES.txt — Post-Install Instructions

`templates/NOTES.txt` is printed after `helm install`. Use it for connection instructions:

```
Thank you for installing {{ .Chart.Name }} v{{ .Chart.AppVersion }}!

{{- if .Values.ingress.enabled }}
Access your application at:
  https://{{ (index .Values.ingress.hosts 0).host }}
{{- else }}
To access locally:
  kubectl port-forward svc/{{ include "myapp.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
  Then open: http://localhost:8080
{{- end }}

Monitor pods:
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

View logs:
  kubectl logs -f -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}
```

---

## Quick Reference

### Helm Template Debugging

```bash
# Render templates without deploying (most useful command)
helm template my-release ./chart -f values-prod.yaml

# Render with debug output (shows computed values)
helm template my-release ./chart --debug

# Dry run against cluster (validates YAML against K8s API)
helm install my-release ./chart --dry-run --debug

# Lint for syntax errors
helm lint ./chart
helm lint ./chart -f values-prod.yaml    # Lint with specific values

# After deploy: see what was actually applied
helm get manifest my-release -n prod
helm get values my-release -n prod
```

### Helm Lifecycle Commands

```bash
helm install <name> ./chart -f values.yaml -n <ns> --create-namespace
helm upgrade <name> ./chart -f values.yaml -n <ns>
helm upgrade --install <name> ./chart     # Install if not exists, upgrade if does
helm rollback <name> 2 -n <ns>            # Roll back to revision 2
helm history <name> -n <ns>              # See all revisions
helm uninstall <name> -n <ns>
```

### Common Template Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `wrong number of args` | Piping to a function with wrong signature | Check function docs |
| `nil pointer evaluating` | Accessing a value that doesn't exist | Add `if` check or default |
| Indentation error in rendered YAML | Missing `nindent` | Add `\| nindent N` after `include` |
| Hook job left running | Missing `hook-delete-policy` | Add annotation |
| Subchart values not applying | Wrong nesting level | Check Chart.yaml dependency name |
