## Part 10: Production Playbook - From Development to Production

### Prerequisites
- Completed Parts 1-9 (or equivalent experience)
- Access to cloud provider (AWS, GCP, or Azure)
- Domain name (optional but recommended)
- Production mindset

### What You'll Learn
- ✅ Complete production deployment checklist
- ✅ CI/CD pipeline integration
- ✅ Security hardening guide
- ✅ Monitoring and alerting setup
- ✅ Runbooks for common incidents
- ✅ Cost optimization strategies
- ✅ Multi-cloud deployment
- ✅ Production readiness review

---

## Chapter 1: Production Readiness Checklist

### The Production Readiness Framework

```
┌─────────────────────────────────────────────────────────────┐
│              Production Readiness Review                    │
├─────────────────────────────────────────────────────────────┤
│  □ Application Code                                         │
│  □ Container Configuration                                  │
│  □ Kubernetes Manifests                                     │
│  □ Security & Compliance                                    │
│  □ Observability                                            │
│  □ Scalability & Resilience                                 │
│  □ CI/CD Pipeline                                           │
│  □ Disaster Recovery                                        │
│  □ Cost Optimization                                        │
│  □ Documentation                                            │
└─────────────────────────────────────────────────────────────┘
```

### Pre-Production Validation Script

```bash
#!/bin/bash
# production-readiness-check.sh

set -e

echo "🔍 Running Production Readiness Checks..."

# 1. Application Checks
echo "✓ Checking application configuration..."
kubectl get deployment myapp -n production -o json | jq '.spec.template.spec.containers[0].resources'
# Must have requests and limits set

# 2. Health Checks
echo "✓ Verifying health endpoints..."
kubectl get pods -n production -l app=myapp -o name | head -1 | xargs -I {} kubectl exec {} -n production -- curl -s http://localhost:8080/actuator/health

# 3. Security Checks
echo "✓ Running security validation..."
kubectl auth can-i list secrets --as=system:serviceaccount:production:myapp-sa
kubectl get networkpolicies -n production

# 4. HPA Configuration
echo "✓ Checking HPA..."
kubectl get hpa -n production

# 5. PDB Configuration
echo "✓ Checking PDB..."
kubectl get pdb -n production

# 6. Monitoring
echo "✓ Verifying metrics scraping..."
kubectl get servicemonitors -n monitoring | grep myapp

# 7. Backup Configuration
echo "✓ Checking backup schedules..."
velero schedule get

# 8. Resource Quotas
echo "✓ Checking resource quotas..."
kubectl get resourcequota -n production

echo "✅ Production readiness check completed!"
```

---

## Chapter 2: Complete Production Values

### Production-Ready Helm Values

**`values-prod-final.yaml`:**
```yaml
# Complete production configuration
global:
  environment: production
  imageRegistry: myregistry.com
  storageClass: fast-ssd

# Application configuration
replicaCount: 5

image:
  repository: myregistry.com/myapp
  tag: v1.0.0  # Specific version, never latest
  pullPolicy: IfNotPresent
  pullSecrets:
    - regcred

# Resource management
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilization: 60
  targetMemoryUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 3

# Rolling update strategy
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# Probes
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 60
    periodSeconds: 10
    failureThreshold: 3
    timeoutSeconds: 5
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 30
    periodSeconds: 5
    failureThreshold: 3
    timeoutSeconds: 5
  startup:
    enabled: true
    failureThreshold: 30
    periodSeconds: 10

# Security
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

containerSecurityContext:
  allowPrivilegeEscalation: false
  privileged: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"

# Ingress
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
  hosts:
  - host: api.myapp.com
    paths:
    - path: /
      pathType: Prefix
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls

# Network Policies
networkPolicies:
  enabled: true
  defaultDeny: true
  allowIngress:
    fromIngressController: true
    fromMonitoring: true
  allowEgress:
    toDNS: true
    toDatabase: true
    toRedis: true

# Service Account
serviceAccount:
  create: true
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-prod-role

# ConfigMap
configMap:
  enabled: true
  data:
    SPRING_PROFILES_ACTIVE: production
    LOG_LEVEL: INFO
    APP_NAME: myapp-prod

# Secrets (using external secrets)
secrets:
  external:
    enabled: true
    provider: aws
    secretsManager:
      - name: db-password
        key: production/myapp/db-password
      - name: api-keys
        key: production/myapp/api-keys

# Monitoring
monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s
  prometheusRules:
    enabled: true
  grafanaDashboard:
    enabled: true

# Logging
logging:
  enabled: true
  loki:
    enabled: true
  additionalLabels:
    environment: production

# Tracing
tracing:
  enabled: true
  provider: tempo
  endpoint: tempo.monitoring.svc:4317
  sampleRate: 0.1  # 10% sampling

# Backup
backup:
  enabled: true
  schedule: "0 2 * * *"  # 2 AM daily
  retention: 30  # days
  s3:
    bucket: myapp-backups-prod

# Pod Anti-Affinity (spread across nodes)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myapp
        topologyKey: kubernetes.io/hostname

# Topology Spread Constraints
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp

# Priority Class (for critical workloads)
priorityClassName: production-high

# Termination Grace Period
terminationGracePeriodSeconds: 60

# Node Selector (for specific hardware)
nodeSelector:
  node-type: application

# Tolerations (for dedicated nodes)
tolerations:
- key: "production"
  operator: "Equal"
  value: "true"
  effect: "NoSchedule"
```

---

## Chapter 3: Complete CI/CD Pipeline

### GitHub Actions Production Pipeline

**`.github/workflows/production-deploy.yml`:**
```yaml
name: Production Deployment

on:
  push:
    tags:
      - 'v*'  # Trigger on version tags only
  workflow_dispatch:  # Manual trigger for hotfixes
    inputs:
      version:
        description: 'Version to deploy'
        required: true
      hotfix:
        description: 'Is this a hotfix?'
        required: false
        default: 'false'

env:
  REGISTRY: myregistry.com
  IMAGE_NAME: myapp
  CLUSTER_NAME: production-cluster
  NAMESPACE: production

jobs:
  # Validate version
  validate:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      with:
        fetch-depth: 0
    
    - name: Extract version
      id: version
      run: |
        if [[ $GITHUB_REF == refs/tags/* ]]; then
          VERSION=${GITHUB_REF#refs/tags/v}
        else
          VERSION=${{ github.event.inputs.version }}
        fi
        echo "version=$VERSION" >> $GITHUB_OUTPUT
        
        # Validate semantic versioning
        if ! [[ $VERSION =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
          echo "Invalid version format. Use semantic versioning (e.g., 1.2.3)"
          exit 1
        fi

  # Build and test
  build-and-test:
    needs: validate
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    
    - name: Run tests
      run: |
        ./mvnw clean test
        ./mvnw verify -Pintegration-tests
    
    - name: Build JAR
      run: ./mvnw package -DskipTests
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: app-jar
        path: target/*.jar

  # Security scan
  security-scan:
    needs: build-and-test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
        severity: 'CRITICAL,HIGH'
    
    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Check for critical vulnerabilities
      run: |
        if grep -q '"severity":"CRITICAL"' trivy-results.sarif; then
          echo "Critical vulnerabilities found! Aborting deployment."
          exit 1
        fi

  # Build and push image
  build-push:
    needs: [validate, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image_digest: ${{ steps.digest.outputs.digest }}
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}
    
    - name: Build and push
      id: docker_build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.validate.outputs.version }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        labels: |
          version=${{ needs.validate.outputs.version }}
          build-date=${{ github.event.head_commit.timestamp }}
          commit=${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Get image digest
      id: digest
      run: |
        DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.validate.outputs.version }} | cut -d'@' -f2)
        echo "digest=$DIGEST" >> $GITHUB_OUTPUT

  # Deploy to staging (always)
  deploy-staging:
    needs: build-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure cluster
      run: |
        echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config
    
    - name: Deploy to staging
      run: |
        helm upgrade --install myapp ./helm-chart \
          --namespace staging \
          --set image.tag=${{ needs.validate.outputs.version }} \
          --set image.digest=${{ needs.build-push.outputs.image_digest }} \
          --set environment=staging \
          --wait \
          --timeout 10m
    
    - name: Run smoke tests
      run: |
        kubectl run smoke-test --image=curlimages/curl --rm -it --restart=Never -n staging -- \
          curl -f http://myapp:8080/actuator/health || exit 1
    
    - name: Run integration tests
      run: |
        helm test myapp -n staging

  # Deploy to production (manual approval)
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://api.myapp.com
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure kubectl
      uses: azure/setup-kubectl@v3
    
    - name: Configure production cluster
      run: |
        echo "${{ secrets.PROD_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config
    
    - name: Backup current state
      run: |
        velero backup create pre-deploy-backup-$(date +%Y%m%d-%H%M%S) \
          --include-namespaces ${{ env.NAMESPACE }}
    
    - name: Deploy to production
      run: |
        helm upgrade --install myapp ./helm-chart \
          --namespace ${{ env.NAMESPACE }} \
          -f values-prod-final.yaml \
          --set image.tag=${{ needs.validate.outputs.version }} \
          --set image.digest=${{ needs.build-push.outputs.image_digest }} \
          --atomic \
          --wait \
          --timeout 15m
    
    - name: Verify deployment
      run: |
        # Check rollout status
        kubectl rollout status deployment/myapp -n ${{ env.NAMESPACE }} --timeout=5m
        
        # Run canary verification
        for i in {1..30}; do
          if curl -s -f https://api.myapp.com/actuator/health | grep -q "UP"; then
            echo "✅ Health check passed"
            break
          fi
          if [ $i -eq 30 ]; then
            echo "❌ Health check failed"
            kubectl rollout undo deployment/myapp -n ${{ env.NAMESPACE }}
            exit 1
          fi
          sleep 10
        done
    
    - name: Notify success
      uses: slackapi/slack-github-action@v1.24
      with:
        payload: |
          {
            "text": "✅ Production deployment successful!",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "✅ *Production Deployment Successful*\nVersion: ${{ needs.validate.outputs.version }}\nDigest: ${{ needs.build-push.outputs.image_digest }}\nEnvironment: Production"
                }
              }
            ]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}

  # Post-deployment validation
  post-deploy-validation:
    needs: deploy-production
    runs-on: ubuntu-latest
    steps:
    - name: Run post-deployment tests
      run: |
        # Run performance tests
        echo "Running performance validation..."
        
        # Verify metrics are being collected
        kubectl port-forward -n monitoring svc/prometheus-server 9090:9090 &
        sleep 5
        curl -s http://localhost:9090/api/v1/query?query=up{namespace="production"} | jq '.data.result | length'
        
        # Check error rate
        ERROR_RATE=$(curl -s "http://localhost:9090/api/v1/query?query=sum(rate(http_server_requests_seconds_count{status=~\"5..\",namespace=\"production\"}[5m]))/sum(rate(http_server_requests_seconds_count{namespace=\"production\"}[5m]))" | jq '.data.result[0].value[1]')
        
        if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
          echo "⚠️ High error rate detected: $ERROR_RATE"
          # Notify but don't fail
        else
          echo "✅ Error rate normal: $ERROR_RATE"
        fi
```

---

## Chapter 4: Security Hardening Guide

### Production Security Checklist

```yaml
# security-hardening.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Pod Security Standards
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
    
    # Network policy isolation
    network-policy: enabled
    
    # Monitoring
    monitoring: enabled
    
    # Backup
    backup: required

---
# Network Policy - Zero Trust
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# OPA Constraints
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: production-requires-labels
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - "production"
  parameters:
    labels:
    - "app"
    - "team"
    - "environment"

---
# Resource Quotas
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    pods: "50"
    services.loadbalancers: "2"

---
# Limit Range
apiVersion: v1
kind: LimitRange
metadata:
  name: production-limits
  namespace: production
spec:
  limits:
  - default:
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:
      cpu: "250m"
      memory: "256Mi"
    max:
      cpu: "2000m"
      memory: "2Gi"
    min:
      cpu: "100m"
      memory: "128Mi"
    type: Container
```

### Security Scanning Automation

```yaml
# security-scan-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: security-scanner
  namespace: security
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
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
              kubectl get pods -n production -o jsonpath='{.items[*].spec.containers[*].image}' | \
                tr ' ' '\n' | sort -u | \
                while read image; do
                  echo "Scanning $image..."
                  trivy image --severity HIGH,CRITICAL --exit-code 0 --format json $image > /tmp/scan-$RANDOM.json
                  
                  # Check for critical vulnerabilities
                  if grep -q '"Severity":"CRITICAL"' /tmp/scan-*.json; then
                    echo "⚠️ Critical vulnerabilities found in $image"
                    # Send alert
                    curl -X POST -H 'Content-type: application/json' \
                      --data "{\"text\":\"CRITICAL VULNERABILITY in $image\"}" \
                      $SLACK_WEBHOOK
                  fi
                done
          restartPolicy: Never
```

---

## Chapter 5: Monitoring & Alerting Runbooks

### Production Alerts Configuration

```yaml
# production-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: production-critical-alerts
  namespace: monitoring
spec:
  groups:
  - name: production-slo
    rules:
    - alert: ProductionServiceDown
      expr: up{namespace="production"} == 0
      for: 1m
      labels:
        severity: critical
        pager: true
      annotations:
        summary: "Production service {{ $labels.pod }} is down"
        runbook: "https://wiki.mycompany.com/runbooks/service-down"
    
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_server_requests_seconds_count{status=~"5..",namespace="production"}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{namespace="production"}[5m]))
        ) > 0.01
      for: 5m
      labels:
        severity: critical
        pager: true
      annotations:
        summary: "High error rate in production"
        description: "Error rate is {{ $value | humanizePercentage }}"
        runbook: "https://wiki.mycompany.com/runbooks/high-error-rate"
    
    - alert: HighLatency
      expr: |
        histogram_quantile(0.99, 
          sum(rate(http_server_requests_seconds_bucket{namespace="production"}[5m])) by (le)
        ) > 1
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "High latency detected"
        description: "P99 latency is {{ $value }} seconds"
    
    - alert: PodCrashLooping
      expr: |
        kube_pod_container_status_restarts_total{namespace="production"} > 5
      for: 15m
      labels:
        severity: critical
        pager: true
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        runbook: "https://wiki.mycompany.com/runbooks/crash-loop"
    
    - alert: MemoryPressure
      expr: |
        (
          sum(container_memory_working_set_bytes{namespace="production",container!=""}) 
          / 
          sum(container_spec_memory_limit_bytes{namespace="production",container!=""})
        ) > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Memory pressure in production"
        description: "Memory usage is at {{ $value | humanizePercentage }}"
    
    - alert: CertificateExpiring
      expr: |
        (cert_manager_certificate_expiration_timestamp_seconds - time()) / 86400 < 7
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Certificate {{ $labels.name }} expires in {{ $value | humanizeDuration }}"
        runbook: "https://wiki.mycompany.com/runbooks/certificate-expiry"
```

### Incident Response Runbook

```yaml
# incident-response.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: incident-response-runbook
data:
  runbook.md: |
    # Production Incident Response Runbook
    
    ## Severity Levels
    
    ### SEV0 (Critical) - Page immediately
    - Complete service outage
    - Data loss
    - Security breach
    
    ### SEV1 (High) - Page within 15min
    - Partial service degradation
    - High error rates (>5%)
    - Slow performance
    
    ### SEV2 (Medium) - Page during business hours
    - Minor feature broken
    - Warning-level alerts
    - Non-critical component down
    
    ### SEV3 (Low) - Email notification
    - Cosmetic issues
    - Documentation errors
    - Non-urgent alerts
    
    ## Incident Response Steps
    
    ### 1. Detection (0-5 min)
    - Alert received via PagerDuty/Slack
    - Check monitoring dashboards
    - Review recent changes
    
    ### 2. Triage (5-10 min)
    - Determine severity
    - Assign incident commander
    - Create incident channel
    
    ### 3. Mitigation (10-30 min)
    - Rollback if recent deployment
    - Scale up resources
    - Restart failing services
    
    ### 4. Resolution (30-60 min)
    - Implement fix
    - Verify system health
    - Document resolution
    
    ### 5. Post-mortem (24-48 hours)
    - Root cause analysis
    - Action items
    - Improve monitoring
    
    ## Common Incident Runbooks
    
    ### High Error Rate
    
    1. Check recent deployments
       ```bash
       kubectl rollout history deployment/myapp -n production
       ```
    
    2. Rollback if needed
       ```bash
       kubectl rollout undo deployment/myapp -n production
       ```
    
    3. Check pod logs
       ```bash
       kubectl logs -f deployment/myapp -n production --tail=100
       ```
    
    4. Scale up temporarily
       ```bash
       kubectl scale deployment/myapp --replicas=10 -n production
       ```
    
    ### Database Connection Issues
    
    1. Check database pods
       ```bash
       kubectl get pods -l app=postgres -n production
       ```
    
    2. Check database logs
       ```bash
       kubectl logs postgres-0 -n production --tail=100
       ```
    
    3. Verify connectivity
       ```bash
       kubectl exec -it myapp-abc12 -n production -- nc -zv postgres-primary 5432
       ```
    
    4. Restart database if needed
       ```bash
       kubectl delete pod postgres-0 -n production
       ```
    
    ### Performance Degradation
    
    1. Check resource usage
       ```bash
       kubectl top pods -n production
       kubectl top nodes
       ```
    
    2. Check HPA status
       ```bash
       kubectl get hpa -n production
       kubectl describe hpa myapp-hpa -n production
       ```
    
    3. Check for throttling
       ```bash
       kubectl get pods -n production -o json | jq '.items[].status.containerStatuses[].restartCount'
       ```
    
    4. Scale manually if needed
       ```bash
       kubectl scale deployment/myapp --replicas=15 -n production
       ```
    
    ## Communication Templates
    
    ### Initial Alert
    ```
    🚨 [SEV0] Production service down
    Time: {{ time }}
    Impact: 100% of users
    Action: Investigating
    Channel: #incident-{{ timestamp }}
    ```
    
    ### Update Message
    ```
    📍 Update: [SEV0] Production service
    Status: Mitigating
    Action: Rolling back deployment
    ETA: 5 minutes
    ```
    
    ### Resolution
    ```
    ✅ Resolved: [SEV0] Production service
    Duration: 25 minutes
    Root cause: Memory leak in v1.2.3
    Action: Reverted to v1.2.2
    Post-mortem: {{ link }}
    ```
```

---

## Chapter 6: Cost Optimization Strategies

### Cost Optimization Checklist

```yaml
# cost-optimization.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cost-optimization
data:
  strategies: |
    ## Compute Optimization
    
    ### Right-size Resources
    - Use VPA recommendations to find optimal requests
    - Remove unused resources
    - Consider spot instances for batch workloads
    
    ### Node Optimization
    - Use larger instances for better density
    - Enable cluster autoscaler
    - Use reserved instances for baseline capacity
    
    ## Storage Optimization
    
    ### Tiered Storage
    - Hot data: SSD (gp3, pd-ssd)
    - Warm data: Standard HDD
    - Cold data: Archive storage
    
    ### Data Lifecycle
    - Delete old PVCs
    - Compress logs
    - Implement retention policies
    
    ## Network Optimization
    
    ### Reduce Egress
    - Use internal load balancers
    - Cache external requests
    - Compress responses
    
    ### Service Mesh
    - mTLS overhead
    - Sidecar resource consumption
    - Consider if needed
    
    ## Kubernetes Optimization
    
    ### Resource Efficiency
    - Set resource limits (avoid waste)
    - Use HPA to match demand
    - Bin pack pods
    
    ### Cluster Management
    - Delete unused namespaces
    - Remove old container images
    - Prune failed jobs
    
    ## Monitoring Costs
    
    ### Tools
    - Kubecost (recommended)
    - Cloud provider cost explorer
    - Custom metrics
    
    ### Cost Allocation
    - Label all resources
    - Track by team/environment
    - Implement chargeback
    
    ## Cost-Saving Actions
    
    ### Immediate (Today)
    1. Delete unused PVCs
       ```bash
       kubectl get pvc -A | grep Released | awk '{print "kubectl delete pvc " $2 " -n " $1}'
       ```
    
    2. Scale down non-production at night
       ```bash
       kubectl scale deployment/myapp-dev --replicas=0 -n dev
       ```
    
    3. Remove unused load balancers
       ```bash
       kubectl get svc -A | grep LoadBalancer | grep -v production
       ```
    
    ### Weekly
    1. Review VPA recommendations
    2. Check for idle resources
    3. Optimize database size
    
    ### Monthly
    1. Review reserved instance coverage
    2. Analyze cost trends
    3. Right-size node groups
```

### Kubecost Installation

```bash
# Install Kubecost for cost monitoring
helm repo add kubecost https://kubecost.github.io/cost-analyzer
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set prometheus.server.persistentVolume.enabled=false \
  --set prometheus.server.nodeSelector."node-type"="monitoring"

# Access Kubecost UI
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
# http://localhost:9090
```

---

## Chapter 7: Disaster Recovery Runbook

### Complete DR Configuration

```yaml
# dr-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: disaster-recovery
data:
  dr-plan.md: |
    # Disaster Recovery Plan
    
    ## RTO/RPO
    - Recovery Time Objective (RTO): 4 hours
    - Recovery Point Objective (RPO): 1 hour
    
    ## Backup Strategy
    
    ### Velero Backups
    ```bash
    # Daily full backup
    velero schedule create daily-full \
      --schedule="0 2 * * *" \
      --include-namespaces production \
      --ttl 168h \
      --default-volumes-to-restic
    
    # Hourly incremental
    velero schedule create hourly-inc \
      --schedule="0 * * * *" \
      --include-namespaces production \
      --ttl 24h \
      --default-volumes-to-restic
    ```
    
    ## Disaster Scenarios
    
    ### Scenario 1: Single Pod Failure (Self-healing)
    - **Expected behavior**: Pod restarts automatically
    - **RTO**: < 1 minute
    - **Action**: Monitor only
    
    ### Scenario 2: Node Failure
    - **Expected behavior**: Pods reschedule to healthy nodes
    - **RTO**: 5-10 minutes
    - **Action**: 
      1. Check node status
         ```bash
         kubectl get nodes
         kubectl describe node failed-node
         ```
      2. Verify pod rescheduling
         ```bash
         kubectl get pods -n production -o wide
         ```
      3. Cluster autoscaler adds new node
    
    ### Scenario 3: Namespace Deletion
    - **RTO**: 30-60 minutes
    - **Action**:
      1. Restore from Velero backup
         ```bash
         velero backup get
         velero restore create --from-backup daily-full-20240315
         ```
      2. Verify restoration
         ```bash
         kubectl get all -n production
         ```
    
    ### Scenario 4: Regional Outage
    - **RTO**: 2-4 hours
    - **Action**:
      1. Activate DR region
         ```bash
         kubectl config use-context dr-cluster
         ```
      2. Restore from cross-region backups
         ```bash
         velero restore create --from-backup daily-full --storage-location s3-dr-region
         ```
      3. Update DNS
         ```bash
         aws route53 change-resource-record-sets --hosted-zone-id ZONEID --change-batch file://dns-failover.json
         ```
    
    ## DR Testing Schedule
    - Monthly: Test backup restoration
    - Quarterly: Full DR simulation
    - Annually: Cross-region failover test
    
    ## Emergency Contacts
    - Primary: +1-555-123-4567
    - Secondary: +1-555-987-6543
    - Slack: #production-oncall
```

---

## Chapter 8: Documentation Template

### Service Documentation

```yaml
# service-documentation.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: service-documentation
data:
  README.md: |
    # MyApp Production Service
    
    ## Overview
    - **Service**: MyApp API
    - **Version**: 1.2.3
    - **Owner**: Platform Team
    - **Criticality**: High
    
    ## Architecture
    - 5 pods (HPA: 5-20)
    - PostgreSQL primary-replica
    - Redis cache
    - Ingress with TLS
    
    ## URLs
    - Production: https://api.myapp.com
    - Staging: https://staging-api.myapp.com
    - Health: https://api.myapp.com/actuator/health
    
    ## Runbooks
    - [Deployment](runbooks/deployment.md)
    - [Rollback](runbooks/rollback.md)
    - [Scaling](runbooks/scaling.md)
    - [Troubleshooting](runbooks/troubleshooting.md)
    
    ## Dashboards
    - Grafana: https://grafana.mycompany.com/d/myapp
    - Prometheus: https://prometheus.mycompany.com
    - Jaeger: https://jaeger.mycompany.com
    
    ## Alerts
    | Alert | Severity | Runbook |
    |-------|----------|---------|
    | High Error Rate | Critical | [link] |
    | Pod Down | Critical | [link] |
    | High Latency | Warning | [link] |
    
    ## Dependencies
    - PostgreSQL: postgres-primary.production.svc
    - Redis: redis-master.production.svc
    - Kafka: kafka-brokers.production.svc
    
    ## SLIs/SLOs
    | Metric | Target | Current |
    |--------|--------|---------|
    | Availability | 99.9% | 99.95% |
    | Latency (p99) | <500ms | 320ms |
    | Error Rate | <0.1% | 0.05% |
    
    ## Troubleshooting
    
    ### Check pod status
    ```bash
    kubectl get pods -n production -l app=myapp
    ```
    
    ### View logs
    ```bash
    kubectl logs -f deployment/myapp -n production
    ```
    
    ### Check metrics
    ```bash
    kubectl top pods -n production
    ```
    
    ### Connect to database
    ```bash
    kubectl exec -it postgres-0 -n production -- psql
    ```
```

---

## Chapter 9: Production Deployment Script

### One-Command Production Deployment

```bash
#!/bin/bash
# deploy-to-production.sh

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${GREEN}🚀 Starting Production Deployment${NC}"

# Validate environment
echo -e "${YELLOW}📋 Validating production environment...${NC}"
if [ "$1" == "" ]; then
    echo -e "${RED}❌ Please provide version to deploy (e.g., ./deploy.sh v1.2.3)${NC}"
    exit 1
fi

VERSION=$1

# 1. Run tests
echo -e "${YELLOW}🧪 Running tests...${NC}"
./mvnw clean test
if [ $? -ne 0 ]; then
    echo -e "${RED}❌ Tests failed!${NC}"
    exit 1
fi

# 2. Build and push image
echo -e "${YELLOW}🐳 Building Docker image...${NC}"
docker build -t myregistry.com/myapp:$VERSION .
docker push myregistry.com/myapp:$VERSION

# 3. Security scan
echo -e "${YELLOW}🔒 Running security scan...${NC}"
trivy image --severity HIGH,CRITICAL --exit-code 1 myregistry.com/myapp:$VERSION

# 4. Deploy to staging
echo -e "${YELLOW}📦 Deploying to staging...${NC}"
helm upgrade --install myapp ./helm-chart \
  --namespace staging \
  --set image.tag=$VERSION \
  --wait

# 5. Run integration tests
echo -e "${YELLOW}✅ Running integration tests...${NC}"
sleep 30  # Wait for pods to be ready
kubectl run integration-test --image=curlimages/curl --rm -it --restart=Never -n staging -- \
  curl -f http://myapp:8080/actuator/health

# 6. Ask for production approval
echo -e "${YELLOW}⚠️  Ready for production deployment. Continue? (y/n)${NC}"
read -r response
if [[ ! "$response" =~ ^[Yy]$ ]]; then
    echo -e "${RED}❌ Deployment cancelled${NC}"
    exit 1
fi

# 7. Backup current production
echo -e "${YELLOW}💾 Backing up current production...${NC}"
velero backup create pre-deploy-$(date +%Y%m%d-%H%M%S) \
  --include-namespaces production

# 8. Deploy to production
echo -e "${YELLOW}🚀 Deploying to production...${NC}"
helm upgrade --install myapp ./helm-chart \
  --namespace production \
  -f values-prod-final.yaml \
  --set image.tag=$VERSION \
  --atomic \
  --wait \
  --timeout 15m

# 9. Verify deployment
echo -e "${YELLOW}🔍 Verifying production deployment...${NC}"
for i in {1..30}; do
    if curl -s -f https://api.myapp.com/actuator/health | grep -q "UP"; then
        echo -e "${GREEN}✅ Production deployment successful!${NC}"
        break
    fi
    if [ $i -eq 30 ]; then
        echo -e "${RED}❌ Health check failed! Rolling back...${NC}"
        helm rollback myapp -n production
        exit 1
    fi
    sleep 10
done

# 10. Send notification
echo -e "${GREEN}✅ Deployment complete!${NC}"
curl -X POST -H 'Content-type: application/json' \
  --data "{\"text\":\"✅ Production deployment of $VERSION successful\"}" \
  $SLACK_WEBHOOK

echo -e "${GREEN}🎉 All done! Version $VERSION is live in production${NC}"
```

---

## Summary: Production Playbook Checklist

| Area | Status | Verification |
|------|--------|--------------|
| **Application** | ✅ | All tests pass, no critical bugs |
| **Container** | ✅ | Image scanned, no vulnerabilities |
| **Kubernetes** | ✅ | HPA, PDB, resources configured |
| **Security** | ✅ | Network policies, PSA enforced |
| **Monitoring** | ✅ | Metrics, logs, traces working |
| **Alerting** | ✅ | Critical alerts configured |
| **Backup** | ✅ | Daily backups running |
| **DR** | ✅ | DR plan documented and tested |
| **CI/CD** | ✅ | Pipeline automated |
| **Documentation** | ✅ | Runbooks, playbooks ready |
| **Cost** | ✅ | Optimization implemented |

## Final Deployment Command

```bash
# One command to rule them all!
./deploy-to-production.sh v1.0.0
```

---

## Congratulations! 🎉

You've completed the entire 10-part Kubernetes mastery series!

### What You've Learned

1. ✅ **Spring Boot + Docker + Helm Basics** - Containerized your first app
2. ✅ **Kubernetes Core Concepts** - Pods, Deployments, Services
3. ✅ **Helm Deep Dive** - Templates, subcharts, hooks
4. ✅ **Container Registry & CI/CD** - Automated pipelines
5. ✅ **Networking & Ingress** - TLS, DNS, routing
6. ✅ **Storage & Stateful Apps** - Databases, StatefulSets
7. ✅ **Security Deep Dive** - RBAC, OPA, zero-trust
8. ✅ **Observability** - Metrics, logs, traces
9. ✅ **Scaling & Resilience** - HPA, chaos, DR
10. ✅ **Production Playbook** - Complete deployment guide

### Next Steps

- **Practice**: Deploy your own apps using these patterns
- **Contribute**: Share your learnings and improvements
- **Certify**: Consider CKA/CKAD certification
- **Explore**: Service mesh (Istio), Serverless (Knative), GitOps (ArgoCD)

### Keep Learning

- Join Kubernetes community (Slack, Discord)
- Follow CNCF projects
- Attend KubeCon
- Build side projects
