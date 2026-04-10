# Part 3: Helm Deep Dive — Beyond Basic Templating

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Intermediate
> **Prerequisites:** Completed Parts 1 and 2, or familiar with Kubernetes Deployments, Services, ConfigMaps, and basic Helm install/upgrade
> **Time to complete:** 4–5 hours
> **What you'll learn:** Go template engine internals, named templates, subchart dependencies, lifecycle hooks, advanced patterns, and the `helm diff` plugin for safe production changes

---

## What This Part Covers

In Parts 1 and 2, you used Helm like a deployment tool — `helm install`, `helm upgrade`, a `values.yaml` with some overrides. That's the surface. Helm's real power is as a templating and packaging system capable of managing complex multi-service applications across multiple environments with lifecycle hooks, dependency management, and safe upgrade previews.

This part goes deep. By the end you'll understand exactly how Helm's template engine works, how to write reusable named templates that eliminate duplication, how to bundle application dependencies (PostgreSQL, Redis) into a single deployable chart, how to run database migrations safely before new code deploys, and how to preview every change before it touches your cluster.

---

## Chapter 1: The Go Template Engine

### What Go Templates Are and Why Helm Uses Them

When Helm renders a chart, it takes your template files — which look like Kubernetes YAML with `{{ }}` blocks mixed in — and replaces every `{{ }}` expression with computed values to produce valid YAML. The engine that does this processing is Go's standard `text/template` package, extended with Helm-specific functions.

Helm chose Go templates for two reasons: Helm itself is written in Go (zero extra dependency), and Go templates are powerful enough to express conditionals, loops, and function pipelines without becoming a full programming language. The limitation is that they're verbose and whitespace-sensitive in ways that bite beginners repeatedly. This chapter covers all of that explicitly.

### The Four Top-Level Objects

Every Helm template has access to four top-level objects. Understanding what each one contains saves a lot of confusion.

**`.Values`** — Everything from `values.yaml` (and any `-f overrides.yaml` files you pass). This is the primary object you work with.

```yaml
# values.yaml
replicaCount: 3
image:
  repository: myapp
  tag: "1.0.0"
database:
  host: postgres-svc
  port: 5432
```

```yaml
# In a template:
replicas: {{ .Values.replicaCount }}                  # → 3
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"  # → "myapp:1.0.0"
host: {{ .Values.database.host }}                     # → postgres-svc
```

**`.Release`** — Information about this specific Helm release (set by Helm at install/upgrade time):

```yaml
{{ .Release.Name }}       # Name given at: helm install THIS-NAME ./chart
{{ .Release.Namespace }}  # Namespace deployed into
{{ .Release.Service }}    # Always "Helm"
{{ .Release.Revision }}   # 1 on install, 2 on first upgrade, etc.
{{ .Release.IsInstall }}  # true on first install, false on upgrades
{{ .Release.IsUpgrade }}  # true on upgrades, false on first install
```

**`.Chart`** — Contents of `Chart.yaml`:

```yaml
{{ .Chart.Name }}        # Chart name
{{ .Chart.Version }}     # Chart version (e.g., "1.2.0")
{{ .Chart.AppVersion }}  # Application version (e.g., "2.1.3")
{{ .Chart.Description }} # Chart description
```

**`.Capabilities`** — Information about the Kubernetes cluster Helm is deploying to:

```yaml
{{ .Capabilities.KubeVersion.Major }}   # Kubernetes major version
{{ .Capabilities.KubeVersion.Minor }}   # Kubernetes minor version

# Check if an API is available on this cluster (used for conditional resources)
{{ .Capabilities.APIVersions.Has "autoscaling/v2" }}  # true or false
```

### Syntax: The `{{ }}` Delimiters

Everything between `{{` and `}}` is a template expression — evaluated and replaced with output. Everything outside is literal text output as-is.

```yaml
# Literal text — output unchanged
metadata:
  name: my-app         # ← literal, always "my-app"

# Template expression — replaced with computed value
metadata:
  name: {{ .Release.Name }}   # ← replaced with the release name
```

### Whitespace Control — The Most Common Source of Bugs

Go templates preserve whitespace around `{{ }}` blocks by default. This means expressions on their own line produce blank lines in the output, which can produce invalid YAML or unexpected formatting.

Consider this template:

```yaml
metadata:
  labels:
    app: myapp
{{ if .Values.extraLabel }}
    extra: true
{{ end }}
```

When `.Values.extraLabel` is true, the output is:

```yaml
metadata:
  labels:
    app: myapp

    extra: true

```

Two blank lines appear — one before `extra:` and one after `end`. While Kubernetes tolerates blank lines in YAML, this is noise that makes `helm template` output hard to read and can occasionally cause parsing issues.

The `-` trim marker removes all whitespace (including newlines) on the side it's on:

```yaml
metadata:
  labels:
    app: myapp
    {{- if .Values.extraLabel }}
    extra: true
    {{- end }}
```

Now the output is clean:

```yaml
metadata:
  labels:
    app: myapp
    extra: true
```

The rule: **use `{{-` to trim the whitespace before the expression, and `-}}` to trim after it.** Most Helm blocks use `{{-` at minimum. Use both `{{-` and `-}}` inside `define` blocks (covered in Chapter 2) to prevent any output escaping the template.

```yaml
# Trim only before (most common — trims the newline from the preceding line)
{{- if .Values.enabled }}

# Trim only after (less common — trims the newline that follows)
{{ if .Values.enabled -}}

# Trim both sides (used inside define blocks)
{{- define "myapp.name" -}}
{{- end -}}
```

### Pipelines — Chaining Functions

The `|` pipe passes the output of one function as the last argument of the next, exactly like Unix pipes. Most Helm template work uses pipelines to transform values.

```yaml
# Single function
name: {{ .Values.name | upper }}         # "myapp" → "MYAPP"
name: {{ .Values.name | lower }}         # "MyApp" → "myapp"
name: {{ .Values.name | title }}         # "my app" → "My App"

# Chained pipeline — read left to right
name: {{ .Values.name | lower | replace " " "-" | trunc 63 }}
#        ↑ value        ↑ lowercase  ↑ spaces→dashes  ↑ max 63 chars

# quote: wraps in double quotes — important for values that look like numbers
#        or booleans (e.g., "true", "3000") to force YAML string type
port: {{ .Values.service.port | quote }}   # 8080 → "8080"

# default: use a fallback if the value is empty/nil
tag: {{ .Values.image.tag | default "latest" }}
```

### Important Functions

**`default` — provide fallback values:**
```yaml
tag: {{ .Values.image.tag | default "latest" }}
replicas: {{ .Values.replicaCount | default 1 }}
```

**`required` — fail loudly if a value is missing:**

Instead of silently deploying with a missing value, `required` stops the render and prints your message:

```yaml
# If .Values.database.host is empty, helm install fails with this message
host: {{ required "database.host is required — set it in values.yaml" .Values.database.host }}
```

This is far better than discovering the problem when the pod starts and can't connect to anything. Use `required` for every value that has no reasonable default and will break the application if missing.

**`quote` and `toYaml`:**
```yaml
# quote: string-safe encoding of scalar values
value: {{ .Values.logLevel | quote }}    # INFO → "INFO"

# toYaml: render a complex value (map, list) as indented YAML
# MUST be paired with nindent or indent
env:
  {{- toYaml .Values.extraEnv | nindent 2 }}
```

**`nindent` vs `indent`:**

Both add indentation. The difference is `nindent` also adds a leading newline before the indented content.

```yaml
# indent 4: adds 4 spaces to each line — NO leading newline
labels:
  app: myapp{{ .Values.extraLabels | indent 4 }}  # often breaks formatting

# nindent 4: adds a leading newline, THEN 4 spaces to each line
# This is almost always what you want when including a block
labels:
  {{- include "myapp.labels" . | nindent 4 }}
# Result:
# labels:
#     helm.sh/chart: myapp-1.0.0
#     app.kubernetes.io/name: myapp
```

Use `nindent` when the expression is on its own line (the leading newline fills the gap). Use `indent` only when the expression is inline within existing text.

### Conditionals

```yaml
# Simple if
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# if / else
replicas: {{ if .Values.production }}5{{ else }}1{{ end }}

# if / else if / else
logLevel: {{ if eq .Values.environment "prod" -}}
  WARN
{{- else if eq .Values.environment "staging" -}}
  INFO
{{- else -}}
  DEBUG
{{- end }}

# Checking for empty / nil values
{{- if .Values.podAnnotations }}
annotations:
  {{- toYaml .Values.podAnnotations | nindent 4 }}
{{- end }}

# Comparison operators
{{- if eq .Values.service.type "LoadBalancer" }}
{{- if ne .Values.service.type "ClusterIP" }}
{{- if gt .Values.replicaCount 1 }}
{{- if and .Values.ingress.enabled .Values.tls.enabled }}
{{- if or .Values.debug .Values.verbose }}
{{- if not .Values.productionMode }}
```

### Loops with `range`

**Range over a list:**
```yaml
# values.yaml
extraEnv:
  - name: LOG_FORMAT
    value: json
  - name: MAX_CONNECTIONS
    value: "100"
```

```yaml
# template
env:
{{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
{{- end }}
```

Inside `range`, `.` refers to the current item (each element of the list). To access the parent context (`.Values`, `.Release`, etc.) from inside a range, assign it to a variable first:

```yaml
{{- $root := . }}   ← capture the outer context before entering range
{{- range .Values.extraEnv }}
  - name: {{ .name }}
    value: {{ .value | quote }}
    # Access outer context via $root:
    releaseName: {{ $root.Release.Name }}
{{- end }}
```

**Range over a map:**
```yaml
# values.yaml
labels:
  team: platform
  cost-center: engineering
  tier: backend
```

```yaml
# template
labels:
{{- range $key, $value := .Values.labels }}
  {{ $key }}: {{ $value | quote }}
{{- end }}
```

---

## Chapter 2: Named Templates — Eliminating Duplication

### Why Named Templates Exist

Every Kubernetes resource in your chart needs labels. The Deployment needs them. The Service needs them. The Ingress needs them. The ServiceAccount, HPA, PodDisruptionBudget — all of them.

Without named templates, you copy and paste the same label block into every file. When you need to add a new standard label (a cost-centre tag, a compliance label), you edit every file. You miss one. It's inconsistent. The Deployment has the label, the Service doesn't. A monitoring tool that queries by label now misses the Service.

Named templates solve this by letting you define a block of text once and call it by name from anywhere:

```yaml
# Define once in _helpers.tpl
{{- define "myapp.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

# Use in every resource
metadata:
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
```

Change the definition once → every resource in the chart gets the update.

### Files Starting with `_` Are Not Rendered

Helm renders every `.yaml` file in `templates/` as a Kubernetes manifest — except files whose name starts with `_`. These are template-only files: they define named templates but produce no output themselves.

The convention is `_helpers.tpl` for the main file of named templates, but you can have `_db-helpers.tpl`, `_network-helpers.tpl`, or any `_*.tpl` file. All are ignored during rendering; all can be used from any template file.

### `define` and `end`

A named template is created with `define` and closed with `end`. The name is a string — by convention, prefixed with the chart name to avoid collisions when subcharts are involved:

```
{{- define "myapp.fullname" -}}
... template body ...
{{- end -}}
```

The `-` on both the opening `define` and closing `end` is essential. Without them, every `include` call inserts a blank line before and after the template's output. With them, the template's output is clean — only the content you explicitly wrote.

### `include` vs `template` — Always Use `include`

There are two ways to call a named template:

```yaml
# Method 1: template action
{{ template "myapp.labels" . }}

# Method 2: include function
{{ include "myapp.labels" . }}
```

They produce the same output — but they are not interchangeable.

`template` is an action that outputs its result directly into the document. It returns no value. You cannot pipe it.

`include` is a function that returns the template's output as a string. You can pipe that string.

```yaml
# This DOES NOT WORK — template cannot be piped
metadata:
  labels:
    {{ template "myapp.labels" . | nindent 4 }}   # ← syntax error

# This WORKS — include returns a string, which can be piped
metadata:
  labels:
    {{- include "myapp.labels" . | nindent 4 }}   # ← correct
```

Since you almost always need `nindent` when including a multi-line template block, `include` is nearly always the right choice. The only time `template` is used is when you're certain the output doesn't need any transformation and you want to avoid the minor overhead of the function call — which in practice is never worth the limitation.

**Rule: always use `include`, never `template`.**

### A Complete `_helpers.tpl` — Explained Line by Line

```
{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.name
The chart name. Allows override via .Values.nameOverride.
─────────────────────────────────────────────────────────────────────────────
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}


{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.fullname
The full resource name: <release-name>-<chart-name>
Truncated to 63 characters.

WHY 63 characters: Kubernetes uses resource names as DNS labels in certain
contexts (e.g., pod hostnames, service DNS names). RFC 1123 requires DNS
labels to be at most 63 characters. Exceeding this causes subtle failures
in DNS resolution, not a loud error.

If .Values.fullnameOverride is set, use that directly (allows complete
control over the name without it being prefixed with the release name).
─────────────────────────────────────────────────────────────────────────────
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
{{- end }}


{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.chart
The chart label value: <name>-<version>
The + in chart version strings is replaced with _ because + is not valid
in Kubernetes label values.
─────────────────────────────────────────────────────────────────────────────
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}


{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.labels
Common labels applied to EVERY resource in the chart.
These are the standard app.kubernetes.io/* labels that tools like
kubectl, Helm, and monitoring systems use for querying and discovery.

Note: includes app.kubernetes.io/version which changes on every release.
This is fine for metadata labels — but NOT for selector labels (see below).
─────────────────────────────────────────────────────────────────────────────
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}


{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.selectorLabels
Labels used in:
  - Deployment.spec.selector.matchLabels
  - Service.spec.selector

WHY these must be kept minimal and NEVER change after first deployment:

Kubernetes REJECTS updates to spec.selector.matchLabels on an existing
Deployment. If you added app.kubernetes.io/version here and later tried
to upgrade, Kubernetes would return:
  "field is immutable"

That's why version is in myapp.labels (metadata, can change) but NOT
here in myapp.selectorLabels (selector, immutable after creation).

Only include values that are stable for the entire lifetime of the resource.
─────────────────────────────────────────────────────────────────────────────
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}


{{/*
─────────────────────────────────────────────────────────────────────────────
myapp.serviceAccountName
Determines the service account name to use.
If serviceAccount.create is true, use the explicitly set name or fall back
to the fullname. If not creating one, use the explicitly set name or
fall back to "default" (the Kubernetes built-in default SA).
─────────────────────────────────────────────────────────────────────────────
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### Writing Your Own Named Templates — Practical Patterns

**Pattern 1: Environment variable block**

Avoids repeating the same probe env vars across multiple containers:

```
{{- define "myapp.commonEnv" -}}
- name: SPRING_PROFILES_ACTIVE
  value: {{ .Values.config.profile | quote }}
- name: LOG_LEVEL
  value: {{ .Values.config.logLevel | quote }}
- name: POD_NAME
  valueFrom:
    fieldRef:
      fieldPath: metadata.name
- name: POD_NAMESPACE
  valueFrom:
    fieldRef:
      fieldPath: metadata.namespace
{{- end }}
```

Use in deployment template:
```yaml
containers:
- name: myapp
  env:
  {{- include "myapp.commonEnv" . | nindent 4 }}
  - name: ADDITIONAL_VAR    # app-specific env var added after the shared ones
    value: "extra"
```

**Pattern 2: Resource block**

```
{{- define "myapp.resources" -}}
requests:
  cpu: {{ .Values.resources.requests.cpu | quote }}
  memory: {{ .Values.resources.requests.memory | quote }}
limits:
  cpu: {{ .Values.resources.limits.cpu | quote }}
  memory: {{ .Values.resources.limits.memory | quote }}
{{- end }}
```

```yaml
resources:
  {{- include "myapp.resources" . | nindent 2 }}
```

**Pattern 3: Passing a sub-context to a template**

The second argument to `include` is the context (usually `.`). You can pass a sub-value instead:

```
{{- define "myapp.probeConfig" -}}
httpGet:
  path: {{ .path }}
  port: 8080
periodSeconds: {{ .periodSeconds }}
failureThreshold: {{ .failureThreshold }}
{{- end }}
```

```yaml
readinessProbe:
  {{- include "myapp.probeConfig" .Values.probes.readiness | nindent 2 }}
livenessProbe:
  {{- include "myapp.probeConfig" .Values.probes.liveness | nindent 2 }}
```

---

## Chapter 3: Subcharts — Managing Application Dependencies

### Why Subcharts Exist

Your Spring Boot service doesn't run in isolation. It needs a PostgreSQL database. It needs a Redis cache. It needs RabbitMQ for async messaging. Before your app can start, those services must already be running.

Without subcharts, you deploy dependencies separately and manually before deploying your app:

```bash
# Manual approach — error-prone and order-dependent
helm install postgres bitnami/postgresql -f postgres-values.yaml
helm install redis bitnami/redis -f redis-values.yaml
# Wait for them to be ready...
helm install myapp ./myapp-chart -f values.yaml
```

This creates drift. Different teams use different versions. A new developer setting up their environment forgets a step. CI/CD pipelines become complex orchestration scripts.

Subcharts package your application together with its dependencies. One `helm install` command deploys everything in the right order with the right configuration:

```bash
helm install myapp ./myapp-chart -f values.yaml
# Deploys: myapp + postgresql + redis + rabbitmq — all together
```

### Declaring Dependencies in Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2
name: myapp
description: E-commerce API with all dependencies
version: 1.0.0
appVersion: "2.1.0"

dependencies:
  - name: postgresql
    version: "12.x.x"           # Semver range — x means "any patch version"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled  # Only deploy if postgresql.enabled=true in values

  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled

  - name: rabbitmq
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: rabbitmq.enabled

  - name: prometheus
    version: "25.x.x"
    repository: "https://prometheus-community.github.io/helm-charts"
    condition: monitoring.enabled
    tags:
      - observability              # Tags let you enable/disable groups of dependencies
```

**Download dependencies:**

```bash
# Add the repositories first
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Download — creates charts/ directory with .tgz files for each dependency
helm dependency update

# Verify
helm dependency list
# NAME         VERSION   REPOSITORY                           STATUS
# postgresql   12.x.x    https://charts.bitnami.com/bitnami  ok
# redis        17.x.x    https://charts.bitnami.com/bitnami  ok
# rabbitmq     12.x.x    https://charts.bitnami.com/bitnami  ok
```

The `charts/` directory should be committed to version control (or added to your artifact store) so deployments are reproducible. Don't rely on `helm dependency update` running during deployment — the upstream chart version could change.

### Configuring Subcharts from the Parent values.yaml

Each subchart's configuration lives in a top-level key in the parent's `values.yaml` that matches the subchart's name. The parent's values are passed down to the subchart under that key.

```yaml
# values.yaml

# ────────────────────────────────────────
# Your application
# ────────────────────────────────────────
replicaCount: 2
image:
  repository: yourusername/myapp
  tag: "2.1.0"

# ────────────────────────────────────────
# PostgreSQL subchart configuration
# Nested under "postgresql" — matches the dependency name in Chart.yaml
# See: https://artifacthub.io/packages/helm/bitnami/postgresql for all options
# ────────────────────────────────────────
postgresql:
  enabled: true
  auth:
    username: appuser
    database: appdb
    # Use an existing Kubernetes Secret rather than hardcoding password here
    existingSecret: db-credentials
    secretKeys:
      adminPasswordKey: postgres-password
      userPasswordKey: password
  primary:
    persistence:
      enabled: true
      size: 20Gi
      storageClass: fast-ssd
    resources:
      requests:
        memory: "512Mi"
        cpu: "250m"
      limits:
        memory: "2Gi"
        cpu: "1000m"

# ────────────────────────────────────────
# Redis subchart configuration
# ────────────────────────────────────────
redis:
  enabled: true
  architecture: standalone   # standalone (dev/staging) or replication (prod)
  auth:
    enabled: true
    existingSecret: redis-credentials
    existingSecretPasswordKey: redis-password
  master:
    persistence:
      enabled: true
      size: 5Gi
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "250m"

# ────────────────────────────────────────
# RabbitMQ subchart configuration
# ────────────────────────────────────────
rabbitmq:
  enabled: true
  auth:
    username: appuser
    existingPasswordSecret: rabbitmq-credentials
  persistence:
    enabled: true
    size: 10Gi

# ────────────────────────────────────────
# Disable all dependencies for prod that uses external managed services
# Override in values-prod.yaml:
# postgresql:
#   enabled: false
# externalDatabase:
#   host: myapp.cluster-xyz.us-east-1.rds.amazonaws.com
#   port: 5432
#   username: appuser
#   existingSecret: rds-credentials
# ────────────────────────────────────────
```

### Accessing Subchart Service Names in Your Templates

When Helm installs a subchart, it creates its Kubernetes resources with predictable names based on the release name and chart name. Your application needs to connect to these services, so you must construct the service names correctly.

The pattern is: `{{ .Release.Name }}-{{ subchart-name }}`

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
data:
  # Construct the database URL using the subchart's service name
  DATABASE_URL: |-
    {{- if .Values.postgresql.enabled }}
    jdbc:postgresql://{{ .Release.Name }}-postgresql:5432/{{ .Values.postgresql.auth.database }}
    {{- else }}
    jdbc:postgresql://{{ required "externalDatabase.host required when postgresql.enabled=false" .Values.externalDatabase.host }}:{{ .Values.externalDatabase.port | default 5432 }}/{{ .Values.externalDatabase.name }}
    {{- end }}

  REDIS_URL: |-
    {{- if .Values.redis.enabled }}
    redis://{{ .Release.Name }}-redis-master:6379
    {{- else }}
    redis://{{ required "externalRedis.host required when redis.enabled=false" .Values.externalRedis.host }}:{{ .Values.externalRedis.port | default 6379 }}
    {{- end }}

  RABBITMQ_URL: |-
    {{- if .Values.rabbitmq.enabled }}
    amqp://{{ .Values.rabbitmq.auth.username }}@{{ .Release.Name }}-rabbitmq:5672
    {{- else }}
    {{ required "externalRabbitmq.url required when rabbitmq.enabled=false" .Values.externalRabbitmq.url }}
    {{- end }}
```

This pattern is the key to making the same chart work with in-cluster services (dev/staging) and external managed services (production) — just toggle `postgresql.enabled`.

### Global Values — Sharing Across All Subcharts

Values under `.Values.global` are shared with all subcharts without any nesting. Useful for cluster-wide settings:

```yaml
# values.yaml
global:
  imageRegistry: "myregistry.example.com"   # All subcharts use this registry prefix
  storageClass: "fast-ssd"                   # All subcharts use this storage class
  postgresql:
    auth:
      postgresPassword: ""                   # Shared postgres password
```

Subcharts that support global values will automatically use `global.imageRegistry` for their images, `global.storageClass` for their PVCs, etc. Check each subchart's documentation to see which global values it honours.

### `import-values` — Pulling Subchart Output into the Parent

Some subcharts export values (credentials, service names, ports) that you want accessible in your parent chart without hardcoding. The `import-values` key in `Chart.yaml` pulls them:

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    import-values:
      - child: primary.service   # Path in the subchart's values
        parent: postgresql.service  # Where it appears in parent's .Values
```

After this, `.Values.postgresql.service.port` in your parent templates contains the PostgreSQL service port as exported by the subchart. Less commonly needed, but useful when subcharts compute values dynamically.

### Disabling Subcharts for External Services

The `condition` field in `Chart.yaml` maps to a boolean value in `values.yaml`. Set it to `false` to skip deploying that subchart entirely:

```yaml
# values-prod.yaml
# Production uses RDS instead of in-cluster PostgreSQL
postgresql:
  enabled: false

# Provide connection details for the external database
externalDatabase:
  host: myapp-prod.cluster-xyz.us-east-1.rds.amazonaws.com
  port: 5432
  name: appdb
  username: appuser
  existingSecret: rds-credentials

# Production uses ElastiCache instead of in-cluster Redis
redis:
  enabled: false

externalRedis:
  host: myapp-prod.xyz.cache.amazonaws.com
  port: 6379
  existingSecret: elasticache-credentials
```

The ConfigMap template above already handles both cases with `{{- if .Values.postgresql.enabled }}`. The same chart, with two different values files, connects to in-cluster services in dev and managed cloud services in production.


## Chapter 4: Hooks — Running Logic at Lifecycle Points

### Why Hooks Exist

Consider what happens during `helm upgrade` when you have a database migration:

Without hooks:
1. Helm replaces old pods with new pods (rolling update)
2. New pods start running new application code
3. New application code tries to read/write database columns that don't exist yet
4. Errors, data corruption, or crashes until the migration finishes separately

With hooks:
1. Helm runs the `pre-upgrade` hook — the migration Job
2. Migration completes successfully
3. Helm replaces old pods with new pods
4. New pods start running new application code
5. All database columns exist — everything works

Hooks let you attach Jobs or Pods to specific points in the Helm lifecycle so that prerequisite operations complete before the main deployment proceeds.

### All Hook Types

| Hook | When it runs | Typical use |
|------|-------------|-------------|
| `pre-install` | Before any resources are created on fresh install | Validate environment, check prerequisites |
| `post-install` | After all resources are ready on fresh install | Seed initial data, send deployment notification |
| `pre-upgrade` | Before any resources are replaced during upgrade | **Database migrations**, create backups |
| `post-upgrade` | After all resources are ready after upgrade | Run smoke tests, notify team |
| `pre-rollback` | Before rolling back to a previous release | Reverse data migrations if needed |
| `post-rollback` | After rollback is complete | Notify team of rollback |
| `pre-delete` | Before `helm uninstall` removes resources | Export data, drain connections |
| `post-delete` | After `helm uninstall` completes | Clean up external resources |
| `test` | Only when `helm test` is explicitly run | Integration/smoke tests |

### Hook Weights — Controlling Execution Order

When multiple hooks fire at the same lifecycle point (e.g., two `pre-upgrade` hooks), they run in order of their weight. Lower weight numbers run first.

```yaml
annotations:
  "helm.sh/hook": pre-upgrade
  "helm.sh/hook-weight": "-5"   # Runs first (before weight 0 and weight 5)
```

```yaml
"helm.sh/hook-weight": "0"      # Default weight — runs after negative weights
```

```yaml
"helm.sh/hook-weight": "5"      # Runs last (after -5 and 0)
```

Use this to order hooks when you need multiple things to happen in sequence. For example: backup the database (weight -10) → run migrations (weight 0) → verify migration success (weight 5).

Hooks with the same weight run in alphabetical order by resource name — not a reliable ordering strategy. Always use explicit weights when order matters.

### Hook Delete Policies — What Happens to Hook Resources After They Run

A hook creates a Kubernetes Job or Pod. What should happen to it after it completes?

```yaml
annotations:
  "helm.sh/hook-delete-policy": hook-succeeded,before-hook-creation
```

| Policy | When the resource is deleted |
|--------|------------------------------|
| `hook-succeeded` | After the hook completes successfully |
| `hook-failed` | After the hook fails |
| `before-hook-creation` | Before a new hook resource is created (on next run) |

**Why `before-hook-creation` is almost always needed:**

Without it, the second `helm upgrade` fails with:
```
Error: UPGRADE FAILED: cannot patch "myapp-migration": Job.batch "myapp-migration"
is invalid: spec.template: Forbidden: pod template updates are not allowed
```

Helm can't update an existing Job (Jobs are immutable). The `before-hook-creation` policy deletes the old Job before creating the new one, so each upgrade gets a fresh Job.

**Why keep `hook-failed` resources:**

If a migration Job fails, you need its logs to understand why. If you delete the Job on failure, those logs are gone. Keep failed hooks so you can run `kubectl logs <job-pod>` to diagnose.

Recommended combination for most hooks:
```yaml
"helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
# Deletes before next run (prevents "already exists" error)
# Deletes on success (cleans up completed jobs automatically)
# Keeps on failure (logs are available for debugging)
```

### Database Migration Hook — Complete Example

This is the most common and most important hook. It runs Flyway or Liquibase migrations before the new application version starts:

```yaml
# templates/hooks/db-migration.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-migration
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    # Run on both fresh install and every upgrade
    "helm.sh/hook": pre-install,pre-upgrade

    # Weight 0 (default) — runs after any validation hooks (negative weight)
    "helm.sh/hook-weight": "0"

    # Clean up before next run + after success; keep on failure for debugging
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  # Retry twice on failure before marking the Job as failed
  # Helm waits for the Job to complete before proceeding — if it fails,
  # the upgrade is aborted and Helm rolls back automatically (with --atomic)
  backoffLimit: 2

  template:
    spec:
      restartPolicy: Never   # Required for Jobs — don't restart the pod on failure

      containers:
      - name: migration
        # Use the same application image — it contains Flyway/Liquibase
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

        # Run the app in migration-only mode
        # With Spring Boot + Flyway, this profile runs migrations and exits
        command:
          - java
          - -XX:+UseContainerSupport
          - -XX:MaxRAMPercentage=75.0
          - -jar
          - /app/app.jar
          - --spring.profiles.active=migration
          - --spring.batch.job.enabled=false  # Don't run batch jobs during migration

        env:
        - name: SPRING_PROFILES_ACTIVE
          value: migration
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: {{ include "myapp.fullname" . }}-config
              key: DATABASE_URL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: {{ .Values.database.existingSecret | default (printf "%s-db" (include "myapp.fullname" .)) }}
              key: password

        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**Spring Boot application profile for migration:**

```yaml
# application-migration.yml
spring:
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DB_PASSWORD}
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
  jpa:
    hibernate:
      ddl-auto: validate   # Validate schema after migration, don't auto-create

# Exit immediately after Flyway runs — don't start the web server
spring.main.web-application-type: none
```

### Pre-Install Validation Hook

Catches missing configuration before any resources are created — fast feedback instead of broken pods:

```yaml
# templates/hooks/validate-config.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-validate
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "-5"   # Run BEFORE migration hook
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 0   # Don't retry validation — fail fast
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: validate
        image: busybox:latest
        command:
          - /bin/sh
          - -c
          - |
            echo "Checking required environment..."

            # Fail if database URL is not set
            if [ -z "$DATABASE_URL" ]; then
              echo "ERROR: DATABASE_URL is not set"
              exit 1
            fi

            # Check database connectivity
            # nc (netcat) exits 0 if port is reachable, 1 if not
            DB_HOST=$(echo $DATABASE_URL | sed 's|.*://||' | cut -d: -f1)
            DB_PORT=$(echo $DATABASE_URL | sed 's|.*://||' | cut -d: -f2 | cut -d/ -f1)
            if ! nc -z -w5 $DB_HOST $DB_PORT; then
              echo "ERROR: Cannot reach database at $DB_HOST:$DB_PORT"
              exit 1
            fi

            echo "Validation passed."
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: {{ include "myapp.fullname" . }}-config
              key: DATABASE_URL
```

### Post-Deploy Notification Hook

Sends a Slack message after a successful upgrade:

```yaml
# templates/hooks/notify-deploy.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-notify
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "10"   # Run last, after smoke tests
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  backoffLimit: 2
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
            curl -s -X POST \
              -H 'Content-type: application/json' \
              --data "{
                \"text\": \"✅ Deployed *{{ .Chart.Name }}* v{{ .Chart.AppVersion }} to *{{ .Release.Namespace }}*\",
                \"attachments\": [{
                  \"color\": \"good\",
                  \"fields\": [
                    {\"title\": \"Release\", \"value\": \"{{ .Release.Name }}\", \"short\": true},
                    {\"title\": \"Revision\", \"value\": \"{{ .Release.Revision }}\", \"short\": true}
                  ]
                }]
              }" \
              $(SLACK_WEBHOOK_URL)
        env:
        - name: SLACK_WEBHOOK_URL
          valueFrom:
            secretKeyRef:
              name: slack-credentials
              key: webhook-url
```

### Helm Test Hooks — Smoke Tests

Test hooks run only when you explicitly call `helm test` — never during normal install/upgrade. They verify the deployment is working after it completes.

```yaml
# templates/tests/smoke-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "myapp.fullname" . }}-smoke-test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  restartPolicy: Never
  containers:
  - name: smoke-test
    image: curlimages/curl:latest
    command:
      - /bin/sh
      - -c
      - |
        BASE_URL="http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}"

        echo "Testing health endpoint..."
        curl -sf "$BASE_URL/actuator/health" || { echo "FAIL: health check"; exit 1; }

        echo "Testing liveness endpoint..."
        curl -sf "$BASE_URL/actuator/health/liveness" || { echo "FAIL: liveness"; exit 1; }

        echo "Testing readiness endpoint..."
        curl -sf "$BASE_URL/actuator/health/readiness" || { echo "FAIL: readiness"; exit 1; }

        echo "Testing API endpoint..."
        RESPONSE=$(curl -sf "$BASE_URL/api/hello")
        echo "Response: $RESPONSE"
        echo "$RESPONSE" | grep -q "Hello" || { echo "FAIL: unexpected response"; exit 1; }

        echo "All smoke tests passed."
```

Run after deployment:
```bash
helm test myapp -n staging
# NAME: myapp
# LAST DEPLOYED: Mon Jan 1 10:00:00 2024
# NAMESPACE: staging
# STATUS: deployed
# TEST SUITE:     myapp-smoke-test
# Last Started:   Mon Jan 1 10:01:00 2024
# Last Completed: Mon Jan 1 10:01:15 2024
# Phase:          Succeeded

# See test output
kubectl logs myapp-smoke-test -n staging
# Testing health endpoint...
# Testing liveness endpoint...
# Testing readiness endpoint...
# Testing API endpoint...
# Response: Hello, World from Spring Boot!
# All smoke tests passed.
```

---

## Chapter 5: Advanced Patterns

### Feature Flags — Conditional Configuration

Feature flags let you enable or disable application features per environment without code changes:

```yaml
# values.yaml
features:
  newCheckout: false
  darkMode: true
  betaAnalytics: false
  maintenanceMode: false
```

```yaml
# templates/configmap.yaml
data:
  application.properties: |
    {{- if .Values.features.newCheckout }}
    feature.checkout.version=v2
    {{- else }}
    feature.checkout.version=v1
    {{- end }}
    {{- if .Values.features.maintenanceMode }}
    maintenance.enabled=true
    maintenance.message=We are currently performing scheduled maintenance.
    {{- end }}

# templates/deployment.yaml
env:
{{- if .Values.features.darkMode }}
- name: DARK_MODE_ENABLED
  value: "true"
{{- end }}
{{- if .Values.features.betaAnalytics }}
- name: ANALYTICS_ENDPOINT
  value: {{ .Values.analytics.betaEndpoint | quote }}
{{- end }}
```

Enable features per environment:
```yaml
# values-staging.yaml
features:
  newCheckout: true    # Test new checkout in staging only
  betaAnalytics: true  # Test beta analytics in staging only
```

### Multi-Environment Pattern — Base + Overlays

The cleanest multi-environment setup uses a base `values.yaml` with safe defaults and environment-specific overlay files that override only what differs:

```yaml
# values.yaml — safe defaults for all environments
replicaCount: 1
image:
  pullPolicy: IfNotPresent
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"
postgresql:
  enabled: true
  primary:
    persistence:
      size: 1Gi
monitoring:
  enabled: false
```

```yaml
# values-staging.yaml — override only staging differences
replicaCount: 2
image:
  tag: "staging-latest"
  pullPolicy: Always
postgresql:
  primary:
    persistence:
      size: 10Gi
monitoring:
  enabled: true
```

```yaml
# values-prod.yaml — override only production differences
replicaCount: 5
image:
  repository: myregistry.com/myapp
  tag: "v2.1.0"
  pullPolicy: Always
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"
postgresql:
  enabled: false   # Use RDS in production
externalDatabase:
  host: myapp.cluster-xyz.us-east-1.rds.amazonaws.com
monitoring:
  enabled: true
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
```

Deploy with Helm merging the files in order (later files win):
```bash
# Staging: base + staging overlay
helm upgrade --install myapp ./chart \
  -f values.yaml \
  -f values-staging.yaml \
  --namespace staging

# Production: base + prod overlay
helm upgrade --install myapp ./chart \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace production \
  --atomic
```

### The `lookup` Function — Checking Existing Resources

`lookup` queries the Kubernetes API for existing resources. Useful for checking whether a resource already exists before trying to create it, or reading values from existing resources:

```yaml
{{- $existingSecret := lookup "v1" "Secret" .Release.Namespace "my-db-secret" }}
{{- if $existingSecret }}
# Secret already exists — use its existing password, don't regenerate
password: {{ $existingSecret.data.password }}
{{- else }}
# Secret doesn't exist — generate a new password
password: {{ randAlphaNum 32 | b64enc | quote }}
{{- end }}
```

**Critical warning: `lookup` does not work during `--dry-run`.**

When you run `helm install --dry-run` or `helm template`, the template is rendered without connecting to a live cluster (or with limited API access). `lookup` always returns an empty result in dry-run mode — meaning code paths that depend on it may produce different output than the real install.

```yaml
# This block will ALWAYS take the "else" branch during --dry-run
{{- $secret := lookup "v1" "Secret" .Release.Namespace "my-secret" }}
{{- if $secret }}
  # This branch never executes in dry-run
{{- else }}
  # This branch always executes in dry-run
{{- end }}
```

Test with `--dry-run` to validate syntax, but always do a real dry-run against a test cluster to validate `lookup`-dependent logic.

### Generating Random Passwords — With Stable Identity

Generating random values in Helm templates has a subtle problem: `randAlphaNum` generates a new random value on every `helm upgrade`. Each upgrade regenerates the password, which breaks any applications using the old one.

The solution combines `lookup` with `helm.sh/resource-policy: keep`:

```yaml
# templates/secrets/generated-password.yaml
{{- $secretName := printf "%s-generated-secrets" (include "myapp.fullname" .) }}
{{- $existingSecret := lookup "v1" "Secret" .Release.Namespace $secretName }}
apiVersion: v1
kind: Secret
metadata:
  name: {{ $secretName }}
  annotations:
    # keep: Helm will NOT delete this Secret during helm uninstall
    # WHY: the generated password lives here; deleting it loses the password
    "helm.sh/resource-policy": keep
type: Opaque
data:
  {{- if $existingSecret }}
  # Secret exists — preserve the existing values, don't regenerate
  redis-password: {{ index $existingSecret.data "redis-password" }}
  jwt-secret: {{ index $existingSecret.data "jwt-secret" }}
  {{- else }}
  # First install — generate fresh random values
  redis-password: {{ randAlphaNum 32 | b64enc | quote }}
  jwt-secret: {{ randAlphaNum 64 | b64enc | quote }}
  {{- end }}
```

With this pattern:
- First install: passwords are generated randomly and stored
- Subsequent upgrades: existing passwords are preserved (looked up and reused)
- `helm uninstall`: the Secret is kept (not deleted) due to `helm.sh/resource-policy: keep`

### Conditional Resources Based on Kubernetes API Availability

Some Kubernetes resources only exist in certain API versions. Rather than failing on older clusters, check API availability first:

```yaml
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
{{- if .Capabilities.APIVersions.Has "autoscaling/v2" }}
apiVersion: autoscaling/v2
{{- else if .Capabilities.APIVersions.Has "autoscaling/v2beta2" }}
apiVersion: autoscaling/v2beta2
{{- else }}
apiVersion: autoscaling/v1
{{- end }}
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  # ...
{{- end }}
```

This chart deploys correctly on Kubernetes 1.20 (which has `autoscaling/v2beta2`) and 1.23+ (which has `autoscaling/v2`) without modification.

### NOTES.txt — Post-Install Instructions

`templates/NOTES.txt` is a special file: it's printed to the terminal after `helm install` or `helm upgrade` completes. It supports Go templates, so you can tailor the output to the actual deployment:

```
╔════════════════════════════════════════════════════════╗
║        {{ .Chart.Name | upper }} v{{ .Chart.AppVersion }} Deployed                  ║
╚════════════════════════════════════════════════════════╝

Release:   {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

{{- if .Values.ingress.enabled }}
Application URL:
  https://{{ (index .Values.ingress.hosts 0).host }}

{{- else }}
To access the application locally:
  kubectl port-forward svc/{{ include "myapp.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
  Then open: http://localhost:8080

{{- end }}

Useful commands:
  # Stream application logs
  kubectl logs -f -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }}

  # Watch pod status
  kubectl get pods -n {{ .Release.Namespace }} -l app.kubernetes.io/instance={{ .Release.Name }} -w

  # Run smoke tests
  helm test {{ .Release.Name }} -n {{ .Release.Namespace }}

{{- if .Values.monitoring.enabled }}

Monitoring:
  # Access Grafana (if port-forwarding)
  kubectl port-forward svc/{{ .Release.Name }}-grafana 3000:80 -n {{ .Release.Namespace }}
  Default credentials: admin / {{ .Values.monitoring.grafana.adminPassword | default "check-secret" }}
{{- end }}
```

---

## Chapter 6: Complete Real-World Example

### The Scenario

An e-commerce API with these components:
- Spring Boot application (the main service)
- PostgreSQL (orders, users, products — stateful)
- Redis (session cache, rate limiting — stateful)
- RabbitMQ (async order processing — stateful)
- Prometheus + Grafana (monitoring — optional)

Everything deployed from one chart, one command, across dev/staging/prod.

### Complete Chart.yaml

```yaml
apiVersion: v2
name: ecommerce-api
description: E-commerce API with PostgreSQL, Redis, and RabbitMQ
type: application
version: 3.1.0
appVersion: "2.4.1"

dependencies:
  - name: postgresql
    version: "12.5.6"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled

  - name: redis
    version: "17.11.6"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled

  - name: rabbitmq
    version: "12.0.4"
    repository: "https://charts.bitnami.com/bitnami"
    condition: rabbitmq.enabled

  - name: kube-prometheus-stack
    version: "48.x.x"
    repository: "https://prometheus-community.github.io/helm-charts"
    condition: monitoring.enabled
```

### Complete values.yaml

```yaml
# ─────────────────────────────────────────
# Application
# ─────────────────────────────────────────
replicaCount: 2

image:
  repository: yourusername/ecommerce-api
  tag: "2.4.1"
  pullPolicy: IfNotPresent

imagePullSecrets: []

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  name: ""

# ─────────────────────────────────────────
# Networking
# ─────────────────────────────────────────
service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: api.ecommerce.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

# ─────────────────────────────────────────
# Resources
# ─────────────────────────────────────────
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"

terminationGracePeriodSeconds: 60

# ─────────────────────────────────────────
# Health probes
# ─────────────────────────────────────────
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

# ─────────────────────────────────────────
# Application configuration
# ─────────────────────────────────────────
config:
  profile: "kubernetes"
  logLevel: "INFO"
  cacheType: "redis"

features:
  newCheckout: false
  darkMode: false
  betaAnalytics: false

# ─────────────────────────────────────────
# Autoscaling (disabled by default)
# ─────────────────────────────────────────
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 60

# ─────────────────────────────────────────
# PostgreSQL subchart
# ─────────────────────────────────────────
postgresql:
  enabled: true
  auth:
    username: apiuser
    database: ecommercedb
    existingSecret: db-credentials
    secretKeys:
      userPasswordKey: password
  primary:
    persistence:
      enabled: true
      size: 10Gi
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "1Gi"
        cpu: "500m"

# Used when postgresql.enabled=false
externalDatabase:
  host: ""
  port: 5432
  name: ecommercedb
  username: apiuser
  existingSecret: db-credentials

# ─────────────────────────────────────────
# Redis subchart
# ─────────────────────────────────────────
redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: redis-credentials
    existingSecretPasswordKey: password
  master:
    persistence:
      enabled: true
      size: 2Gi

externalRedis:
  host: ""
  port: 6379
  existingSecret: redis-credentials

# ─────────────────────────────────────────
# RabbitMQ subchart
# ─────────────────────────────────────────
rabbitmq:
  enabled: true
  auth:
    username: apiuser
    existingPasswordSecret: rabbitmq-credentials
  persistence:
    enabled: true
    size: 5Gi

externalRabbitmq:
  url: ""

# ─────────────────────────────────────────
# Monitoring
# ─────────────────────────────────────────
monitoring:
  enabled: false
```

### Multi-Environment Deploy Commands

```bash
# Add required repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Download dependencies
helm dependency update

# Validate
helm lint ./ecommerce-api-chart

# Preview rendered YAML
helm template ecommerce-dev ./ecommerce-api-chart \
  -f values.yaml \
  -f values-dev.yaml

# Deploy to dev
helm upgrade --install ecommerce-dev ./ecommerce-api-chart \
  -f values.yaml \
  -f values-dev.yaml \
  --namespace dev \
  --create-namespace \
  --wait \
  --timeout 10m

# Run smoke tests
helm test ecommerce-dev -n dev

# Deploy to staging
helm upgrade --install ecommerce-staging ./ecommerce-api-chart \
  -f values.yaml \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace \
  --wait \
  --timeout 10m

# Deploy to production (with atomic rollback)
helm upgrade --install ecommerce-prod ./ecommerce-api-chart \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace production \
  --create-namespace \
  --atomic \
  --wait \
  --timeout 15m
```

### `helm history` and `helm rollback`

Every `helm install` and `helm upgrade` creates a numbered revision. Helm keeps the full history, so you can inspect past states and roll back to any of them:

```bash
# See all revisions for a release
helm history ecommerce-prod -n production
# REVISION  UPDATED                  STATUS     CHART                   APP VERSION  DESCRIPTION
# 1         Mon Jan 1 09:00:00 2024  superseded ecommerce-api-3.0.0     2.3.0        Install complete
# 2         Tue Jan 2 14:00:00 2024  superseded ecommerce-api-3.1.0     2.4.0        Upgrade complete
# 3         Wed Jan 3 16:00:00 2024  deployed   ecommerce-api-3.1.0     2.4.1        Upgrade complete

# Get values used for a specific revision
helm get values ecommerce-prod --revision 2 -n production

# Get the full manifest for a specific revision
helm get manifest ecommerce-prod --revision 1 -n production

# Roll back to revision 2
helm rollback ecommerce-prod 2 -n production
# Rollback was a success! Happy Helming.

# Roll back to the immediately previous revision
helm rollback ecommerce-prod -n production
```

When you rollback, Helm creates a new revision (revision 4 in this case) that contains the configuration from revision 2. The old revision is not deleted — the history stays intact.

### The `helm diff` Plugin — Preview Changes Before Applying

`helm diff` is a third-party plugin that shows you exactly what will change in the cluster before you run `helm upgrade`. It's one of the most valuable tools in a production Helm workflow.

```bash
# Install the plugin (one-time)
helm plugin install https://github.com/databus23/helm-diff

# Preview what a values change will do to the cluster
helm diff upgrade ecommerce-prod ./ecommerce-api-chart \
  -f values.yaml \
  -f values-prod.yaml \
  --namespace production

# Sample output:
# default, ecommerce-prod-deployment, Deployment (apps) has changed:
#   ...
#   spec:
#     template:
#       spec:
#         containers:
#         - image: yourusername/ecommerce-api:2.4.0  ← old
#         + image: yourusername/ecommerce-api:2.4.1  ← new
#           name: ecommerce-api
#   ...
# default, ecommerce-prod-config, ConfigMap () has changed:
#   ...
#   data:
#   - LOG_LEVEL: INFO                 ← old
#   + LOG_LEVEL: WARN                 ← new
```

The diff shows you:
- Which resources change
- What the old value was and what the new value will be
- Which resources are added or deleted

This is invaluable for catching accidental changes. If you updated `values-prod.yaml` intending to change the image tag but the diff shows the memory limit also changed (perhaps from an edit you forgot about), you catch it before it hits production.

In Part 4, we'll integrate `helm diff` into the CI/CD pipeline as a pull request step — reviewers see the exact cluster changes as part of the code review.

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `nil pointer evaluating` during `helm template` | Accessing a nested value that doesn't exist in values.yaml | `helm template --debug` — shows the line number | Add a `default` or wrap in `{{- if .Values.x.y }}` check |
| Blank lines in rendered YAML | Missing `{{-` whitespace trim on conditional/loop blocks | `helm template \| cat -A` — shows whitespace | Add `-` to `{{` on lines that should not produce output |
| Wrong indentation — YAML parse error | Missing `nindent` after `include`, or wrong indent count | `helm template \| kubectl apply --dry-run=client -f -` | Add `\| nindent N` — count the spaces in the context |
| `selector is immutable` on upgrade | Added a label to `selectorLabels` that didn't exist before | `kubectl get deploy <n> -o yaml \| grep selector` | Labels in `selectorLabels` / `matchLabels` can NEVER change; delete and recreate the Deployment |
| Hook `Job already exists` on upgrade | Missing `before-hook-creation` delete policy | `kubectl get jobs -n <ns>` | Add `"helm.sh/hook-delete-policy": before-hook-creation` to the hook |
| Migration hook fails silently | Job backoff limit reached, pods deleted | `kubectl get pods -n <ns> --show-all` | Add `hook-failed` to delete policy so pods persist; check `kubectl logs` |
| Subchart values not applied | Wrong nesting key in values.yaml | `helm get values <release>` — inspect merged values | The key must exactly match the dependency `name` in Chart.yaml |
| `lookup` returns empty during dry-run | Expected — lookup doesn't work without live cluster | Run against a test cluster instead | Design `lookup`-dependent code to handle empty returns gracefully |
| `helm dependency update` fails | Repository not added | `helm repo list` | Run `helm repo add <name> <url>` then `helm repo update` |
| `helm diff` shows unexpected changes | Out-of-date local chart or values | Compare `helm get manifest` with your templates | Run `helm dependency update` to sync subchart versions |
| Random password regenerated on every upgrade | `lookup` for existing secret is missing or always fails | `helm template --debug` — check lookup output | Ensure `before-hook-creation` delete policy on generated secret; verify RBAC allows `get secrets` |

---

## Practice Exercises

**Exercise 1 — Whitespace debugging:**
Take this template snippet and predict the output with `.Values.debug = true` and `.Values.debug = false`. Then render it with `helm template` to verify your prediction:

```yaml
spec:
  template:
    spec:
      containers:
      - name: app
        {{ if .Values.debug }}
        args: ["--debug"]
        {{ end }}
        image: myapp:1.0.0
```

Now fix the whitespace so the output is clean YAML in both cases.

**Exercise 2 — Named template extraction:**
Take the deployment template from Part 1 or 2 and extract the following into named templates in `_helpers.tpl`:
- The complete env var block (all environment variables)
- The complete probe configuration (all three probes)
- The resource limits block

Verify with `helm template` that the rendered output is identical before and after extraction.

**Exercise 3 — Subchart integration:**
Add the Bitnami PostgreSQL subchart to your `hello-app-chart`. Configure it with a 1Gi persistent volume and an existing Secret for the password. Add a `externalDatabase` fallback configuration. Deploy to Minikube with `postgresql.enabled=true` and verify both the app pod and postgres pod are running. Then deploy again with `postgresql.enabled=false` and `externalDatabase.host=localhost` — verify the app deploys without the postgres pod (it will fail to connect, but it should deploy).

**Exercise 4 — Migration hook:**
Write a `pre-upgrade` hook for your chart that runs a Job printing `"Running database migration for version {{ .Chart.AppVersion }}"` to stdout. Run `helm upgrade` and verify: (1) the Job runs and completes before pods are updated, (2) the Job is deleted after success, (3) `helm history` shows a new revision. Then break the Job intentionally (set `command: ["exit", "1"]`) and observe what `--atomic` does.

**Exercise 5 — `helm diff` in practice:**
Install `helm diff`. Make three simultaneous changes to `values-prod.yaml`: change the image tag, increase the replica count, and add a new environment variable. Before running `helm upgrade`, run `helm diff upgrade` and verify all three changes appear in the diff output. Then make the upgrade and use `helm history` and `helm get manifest` to confirm what changed.
