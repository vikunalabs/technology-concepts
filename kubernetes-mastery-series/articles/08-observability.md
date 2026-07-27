# Part 8: Observability — Monitoring, Logging, and Tracing

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Advanced
> **Prerequisites:** Completed Parts 1–7, or comfortable with Kubernetes workloads, Helm, and Spring Boot
> **Time to complete:** 5–6 hours
> **What you'll learn:** The three observability pillars (metrics, logs, traces), Prometheus with PromQL, Grafana dashboards and alerts, Alertmanager routing, Loki log aggregation, distributed tracing with Tempo and OpenTelemetry, SLI/SLO implementation, and a complete production observability stack

---

## What This Part Covers

Running an application is only half the job. Knowing what it's doing — whether it's healthy, why it's slow, which service caused that error at 2 AM — is the other half. Without observability, you're operating blind. You find out about problems when users complain, and you diagnose them by guessing.

This part builds the complete observability stack: Prometheus collects metrics so you know the system's health at a glance, Loki aggregates logs so you can investigate what happened, and Tempo traces requests across services so you know exactly where time was spent. These three pillars work together — a Prometheus alert fires, you look at the Loki logs for that time window, you open the distributed trace to find the root cause. That workflow is what this part teaches end to end.

---

## Chapter 1: The Three Pillars of Observability

### You Can't Fix What You Can't See

A production incident in an unobserved system goes like this: a user emails support saying the checkout is slow. You SSH into a server. You look at logs. You grep for errors. You notice high CPU in `top`. You guess it might be the database. You look at the database server. You make a change. You hope it helps.

This is not a system — it is detective work under pressure, and it scales to exactly one engineer who knows exactly one system they've been running for years.

Observability replaces guesswork with data. With a properly instrumented system, the same incident goes: Prometheus fires an alert (P99 latency > 500ms). You open the Grafana dashboard — checkout service error rate is elevated. You open the Loki logs for checkout-service in the last 15 minutes — there are connection timeout errors to the payment-service. You click the trace ID in the log line — the distributed trace shows the payment-service taking 4 seconds on a database query. You look at the slow query log. Root cause identified in 3 minutes.

### The Three Pillars — What Each One Answers

**Metrics** answer: *Is the system healthy? How fast is it? How much of it is there?*

Metrics are numbers that change over time: request rate, error rate, latency percentiles, JVM heap size, CPU usage, number of active database connections. They are cheap to collect (one number per measurement), cheap to store (time-series databases compress well), and fast to query. They cannot tell you *why* something is wrong — they only tell you *that* something is wrong, and approximately *what* is wrong.

**Logs** answer: *What actually happened? What was the exact error?*

Logs are timestamped records of events: "User 123 added item 456 to cart", "SQLException: connection refused", "Payment gateway returned 402". They provide the narrative. When metrics tell you error rate is elevated, logs tell you which users are affected, what the exact error is, and which code path produced it. Logs are expensive to store (every event is a full text record) and slower to query at scale.

**Traces** answer: *Which service caused the slowdown? Where did the time go?*

A trace follows a single request as it travels through multiple services — from the browser, through the API gateway, to the order service, to the inventory service, to the database. Each step is a span with a start time, duration, and metadata. When logs tell you there's a timeout, traces tell you which service is timing out and how much time was spent at every hop.

### The Car Dashboard Analogy

Think of your car's instrumentation panel:

- **Dashboard gauges** (speedometer, fuel, temperature) = **Metrics**: they give you continuous numbers at a glance. You know immediately if your engine temperature is in the red zone without reading any text.
- **Trip computer** (distance, fuel consumption history, "turn left in 200m") = **Logs**: a record of what happened and where you are in a journey. You check it to understand a sequence of events.
- **GPS route** (the full path from origin to destination, each road segment timed) = **Traces**: shows you the complete journey broken into segments, so you know exactly where the traffic jam was, not just that you arrived late.

None of these alone is sufficient. Metrics without logs tell you something is wrong but not what. Logs without metrics mean you're searching through millions of lines without knowing which ones matter. Traces without metrics tell you about individual requests but miss patterns across thousands.

### The Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Grafana (UI)                                  │
│    Dashboards · Alert Rules · Log Explorer · Trace Viewer           │
└──────────┬──────────────────┬────────────────────┬──────────────────┘
           │                  │                    │
           ▼                  ▼                    ▼
    ┌─────────────┐   ┌──────────────┐   ┌──────────────────┐
    │  Prometheus │   │     Loki     │   │  Tempo / Jaeger  │
    │  (metrics)  │   │   (logs)     │   │   (traces)       │
    └──────┬──────┘   └──────┬───────┘   └────────┬─────────┘
           │                 │                    │
           ▼                 ▼                    ▼
    ┌──────────────────────────────────────────────────────────┐
    │                  Your Application                        │
    │  /actuator/prometheus  stdout logs  OTLP trace exporter  │
    └──────────────────────────────────────────────────────────┘
           │
           ▼
    ┌─────────────────┐
    │  Alertmanager   │
    │  Slack/PagerDuty│
    └─────────────────┘
```

Everything in this part is wired through Grafana as the single pane of glass. You open Grafana for metrics dashboards, log exploration, and trace viewing. One tool, three pillars.

---

## Chapter 2: Prometheus — Metrics Collection

### Why Pull, Not Push

Prometheus scrapes metrics from your application on a schedule rather than having your application push metrics to Prometheus. This seems counter-intuitive — why not have the app send data when it has it?

The pull model has three significant advantages. First, if an application stops responding to scrapes, Prometheus immediately knows it's down — the scrape fails. With a push model, silence looks identical to the app being healthy and quiet. Second, Prometheus controls the scrape interval, so it cannot be overwhelmed by an application pushing metrics at an uncontrolled rate. Third, the scrape endpoint (`/actuator/prometheus`) is a standard HTTP endpoint you can inspect with `curl` at any time for debugging — no special tooling needed.

### What Prometheus Stores

Prometheus stores **time series** — streams of timestamped values identified by a metric name and a set of key-value labels:

```
http_server_requests_seconds_count{
  method="GET",
  uri="/api/orders",
  status="200",
  namespace="production",
  pod="orders-service-7d9b4f-xj8mk"
} 1234  @timestamp
```

The metric name (`http_server_requests_seconds_count`) combined with the label set `{method="GET", uri="...", status="200", ...}` forms a unique time series. Prometheus stores all samples for this series in its TSDB (Time Series Database), compressed efficiently on disk.

Labels are what make Prometheus powerful — you can filter, aggregate, and slice metrics by any label combination. They are also what makes it dangerous, which we'll cover in the cardinality section below.

### Installing kube-prometheus-stack

The `kube-prometheus-stack` Helm chart installs Prometheus, Grafana, Alertmanager, node-exporter (hardware metrics from every node), kube-state-metrics (Kubernetes resource metrics), and a full set of pre-built dashboards and alert rules — everything you need to start observing a Kubernetes cluster immediately.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=changeme-in-production \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=fast-ssd \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --wait

kubectl get pods -n monitoring
# NAME                                                    READY   STATUS
# alertmanager-monitoring-kube-prometheus-alertmanager-0  2/2     Running
# monitoring-grafana-5d9c7f6b8-xj8mk                     3/3     Running
# monitoring-kube-prometheus-operator-7d9b4f9c6-2xmpq    1/1     Running
# monitoring-kube-state-metrics-6b8c7d9f7-mklpq          1/1     Running
# monitoring-prometheus-node-exporter-abcd1              1/1     Running  (one per node)
# prometheus-monitoring-kube-prometheus-prometheus-0      2/2     Running

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# http://localhost:3000   admin / changeme-in-production
```

### Spring Boot + Prometheus Integration

Add the dependency to `pom.xml`:

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Or to `build.gradle`:

```groovy
implementation 'io.micrometer:micrometer-registry-prometheus'
```

Configure in `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    # Add common tags to every metric this app exports.
    # These become labels on every time series — enabling filtering by app and environment.
    tags:
      application: ${spring.application.name}
      environment: ${SPRING_PROFILES_ACTIVE:local}
  endpoint:
    prometheus:
      enabled: true
```

Verify the endpoint:
```bash
curl http://localhost:8080/actuator/prometheus | head -30
# # HELP jvm_memory_used_bytes The amount of used memory
# # TYPE jvm_memory_used_bytes gauge
# jvm_memory_used_bytes{application="myapp",area="heap",environment="prod",...} 1.45e+08
# # HELP http_server_requests_seconds  
# # TYPE http_server_requests_seconds summary
# http_server_requests_seconds_count{application="myapp",method="GET",status="200",...} 1234
```

Hundreds of metrics are exposed automatically: JVM memory and GC, HTTP request rates and latencies, connection pool sizes, thread pool stats, and more.

### ServiceMonitor — Telling Prometheus What to Scrape

The `kube-prometheus-stack` uses the Prometheus Operator pattern: instead of editing Prometheus config files, you create `ServiceMonitor` Kubernetes objects and the operator automatically updates Prometheus's scrape configuration.

```yaml
# servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp-monitor
  namespace: monitoring          # ServiceMonitor lives in monitoring namespace
  labels:
    # This label must match what kube-prometheus-stack uses to discover ServiceMonitors.
    # Check with: helm get values monitoring -n monitoring | grep serviceMonitorSelector
    release: monitoring
spec:
  selector:
    matchLabels:
      app: myapp                 # Selects Services with this label
  namespaceSelector:
    matchNames:
    - production                 # In the production namespace
  endpoints:
  - port: http                   # Named port in the Service spec
    path: /actuator/prometheus
    interval: 30s                # Scrape every 30 seconds
    scrapeTimeout: 10s
```

Alternatively, annotate your Service for automatic discovery without a ServiceMonitor:

```yaml
# service.yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"
```

After applying, verify Prometheus is scraping your app:
```bash
# Access Prometheus UI
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
# http://localhost:9090/targets — look for your app in the target list
```

### Custom Metrics in Spring Boot

Micrometer provides Counter, Timer, Gauge, and DistributionSummary for custom instrumentation:

```java
package com.example.myapp;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Service;
import java.util.concurrent.atomic.AtomicInteger;

@Service
public class OrderService {

    private final Counter orderCreatedCounter;
    private final Counter orderFailedCounter;
    private final Timer orderProcessingTimer;
    private final AtomicInteger activeOrders;

    public OrderService(MeterRegistry registry) {
        // Counter: monotonically increasing count
        // Tags become Prometheus labels: orders_created_total{status="success"}
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Total number of orders created")
            .tag("status", "success")
            .register(registry);

        this.orderFailedCounter = Counter.builder("orders.created")
            .description("Total number of orders created")
            .tag("status", "failed")
            .register(registry);

        // Timer: measures both count and duration distribution
        // Produces: orders_processing_seconds_count, _sum, _bucket (histogram)
        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time to process an order end-to-end")
            .publishPercentiles(0.5, 0.95, 0.99)   // Pre-compute percentiles
            .publishPercentileHistogram()            // Enables histogram_quantile() in PromQL
            .register(registry);

        // Gauge: current value (not cumulative)
        // Reports the current value of activeOrders.get() on every scrape
        this.activeOrders = new AtomicInteger(0);
        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Number of orders currently being processed")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        activeOrders.incrementAndGet();
        try {
            return orderProcessingTimer.record(() -> {
                Order order = processOrder(request);
                orderCreatedCounter.increment();
                return order;
            });
        } catch (Exception e) {
            orderFailedCounter.increment();
            throw e;
        } finally {
            activeOrders.decrementAndGet();
        }
    }
}
```

These produce Prometheus metrics:
```
orders_created_total{status="success"} 1542
orders_created_total{status="failed"} 23
orders_processing_duration_seconds_count 1565
orders_processing_duration_seconds_sum 782.3
orders_active 12
```

### PromQL — The Query Language

PromQL is how you interrogate Prometheus data. The queries in this section are the ones you'll use most often.

**Instant vectors vs range vectors:**
```promql
# Instant vector: the current value of a metric
http_server_requests_seconds_count

# Range vector: values over a time window — used as input to functions
http_server_requests_seconds_count[5m]
# Returns all samples from the last 5 minutes for each time series
```

**`rate()` and `irate()` — the most important functions:**

`rate()` calculates the per-second average rate of increase over the range window. It smooths out spikes and is appropriate for alerting and trend dashboards:

```promql
# Requests per second (averaged over 5 minutes)
rate(http_server_requests_seconds_count[5m])
```

`irate()` calculates the per-second rate using only the last two data points. It reflects instantaneous spikes but is noisy. Use it for dashboards showing "right now" activity:

```promql
# Requests per second right now (instantaneous)
irate(http_server_requests_seconds_count[2m])
```

**Use `rate()` for alerting, `irate()` for "current activity" dashboards.** Alerting on `irate()` causes false positives because a single momentary spike triggers the alert even if the 5-minute average is fine.

**Essential PromQL queries:**

```promql
# ── Request Rates ─────────────────────────────────────────────────────────────

# Total requests per second across all endpoints
rate(http_server_requests_seconds_count{namespace="production"}[5m])

# Requests per second broken down by HTTP status code
sum by (status) (
  rate(http_server_requests_seconds_count{namespace="production"}[5m])
)

# ── Error Rate ────────────────────────────────────────────────────────────────

# Percentage of requests that returned 5xx errors
sum(rate(http_server_requests_seconds_count{status=~"5..",namespace="production"}[5m]))
/
sum(rate(http_server_requests_seconds_count{namespace="production"}[5m]))
* 100

# ── Latency Percentiles ───────────────────────────────────────────────────────

# P99 latency across all endpoints (requires publishPercentileHistogram=true)
histogram_quantile(0.99,
  sum by (le) (
    rate(http_server_requests_seconds_bucket{namespace="production"}[5m])
  )
)

# P95 and P99 per endpoint
histogram_quantile(0.95,
  sum by (le, uri) (
    rate(http_server_requests_seconds_bucket{namespace="production"}[5m])
  )
)

# ── JVM Health ────────────────────────────────────────────────────────────────

# Heap usage as percentage of max
jvm_memory_used_bytes{area="heap",namespace="production"}
/
jvm_memory_max_bytes{area="heap",namespace="production"}
* 100

# GC pause time rate (high values = GC pressure)
rate(jvm_gc_pause_seconds_sum{namespace="production"}[5m])

# ── Pod Resources ─────────────────────────────────────────────────────────────

# CPU usage per pod (cores)
sum by (pod) (
  rate(container_cpu_usage_seconds_total{namespace="production",container!=""}[5m])
)

# Memory usage as percentage of limit
sum by (pod) (container_memory_working_set_bytes{namespace="production",container!=""})
/
sum by (pod) (container_spec_memory_limit_bytes{namespace="production",container!=""})
* 100
```

### The HIGH CARDINALITY Warning

Cardinality is the number of unique time series Prometheus tracks. Every unique combination of metric name and label values is a separate time series. High cardinality is the single most common cause of Prometheus running out of memory.

Consider this: you add a `user_id` label to your HTTP request counter.

```java
// ❌ NEVER do this — catastrophic cardinality
Counter.builder("http.requests")
    .tag("user_id", userId)    // 10,000 users = 10,000 time series per endpoint
    .register(registry);
```

With 10,000 users, 50 endpoints, and Prometheus scraping every 30 seconds:
- `10,000 users × 50 endpoints = 500,000 time series`
- Each time series gets a new sample every 30 seconds
- That is 1,000,000 new samples per minute stored in memory
- Prometheus runs out of memory within hours and crashes

**The rule: metric labels must have low cardinality — a small, bounded set of possible values.**

| ✅ Low cardinality — safe | ❌ High cardinality — dangerous |
|--------------------------|-------------------------------|
| `status="200"` (handful of values) | `user_id="12345"` (millions of users) |
| `method="GET"` (7 HTTP methods) | `request_id="uuid-..."` (unique per request) |
| `endpoint="/api/orders"` (dozens of routes) | `customer_email="..."` (unbounded) |
| `environment="prod"` (3 environments) | `trace_id="..."` (unique per trace) |
| `region="us-east-1"` (dozens of regions) | `session_id="..."` (unique per session) |

If you need to analyse per-user behaviour, that data belongs in logs or a dedicated analytics database — not in Prometheus metrics.

### Prometheus Adapter — Custom Metrics for HPA

The Prometheus Adapter bridges Prometheus metrics into the Kubernetes custom metrics API, enabling HPA to scale on application-level metrics like requests per second rather than just CPU.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://monitoring-kube-prometheus-prometheus.monitoring.svc \
  --set prometheus.port=9090
```

Configure a custom metric mapping:

```yaml
# prometheus-adapter-config.yaml (passed as Helm values)
rules:
  custom:
  - seriesQuery: 'http_server_requests_seconds_count{namespace!="",pod!=""}'
    resources:
      overrides:
        namespace:
          resource: namespace
        pod:
          resource: pod
    name:
      matches: "^(.*)_seconds_count"
      as: "${1}_per_second"     # → http_server_requests_per_second
    # metricsQuery uses the Prometheus Adapter's own Go template syntax — <<...>> —
    # which is distinct from PromQL's {label="value"} syntax. The adapter expands
    # these placeholders before sending the query to Prometheus:
    #   <<.Series>>        → the metric name from seriesQuery
    #   <<.LabelMatchers>> → the label selectors the adapter adds for the specific
    #                        pod/namespace being queried
    #   <<.GroupBy>>       → the "group by" dimension (pod or namespace)
    metricsQuery: |
      sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)
```

Verify the metric is available to HPA:
```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | jq '.resources[].name'
# "pods/http_server_requests_per_second"
# "namespaces/http_server_requests_per_second"
```

---

## Chapter 3: Grafana — Dashboards and Alerts

### Accessing Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Get the admin password (if not set during install)
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

### Importing Pre-Built Dashboards

Grafana has a community dashboard library at grafana.com/grafana/dashboards. Import by ID:

| Dashboard | ID | What it shows |
|-----------|-----|---------------|
| **Spring Boot 3.x / Micrometer** | 17175 | HTTP rates, latency, JVM heap, GC, thread pools |
| **Kubernetes Cluster Overview** | 315 | Node CPU/memory, pod counts, namespace summaries |
| **Kubernetes Pod Resources** | 6417 | Per-pod CPU, memory, network, storage |
| **JVM (Micrometer)** | 4701 | Detailed JVM internals: heap generations, GC pauses |
| **Kubernetes Deployment** | 8588 | Rolling updates, replica counts, restart rates |
| **PostgreSQL** | 9628 | Query rates, connection counts, cache hit ratios |

To import: Grafana → Dashboards → Import → Enter ID → Load → Select Prometheus data source → Import.

### Creating a Custom Dashboard

Custom dashboards let you combine your application's business metrics (orders per second, checkout conversion rate) with infrastructure metrics (CPU, memory) into a single view relevant to your team.

**Creating a panel:**
1. Dashboards → New Dashboard → Add Panel
2. In the query editor, write a PromQL expression
3. Choose visualization: Time series, Stat, Gauge, Bar chart, Table
4. Set thresholds: green below 1%, yellow below 5%, red above 5%
5. Set the unit: `percent`, `requests/sec`, `milliseconds`

**Example: error rate panel**
```promql
# Query:
sum(rate(http_server_requests_seconds_count{status=~"5..",namespace="$namespace"}[5m]))
/
sum(rate(http_server_requests_seconds_count{namespace="$namespace"}[5m]))
* 100

# Visualization: Stat or Gauge
# Unit: percent (0-100)
# Thresholds:
#   green: 0–1%
#   yellow: 1–5%
#   red: >5%
```

### Dashboard Variables

Variables make dashboards interactive — users select a namespace, pod, or time interval from a dropdown rather than editing queries:

In Dashboard Settings → Variables → Add Variable:

```
# Variable: namespace
Type: Query
Data source: Prometheus
Query: label_values(kube_pod_info, namespace)
# Populates dropdown with all namespaces that have pod data

# Variable: pod
Type: Query
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
# Populates with pods in the selected namespace

# Variable: interval
Type: Interval
Values: 1m,5m,15m,30m,1h
# Used in queries as: rate(...[$interval])
```

Use variables in panel queries:
```promql
rate(http_server_requests_seconds_count{namespace="$namespace", pod="$pod"}[$interval])
```

Now the same dashboard works for every namespace and pod without duplication.

### Alert Rules with PrometheusRule

kube-prometheus-stack uses the `PrometheusRule` CRD to define alerts as code, version-controllable alongside your application:

```yaml
# prometheusrule-myapp.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    release: monitoring    # Must match kube-prometheus-stack's ruleSelector
spec:
  groups:
  - name: myapp.rules
    interval: 30s          # How often to evaluate these rules
    rules:

    # ── Availability ──────────────────────────────────────────────────────────
    - alert: ServiceDown
      expr: |
        absent(up{job="myapp", namespace="production"} == 1)
      for: 1m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "myapp is down in production"
        description: "No healthy myapp instances have been scraped in the last minute."
        runbook: "https://wiki.mycompany.com/runbooks/myapp-down"

    # ── Error Rate ────────────────────────────────────────────────────────────
    - alert: HighErrorRate
      expr: |
        sum(rate(http_server_requests_seconds_count{
          namespace="production", application="myapp", status=~"5.."}[5m]))
        /
        sum(rate(http_server_requests_seconds_count{
          namespace="production", application="myapp"}[5m]))
        * 100 > 5
      for: 5m
      # for: the condition must be true for this long before the alert fires.
      # Prevents false positives from momentary spikes.
      labels:
        severity: critical
      annotations:
        summary: "Error rate is {{ $value | humanizePercentage }}"
        description: "5xx error rate has been above 5% for 5 minutes."

    # ── Latency ───────────────────────────────────────────────────────────────
    - alert: HighP99Latency
      expr: |
        histogram_quantile(0.99,
          sum by (le) (
            rate(http_server_requests_seconds_bucket{
              namespace="production", application="myapp"}[5m])
          )
        ) > 0.5
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "P99 latency is {{ $value | humanizeDuration }}"
        description: "P99 response time has been above 500ms for 5 minutes."

    # ── Crash Loop ────────────────────────────────────────────────────────────
    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total{
          namespace="production"}[15m]) > 3
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "{{ $labels.pod }} has restarted {{ $value }} times in the last 15 minutes."

    # ── CPU Throttling ─────────────────────────────────────────────────────────
    - alert: CPUThrottling
      expr: |
        sum by (pod, container) (
          rate(container_cpu_throttled_seconds_total{
            namespace="production", container!=""}[5m])
        )
        /
        sum by (pod, container) (
          rate(container_cpu_usage_seconds_total{
            namespace="production", container!=""}[5m])
          +
          rate(container_cpu_throttled_seconds_total{
            namespace="production", container!=""}[5m])
        )
        * 100 > 50
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.container }} is CPU throttled {{ $value | humanizePercentage }}"
        description: "CPU throttling > 50% for 15 minutes — consider increasing CPU limit."

    # ── JVM Memory ────────────────────────────────────────────────────────────
    - alert: JVMHeapNearLimit
      expr: |
        jvm_memory_used_bytes{area="heap", namespace="production"}
        /
        jvm_memory_max_bytes{area="heap", namespace="production"}
        * 100 > 90
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "JVM heap is {{ $value | humanizePercentage }} full"
        description: "Pod {{ $labels.pod }} heap usage above 90% for 5 minutes. Risk of OOMKill."
```

```bash
kubectl apply -f prometheusrule-myapp.yaml

# Verify the rules were loaded by Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring
# http://localhost:9090/rules — check myapp.rules group appears
```


## Chapter 4: Alertmanager — Routing Alerts to the Right People

### What Alertmanager Does

Prometheus evaluates alert rules and fires alerts when conditions are met. Alertmanager receives those alerts and handles everything that comes next: deduplication (don't send the same alert 100 times), grouping (bundle related alerts into one notification), routing (send critical alerts to PagerDuty, warnings to Slack), and silencing (suppress alerts during planned maintenance).

Without Alertmanager, every Prometheus alert would fire a separate notification immediately. During an incident where 20 services all alert simultaneously, you'd receive 20 separate pages. Alertmanager groups them into one notification: "20 services are unhealthy — here's the summary."

### The Configuration Structure

Alertmanager configuration has three top-level sections:

```yaml
global:
  # Defaults applied to all receivers
  resolve_timeout: 5m          # How long after an alert stops firing before it's considered resolved
  slack_api_url: 'https://hooks.slack.com/services/...'   # Default Slack webhook

route:
  # The routing tree — determines which receiver handles each alert
  receiver: default-receiver   # Catch-all if no child route matches
  group_by: ['alertname', 'namespace', 'severity']
  group_wait: 30s              # Wait 30s before sending the first notification (allows grouping)
  group_interval: 5m           # How long to wait before sending new notifications for the same group
  repeat_interval: 4h          # How long before repeating a notification for a still-firing alert

  routes:                      # Child routes (evaluated in order, first match wins)
  - match:
      severity: critical
    receiver: pagerduty
    repeat_interval: 1h        # Override: page every hour if still firing

  - match:
      severity: warning
    receiver: slack-warnings
    repeat_interval: 24h       # Slack once per day if still warning

receivers:
  - name: default-receiver
    slack_configs:
    - ...

  - name: pagerduty
    pagerduty_configs:
    - ...

  - name: slack-warnings
    slack_configs:
    - ...
```

### Understanding the Timing Parameters

These three parameters are the most commonly misunderstood:

**`group_wait: 30s`** — When a new alert group fires, wait 30 seconds before sending the first notification. During this window, other alerts in the same group are collected and sent together. Without this, the first alert in a cascade sends immediately, before you know how widespread the incident is.

**`group_interval: 5m`** — After the initial notification, wait 5 minutes before sending another notification about changes to an existing alert group (new alerts added, some resolved). Prevents notification spam during ongoing incidents.

**`repeat_interval: 4h`** — If an alert is still firing and nothing has changed, repeat the notification after 4 hours. This is a reminder that the problem is still present. Set shorter for critical alerts (1h), longer for warnings (24h or longer).

### Complete Slack + PagerDuty Configuration

This is deployed as a Kubernetes Secret referenced by Alertmanager:

```yaml
# alertmanager-config-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-monitoring-kube-prometheus-alertmanager
  namespace: monitoring
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      # Slack default webhook (overridden per receiver below if needed)
      slack_api_url: 'https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX'

    templates:
    - '/etc/alertmanager/config/*.tmpl'

    route:
      receiver: slack-default
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h

      routes:
      # Critical alerts → PagerDuty (wakes someone up)
      - match:
          severity: critical
        receiver: pagerduty-critical
        group_wait: 10s
        repeat_interval: 1h
        continue: false         # Stop routing once matched

      # Warning alerts → Slack #alerts-warning channel
      - match:
          severity: warning
        receiver: slack-warnings
        repeat_interval: 12h

      # Watchdog alert (always-firing heartbeat) → dedicated receiver so it never pages
      - match:
          alertname: Watchdog
        receiver: watchdog
        repeat_interval: 10m

    inhibit_rules:
    # If a critical alert is firing, inhibit warnings about the same service.
    # Prevents getting paged twice for the same root cause.
    - source_match:
        severity: critical
      target_match:
        severity: warning
      equal: ['namespace', 'alertname']

    receivers:
    - name: slack-default
      slack_configs:
      - channel: '#alerts'
        send_resolved: true
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

    - name: slack-warnings
      slack_configs:
      - channel: '#alerts-warnings'
        send_resolved: true
        title: '[WARNING] {{ .GroupLabels.alertname }} ({{ .GroupLabels.namespace }})'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Runbook:* {{ .Annotations.runbook }}
          {{ end }}

    - name: pagerduty-critical
      pagerduty_configs:
      - service_key: 'your-pagerduty-integration-key'
        send_resolved: true
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
        details:
          namespace: '{{ .GroupLabels.namespace }}'
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'

    - name: watchdog
      slack_configs:
      - channel: '#monitoring-heartbeat'
        send_resolved: false
        title: '🟢 Alertmanager heartbeat'
        text: 'Alertmanager is alive and routing alerts.'
```

### Silences — Suppressing Alerts During Maintenance

During planned maintenance (cluster upgrade, database migration), you don't want alerts firing for expected downtime:

```bash
# Create a silence via the Alertmanager API (or UI at http://localhost:9093)
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring

# CLI approach using amtool
kubectl exec -it alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -- \
  amtool --alertmanager.url=http://localhost:9093 silence add \
  --comment="Planned DB maintenance window" \
  --author="ops-team" \
  --duration="2h" \
  alertname="HighErrorRate" namespace="production"
# Created silence: abc123-def456

# List active silences
kubectl exec -it alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -- \
  amtool --alertmanager.url=http://localhost:9093 silence list

# Expire a silence early
kubectl exec -it alertmanager-monitoring-kube-prometheus-alertmanager-0 -n monitoring -- \
  amtool --alertmanager.url=http://localhost:9093 silence expire abc123-def456
```

---

## Chapter 5: Loki — Log Aggregation

### Why Centralised Logging

Without centralised logging:
- Logs live on individual pods that get deleted during rolling updates
- To see logs for a failed pod, you need `kubectl logs --previous` — if the pod was replaced, those logs may be gone
- Searching logs across 20 pods means running 20 separate `kubectl logs` commands and grep-ing through them mentally
- Logs from two hours ago require the pod to still be running — or they're lost

Loki collects logs from every pod and stores them centrally. You can query logs from any pod, any time window, even after the pod was deleted, from a single Grafana interface.

### What Loki Is — and Isn't

Loki's design is explicitly modelled on Prometheus: instead of indexing log content (like Elasticsearch), Loki indexes only the labels attached to log streams. Log content is stored compressed, queried by scanning lines that match the label selector.

This makes Loki dramatically cheaper to run than Elasticsearch for most Kubernetes use cases — you need far less memory and storage. The tradeoff: full-text search across arbitrary log content is slower than in Elasticsearch. For "find all logs from the `production` namespace in the last hour that contain the word ERROR", Loki is perfectly fast. For "find every log line across all time that matches this complex regex", Elasticsearch is faster.

For most teams, Loki's performance is more than adequate and the operational simplicity is worth it.

### Installing Loki Stack

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.enabled=true \
  --set promtail.enabled=true \     # Promtail: log collector DaemonSet
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=50Gi \
  --set loki.persistence.storageClassName=fast-ssd

kubectl get pods -n monitoring | grep loki
# loki-0                          1/1     Running   ← Loki server
# loki-promtail-abcd1             1/1     Running   ← one per node (DaemonSet)
# loki-promtail-abcd2             1/1     Running
```

Promtail runs as a DaemonSet — one pod per node — and tails `/var/log/pods/` collecting every container's stdout/stderr. Zero configuration needed in your application. Your logs appear in Loki automatically.

**Add Loki as a Grafana data source:**
Grafana → Configuration → Data Sources → Add data source → Loki
URL: `http://loki.monitoring.svc.cluster.local:3100`
Save & Test → "Data source connected and labels found"

### LogQL — Querying Logs

LogQL is Loki's query language. It has two parts: a stream selector (which log streams to look at) and optional filter expressions (what to look for within them).

**Stream selectors — always required:**
```logql
# All logs from the production namespace
{namespace="production"}

# Logs from a specific pod
{namespace="production", pod="myapp-7d9b4f-xj8mk"}

# Logs from all myapp pods (label selector)
{namespace="production", app="myapp"}

# Logs from the ingress controller
{namespace="ingress-nginx", app_kubernetes_io_name="ingress-nginx"}
```

**Filter expressions — narrow down within the stream:**
```logql
# Lines containing the exact string "ERROR"
{namespace="production"} |= "ERROR"

# Lines NOT containing "healthcheck"
{namespace="production"} != "healthcheck"

# Lines matching a regex
{namespace="production"} |~ "Exception|Error|WARN"

# Lines NOT matching a regex
{namespace="production"} !~ "GET /actuator/health"
```

**JSON parsing — structured log queries:**

When your application logs JSON, Loki can parse fields and filter on them:
```logql
# Parse JSON and filter by log level
{namespace="production", app="myapp"} | json | level="ERROR"

# Parse JSON and filter by multiple fields
{namespace="production"} | json | level="ERROR" | duration > 1000

# Parse JSON and extract a field for display
{namespace="production"} | json | line_format "{{.level}} {{.message}} ({{.traceId}})"
```

**Rate queries — metrics from logs:**
```logql
# Error log rate per minute
rate({namespace="production"} |= "ERROR" [1m])

# Count of log lines by level
sum by (level) (
  rate({namespace="production"} | json [5m])
)
```

### Structured Logging in Spring Boot

Plain text logs work with Loki, but JSON structured logs unlock the full power of LogQL's JSON parser. Instead of grepping for text patterns, you filter on typed fields.

Add the dependency:
```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.4</version>
</dependency>
```

```groovy
// build.gradle
implementation 'net.logstash.logback:logstash-logback-encoder:7.4'
```

Configure Logback to output JSON:
```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="kubernetes,prod,staging">
        <!-- JSON output for Kubernetes environments -->
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder">
                <!-- Include standard Spring Boot fields -->
                <includeMdcKeyName>traceId</includeMdcKeyName>
                <includeMdcKeyName>spanId</includeMdcKeyName>
                <!-- Add custom fields -->
                <customFields>{"application":"${spring.application.name}","environment":"${spring.profiles.active}"}</customFields>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>

    <springProfile name="local,test">
        <!-- Human-readable output for local development -->
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
```

Your logs now look like this in Kubernetes:
```json
{
  "@timestamp": "2024-01-15T14:30:22.456Z",
  "level": "ERROR",
  "logger_name": "com.example.OrderService",
  "message": "Failed to process order",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "application": "myapp",
  "environment": "prod",
  "orderId": "ORD-12345",
  "customerId": "CUST-67890",
  "exception": "java.sql.SQLException: Connection refused"
}
```

Now in Loki you can query:
```logql
{namespace="production"} | json | level="ERROR" | orderId="ORD-12345"
{namespace="production"} | json | traceId="abc123def456"
```

### Log-to-Trace Correlation

When your application includes `traceId` in log output (via the logback encoder with MDC integration), Grafana's Explore view can link directly from a log line to the full distributed trace:

```yaml
# In Grafana Loki data source configuration:
# Derived fields → add a derived field:
# Name: TraceID
# Regex: "traceId":"(\w+)"
# Internal link: Tempo data source, query: ${__value.raw}
```

Now in Grafana Explore, any log line with a `traceId` field shows a clickable "Tempo" link. Click it → the full distributed trace opens showing exactly what happened during that request.

---

## Chapter 6: Distributed Tracing — Tempo and OpenTelemetry

### Why Tracing When You Have Logs?

Logs tell you what happened inside one service. When a request passes through five services, each service's logs have only its portion of the story. You can find the error in the payment-service logs — but you don't know which upstream call triggered it, how long the checkout-service waited, or whether it was a timeout or a bug.

A distributed trace captures the full journey: every service the request touched, how long each took, what the parent-child relationship between calls was, and which specific call failed. It is the only tool that answers "where did this 3-second request actually spend its time?"

### Traces, Spans, and Context Propagation

A **trace** is a directed acyclic graph of **spans**. Each span represents a unit of work — an HTTP call, a database query, a cache lookup — with a start time, duration, and metadata (status, error, attributes).

```
Trace: checkout-order (total: 2.3s)
├── checkout-service.processOrder (2.3s)
│   ├── inventory-service.checkStock (0.1s)     ← fast, ok
│   ├── payment-service.charge (1.8s)           ← SLOW — this is the problem
│   │   └── postgres.query (1.75s)              ← the actual bottleneck: slow query
│   └── notification-service.sendConfirmation (0.1s)
```

One glance at this tree tells you the checkout took 2.3 seconds because the payment service made a 1.75-second database query. No grepping, no cross-referencing between five log files.

**Context propagation** is what connects spans across services. When the checkout-service calls the payment-service, it includes a `traceparent` HTTP header (`W3C Trace Context` standard) containing the current trace ID and span ID. The payment-service reads this header, creates a child span under the parent, and continues propagating it to any downstream calls. Every service in the chain contributes spans to the same trace.

### Tempo vs Jaeger

| | Tempo | Jaeger |
|-|-------|--------|
| **Storage** | Object storage (S3, GCS) — very cheap at scale | Multiple backends (Cassandra, Elasticsearch, local) |
| **Integration** | Native Grafana data source | Own UI + Grafana plugin |
| **Query language** | TraceQL (powerful, newer) | Jaeger UI queries |
| **Best for** | Teams already using Grafana stack | Teams wanting a standalone tracing UI |

Tempo is the natural choice if you're already using Prometheus and Loki — everything lives in Grafana.

### Installing Tempo

```bash
helm install tempo grafana/tempo \
  --namespace monitoring \
  --set tempo.storage.trace.backend=local \    # For learning; use S3/GCS in production
  --set tempo.storage.trace.local.path=/var/tempo

# For production with S3:
helm install tempo grafana/tempo-distributed \
  --namespace monitoring \
  --set storage.trace.backend=s3 \
  --set storage.trace.s3.bucket=myapp-traces \
  --set storage.trace.s3.region=us-east-1
```

Add Tempo as a Grafana data source:
Grafana → Configuration → Data Sources → Add → Tempo
URL: `http://tempo.monitoring.svc.cluster.local:3100`

### Spring Boot + OpenTelemetry Auto-Instrumentation

OpenTelemetry auto-instrumentation means you add a dependency and configure an endpoint — Spring Boot automatically creates traces for every HTTP request, database call, and messaging operation without any code changes.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```groovy
// build.gradle
implementation 'io.micrometer:micrometer-tracing-bridge-otel'
implementation 'io.opentelemetry:opentelemetry-exporter-otlp'
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0    # Sample 100% of traces (reduce in production — see sampling section)
  otlp:
    tracing:
      endpoint: http://tempo.monitoring.svc.cluster.local:4318/v1/traces

spring:
  application:
    name: checkout-service   # Appears as the service name in traces
```

With this configuration, every incoming HTTP request automatically creates a trace that is exported to Tempo. Spring Boot also propagates the `traceparent` header on outgoing `RestTemplate` and `WebClient` calls, so distributed traces across services are assembled automatically.

### Manual Span Creation with `@WithSpan`

For business logic that spans multiple operations and where you want granular timing:

```java
import io.micrometer.tracing.annotation.WithSpan;
import io.micrometer.tracing.annotation.SpanTag;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.common.AttributeKey;

@Service
public class PaymentService {

    // @WithSpan creates a new span for this method call.
    // The span name defaults to the method name but can be overridden.
    @WithSpan("payment.process")
    public PaymentResult processPayment(
        @SpanTag("payment.amount") BigDecimal amount,   // Adds amount as a span attribute
        @SpanTag("payment.currency") String currency,
        String customerId
    ) {
        // Add custom attributes to the current span
        Span.current().setAttribute(AttributeKey.stringKey("customer.id"), customerId);
        Span.current().setAttribute(AttributeKey.stringKey("payment.provider"), "stripe");

        try {
            PaymentResult result = stripeClient.charge(amount, currency, customerId);
            Span.current().setAttribute(AttributeKey.stringKey("payment.transaction_id"),
                result.getTransactionId());
            return result;
        } catch (StripeException e) {
            // Mark the span as an error — appears in trace as a failed span
            Span.current().recordException(e);
            Span.current().setStatus(StatusCode.ERROR, e.getMessage());
            throw new PaymentException("Payment failed", e);
        }
    }
}
```

This produces a span in the trace with:
- Name: `payment.process`
- Duration: exact time the method took
- Attributes: `payment.amount`, `payment.currency`, `customer.id`, `payment.provider`, `payment.transaction_id`
- Status: ERROR with the exception if it throws

### Trace Sampling — Don't Collect Everything

At high traffic volumes (1000 req/sec), 100% trace sampling means 1000 traces per second stored. At a typical trace size of 2KB, that's 2MB/second → 170GB/day of trace data. This is expensive both in storage and in the overhead of exporting traces.

```yaml
# Sampling strategies:

# 1. Always-on (100%) — for development and low-traffic production
management:
  tracing:
    sampling:
      probability: 1.0

# 2. Probabilistic (10%) — for high-traffic production
management:
  tracing:
    sampling:
      probability: 0.1    # Sample 10% of all requests

# 3. Rate-limited — sample N traces per second regardless of traffic
# Requires configuring the OpenTelemetry SDK directly
```

**Head-based vs tail-based sampling:**

*Head-based* (the default above): the sampling decision is made at the first service when the trace starts. Simple but you might miss the 0.5% of requests that are slow or fail.

*Tail-based*: collect all spans from all services, then make the sampling decision after the full trace is assembled — keeping all traces that were slow, errored, or otherwise interesting. More complex (requires a trace collector like OpenTelemetry Collector), but guarantees you keep the traces that matter.

For most teams, probabilistic head-based sampling (10–20%) is sufficient. Add the rule "always sample if HTTP status is 5xx" to ensure errors are never dropped.

### Viewing Traces in Grafana

1. Open Grafana → Explore → Select Tempo data source
2. Search by service name: `{service.name="checkout-service"}`
3. Click any trace to open the waterfall view
4. Each span shows: name, duration, service, attributes, events
5. Click a failed span (shown in red) to see the exception details

**From logs to trace:** With trace IDs in your logs (via the JSON Logback encoder and MDC), click the "Tempo" link in any log line to jump directly to that request's trace.

**TraceQL — querying traces directly:**
```
# Find all traces where the payment service took > 1 second
{span.service.name="payment-service"} | duration > 1s

# Find all failed traces
{rootServiceName="checkout-service"} | status=error

# Find traces containing a specific attribute
{span.http.url=~".*checkout.*"} | duration > 500ms
```


## Chapter 7: SLI/SLO — Measuring What Actually Matters

### The Problem with Alert-Driven Operations

Most teams alert on symptoms: CPU > 80%, error count > 100, latency > 500ms. These alerts are useful, but they don't answer the question that actually matters: are our users experiencing acceptable service?

A CPU spike to 95% for 10 seconds may have no impact on users if requests were still served quickly. A 2% error rate sounds small but could mean 20,000 failed requests per hour for a high-traffic service. Resource metrics and raw counts are proxies for user experience — they're one step removed from what you actually care about.

SLOs (Service Level Objectives) close this gap by defining reliability targets directly in terms of user experience.

### SLIs and SLOs — The Vocabulary

**SLI (Service Level Indicator):** A quantitative measure of service behaviour. Examples:
- Request success rate: `(successful requests) / (total requests)`
- Availability: `(minutes service is up) / (total minutes)`
- Latency: `(requests served in < 500ms) / (total requests)`

**SLO (Service Level Objective):** A target value for an SLI over a time window. Examples:
- 99.9% of requests succeed over a rolling 30-day window
- 95% of requests are served in < 200ms
- Service is available 99.95% of the time monthly

**SLA (Service Level Agreement):** A contractual commitment to an SLO, with consequences for violations. SLAs are business agreements; SLOs are internal engineering targets (typically set higher than SLAs to leave headroom).

### Error Budget — What You're Allowed to Fail

An SLO of 99.9% availability over 30 days means you're allowed to be unavailable for:

```
30 days × 24 hours × 60 minutes = 43,200 minutes total
43,200 × (1 - 0.999) = 43.2 minutes of allowed downtime
```

Those 43.2 minutes are your **error budget**. It is not a punishment — it is a licence to take risks. You can deploy risky changes, run experiments, and accept failures as long as you stay within budget. When the budget is exhausted, stop taking risks and focus on reliability until it recovers at the start of the next window.

Error budgets change the conversation from "we can never have downtime" (which leads to change aversion and stagnation) to "we have X minutes of budget — should we spend it on this deployment?"

### Defining SLOs in Prometheus

```yaml
# prometheusrule-slos.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-slos
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  # ── Recording Rules: pre-compute SLI values ────────────────────────────────
  # Recording rules store computed values as new metrics, making dashboards fast
  - name: myapp.slo.recording
    interval: 30s
    rules:
    # Success rate over 5-minute window
    - record: job:http_requests:success_rate5m
      expr: |
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp",status!~"5.."}[5m]))
        /
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp"}[5m]))

    # Success rate over 1-hour window (for burn rate calculation)
    - record: job:http_requests:success_rate1h
      expr: |
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp",status!~"5.."}[1h]))
        /
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp"}[1h]))

    # Success rate over 6-hour window
    - record: job:http_requests:success_rate6h
      expr: |
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp",status!~"5.."}[6h]))
        /
        sum(rate(http_server_requests_seconds_count{
          namespace="production",application="myapp"}[6h]))

  # ── Alerting Rules: burn rate alerts ──────────────────────────────────────
  # These four alerts cover fast-burn (immediate crisis) and slow-burn (gradual erosion)
  - name: myapp.slo.alerts
    rules:
    # Fast burn: 14.4× budget rate = exhausted in 2 hours
    # This means 2% of requests are failing right now — page immediately
    - alert: SLOFastBurn
      expr: |
        (1 - job:http_requests:success_rate5m) > 14.4 * (1 - 0.999)
        and
        (1 - job:http_requests:success_rate1h) > 14.4 * (1 - 0.999)
      for: 2m
      labels:
        severity: critical
        slo: availability
      annotations:
        summary: "SLO fast burn: error budget exhausted in ~2 hours"
        description: |
          Error rate is {{ $value | humanizePercentage }} — 14.4x the SLO budget rate.
          At this rate, the 30-day error budget will be exhausted in approximately 2 hours.

    # Slow burn: 6× budget rate = exhausted in 5 days
    # Worth waking someone up, but not an immediate crisis
    - alert: SLOSlowBurn
      expr: |
        (1 - job:http_requests:success_rate1h) > 6 * (1 - 0.999)
        and
        (1 - job:http_requests:success_rate6h) > 6 * (1 - 0.999)
      for: 15m
      labels:
        severity: warning
        slo: availability
      annotations:
        summary: "SLO slow burn: error budget will be exhausted in ~5 days"
        description: |
          Error rate is {{ $value | humanizePercentage }} sustained over the past hour.
          At this rate, the 30-day error budget will be exhausted in approximately 5 days.
```

### SLO Dashboard in Grafana

A clear SLO dashboard shows four things:
1. Current SLI value (is it above or below the target right now?)
2. Error budget remaining (how much of the 30-day budget is left?)
3. Burn rate (are we consuming budget faster than we're earning it?)
4. Historical SLI (trend over the last 30 days)

```promql
# Panel 1: Current success rate (Stat visualization)
job:http_requests:success_rate5m * 100

# Panel 2: Error budget remaining (percentage of monthly budget left)
# SLO = 99.9%, allowed error rate = 0.1%
# Budget consumed = actual errors / allowed errors
(
  1 -
  (
    sum(increase(http_server_requests_seconds_count{
      status=~"5..",namespace="production"}[30d]))
    /
    sum(increase(http_server_requests_seconds_count{namespace="production"}[30d]))
  ) / 0.001   # 0.001 = 0.1% allowed error rate
) * 100

# Panel 3: Error budget burn rate (Time series)
(1 - job:http_requests:success_rate1h) / (1 - 0.999)

# Panel 4: Latency SLO — % of requests under 500ms
sum(rate(http_server_requests_seconds_bucket{
  namespace="production", le="0.5"}[5m]))
/
sum(rate(http_server_requests_seconds_count{namespace="production"}[5m]))
* 100
```

---

## Chapter 8: Complete Observability Stack

### The Full Deploy Sequence

Deploy the complete stack in this order (each component depends on the previous):

```bash
# 1. kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm upgrade --install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values-monitoring.yaml \
  --wait

# 2. Loki stack (log aggregation + Promtail collector)
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  -f values-loki.yaml \
  --wait

# 3. Tempo (distributed tracing backend)
helm upgrade --install tempo grafana/tempo \
  --namespace monitoring \
  -f values-tempo.yaml \
  --wait

# 4. Prometheus Adapter (custom metrics for HPA) — only if needed
helm upgrade --install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  -f values-prometheus-adapter.yaml \
  --wait

# 5. Deploy your application PrometheusRules and ServiceMonitors
kubectl apply -f prometheusrule-myapp.yaml -n monitoring
kubectl apply -f prometheusrule-slos.yaml -n monitoring
kubectl apply -f servicemonitor-myapp.yaml -n monitoring

# 6. Verify everything is connected
kubectl get servicemonitor -n monitoring
kubectl get prometheusrule -n monitoring
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# Check http://localhost:9090/targets — your app should be listed as UP
```

### Production values-monitoring.yaml

```yaml
# values-monitoring.yaml — production kube-prometheus-stack configuration
grafana:
  adminPassword: ""    # Set via secret in production — don't hardcode
  persistence:
    enabled: true
    size: 10Gi
    storageClassName: fast-ssd
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/auth-type: basic    # Add basic auth or use SSO
    hosts:
      - grafana.myapp.com
    tls:
      - secretName: grafana-tls
        hosts: [grafana.myapp.com]
  # Import dashboards automatically
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: default
        folder: MyApp
        type: file
        options:
          path: /var/lib/grafana/dashboards/default
  dashboards:
    default:
      spring-boot:
        gnetId: 17175
        revision: 1
        datasource: Prometheus
      kubernetes-cluster:
        gnetId: 315
        revision: 3
        datasource: Prometheus

prometheus:
  prometheusSpec:
    retention: 15d
    retentionSize: "45GB"
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    # Scrape all ServiceMonitors — not just ones with the release label
    serviceMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: 2000m

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 10Gi
```

### The Incident Investigation Workflow

When an alert fires, the observability stack gives you a structured path to root cause:

```
Alert: HighErrorRate fires at 14:32
"Error rate is 8.3% in production"
        │
        ▼
Step 1: Open Grafana → Spring Boot dashboard
        - Error rate panel: confirms 8.3%, started at ~14:28
        - P99 latency: jumped from 120ms to 3.2s at 14:28
        - Orders per second: flat (same traffic, more errors)
        │
        ▼
Step 2: Open Loki → Explore → {namespace="production",app="myapp"}
        Filter: | json | level="ERROR"
        Time range: 14:25–14:35
        - See: "java.sql.SQLException: Connection refused to postgres:5432"
        - First occurrence: 14:27:43
        │
        ▼
Step 3: Click traceId in a log line → Tempo opens
        Trace: checkout-order (3.2s total)
        ├── checkout-service (3.2s)
        │   ├── postgres query FAILED (timeout after 3s)  ← root cause
        └── ...
        │
        ▼
Step 4: Open Kubernetes dashboard → postgres StatefulSet
        kubectl describe pod postgres-0 -n production
        - Events: "Liveness probe failed: connection refused"
        - postgres-0 is restarting
        │
        ▼
Step 5: Check postgres logs
        kubectl logs postgres-0 -n production --previous
        - "FATAL: database system is in recovery mode"
        │
        ▼
Root cause: postgres-0 restarted during a node maintenance event,
            is in recovery mode, not accepting connections yet.

Resolution: Wait for recovery (3 minutes), traffic normalises automatically
            (readiness probe prevents traffic until postgres is ready).
            Action item: Add PDB to prevent this during future maintenance.

Total diagnosis time: 8 minutes.
Without observability: hours of guessing.
```

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| Prometheus not scraping your app | ServiceMonitor not found, or wrong `release` label | `kubectl get servicemonitor -n monitoring` — check it exists; check Prometheus targets at `:9090/targets` | Add `labels: release: monitoring` to ServiceMonitor; check `serviceMonitorSelector` in Prometheus spec |
| `/actuator/prometheus` returns 404 | `prometheus` endpoint not exposed | `curl localhost:8080/actuator` — check available endpoints | Add `prometheus` to `management.endpoints.web.exposure.include` |
| Grafana shows "no data" for custom metrics | Metric name wrong, or labels don't match | Run the PromQL query directly at `localhost:9090` | Verify metric name with `curl localhost:8080/actuator/prometheus \| grep metric_name` |
| High cardinality causes Prometheus OOM | Unbounded label values — user IDs, request IDs, trace IDs as labels | `prometheus_tsdb_head_series` metric — if millions, you have a cardinality problem | Find the high-cardinality metric: `topk(10, count by (__name__)({__name__=~".+"}))` then fix the label |
| Loki shows no logs | Promtail not running or not collecting from namespace | `kubectl get pods -n monitoring \| grep promtail`; check Promtail logs | Verify Promtail DaemonSet is running on all nodes; check Promtail config includes your namespace |
| Loki JSON parsing returns no results | Logs not actually JSON, or wrong field names | Run `{namespace="production"} \| json` in Explore — see if fields appear | Check if Spring Boot is using the JSON Logback encoder; verify `logback-spring.xml` profile is active |
| Traces not appearing in Tempo | OTLP endpoint wrong, or app not started with tracing | Check app logs for OTLP export errors; verify `management.otlp.tracing.endpoint` | Verify endpoint: `http://tempo.monitoring.svc.cluster.local:4318/v1/traces`; check Tempo is running |
| Alertmanager not sending notifications | Config wrong, or receiver unreachable | Check Alertmanager UI at `:9093`; check logs: `kubectl logs -n monitoring alertmanager-*` | Test webhook manually with curl; check firewall rules if webhook is external |
| SLO burn rate alert fires constantly | SLO target too aggressive, or genuine reliability problem | Check error budget consumption rate over the past 7 days | If truly too aggressive: recalibrate SLO; if genuine problem: work on reliability |
| `rate()` shows zero for new metric | Metric has < 2 samples — not enough data for rate() | Check `http_server_requests_seconds_count` has data | Wait for 2+ scrape cycles (60s at 30s interval); confirm metric appears in raw form |

---

## Practice Exercises

**Exercise 1 — Prometheus end-to-end:**
Install `kube-prometheus-stack` in your kind cluster. Add the `micrometer-registry-prometheus` dependency to your `hello-app`. Expose `/actuator/prometheus`. Create a `ServiceMonitor`. Verify the target appears in Prometheus at `localhost:9090/targets`. Write a PromQL query that shows the request rate for your `/api/hello` endpoint. Watch it change as you run `curl` in a loop.

**Exercise 2 — Custom metrics and alerting:**
Add a `Counter` metric to your Spring Boot app that counts requests to each endpoint, tagged with `endpoint` and `status` labels. Write a `PrometheusRule` that fires a `HighErrorRate` alert when the error percentage exceeds 5% for 2 minutes. Deliberately trigger the alert by temporarily making your endpoint return 500 errors. Watch the alert appear in Prometheus, then in Alertmanager, and verify a Slack notification is sent (or use a webhook.site URL for testing).

**Exercise 3 — Structured logging with Loki:**
Configure Spring Boot JSON logging with the Logstash Logback encoder. Deploy to your kind cluster with Loki installed. Open Grafana Explore with the Loki data source. Write a LogQL query that returns only ERROR-level logs from your namespace. Add a `orderId` field to some log lines and filter by it. Verify the JSON field appears as a Loki label after parsing.

**Exercise 4 — Distributed tracing:**
Deploy two Spring Boot services: `service-a` (receives HTTP requests and calls `service-b`) and `service-b` (does some work and returns). Install Tempo. Add the OpenTelemetry tracing dependency to both services. Call `service-a` from your browser, then find the trace in Grafana. Verify the trace spans both services and shows the parent-child relationship. Add `@WithSpan` to a method in `service-b` and verify the new span appears in the trace.

**Exercise 5 — SLO dashboard:**
Define an SLO for your hello-app: 99.9% of requests succeed over a 30-day window. Create the recording rules and burn rate alerts from Chapter 7. Build a Grafana dashboard with four panels: current success rate (Stat), error budget remaining (Gauge), burn rate (Time series), and historical SLI (Time series showing the last 7 days). Deliberately inject a 10% error rate for 5 minutes and watch the burn rate spike and the error budget decrease.

