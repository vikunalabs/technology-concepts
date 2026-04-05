## Part 8: Observability - Monitoring, Logging, and Tracing

### Prerequisites
- Completed Parts 1-7 (or equivalent experience)
- Kubernetes cluster (EKS, GKE, AKS, or Minikube)
- Basic understanding of metrics and logging concepts

### What You'll Learn
- ✅ Observability pillars (Metrics, Logs, Traces)
- ✅ Prometheus for metrics collection
- ✅ Grafana for visualization and dashboards
- ✅ Loki for log aggregation
- ✅ Tempo/Jaeger for distributed tracing
- ✅ Alerting with Alertmanager
- ✅ SLI/SLO implementation
- ✅ Kubernetes monitoring with kube-state-metrics

---

## Chapter 1: The Three Pillars of Observability

### Understanding Observability

**Analogy - Car Dashboard vs. Mechanic:**
- **Monitoring** = Dashboard (speed, fuel, temperature)
- **Logging** = Trip computer (detailed events)
- **Tracing** = GPS route tracking (path through system)

```
┌─────────────────────────────────────────────────────────────┐
│                      User Request                           │
│                   GET /api/orders/123                       │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  METRICS: Request count, duration, error rate               │
│  LOGS: "2024-01-15 10:30:45 INFO Processing order 123"      │
│  TRACES: API Gateway → Auth Service → Order Service → DB    │
└─────────────────────────────────────────────────────────────┘
```

### The Observability Stack

| Component | Tool | Purpose |
|-----------|------|---------|
| **Metrics** | Prometheus | Collect numerical data |
| **Visualization** | Grafana | Dashboards and graphs |
| **Logs** | Loki | Centralized logging |
| **Traces** | Tempo/Jaeger | Distributed tracing |
| **Alerting** | Alertmanager | Notifications |
| **K8s Metrics** | kube-state-metrics | Cluster health |

---

## Chapter 2: Prometheus - Metrics Collection

### What is Prometheus?

**Pull-based architecture:**
```
Prometheus ──scrape──▶  App (metrics endpoint)
     │                      :8080/metrics
     │
     └─────▶  Storage (TSDB)
              │
              └─────▶  Grafana (query)
```

### Installing Prometheus Stack

**Method 1: kube-prometheus-stack (Recommended)**

```bash
# Add Prometheus Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=LoadBalancer \
  --set grafana.service.type=LoadBalancer \
  --set grafana.adminPassword=prom-operator \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.resources.requests.memory=2Gi

# Check installation
kubectl get pods -n monitoring
# prometheus-kube-prometheus-stack-prometheus-0   2/2     Running
# prometheus-grafana-xxx                         1/1     Running
# prometheus-kube-state-metrics-xxx              1/1     Running
# prometheus-prometheus-node-exporter-xxx        1/1     Running

# Get access URLs
kubectl get svc -n monitoring
# prometheus-kube-prometheus-stack-prometheus   LoadBalancer   10.0.0.10    a1b2c3d4.elb.amazonaws.com   9090:32258/TCP
# prometheus-grafana                            LoadBalancer   10.0.0.11    a1b2c3d5.elb.amazonaws.com   80:30242/TCP
```

**Method 2: Minikube (Local Development)**

```bash
# Enable metrics-server
minikube addons enable metrics-server

# Install Prometheus with NodePort
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.service.type=NodePort \
  --set grafana.service.type=NodePort

# Get URLs
minikube service prometheus-grafana -n monitoring
minikube service prometheus-kube-prometheus-stack-prometheus -n monitoring
```

### Spring Boot Metrics Integration

**Add dependencies to `pom.xml`:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Configure `application.yml`:**
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active}
  endpoint:
    metrics:
      enabled: true
    prometheus:
      enabled: true
```

**Custom Metrics Example:**
```java
@RestController
public class OrderController {
    
    private final Counter orderCounter;
    private final Timer orderTimer;
    private final DistributionSummary orderSize;
    
    public OrderController(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.total")
            .description("Total number of orders")
            .tag("type", "ecommerce")
            .register(registry);
        
        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Time to process order")
            .register(registry);
        
        this.orderSize = DistributionSummary.builder("orders.size")
            .description("Order size in items")
            .baseUnit("items")
            .register(registry);
    }
    
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest request) {
        orderCounter.increment();
        orderSize.record(request.getItems().size());
        
        return orderTimer.record(() -> {
            // Process order
            return orderService.create(request);
        });
    }
}
```

### Prometheus ServiceMonitor

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
  - port: metrics
    path: /actuator/prometheus
    interval: 30s
    scrapeTimeout: 10s
  namespaceSelector:
    matchNames:
    - production
    - staging
```

**Annotate your service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
spec:
  selector:
    app: myapp
  ports:
  - name: metrics
    port: 8080
    targetPort: 8080
```

### Important Prometheus Metrics

```promql
# Common queries
# Request rate (per second)
rate(http_server_requests_seconds_count[5m])

# 95th percentile latency
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))

# Error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) / rate(http_server_requests_seconds_count[5m])

# Pod CPU usage
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)

# Memory usage
sum(container_memory_working_set_bytes{container!=""}) by (pod)

# JVM heap usage
jvm_memory_used_bytes{area="heap"}
```

---

## Chapter 3: Grafana - Visualization

### Accessing Grafana

```bash
# Get Grafana admin password
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode

# Port forward for local access
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Access: http://localhost:3000
# Username: admin
# Password: (from secret)
```

### Pre-built Dashboards

**Import Kubernetes Dashboard:**
```bash
# Dashboard ID: 315 (Kubernetes Cluster)
# Dashboard ID: 6417 (Kubernetes Pods)
# Dashboard ID: 15758 (Spring Boot 2.1+)
```

**Create Custom Dashboard:**

```json
{
  "dashboard": {
    "title": "Spring Boot Application Dashboard",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count{application=\"myapp\"}[5m])",
            "legendFormat": "{{method}} {{uri}}"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_server_requests_seconds_count{status=~\"5..\"}[5m]) / rate(http_server_requests_seconds_count[5m]) * 100",
            "legendFormat": "Error %"
          }
        ],
        "type": "stat",
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                { "value": 0, "color": "green" },
                { "value": 1, "color": "yellow" },
                { "value": 5, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "title": "P99 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri))",
            "legendFormat": "{{uri}}"
          }
        ],
        "type": "graph",
        "fieldConfig": {
          "defaults": {
            "unit": "s"
          }
        }
      },
      {
        "title": "JVM Memory",
        "targets": [
          {
            "expr": "jvm_memory_used_bytes{area=\"heap\"}",
            "legendFormat": "Used"
          },
          {
            "expr": "jvm_memory_max_bytes{area=\"heap\"}",
            "legendFormat": "Max"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

### Alert Rules in Grafana

```yaml
# alert-rule.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
spec:
  groups:
  - name: myapp
    rules:
    - alert: HighErrorRate
      expr: |
        rate(http_server_requests_seconds_count{status=~"5.."}[5m]) 
        / 
        rate(http_server_requests_seconds_count[5m]) 
        > 0.05
      for: 5m
      labels:
        severity: critical
        team: backend
      annotations:
        summary: "High error rate on {{ $labels.uri }}"
        description: "Error rate is {{ $value | humanizePercentage }} on {{ $labels.pod }}"
    
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99, 
          sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
        ) > 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High latency on {{ $labels.uri }}"
        description: "P99 latency is {{ $value }}s"
    
    - alert: PodCrashLooping
      expr: |
        kube_pod_container_status_restarts_total > 5
      for: 15m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Container {{ $labels.container }} has restarted {{ $value }} times"
    
    - alert: CPUThrottling
      expr: |
        sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod)
        /
        sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)
        > 0.5
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "CPU throttling detected on {{ $labels.pod }}"
```

---

## Chapter 4: Loki - Log Aggregation

### Installing Loki

```bash
# Install Loki stack
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=true \
  --set prometheus.enabled=true \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi

# Verify
kubectl get pods -n monitoring | grep loki
# loki-0                             1/1     Running
# loki-promtail-xxx                  1/1     Running
```

### LogQL Queries

```logql
# Basic queries
{namespace="production", app="myapp"}

# Filter by log level
{namespace="production"} |= "ERROR"
{namespace="production"} |~ ".*Exception.*"

# Regex matching
{namespace="production"} |~ "HTTP/\\d\\.\\d\" 5\\d\\d"

# Time-based
{namespace="production"} |= "OutOfMemoryError" | json | duration > 5s

# Rate of errors
rate({namespace="production"} |= "ERROR"[5m])

# Count by pod
sum by (pod) (count_over_time({namespace="production"} |= "ERROR"[1h]))
```

### Log Aggregation Example

```yaml
# promtail-config.yaml - Custom log scraping
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
  namespace: monitoring
data:
  promtail.yaml: |
    scrape_configs:
    - job_name: kubernetes-pods
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: node
      
      # Parse JSON logs
      - action: replace
        source_labels: [__meta_kubernetes_pod_annotation_json_parser]
        target_label: __path_json
      
      pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            trace_id: trace_id
      - labels:
          level: level
          trace_id: trace_id
```

### Viewing Logs in Grafana

```bash
# Configure Loki data source in Grafana
# 1. Go to Configuration → Data Sources
# 2. Add Loki
# 3. URL: http://loki:3100
# 4. Save & Test

# Example LogQL queries in Grafana:
# Show errors in last hour
{namespace="production"} |= "ERROR" | json | level="ERROR"

# Show logs with trace ID
{namespace="production"} | json | trace_id="abc123"

# Show slow requests (>1s)
{namespace="production"} | json | duration > 1

# Count errors per pod
sum by (pod) (count_over_time({namespace="production"} |= "ERROR"[1h]))
```

---

## Chapter 5: Distributed Tracing with Tempo/Jaeger

### Why Distributed Tracing?

**Problem:** Request spans multiple services, hard to debug

```
User → API Gateway → Auth Service → Order Service → Payment Service → DB
                    ↓                ↓
                 Redis Cache      Inventory Service
```

**Solution:** Trace shows complete path and timing

```
Trace ID: abc123
├── API Gateway: 10ms
├── Auth Service: 5ms
├── Order Service: 45ms
│   ├── Redis: 2ms
│   └── DB Query: 38ms
├── Payment Service: 120ms
└── Inventory Service: 15ms
Total: 195ms
```

### Installing Tempo

```bash
# Install Tempo (Grafana's tracing backend)
helm install tempo grafana/tempo \
  --namespace monitoring \
  --set tempo.query.frontend.service.type=LoadBalancer \
  --set tempo.storage.trace.backend=s3 \
  --set tempo.storage.trace.s3.bucket=tempo-traces \
  --set tempo.storage.trace.s3.endpoint=s3.amazonaws.com

# Configure Grafana to use Tempo
# Data Sources → Add Tempo
# URL: http://tempo-query-frontend:16686
```

### Installing Jaeger (Alternative)

```bash
# Install Jaeger
helm install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --set allInOne.enabled=true \
  --set allInOne.service.type=LoadBalancer \
  --set storage.type=elasticsearch \
  --set storage.elasticsearch.host=elasticsearch-master \
  --set storage.elasticsearch.port=9200

# Access Jaeger UI
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
# http://localhost:16686
```

### Spring Boot with OpenTelemetry

**Add dependencies to `pom.xml`:**
```xml
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-sdk</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-spring-boot-starter</artifactId>
    <version>1.32.0-alpha</version>
</dependency>
```

**Configure `application.yml`:**
```yaml
opentelemetry:
  traces:
    exporter: otlp
  exporters:
    otlp:
      endpoint: http://tempo:4317  # OTLP gRPC
  service:
    name: ${spring.application.name}
  resource:
    attributes:
      service.version: ${app.version}
      environment: ${spring.profiles.active}
```

**Manual Instrumentation:**
```java
@Service
public class OrderService {
    
    private final Tracer tracer;
    
    public OrderService(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer(OrderService.class.getName());
    }
    
    @WithSpan  // Automatic span creation
    public Order createOrder(OrderRequest request) {
        // Add custom attributes
        Span.current().setAttribute("order.id", request.getId());
        Span.current().setAttribute("order.amount", request.getAmount());
        
        // Create nested spans
        Span validationSpan = tracer.spanBuilder("validateOrder")
            .setParent(Context.current())
            .startSpan();
        
        try (Scope scope = validationSpan.makeCurrent()) {
            validateOrder(request);
        } catch (Exception e) {
            validationSpan.recordException(e);
            throw e;
        } finally {
            validationSpan.end();
        }
        
        return saveOrder(request);
    }
    
    @WithSpan
    private void validateOrder(OrderRequest request) {
        // Validation logic
    }
}
```

### Trace Propagation

```java
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id, 
                          @RequestHeader(value = "traceparent", required = false) String traceParent) {
        // Extract trace context from headers
        TextMapGetter<HttpServletRequest> getter = new TextMapGetter<>() {
            @Override
            public Iterable<String> keys(HttpServletRequest request) {
                return Collections.list(request.getHeaderNames());
            }
            
            @Override
            public String get(HttpServletRequest request, String key) {
                return request.getHeader(key);
            }
        };
        
        Context context = OpenTelemetry.getGlobalPropagators()
            .getTextMapPropagator()
            .extract(Context.current(), request, getter);
        
        // Continue trace
        try (Scope scope = context.makeCurrent()) {
            return orderService.getOrder(id);
        }
    }
}
```

### Jaeger Queries

```bash
# Find traces by service
service=myapp

# Find traces with errors
service=myapp error=true

# Find slow traces
service=myapp duration>1s

# Find traces with specific tag
service=myapp http.status_code=500

# Find traces by operation
operation=POST /api/orders

# Combine filters
service=myapp AND operation=GET /api/orders AND duration>500ms
```

---

## Chapter 6: Alerting with Alertmanager

### Configuring Alertmanager

```yaml
# alertmanager-config.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      slack_api_url: https://hooks.slack.com/services/XXX/YYY/ZZZ
      pagerduty_url: https://events.pagerduty.com/v2/enqueue
    
    route:
      group_by: ['alertname', 'cluster']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: 'slack-notifications'
      
      routes:
      - match:
          severity: critical
        receiver: pagerduty-critical
        continue: true
      
      - match:
          severity: warning
        receiver: slack-notifications
    
    receivers:
    - name: slack-notifications
      slack_configs:
      - channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: |-
          *Alert:* {{ .CommonAnnotations.summary }}
          *Description:* {{ .CommonAnnotations.description }}
          *Severity:* {{ .CommonLabels.severity }}
          *Time:* {{ .StartsAt }}
    
    - name: pagerduty-critical
      pagerduty_configs:
      - service_key: <PAGERDUTY_KEY>
        description: '{{ .CommonAnnotations.summary }}'
        
    - name: email-alerts
      email_configs:
      - to: 'oncall@mycompany.com'
        from: 'alerts@mycompany.com'
        smarthost: 'smtp.gmail.com:587'
        auth_username: 'alerts@mycompany.com'
        auth_password: 'password'

---
kubectl apply -f alertmanager-config.yaml
kubectl rollout restart -n monitoring deployment/alertmanager
```

### Slack Integration

```yaml
# Create Slack webhook
# 1. Go to https://api.slack.com/apps
# 2. Create app → Incoming Webhooks
# 3. Add to channel → Copy webhook URL

# Create secret
kubectl create secret generic slack-webhook \
  --from-literal=url=https://hooks.slack.com/services/XXX/YYY/ZZZ \
  -n monitoring

# Update Alertmanager config
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-config
stringData:
  alertmanager.yaml: |
    receivers:
    - name: slack
      slack_configs:
      - api_url: ${SLACK_WEBHOOK_URL}
        channel: '#alerts'
        title: '🚨 {{ .GroupLabels.alertname }}'
        text: |-
          *Severity:* {{ .CommonLabels.severity }}
          *Service:* {{ .CommonLabels.service }}
          *Description:* {{ .CommonAnnotations.description }}
          *Graph:* <{{ .GeneratorURL }}|View in Prometheus>
```

### PagerDuty Integration

```yaml
# Create PagerDuty integration
# 1. Services → Service Directory
# 2. Add Service → Integration → Events API v2
# 3. Copy Integration Key

receivers:
- name: pagerduty
  pagerduty_configs:
  - service_key: <PAGERDUTY_KEY>
    severity: critical
    description: '{{ .CommonAnnotations.summary }}'
    details:
      alert: '{{ .CommonLabels.alertname }}'
      pod: '{{ .CommonLabels.pod }}'
      value: '{{ .CommonAnnotations.value }}'
```

---

## Chapter 7: SLI/SLO Implementation

### Defining SLOs

```yaml
# slo-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: slo-config
data:
  slo.yaml: |
    services:
      - name: order-api
        slos:
          - name: availability
            target: 99.9
            window: 30d
            query: |
              sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
              /
              sum(rate(http_server_requests_seconds_count[5m]))
          
          - name: latency
            target: 99
            window: 30d
            query: |
              histogram_quantile(0.99, 
                sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
              )
            threshold: 0.5  # 500ms
          
          - name: error-budget
            target: 95
            window: 30d
            query: |
              1 - (
                sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
                /
                sum(rate(http_server_requests_seconds_count[5m]))
              )
```

### Error Budget Alerts

```yaml
# error-budget-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: slo-alerts
spec:
  groups:
  - name: slo
    rules:
    - alert: ErrorBudgetBurnRate
      expr: |
        (
          (1 - (
            sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
            /
            sum(rate(http_server_requests_seconds_count[5m]))
          ))
          /
          (1 - 0.999)
        ) > 14.4
      labels:
        severity: critical
      annotations:
        summary: "Error budget burn rate too high"
        description: "Would exhaust 30d error budget in 2 hours"
    
    - alert: SLOMissed
      expr: |
        (
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[30d]))
          /
          sum(rate(http_server_requests_seconds_count[30d]))
        ) > 0.001  # 0.1% error rate
      labels:
        severity: high
      annotations:
        summary: "SLO being violated"
        description: "Error rate over last 30 days: {{ $value | humanizePercentage }}"
```

---

## Chapter 8: Complete Observability Stack

### Production Configuration

**`values-observability.yaml`:**
```yaml
# Complete observability stack values
prometheus:
  enabled: true
  prometheusSpec:
    retention: 30d
    retentionSize: 50GB
    resources:
      requests:
        memory: 4Gi
        cpu: 2
      limits:
        memory: 8Gi
        cpu: 4
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi
    additionalScrapeConfigs:
      - job_name: 'spring-boot'
        kubernetes_sd_configs:
        - role: pod
        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true

grafana:
  enabled: true
  adminPassword: ${GRAFANA_PASSWORD}
  ingress:
    enabled: true
    hosts:
      - grafana.mycompany.com
  dashboards:
    default:
      spring-boot:
        url: https://raw.githubusercontent.com/grafana/jsonnet-libs/master/grafonnet/spring-boot-dashboard.json
      kubernetes:
        url: https://raw.githubusercontent.com/kubernetes-monitoring/kubernetes-mixin/master/dashboards/kubernetes.json
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
      - name: Prometheus
        type: prometheus
        url: http://prometheus-server:9090
      - name: Loki
        type: loki
        url: http://loki:3100
      - name: Tempo
        type: tempo
        url: http://tempo-query-frontend:16686

loki:
  enabled: true
  persistence:
    enabled: true
    size: 100Gi
  limits_config:
    ingestion_rate_mb: 10
    ingestion_burst_size_mb: 20

tempo:
  enabled: true
  storage:
    trace:
      backend: s3
      s3:
        bucket: tempo-traces
        endpoint: s3.amazonaws.com
        region: us-east-1
  ingester:
    max_block_duration: 10m

alertmanager:
  enabled: true
  config:
    global:
      slack_api_url: ${SLACK_WEBHOOK}
    route:
      group_by: ['alertname']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: slack
    receivers:
    - name: slack
      slack_configs:
      - channel: '#alerts'
```

### Deploy Complete Stack

```bash
# 1. Create monitoring namespace
kubectl create namespace monitoring

# 2. Create secrets
kubectl create secret generic grafana-admin \
  --from-literal=admin-password=$(openssl rand -base64 32) \
  -n monitoring

kubectl create secret generic slack-webhook \
  --from-literal=url=https://hooks.slack.com/services/XXX \
  -n monitoring

# 3. Deploy observability stack
helm upgrade --install observability prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values-observability.yaml

# 4. Deploy Loki
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  -f loki-values.yaml

# 5. Deploy Tempo
helm upgrade --install tempo grafana/tempo \
  --namespace monitoring \
  -f tempo-values.yaml

# 6. Annotate your app for scraping
kubectl annotate service myapp-service \
  prometheus.io/scrape="true" \
  prometheus.io/port="8080" \
  prometheus.io/path="/actuator/prometheus"

# 7. Access dashboards
echo "Grafana: http://$(kubectl get svc -n monitoring prometheus-grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
echo "Prometheus: http://$(kubectl get svc -n monitoring prometheus-kube-prometheus-stack-prometheus -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'):9090"
echo "Jaeger: http://$(kubectl get svc -n monitoring jaeger-query -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'):16686"
```

### Dashboard Examples

**Grafana Dashboard Variables:**
```json
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "query": "label_values(kube_namespace_labels)",
        "refresh": 1
      },
      {
        "name": "pod",
        "type": "query",
        "query": "label_values(container_memory_working_set_bytes{namespace=\"$namespace\"}, pod)",
        "refresh": 1
      },
      {
        "name": "interval",
        "type": "interval",
        "auto": true,
        "auto_min": "30s",
        "options": ["30s", "1m", "5m", "10m", "30m", "1h"]
      }
    ]
  }
}
```

---

## Summary: Observability Checklist

| Component | Tool | Status |
|-----------|------|--------|
| **Metrics Collection** | Prometheus | ✅ |
| **Metrics Storage** | Prometheus TSDB | ✅ |
| **Visualization** | Grafana | ✅ |
| **Log Aggregation** | Loki + Promtail | ✅ |
| **Distributed Tracing** | Tempo/Jaeger | ✅ |
| **Alerting** | Alertmanager | ✅ |
| **K8s Metrics** | kube-state-metrics | ✅ |
| **Node Metrics** | node-exporter | ✅ |
| **Service Monitors** | Prometheus Operator | ✅ |
| **SLI/SLO** | Custom metrics | ✅ |

## Common Observability Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **Metrics missing** | No data in Grafana | Check ServiceMonitor, annotations |
| **High cardinality** | Prometheus OOM | Use aggregations, drop high-card labels |
| **Logs not showing** | No logs in Loki | Check Promtail config, label selectors |
| **Traces incomplete** | Missing spans | Check propagation, OTLP endpoint |
| **Alerts not firing** | Silence | Check Alertmanager config, routing |
| **Dashboard slow** | Slow queries | Reduce time range, add aggregations |

## Practice Exercises

### Exercise 1: Instrument Spring Boot
Add Prometheus metrics to your app and create Grafana dashboard.

### Exercise 2: Centralized Logging
Configure Loki to collect logs from all pods and create log alert for errors.

### Exercise 3: Distributed Tracing
Add OpenTelemetry to two services and trace a request across them.

### Exercise 4: Alert Configuration
Create alert for high error rate and test with Slack notification.

### Exercise 5: SLO Implementation
Define SLO for your service and track error budget.

## Next Steps

After mastering observability, you're ready for:
- **Part 9: Scaling & Resilience** - HPA, VPA, chaos engineering
- **Part 10: Production Playbook** - Complete deployment guide

---