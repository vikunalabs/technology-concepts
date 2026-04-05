# Part 10: Production Playbook — From Development to Production

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Expert
> **Prerequisites:** Completed Parts 1–9 — this part synthesises everything
> **Time to complete:** 4–5 hours
> **What this part is:** A complete production reference. Every chapter is a working artefact — a checklist, a values file, a pipeline, a runbook, a script — that you copy, adapt, and use directly. This is the playbook you reach for when something goes wrong at 2 AM, when you're onboarding a new service, and when you're trying to justify a budget for the infrastructure team.

---

## What This Part Covers

The previous nine parts taught concepts and techniques individually. This part assembles them into a coherent whole — the production system as it should actually look when everything is working together correctly.

There is no new theory here. Everything referenced in this part was covered in a previous chapter. What is new is the integration: seeing how the security hardening from Part 7 fits into the same values file as the observability configuration from Part 8, and how both feed into the CI/CD pipeline from Part 4. The production system is not nine separate concerns — it is one.

Use this part as a reference. When deploying a new service, start from Chapter 2's values template. When an alert fires, reach for Chapter 5's runbooks. When something breaks badly, the DR playbook in Chapter 7 tells you exactly what to do.

---

## Chapter 1: Production Readiness Framework

### The Minimum Viable Production Checklist

Before going live, these 20 items are non-negotiable. A system missing any one of them has a known gap that will eventually cause an incident.

**Application:**
- [ ] All API endpoints return appropriate HTTP status codes (not 500 for client errors)
- [ ] Graceful shutdown configured: `spring.shutdown=graceful`, `preStop` sleep, `terminationGracePeriodSeconds: 60`
- [ ] All three health probes configured: `startupProbe`, `readinessProbe`, `livenessProbe`
- [ ] Structured JSON logging to stdout (not to files)
- [ ] Trace ID included in all log lines

**Container:**
- [ ] Multi-stage Dockerfile: build stage separate from runtime stage
- [ ] Non-root user in runtime image
- [ ] JVM flags set: `-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0`
- [ ] Image tagged with immutable version (not `latest`)
- [ ] Image scanned for CRITICAL CVEs — none present

**Kubernetes:**
- [ ] `resources.requests` and `resources.limits` set on every container
- [ ] `readOnlyRootFilesystem: true` with `emptyDir` volumes for temp directories
- [ ] `runAsNonRoot: true` and `automountServiceAccountToken: false`
- [ ] Pod Disruption Budget in place
- [ ] Network policies: default-deny-all + DNS allow + explicit allow rules

**Operations:**
- [ ] Prometheus `ServiceMonitor` deployed — app appears in Prometheus targets
- [ ] At minimum: error rate alert and pod crash-loop alert configured
- [ ] Helm chart version-controlled in Git, deployed via CI/CD (not manual `helm upgrade`)
- [ ] Velero backup schedule active for the namespace
- [ ] Runbook exists for: app down, high error rate, database unreachable

### The Full Production Checklist

**Application layer:**
- [ ] Health check endpoints test actual dependencies (database, cache) for readiness
- [ ] Circuit breakers and timeouts configured for all external calls
- [ ] Idempotent retry logic for non-idempotent operations
- [ ] Database connection pool sized correctly (not default)
- [ ] Sensitive data never logged (passwords, tokens, PII)
- [ ] Correlation IDs propagated across service boundaries

**Container layer:**
- [ ] `.dockerignore` excludes `target/`, `.git/`, `*.md`
- [ ] `allowPrivilegeEscalation: false`
- [ ] `capabilities: drop: [ALL]`
- [ ] `seccompProfile: type: RuntimeDefault`
- [ ] Base image pinned to a specific digest (not just tag)
- [ ] Image signed with Cosign

**Kubernetes layer:**
- [ ] Separate namespace per environment (dev/staging/prod)
- [ ] ResourceQuota on production namespace
- [ ] LimitRange with default requests/limits
- [ ] Pod anti-affinity: prefer spreading across nodes
- [ ] Topology spread constraints: spread across availability zones
- [ ] HPA configured with appropriate min/max and custom metric (not just CPU)

**Security layer:**
- [ ] Dedicated ServiceAccount with IRSA/Workload Identity annotation
- [ ] RBAC: principle of least privilege — no wildcard permissions
- [ ] Secrets managed via External Secrets Operator or Vault (not plain K8s Secrets)
- [ ] OPA Gatekeeper: required labels + allowed registries constraints active
- [ ] Pod Security Standards: `enforce: restricted` on production namespace
- [ ] Falco installed and alerts routing to incident channel

**Observability layer:**
- [ ] Custom business metrics (orders processed, payment failures) — not just infrastructure metrics
- [ ] P99 latency dashboard panel visible to the team
- [ ] SLO defined and error budget dashboard deployed
- [ ] Log retention policy defined (cost vs compliance)
- [ ] Alert runbooks linked from `annotations.runbook` in PrometheusRule

**CI/CD layer:**
- [ ] All tests pass before any deployment
- [ ] `helm diff` runs as PR check — cluster changes visible in code review
- [ ] Manual approval gate before production deployment
- [ ] `--atomic` on all production Helm upgrades (auto-rollback on failure)
- [ ] Semantic versioning and automated changelog generation

**Cost layer:**
- [ ] All resources tagged with `team`, `environment`, `service` labels
- [ ] VPA recommendations reviewed and applied quarterly
- [ ] Non-production environments scaled to zero outside business hours
- [ ] Unused PVCs and load balancers audited monthly

**Documentation layer:**
- [ ] Service README exists with architecture diagram, URLs, runbooks, SLOs
- [ ] On-call rotation documented
- [ ] Post-mortem template available and used after every SEV1+

### Pre-Production Validation Script

Run this before every production deployment:

```bash
#!/bin/bash
# validate-production.sh
set -euo pipefail

NAMESPACE="production"
RELEASE="myapp-prod"
CHART="./helm-chart"
VALUES_FILE="values-prod.yaml"

echo "══════════════════════════════════════════════"
echo "  Production Deployment Validation"
echo "══════════════════════════════════════════════"
ERRORS=0

# ── Helm chart validation ──────────────────────────────────────────────────
echo ""
echo "▶ Helm lint..."
if helm lint "$CHART" -f "$VALUES_FILE" --quiet; then
  echo "  ✓ Chart syntax valid"
else
  echo "  ✗ Chart lint failed"
  ERRORS=$((ERRORS + 1))
fi

# ── Preview changes ────────────────────────────────────────────────────────
echo ""
echo "▶ Helm diff (changes to cluster)..."
helm diff upgrade "$RELEASE" "$CHART" \
  -f "$VALUES_FILE" \
  --namespace "$NAMESPACE" \
  --no-color \
  --allow-unreleased 2>/dev/null || true

# ── Kubernetes manifest validation ────────────────────────────────────────
echo ""
echo "▶ kubectl diff (current vs incoming)..."
helm template "$RELEASE" "$CHART" -f "$VALUES_FILE" --namespace "$NAMESPACE" \
  | kubectl diff -f - -n "$NAMESPACE" || echo "  (differences shown above)"

# ── Image exists in registry ───────────────────────────────────────────────
echo ""
echo "▶ Checking image exists..."
IMAGE=$(helm show values "$CHART" -f "$VALUES_FILE" | \
  grep -E "repository:|tag:" | \
  awk -F': ' '{print $2}' | tr -d '"' | paste -s -d':')
if docker manifest inspect "$IMAGE" > /dev/null 2>&1; then
  echo "  ✓ Image found: $IMAGE"
else
  echo "  ✗ Image not found: $IMAGE"
  ERRORS=$((ERRORS + 1))
fi

# ── Cluster connectivity ───────────────────────────────────────────────────
echo ""
echo "▶ Cluster connectivity..."
if kubectl cluster-info --request-timeout=5s > /dev/null 2>&1; then
  echo "  ✓ Cluster reachable"
else
  echo "  ✗ Cannot reach cluster"
  ERRORS=$((ERRORS + 1))
fi

# ── Namespace exists ───────────────────────────────────────────────────────
if kubectl get namespace "$NAMESPACE" > /dev/null 2>&1; then
  echo "  ✓ Namespace exists: $NAMESPACE"
else
  echo "  ✗ Namespace missing: $NAMESPACE"
  ERRORS=$((ERRORS + 1))
fi

# ── Required secrets exist ─────────────────────────────────────────────────
echo ""
echo "▶ Required secrets..."
for SECRET in regcred db-credentials external-secrets-token; do
  if kubectl get secret "$SECRET" -n "$NAMESPACE" > /dev/null 2>&1; then
    echo "  ✓ Secret exists: $SECRET"
  else
    echo "  ✗ Secret missing: $SECRET"
    ERRORS=$((ERRORS + 1))
  fi
done

# ── Velero backup ─────────────────────────────────────────────────────────
echo ""
echo "▶ Creating pre-deployment backup..."
BACKUP_NAME="pre-deploy-$(date +%Y%m%d-%H%M%S)"
if velero backup create "$BACKUP_NAME" \
  --include-namespaces "$NAMESPACE" \
  --wait --timeout 10m > /dev/null 2>&1; then
  echo "  ✓ Backup created: $BACKUP_NAME"
else
  echo "  ⚠ Backup failed — proceeding anyway (check Velero)"
fi

# ── Summary ───────────────────────────────────────────────────────────────
echo ""
echo "══════════════════════════════════════════════"
if [ "$ERRORS" -eq 0 ]; then
  echo "  ✓ All checks passed — ready to deploy"
  exit 0
else
  echo "  ✗ $ERRORS check(s) failed — fix before deploying"
  exit 1
fi
```

---

## Chapter 2: Complete Production Helm Values

This is the reference values file for a production Spring Boot service. Every field is present and commented. Copy this, rename it `values-prod.yaml`, and fill in your specifics.

```yaml
# values-prod.yaml
# Complete production values for a Spring Boot service.
# Every field explicitly set — no reliance on chart defaults.

# ─────────────────────────────────────────────────────
# Replica and Image
# ─────────────────────────────────────────────────────
replicaCount: 3    # Minimum 3 for zone redundancy (3 AZs, 1 pod per AZ minimum)

image:
  repository: 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp
  tag: "v1.3.0"       # Always pinned — CI/CD updates this field
  pullPolicy: Always  # Re-pull on every restart — ensures node has latest tag

imagePullSecrets:
  - name: regcred

# ─────────────────────────────────────────────────────
# Service Account (with IRSA for AWS)
# ─────────────────────────────────────────────────────
serviceAccount:
  create: true
  name: myapp-sa
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/myapp-production
  automountServiceAccountToken: false   # Never mount default K8s API token

# ─────────────────────────────────────────────────────
# Resource Requests and Limits
# ─────────────────────────────────────────────────────
resources:
  requests:
    cpu: "500m"       # Scheduler guarantees this — used by HPA for utilisation calculation
    memory: "512Mi"   # Actual observed idle usage from VPA recommendations
  limits:
    cpu: "2000m"      # 4× request — allow bursting, prevent runaway
    memory: "1Gi"     # 2× request — JVM headroom above heap

# ─────────────────────────────────────────────────────
# Pod Security Context
# ─────────────────────────────────────────────────────
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  seccompProfile:
    type: RuntimeDefault

# ─────────────────────────────────────────────────────
# Container Security Context
# ─────────────────────────────────────────────────────
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true   # Requires emptyDir volumes below for temp dirs
  capabilities:
    drop: ["ALL"]

# emptyDir volumes for paths the app writes to at runtime
extraVolumes:
  - name: tmp
    emptyDir: {}
  - name: spring-tmp
    emptyDir:
      sizeLimit: 200Mi

extraVolumeMounts:
  - name: tmp
    mountPath: /tmp
  - name: spring-tmp
    mountPath: /app/tmp

# ─────────────────────────────────────────────────────
# Health Probes
# ─────────────────────────────────────────────────────
probes:
  startup:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    failureThreshold: 30     # 30 × 10s = 5 minutes max startup window
    periodSeconds: 10
  readiness:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    periodSeconds: 5
    failureThreshold: 3      # Remove from Service after 15s of failure
    timeoutSeconds: 3
  liveness:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    periodSeconds: 10
    failureThreshold: 3      # Restart after 30s of liveness failure
    timeoutSeconds: 5

# ─────────────────────────────────────────────────────
# Graceful Shutdown
# ─────────────────────────────────────────────────────
terminationGracePeriodSeconds: 60   # > preStop(15s) + spring shutdown(30s)
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 15"]

# ─────────────────────────────────────────────────────
# Environment Variables
# ─────────────────────────────────────────────────────
env:
  SPRING_PROFILES_ACTIVE: "prod"
  JAVA_OPTS: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

# ─────────────────────────────────────────────────────
# Service
# ─────────────────────────────────────────────────────
service:
  type: ClusterIP
  port: 8080
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/actuator/prometheus"

# ─────────────────────────────────────────────────────
# Ingress
# ─────────────────────────────────────────────────────
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    external-dns.alpha.kubernetes.io/ttl: "300"
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-myapp-com-tls
      hosts:
        - api.myapp.com

# ─────────────────────────────────────────────────────
# Autoscaling
# ─────────────────────────────────────────────────────
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60

# ─────────────────────────────────────────────────────
# Pod Disruption Budget
# ─────────────────────────────────────────────────────
podDisruptionBudget:
  enabled: true
  maxUnavailable: "20%"

# ─────────────────────────────────────────────────────
# Deployment Update Strategy
# ─────────────────────────────────────────────────────
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0

# ─────────────────────────────────────────────────────
# Pod Placement
# ─────────────────────────────────────────────────────
# Spread pods across availability zones
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: myapp

# Prefer different nodes (soft anti-affinity — allows scheduling if needed)
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: kubernetes.io/hostname

# ─────────────────────────────────────────────────────
# Monitoring
# ─────────────────────────────────────────────────────
monitoring:
  serviceMonitor:
    enabled: true
    interval: 30s
    scrapeTimeout: 10s
    labels:
      release: monitoring    # Must match kube-prometheus-stack selector

  prometheusRules:
    enabled: true
    labels:
      release: monitoring

# ─────────────────────────────────────────────────────
# Required Labels (enforced by OPA Gatekeeper)
# ─────────────────────────────────────────────────────
podLabels:
  team: platform
  environment: production
  cost-center: engineering

# ─────────────────────────────────────────────────────
# Priority Class (ensures production pods aren't evicted first)
# ─────────────────────────────────────────────────────
priorityClassName: high-priority   # Create with: kubectl apply -f priorityclass.yaml
```

---

## Chapter 3: Complete CI/CD Pipeline

This pipeline runs on every merge to `main` and covers the full path from validation to production deployment with post-deployment verification.

```yaml
# .github/workflows/production-deploy.yml
name: Production Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      skip_staging:
        description: 'Skip staging (emergency hotfix only)'
        type: boolean
        default: false

env:
  REGISTRY: 123456789.dkr.ecr.us-east-1.amazonaws.com
  IMAGE_NAME: myapp
  CHART_PATH: ./helm-chart

jobs:
  # ──────────────────────────────────────────────────────────────────────────
  # Job 1: Validate — runs on every push
  # ──────────────────────────────────────────────────────────────────────────
  validate:
    name: Validate
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Validate semantic version
      run: |
        # Enforce conventional commits format
        COMMIT_MSG=$(git log -1 --pretty=%B)
        if ! echo "$COMMIT_MSG" | grep -qE \
          "^(feat|fix|docs|chore|refactor|test|ci|build|perf|revert)(\(.+\))?(!)?: .+"; then
          echo "Commit message does not follow conventional commits format"
          echo "Expected: type(scope): description"
          echo "Got: $COMMIT_MSG"
          exit 1
        fi
        echo "✓ Commit message format valid"

    - name: Helm lint
      uses: azure/setup-helm@v4
      with:
        version: '3.13.0'
    - run: helm lint ${{ env.CHART_PATH }} -f values-prod.yaml

  # ──────────────────────────────────────────────────────────────────────────
  # Job 2: Build and Test
  # ──────────────────────────────────────────────────────────────────────────
  build-test:
    name: Build & Test
    runs-on: ubuntu-latest
    needs: validate
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven
    - name: Run tests
      run: ./mvnw verify
    - name: Upload test results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: test-results
        path: target/surefire-reports/

  # ──────────────────────────────────────────────────────────────────────────
  # Job 3: Security Scan
  # ──────────────────────────────────────────────────────────────────────────
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-test
    permissions:
      id-token: write
      contents: read
      security-events: write
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
        aws-region: us-east-1

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Set image tag
      id: tag
      run: echo "tag=git-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

    - name: Build image (no push yet — scan first)
      uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.tag.outputs.tag }}
        load: true    # Load into local Docker daemon for scanning

    - name: Scan with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.tag.outputs.tag }}
        format: sarif
        output: trivy-results.sarif
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
        ignore-unfixed: true

    - name: Upload Trivy results to Security tab
      if: always()
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: trivy-results.sarif

  # ──────────────────────────────────────────────────────────────────────────
  # Job 4: Build and Push (only after security scan passes)
  # ──────────────────────────────────────────────────────────────────────────
  build-push:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: security-scan
    outputs:
      image-tag: ${{ steps.tag.outputs.tag }}
    permissions:
      id-token: write
      contents: read
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
        aws-region: us-east-1

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Set image tag
      id: tag
      run: echo "tag=git-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

    - uses: docker/setup-buildx-action@v3

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.tag.outputs.tag }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Sign image with Cosign
      uses: sigstore/cosign-installer@v3
    - run: |
        cosign sign --yes \
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.tag.outputs.tag }}

  # ──────────────────────────────────────────────────────────────────────────
  # Job 5: Deploy to Staging
  # ──────────────────────────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    needs: build-push
    if: ${{ !inputs.skip_staging }}
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4
      with:
        version: '3.13.0'

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
        aws-region: us-east-1

    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name my-cluster-staging --region us-east-1

    - name: Helm diff
      run: |
        helm plugin install https://github.com/databus23/helm-diff || true
        helm dependency update ${{ env.CHART_PATH }}
        helm diff upgrade myapp-staging ${{ env.CHART_PATH }} \
          -f values.yaml -f values-staging.yaml \
          --set image.tag=${{ needs.build-push.outputs.image-tag }} \
          --namespace staging --allow-unreleased || true

    - name: Deploy to staging
      run: |
        helm upgrade --install myapp-staging ${{ env.CHART_PATH }} \
          -f values.yaml -f values-staging.yaml \
          --set image.tag=${{ needs.build-push.outputs.image-tag }} \
          --namespace staging \
          --create-namespace \
          --atomic \
          --wait \
          --timeout 10m

    - name: Smoke tests
      run: helm test myapp-staging -n staging --timeout 5m

    - name: Integration tests
      run: |
        BASE="https://staging.myapp.com"
        curl -sf "$BASE/actuator/health" | jq -e '.status == "UP"'
        curl -sf "$BASE/api/hello" | grep -q "Hello"
        echo "✓ Integration tests passed"

  # ──────────────────────────────────────────────────────────────────────────
  # Job 6: Deploy to Production — requires manual approval
  # Configured via GitHub Settings → Environments → production → Required reviewers
  # ──────────────────────────────────────────────────────────────────────────
  deploy-production:
    name: Deploy → Production
    runs-on: ubuntu-latest
    needs: [build-push, deploy-staging]
    if: always() && needs.build-push.result == 'success' && (needs.deploy-staging.result == 'success' || needs.deploy-staging.result == 'skipped')
    environment:
      name: production
      url: https://api.myapp.com
    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4
      with:
        version: '3.13.0'

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy-prod
        aws-region: us-east-1

    - name: Update kubeconfig
      run: aws eks update-kubeconfig --name my-cluster-prod --region us-east-1

    - name: Pre-deployment Velero backup
      run: |
        velero backup create "pre-deploy-$(date +%Y%m%d-%H%M%S)" \
          --include-namespaces production \
          --wait --timeout 10m
        echo "✓ Pre-deployment backup complete"

    - name: Helm diff — final preview
      run: |
        helm plugin install https://github.com/databus23/helm-diff || true
        helm dependency update ${{ env.CHART_PATH }}
        echo "=== Changes to production ==="
        helm diff upgrade myapp-prod ${{ env.CHART_PATH }} \
          -f values.yaml -f values-prod.yaml \
          --set image.tag=${{ needs.build-push.outputs.image-tag }} \
          --namespace production

    - name: Deploy to production
      run: |
        helm upgrade --install myapp-prod ${{ env.CHART_PATH }} \
          -f values.yaml -f values-prod.yaml \
          --set image.tag=${{ needs.build-push.outputs.image-tag }} \
          --namespace production \
          --create-namespace \
          --atomic \
          --wait \
          --timeout 15m

    - name: Post-deployment health verification
      run: |
        echo "Waiting for deployment to stabilise..."
        sleep 30

        MAX_RETRIES=12
        RETRY_INTERVAL=15
        for i in $(seq 1 $MAX_RETRIES); do
          STATUS=$(curl -sf https://api.myapp.com/actuator/health/readiness \
            -o /dev/null -w "%{http_code}" 2>/dev/null || echo "000")
          if [ "$STATUS" = "200" ]; then
            echo "✓ Health check passed (attempt $i)"
            break
          fi
          echo "Attempt $i/$MAX_RETRIES: status=$STATUS, retrying in ${RETRY_INTERVAL}s..."
          sleep $RETRY_INTERVAL
          if [ "$i" = "$MAX_RETRIES" ]; then
            echo "✗ Health check failed after $MAX_RETRIES attempts"
            echo "Rolling back..."
            helm rollback myapp-prod -n production
            exit 1
          fi
        done

    - name: Validate metrics via Prometheus API
      run: |
        # Query Prometheus to confirm error rate is not elevated post-deploy
        PROM_URL="http://monitoring-kube-prometheus-prometheus.monitoring:9090"

        # Port-forward Prometheus for this check
        kubectl port-forward svc/monitoring-kube-prometheus-prometheus \
          9090:9090 -n monitoring &
        sleep 5

        ERROR_RATE=$(curl -sf \
          "localhost:9090/api/v1/query?query=sum(rate(http_server_requests_seconds_count{namespace=%22production%22,status=~%225..%22}[5m]))/sum(rate(http_server_requests_seconds_count{namespace=%22production%22}[5m]))*100" \
          | jq -r '.data.result[0].value[1] // "0"')

        echo "Current error rate: ${ERROR_RATE}%"
        if (( $(echo "$ERROR_RATE > 5" | bc -l) )); then
          echo "✗ Error rate is ${ERROR_RATE}% — exceeds 5% threshold"
          helm rollback myapp-prod -n production
          exit 1
        fi
        echo "✓ Error rate acceptable: ${ERROR_RATE}%"

    - name: Notify Slack — success
      if: success()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "✅ *myapp* deployed to production",
            "attachments": [{
              "color": "good",
              "fields": [
                {"title": "Version", "value": "${{ needs.build-push.outputs.image-tag }}", "short": true},
                {"title": "Deployed by", "value": "${{ github.actor }}", "short": true},
                {"title": "Commit", "value": "${{ github.event.head_commit.message }}", "short": false},
                {"title": "Run", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

    - name: Notify Slack — failure
      if: failure()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "❌ *myapp* production deployment FAILED — automatic rollback triggered",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Version attempted", "value": "${{ needs.build-push.outputs.image-tag }}", "short": true},
                {"title": "Run", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```


## Chapter 4: Security Hardening — Namespace Bootstrap

Apply this to every production namespace when it is created. These are the baseline controls that every namespace requires before any workloads run.

```bash
#!/bin/bash
# bootstrap-namespace.sh — run once per namespace before any workloads deploy
NAMESPACE="${1:-production}"
echo "Bootstrapping security controls for namespace: $NAMESPACE"

# ── Step 1: Create namespace with Pod Security Standards ──────────────────
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: $NAMESPACE
  labels:
    # enforce: rejects non-compliant pods at admission
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # audit: logs violations for pods that would be rejected
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    # Required by ExternalDNS, network policies, Prometheus scraping
    kubernetes.io/metadata.name: $NAMESPACE
    environment: production
EOF

# ── Step 2: ResourceQuota — prevent namespace from consuming entire cluster ──
kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: $NAMESPACE
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    persistentvolumeclaims: "20"
    services: "30"
    count/deployments.apps: "30"
EOF

# ── Step 3: LimitRange — default requests/limits for pods that don't specify ──
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: $NAMESPACE
spec:
  limits:
  - type: Container
    default:          # Applied if container omits limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:   # Applied if container omits requests
      cpu: "100m"
      memory: "128Mi"
    max:              # No container may exceed these
      cpu: "4"
      memory: "8Gi"
    min:              # No container may request less than these
      cpu: "10m"
      memory: "32Mi"
EOF

# ── Step 4: Default deny-all network policy ──────────────────────────────
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: $NAMESPACE
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: $NAMESPACE
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF

echo "✓ Namespace $NAMESPACE bootstrapped with security controls"
```

### OPA Gatekeeper Constraints for Production

Apply these cluster-wide constraints after Gatekeeper is installed:

```bash
# Required labels — every Deployment must have team and environment labels
kubectl apply -f - <<EOF
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: deployments-must-have-team-label
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: ["apps"]
      kinds: ["Deployment", "StatefulSet"]
    namespaces: [production, staging]
  parameters:
    labels: [team, environment]
---
# Allowed registries — block images from untrusted sources
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-registries-production
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces: [production]
  parameters:
    repos:
    - "123456789.dkr.ecr.us-east-1.amazonaws.com/"
    - "gcr.io/my-project/"
EOF
```

### Automated Security Scanning CronJob

```yaml
# security-scan-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-security-scan
  namespace: security
spec:
  schedule: "0 6 * * 1"   # Every Monday at 6 AM UTC
  successfulJobsHistoryLimit: 4
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: security-scanner-sa
          containers:
          - name: scanner
            image: aquasec/trivy:latest
            command:
            - /bin/sh
            - -c
            - |
              # Scan all unique images in the production namespace
              IMAGES=$(kubectl get pods -n production \
                -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
                | sort -u)

              FAILED=0
              REPORT=""

              while IFS= read -r IMAGE; do
                RESULT=$(trivy image --severity CRITICAL --ignore-unfixed \
                  --quiet --format json "$IMAGE" 2>/dev/null)
                VULN_COUNT=$(echo "$RESULT" | jq '[.Results[]?.Vulnerabilities[]?] | length' 2>/dev/null || echo "0")

                if [ "$VULN_COUNT" -gt "0" ]; then
                  FAILED=1
                  REPORT="${REPORT}\n⚠️  *${IMAGE}*: ${VULN_COUNT} critical CVEs"
                fi
              done <<< "$IMAGES"

              if [ "$FAILED" = "1" ]; then
                # Send alert to Slack
                curl -s -X POST "$SLACK_WEBHOOK" \
                  -H "Content-type: application/json" \
                  --data "{\"text\":\"🔴 *Weekly Security Scan*: Critical CVEs found in production images:\n${REPORT}\"}"
                exit 1
              else
                echo "✓ No critical CVEs found in production images"
              fi
            env:
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: slack-credentials
                  key: webhook-url
```

---

## Chapter 5: Monitoring & Alerting Runbooks

### Incident Severity Levels

| Severity | Definition | Response time | Examples |
|----------|-----------|---------------|---------|
| **SEV0** | Complete service outage — all users affected | Immediate, all hands | Production down, data loss |
| **SEV1** | Major degradation — significant user impact | < 15 minutes | Error rate > 10%, P99 > 2s |
| **SEV2** | Partial degradation — some users or features affected | < 1 hour | Single region degraded, one feature broken |
| **SEV3** | Minor issue — no current user impact | Next business day | Certificate expiring in 7 days, capacity warning |

### Complete Alert Rules

```yaml
# prometheusrule-production.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: production-alerts
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: production.critical
    rules:
    - alert: ServiceDown
      expr: absent(up{job=~"myapp.*", namespace="production"} == 1)
      for: 1m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Service is down: {{ $labels.job }}"
        description: "No healthy instances of {{ $labels.job }} in production for 1 minute."
        runbook: "https://wiki.mycompany.com/runbooks/service-down"

    - alert: HighErrorRate
      expr: |
        sum(rate(http_server_requests_seconds_count{namespace="production",status=~"5.."}[5m]))
        / sum(rate(http_server_requests_seconds_count{namespace="production"}[5m])) * 100 > 5
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Error rate {{ $value | humanizePercentage }} in production"
        runbook: "https://wiki.mycompany.com/runbooks/high-error-rate"

    - alert: HighP99Latency
      expr: |
        histogram_quantile(0.99,
          sum by (le) (rate(http_server_requests_seconds_bucket{namespace="production"}[5m]))
        ) > 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "P99 latency is {{ $value | humanizeDuration }}"
        runbook: "https://wiki.mycompany.com/runbooks/high-latency"

    - alert: PodCrashLooping
      expr: increase(kube_pod_container_status_restarts_total{namespace="production"}[15m]) > 3
      for: 0m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        runbook: "https://wiki.mycompany.com/runbooks/crash-loop"

  - name: production.warning
    rules:
    - alert: MemoryPressure
      expr: |
        container_memory_working_set_bytes{namespace="production",container!=""}
        / container_spec_memory_limit_bytes{namespace="production",container!=""} * 100 > 85
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Pod {{ $labels.pod }} memory at {{ $value | humanizePercentage }} of limit"
        runbook: "https://wiki.mycompany.com/runbooks/memory-pressure"

    - alert: CertificateExpiringSoon
      expr: certmanager_certificate_expiration_timestamp_seconds - time() < 7 * 24 * 3600
      for: 1h
      labels:
        severity: warning
      annotations:
        summary: "Certificate {{ $labels.name }} expires in less than 7 days"
        runbook: "https://wiki.mycompany.com/runbooks/certificate-expiry"

    - alert: HPANearMaxReplicas
      expr: |
        kube_horizontalpodautoscaler_status_current_replicas{namespace="production"}
        / kube_horizontalpodautoscaler_spec_max_replicas{namespace="production"} > 0.85
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "HPA {{ $labels.horizontalpodautoscaler }} at {{ $value | humanizePercentage }} of max replicas"
        description: "May hit scaling ceiling during traffic spikes — review maxReplicas."

    - alert: PVCUsageHigh
      expr: |
        kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100 > 80
      for: 30m
      labels:
        severity: warning
      annotations:
        summary: "PVC {{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
        runbook: "https://wiki.mycompany.com/runbooks/disk-full"
```

### Incident Response Runbooks

**Runbook: Service Down / High Error Rate**
```
DETECTION: ServiceDown or HighErrorRate alert fires

IMMEDIATE STEPS (first 5 minutes):
1. Acknowledge the alert (prevents duplicate pages)
   kubectl get pods -n production
   kubectl get events -n production --sort-by='.lastTimestamp' | tail -20

2. Check if a recent deployment caused this
   helm history myapp-prod -n production
   git log --oneline -5

3. If recent deployment: ROLLBACK IMMEDIATELY
   helm rollback myapp-prod -n production
   # Wait 2 minutes, check error rate in Grafana

4. If not a deployment issue — check pod health
   kubectl logs deployment/myapp -n production --tail=50
   kubectl describe pod $(kubectl get pod -n production -l app=myapp -o name | head -1) -n production

INVESTIGATION:
5. Check external dependencies
   kubectl exec -it <pod> -n production -- curl -sf http://postgres-service:5432 || echo "DB unreachable"
   kubectl exec -it <pod> -n production -- curl -sf http://redis-service:6379 || echo "Cache unreachable"

6. Check resource pressure
   kubectl top pods -n production
   kubectl top nodes

ESCALATION:
- If not resolved in 15 minutes: SEV1 → wake up second on-call
- If data loss suspected: SEV0 → wake up everyone
```

**Runbook: Database Connection Issues**
```
DETECTION: Application logs show "Connection refused" or "Connection pool exhausted"

STEPS:
1. Check if postgres pods are healthy
   kubectl get pods -n production -l app=postgres
   kubectl logs postgres-0 -n production --tail=30

2. Check if the Service has endpoints
   kubectl get endpoints postgres-service -n production
   # Should show pod IPs — if empty, Service selector is broken

3. Test connectivity from application pod
   kubectl exec -it <app-pod> -n production -- nc -zv postgres-service 5432

4. Check connection pool metrics in Grafana
   Query: hikaricp_connections_active{namespace="production"}
   Query: hikaricp_connections_pending{namespace="production"}

5. If pool exhausted — temporary fix: rolling restart
   kubectl rollout restart deployment/myapp -n production

6. If postgres is down:
   kubectl describe statefulset postgres -n production
   kubectl get pvc -n production  # Check PVC is bound
   kubectl logs postgres-0 -n production --previous  # Check last crash reason
```

**Runbook: Performance Degradation**
```
DETECTION: HighP99Latency alert, or user reports of slowness

STEPS:
1. Identify slow endpoints
   Grafana → Spring Boot dashboard → endpoint latency breakdown
   Tempo → search for traces with duration > 500ms

2. Check if it's resource-related
   kubectl top pods -n production
   # CPU throttling?
   kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/production/pods | jq .

3. Check HPA — is it scaling to handle load?
   kubectl get hpa -n production
   kubectl describe hpa myapp-hpa -n production

4. Check if Cluster Autoscaler is adding nodes
   kubectl get nodes -w
   kubectl logs -n kube-system deployment/cluster-autoscaler | tail -20

5. If CPU throttling: temporary fix
   kubectl set resources deployment/myapp -n production \
     --requests=cpu=1000m --limits=cpu=4000m
   # Then update Helm values permanently

6. Open distributed trace for slow requests in Tempo
   Find the slow span — is it DB, cache, external API, or application logic?
```

### Communication Templates

**Initial alert notification (Slack):**
```
🔴 *INCIDENT — SEV1*
*Service:* myapp (production)
*Symptoms:* Error rate 12% (normal: <0.1%)
*Started:* 14:32 UTC
*Impact:* ~15% of checkout requests failing
*IC:* @alice
*Bridge:* https://meet.google.com/xxx-yyy-zzz
*Status page:* Updated — investigating
```

**Update (every 30 minutes until resolved):**
```
🟡 *INCIDENT UPDATE — 14:58 UTC*
*Status:* Investigating
*Findings:* Database connection pool exhausted — root cause TBD
*Actions taken:* Rolled back v1.3.2 → v1.3.1 (no improvement), restarted pods
*Next update:* 15:30 UTC or sooner if resolved
```

**Resolution:**
```
✅ *INCIDENT RESOLVED — 15:15 UTC*
*Duration:* 43 minutes
*Root cause:* Connection pool misconfiguration in v1.3.2 (max-pool-size set to 2)
*Resolution:* Reverted to v1.3.1
*Fix:* Correct pool size in v1.3.3, deploy in next maintenance window
*Post-mortem:* Friday 10 AM — @alice to lead
```

---

## Chapter 6: Cost Optimisation

### Right-Sizing with VPA

The fastest way to reduce costs without affecting reliability is right-sizing — removing excess CPU and memory reservations that inflate your node requirements.

```bash
# Step 1: Install VPA in Off mode across all production workloads
for DEPLOY in $(kubectl get deployment -n production -o name); do
  NAME=$(basename $DEPLOY)
  kubectl apply -f - <<EOF
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: ${NAME}-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${NAME}
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      controlledResources: ["memory"]
      minAllowed:
        memory: "128Mi"
      maxAllowed:
        memory: "4Gi"
EOF
done

# Step 2: Wait 7 days for recommendations to stabilise, then review
kubectl get vpa -n production -o json | \
  jq '.items[] | {
    name: .metadata.name,
    current_requests: .spec.resourcePolicy.containerPolicies[0].minAllowed,
    recommended_target: .status.recommendation.containerRecommendations[0].target
  }'
```

### Scale Non-Production to Zero

```yaml
# cronjob-scale-down-dev.yaml — scale dev to zero at 8 PM, back up at 8 AM
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-down-dev
  namespace: kube-system
spec:
  schedule: "0 20 * * 1-5"    # 8 PM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: namespace-scaler-sa
          containers:
          - name: scaler
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment --all --replicas=0 -n dev
              kubectl scale deployment --all --replicas=0 -n staging
              echo "✓ Non-production scaled to zero"
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: scale-up-dev
  namespace: kube-system
spec:
  schedule: "0 8 * * 1-5"     # 8 AM weekdays
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: namespace-scaler-sa
          containers:
          - name: scaler
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment --all --replicas=1 -n dev
              kubectl scale deployment --all --replicas=2 -n staging
              echo "✓ Non-production scaled up"
```

### Spot Instances for Non-Critical Workloads

AWS Spot instances cost 60–90% less than On-Demand. Use them for stateless, fault-tolerant workloads (API servers, workers) — not for databases or stateful services.

```yaml
# Node group configuration (Terraform / eksctl)
# Add a spot node group alongside your on-demand group

# Then use node affinity to target spot nodes for appropriate workloads:
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: eks.amazonaws.com/capacityType
                operator: In
                values: [SPOT]
      # Toleration needed if spot nodes have a taint
      tolerations:
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
```

### Kubecost — Cost Visibility

```bash
helm repo add kubecost https://kubecost.github.io/cost-analyzer/
helm install kubecost kubecost/cost-analyzer \
  --namespace kubecost \
  --create-namespace \
  --set kubecostToken="" \       # Free tier, no token needed
  --set persistentVolume.size=32Gi

kubectl port-forward svc/kubecost-cost-analyzer 9090:9090 -n kubecost
# http://localhost:9090 — cost breakdown by namespace, deployment, label
```

### Unused Resource Cleanup

```bash
# Find PVCs in Released state (PV exists but no PVC bound to it)
kubectl get pv -A | grep Released

# Find Services of type LoadBalancer not in production (expensive)
kubectl get svc -A --field-selector spec.type=LoadBalancer | grep -v production

# Find pods consuming no CPU (likely idle, can be scaled down or removed)
kubectl top pods -A --sort-by=cpu | tail -20

# Find images older than 30 days in ECR (lifecycle policy handles this,
# but audit manually if lifecycle policy not configured)
aws ecr describe-images --repository-name myapp \
  --query 'imageDetails[?imagePushedAt<`2024-01-01`].[imageDigest,imageTags]' \
  --output table

# Delete untagged ECR images immediately
aws ecr list-images --repository-name myapp \
  --filter tagStatus=UNTAGGED \
  --query 'imageIds[*]' --output json | \
  xargs -I{} aws ecr batch-delete-image --repository-name myapp --image-ids '{}'
```


## Chapter 7: Disaster Recovery Runbook

### RTO/RPO Targets

Define these for your service before an incident forces the conversation:

| Tier | Services | RTO Target | RPO Target | Recovery mechanism |
|------|---------|-----------|-----------|-------------------|
| Tier 0 — Critical | Payment, authentication | 5 minutes | 0 (no data loss) | Active-active multi-region |
| Tier 1 — Important | Core API, database | 30 minutes | 15 minutes | Warm standby + PITR backups |
| Tier 2 — Standard | Most services | 2 hours | 1 hour | Daily Velero backups |
| Tier 3 — Non-critical | Reporting, analytics | 8 hours | 24 hours | Weekly backups |

### Velero Backup Strategy

```bash
# ── PRODUCTION BACKUP SCHEDULE ─────────────────────────────────────────────

# Daily full backup at 1 AM — kept 30 days
velero schedule create daily-full \
  --schedule="0 1 * * *" \
  --include-namespaces production \
  --ttl 720h

# Hourly application-only backup (excludes PVCs — faster, cheaper)
velero schedule create hourly-config \
  --schedule="0 * * * *" \
  --include-namespaces production \
  --exclude-resources persistentvolumeclaims,persistentvolumes \
  --ttl 48h

# Weekly cross-region backup to secondary S3 bucket
velero schedule create weekly-offsite \
  --schedule="0 2 * * 0" \
  --include-namespaces production \
  --storage-location secondary-s3 \
  --ttl 2160h    # 90 days

# Always take a manual backup before major operations
alias pre-deploy-backup='velero backup create "pre-deploy-$(date +%Y%m%d-%H%M)" \
  --include-namespaces production --wait'
```

### Scenario Playbooks

**Scenario 1: Single pod failure**
```
Expected: Kubernetes self-heals within 60 seconds.
Action: None required. Monitor kubectl get pods -n production.
If not self-healing after 2 minutes:
  kubectl describe pod <pod> -n production   # Check events
  kubectl delete pod <pod> -n production    # Force recreation
```

**Scenario 2: Node failure**
```
Expected: Kubernetes reschedules pods to healthy nodes within 5 minutes.
Verify: kubectl get pods -n production -o wide  (check pods spread across nodes)
If PDB prevents rescheduling: kubectl get pdb -n production
If node is permanently lost:
  kubectl delete node <node-name>
  # CA will provision replacement
```

**Scenario 3: Namespace accidentally deleted**
```
Time pressure: HIGH — restore before anyone notices
1. kubectl create namespace production  (recreate the namespace)
2. velero restore create production-emergency \
     --from-backup $(velero backup get -o json | \
       jq -r '.items | sort_by(.status.completionTimestamp) | last | .metadata.name') \
     --include-namespaces production \
     --wait
3. Verify: kubectl get all -n production
4. Verify: curl https://api.myapp.com/actuator/health
5. Post-mortem: how did this happen? Add RBAC to prevent namespace deletion.
```

**Scenario 4: Database data corruption**
```
1. STOP ALL WRITES IMMEDIATELY
   kubectl scale deployment/myapp --replicas=0 -n production
   (Or update network policy to block API traffic)

2. Assess the damage
   kubectl exec -it postgres-0 -n production -- \
     psql -U appuser -d appdb -c "SELECT count(*) FROM orders;"

3. Identify last known good backup
   velero backup get --selector velero.io/schedule-name=daily-full

4. Restore to a test namespace FIRST
   velero restore create test-restore-$(date +%H%M) \
     --from-backup daily-full-20240101010000 \
     --namespace-mappings production:production-restore

5. Verify data integrity in production-restore namespace
   kubectl exec -it postgres-0 -n production-restore -- \
     psql -U appuser -d appdb -c "SELECT count(*) FROM orders;"

6. If verified good: restore to production namespace
   kubectl delete namespace production
   velero restore create production-restore \
     --from-backup daily-full-20240101010000 \
     --include-namespaces production --wait

7. Restart application
   kubectl scale deployment/myapp --replicas=3 -n production
```

**Scenario 5: Regional outage (activate DR)**
```
1. Confirm outage affects entire primary region (not just our cluster)
   Check AWS Health Dashboard / GCP Status / Azure Status

2. Update Route53 / CloudFlare to point to DR region
   # Route53 health check failover triggers automatically (if configured)
   # Manual override:
   aws route53 change-resource-record-sets --hosted-zone-id ZONE \
     --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
       "Name":"api.myapp.com","Type":"A","TTL":60,
       "ResourceRecords":[{"Value":"DR_REGION_IP"}]}}]}'

3. Restore from latest backup to DR cluster
   # (Run on DR cluster — separate kubeconfig)
   velero restore create dr-restore-$(date +%Y%m%d-%H%M) \
     --from-backup $(velero backup get -o json | \
       jq -r '.items[-1].metadata.name') \
     --include-namespaces production --wait

4. Update external service credentials if needed
   # Database endpoints, message queues, etc. differ between regions

5. Verify DR cluster is serving traffic
   curl https://api.myapp.com/actuator/health

6. Communicate to stakeholders (use communication template from Chapter 5)
```

### DR Testing Schedule

| Test | Frequency | Who runs it | What success looks like |
|------|-----------|------------|------------------------|
| Velero restore to test namespace | Monthly | Platform team | All pods healthy, data intact |
| Full cluster restore to staging | Quarterly | Platform + team leads | Staging cluster restored in < 2h |
| DNS failover simulation | Quarterly | Platform + networking | Traffic routes to DR in < 5 minutes |
| Cross-region DR activation | Annually | All hands | DR cluster serves full production traffic |

---

## Chapter 8: Service Documentation Templates

### Service README Template

```markdown
# Service Name

## Overview
One paragraph describing what this service does, who uses it, and why it exists.

## Architecture
[Diagram or description of how this service fits into the larger system]

**Depends on:** postgres-service, redis-service, payment-gateway-api
**Depended on by:** checkout-service, reporting-service, mobile-api

## Environments

| Environment | URL | Kubernetes namespace |
|-------------|-----|---------------------|
| Production | https://api.myapp.com | production |
| Staging | https://staging.myapp.com | staging |
| Dev | https://dev.myapp.com | dev |

## Deploying

```bash
# Deploy to staging
helm upgrade --install myapp-staging ./helm-chart \
  -f values.yaml -f values-staging.yaml \
  --set image.tag=git-abc1234 \
  --namespace staging --atomic

# Deploy to production (requires pipeline approval)
git push origin main   # CI/CD handles the rest
```

## Monitoring

| Dashboard | Link |
|-----------|------|
| Spring Boot overview | https://grafana.myapp.com/d/spring-boot |
| Error rate / latency | https://grafana.myapp.com/d/myapp-slo |
| Infrastructure | https://grafana.myapp.com/d/kubernetes |

## SLOs

| SLI | SLO | Current |
|-----|-----|---------|
| Availability | 99.9% / 30 days | [link to dashboard] |
| P99 latency | < 500ms | [link to dashboard] |
| Error rate | < 0.1% | [link to dashboard] |

## Runbooks

- [Service Down](https://wiki.mycompany.com/runbooks/myapp-down)
- [High Error Rate](https://wiki.mycompany.com/runbooks/myapp-errors)
- [Database Issues](https://wiki.mycompany.com/runbooks/myapp-database)
- [Performance Degradation](https://wiki.mycompany.com/runbooks/myapp-perf)

## On-Call

Primary: @alice  
Secondary: @bob  
Escalation: @platform-team

## Known Issues / Tech Debt
- [ ] Connection pool size not tuned for peak load (ticket #1234)
- [ ] No circuit breaker on payment-gateway calls (ticket #1235)
```

### Post-Mortem Template

```markdown
# Post-Mortem: [Service] — [Date] — [Brief description]

**Severity:** SEV1
**Duration:** 43 minutes (14:32–15:15 UTC)
**Affected users:** ~15% of checkout traffic
**Authored by:** Alice
**Reviewed by:** Bob, Charlie

---

## Summary
Two sentences describing what happened and the impact.

---

## Timeline

| Time (UTC) | Event |
|-----------|-------|
| 14:28 | Deployment of v1.3.2 completed |
| 14:30 | Error rate begins rising (first sample) |
| 14:32 | PagerDuty alert fires — Alice acknowledged |
| 14:35 | Connection pool exhaustion identified in logs |
| 14:45 | v1.3.2 rolled back to v1.3.1 — no improvement |
| 14:52 | Root cause identified: pool size of 2 in new config |
| 15:00 | Emergency config change applied |
| 15:03 | Error rate returns to normal |
| 15:15 | Incident declared resolved |

---

## Root Cause
The v1.3.2 deployment introduced a misconfigured `HikariCP` max-pool-size of 2
(changed from 20 during a refactor). Under normal production traffic, all 2
connections were immediately exhausted and subsequent requests queued, eventually
timing out and returning 503.

## Contributing Factors
- The configuration change was not caught in code review (pool size not tested)
- Staging uses much lower traffic and did not exhibit pool exhaustion
- The deployment pipeline has no post-deploy load test

---

## Impact
- 43 minutes of 15% error rate on checkout
- ~2,400 failed checkout attempts
- ~£12,000 estimated lost revenue
- No data loss

---

## Action Items

| Action | Owner | Due date | Priority |
|--------|-------|---------|---------|
| Add connection pool health check to integration tests | Alice | Jan 22 | P1 |
| Add post-deploy load test step to staging pipeline | Bob | Jan 29 | P1 |
| Document HikariCP config in service README | Alice | Jan 22 | P2 |
| Add alert for "connection pool near exhaustion" | Charlie | Jan 29 | P2 |

---

## What Went Well
- Alert fired within 2 minutes of error rate rising
- Root cause identified in 20 minutes
- Rollback and fix completed within 45 minutes total

## What Could Be Improved
- Rollback didn't fix the issue — config was not reverted with the image
- No runbook for "connection pool exhaustion" — responder had to improvise
```

---

## Chapter 9: One-Command Production Deployment Script

This script is the human-runnable alternative to the full CI/CD pipeline — useful for emergency hotfixes, environments without CI/CD access, and initial cluster setup.

```bash
#!/bin/bash
# deploy-to-production.sh
# Usage: ./deploy-to-production.sh v1.3.3
#
# Runs: validate → pre-backup → diff → deploy → verify → notify
# Requires: helm, kubectl, velero, jq, curl configured for production cluster

set -euo pipefail

# ── Configuration ─────────────────────────────────────────────────────────
VERSION="${1:-}"
NAMESPACE="production"
RELEASE="myapp-prod"
CHART="./helm-chart"
VALUES_BASE="values.yaml"
VALUES_PROD="values-prod.yaml"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL:-}"

# ── Validation ─────────────────────────────────────────────────────────────
if [ -z "$VERSION" ]; then
  echo "Usage: $0 <version-tag>"
  echo "Example: $0 v1.3.3"
  exit 1
fi

if ! echo "$VERSION" | grep -qE '^v[0-9]+\.[0-9]+\.[0-9]+$'; then
  echo "Version must be a semantic version: vMAJOR.MINOR.PATCH"
  echo "Got: $VERSION"
  exit 1
fi

echo "══════════════════════════════════════════════════════"
echo "  Production Deployment: $VERSION"
echo "  Namespace: $NAMESPACE"
echo "  Operator: $(whoami)"
echo "  Time: $(date -u)"
echo "══════════════════════════════════════════════════════"
echo ""

# ── Step 1: Pre-flight checks ──────────────────────────────────────────────
echo "▶ Pre-flight checks..."
helm lint "$CHART" -f "$VALUES_PROD" --quiet
kubectl cluster-info --request-timeout=5s > /dev/null
echo "  ✓ Cluster reachable and chart is valid"

# ── Step 2: Pre-deployment backup ─────────────────────────────────────────
BACKUP_NAME="pre-deploy-${VERSION}-$(date +%Y%m%d-%H%M%S)"
echo ""
echo "▶ Creating pre-deployment backup: $BACKUP_NAME"
velero backup create "$BACKUP_NAME" \
  --include-namespaces "$NAMESPACE" \
  --wait --timeout 10m
echo "  ✓ Backup complete"

# ── Step 3: Show diff ─────────────────────────────────────────────────────
echo ""
echo "▶ Changes to be applied:"
echo "─────────────────────────────────────────────────────"
helm diff upgrade "$RELEASE" "$CHART" \
  -f "$VALUES_BASE" -f "$VALUES_PROD" \
  --set image.tag="$VERSION" \
  --namespace "$NAMESPACE" \
  --no-color || true
echo "─────────────────────────────────────────────────────"
echo ""

# ── Step 4: Confirmation ──────────────────────────────────────────────────
read -r -p "Deploy $VERSION to production? [yes/N] " CONFIRM
if [ "$CONFIRM" != "yes" ]; then
  echo "Deployment cancelled."
  exit 0
fi

DEPLOY_START=$(date +%s)

# ── Step 5: Deploy ────────────────────────────────────────────────────────
echo ""
echo "▶ Deploying $VERSION to production..."
if helm upgrade --install "$RELEASE" "$CHART" \
  -f "$VALUES_BASE" -f "$VALUES_PROD" \
  --set image.tag="$VERSION" \
  --namespace "$NAMESPACE" \
  --create-namespace \
  --atomic \
  --wait \
  --timeout 15m; then
  echo "  ✓ Helm upgrade successful"
else
  echo "  ✗ Helm upgrade failed — rollback triggered automatically by --atomic"
  DEPLOY_END=$(date +%s)
  DURATION=$((DEPLOY_END - DEPLOY_START))

  [ -n "$SLACK_WEBHOOK" ] && curl -s -X POST "$SLACK_WEBHOOK" \
    -H "Content-type: application/json" \
    --data "{\"text\":\"❌ Production deploy FAILED and rolled back\",\
             \"attachments\":[{\"color\":\"danger\",\"fields\":[\
               {\"title\":\"Version\",\"value\":\"$VERSION\",\"short\":true},\
               {\"title\":\"Operator\",\"value\":\"$(whoami)\",\"short\":true},\
               {\"title\":\"Duration\",\"value\":\"${DURATION}s\",\"short\":true}]}]}"

  exit 1
fi

# ── Step 6: Health verification ───────────────────────────────────────────
echo ""
echo "▶ Verifying production health..."
sleep 30  # Allow deployment to stabilise

HEALTHY=false
for i in $(seq 1 12); do
  STATUS=$(curl -sf https://api.myapp.com/actuator/health/readiness \
    -o /dev/null -w "%{http_code}" 2>/dev/null || echo "000")
  if [ "$STATUS" = "200" ]; then
    HEALTHY=true
    echo "  ✓ Health check passed (attempt $i)"
    break
  fi
  echo "  Attempt $i/12: status=$STATUS, waiting 15s..."
  sleep 15
done

if [ "$HEALTHY" = "false" ]; then
  echo "  ✗ Health checks failed — rolling back"
  helm rollback "$RELEASE" -n "$NAMESPACE"
  exit 1
fi

DEPLOY_END=$(date +%s)
DURATION=$((DEPLOY_END - DEPLOY_START))

# ── Step 7: Post-deploy verification ─────────────────────────────────────
echo ""
echo "▶ Verifying pod rollout..."
kubectl rollout status deployment/myapp-prod -n "$NAMESPACE" --timeout=5m
echo "  ✓ All pods rolled out successfully"

echo ""
echo "▶ Post-deployment summary:"
kubectl get pods -n "$NAMESPACE" -l app=myapp
echo ""
echo "  Release history:"
helm history "$RELEASE" -n "$NAMESPACE" --max 5

# ── Step 8: Notify ────────────────────────────────────────────────────────
[ -n "$SLACK_WEBHOOK" ] && curl -s -X POST "$SLACK_WEBHOOK" \
  -H "Content-type: application/json" \
  --data "{\"text\":\"✅ Production deployment complete\",\
           \"attachments\":[{\"color\":\"good\",\"fields\":[\
             {\"title\":\"Version\",\"value\":\"$VERSION\",\"short\":true},\
             {\"title\":\"Operator\",\"value\":\"$(whoami)\",\"short\":true},\
             {\"title\":\"Duration\",\"value\":\"${DURATION}s\",\"short\":true},\
             {\"title\":\"Pre-deploy backup\",\"value\":\"$BACKUP_NAME\",\"short\":false}]}]}"

echo ""
echo "══════════════════════════════════════════════════════"
echo "  ✓ Deployment complete: $VERSION"
echo "  Duration: ${DURATION} seconds"
echo "  Pre-deploy backup: $BACKUP_NAME"
echo "══════════════════════════════════════════════════════"
```

---

## Production Checklists — Summary Tables

### Before Going Live (Minimum Viable)

| # | Check | How to verify |
|---|-------|--------------|
| 1 | Graceful shutdown configured | `spring.shutdown=graceful` in application.yml |
| 2 | All three probes configured | `kubectl describe pod <n>` — all three listed |
| 3 | Resource requests and limits set | `kubectl get pod <n> -o yaml \| grep -A4 resources` |
| 4 | Non-root container user | `docker inspect` or `runAsNonRoot: true` in spec |
| 5 | Image tagged with version | `image.tag != "latest"` in values-prod.yaml |
| 6 | Image scanned — no critical CVEs | Trivy scan output in CI |
| 7 | `readOnlyRootFilesystem: true` | Pod spec + emptyDir volumes present |
| 8 | `automountServiceAccountToken: false` | Pod spec shows this field |
| 9 | Pod Disruption Budget exists | `kubectl get pdb -n production` |
| 10 | Network policies: deny-all + DNS allow | `kubectl get networkpolicy -n production` |
| 11 | Prometheus scraping the app | `:9090/targets` — app listed as UP |
| 12 | Error rate alert configured | `kubectl get prometheusrule -n monitoring` |
| 13 | Deployed via CI/CD, not manual | Last deploy visible in GitHub Actions |
| 14 | Helm `--atomic` used | CI/CD workflow YAML |
| 15 | Velero backup schedule active | `velero schedule get` |
| 16 | Rollback procedure documented | Link in service README |
| 17 | On-call rotation defined | Runbook / PagerDuty schedule |
| 18 | Secrets in External Secrets or Vault | No raw Secret YAML in Git |
| 19 | `helm diff` in PR checks | `.github/workflows/` directory |
| 20 | Service README exists | Link in repo root |

### Ongoing — Monthly

| Check | Command |
|-------|---------|
| Security audit — root containers | `kubectl get pods -A -o json \| jq '.items[] \| select(.spec.securityContext.runAsNonRoot != true)'` |
| Security audit — latest tags | `kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \| grep :latest` |
| Cost audit — unused PVCs | `kubectl get pvc -A \| grep -v Bound` |
| Cost audit — idle LoadBalancers | `kubectl get svc -A --field-selector spec.type=LoadBalancer` |
| VPA recommendations reviewed | `kubectl get vpa -A -o yaml \| grep -A3 target:` |
| Velero restore tested | Run restore to test namespace, verify data |
| Certificate expiry check | `kubectl get certificate -A` |
| Dependency updates | `helm dependency list` + check for new versions |

---

## The Series in One Diagram

You started Part 1 with a Spring Boot app on your laptop. This is where it lives in production:

```
Developer pushes code
         │
         ▼
  GitHub Actions (Part 4)
  test → scan → build → push
         │
         ▼
  Container Registry (Part 2/4)
  Signed image: v1.3.3
         │
         ▼
  Helm Chart (Parts 1/3) ──────────────────────────────────────┐
  values-prod.yaml                                             │
         │                                                     │
         ▼                                                     ▼
  Kubernetes (Parts 1/2)              Monitoring (Part 8)
  ┌──────────────────────┐            Prometheus scrapes /actuator/prometheus
  │ production namespace │            Grafana shows dashboards
  │ PSA: restricted      │            Loki collects logs
  │ ResourceQuota        │            Tempo receives traces
  │ NetworkPolicy        │            Alertmanager routes alerts
  │                      │
  │  ┌────────────────┐  │
  │  │  3 × Pod       │  │
  │  │  Non-root      │  │
  │  │  ReadOnly FS   │  │
  │  │  Resource lim  │  │
  │  │  IRSA identity │  │
  │  └───────┬────────┘  │
  │          │           │
  │  HPA ────┘           │    Security (Part 7)
  │  VPA recommendations │    OPA Gatekeeper
  │  PDB: max 20% down   │    Falco runtime
  │                      │    Vault secrets
  └──────────┬───────────┘    Cosign signed images
             │
             ▼
  Ingress (Part 5)
  TLS via cert-manager
  DNS via ExternalDNS
  nginx-ingress
             │
             ▼
  Users at https://api.myapp.com

  Storage (Part 6)           Scaling (Part 9)
  PostgreSQL StatefulSet      Cluster Autoscaler (adds nodes)
  Velero backups              Chaos Mesh (tests resilience)
  Volume snapshots            Argo Rollouts (canary/blue-green)
```

Every box in that diagram corresponds to a part of this series. The system is not ten separate concerns — it is one coherent whole where each piece reinforces the others.

The hard work is not understanding any single concept. It is making all of them work together, consistently, reliably, at 2 AM when something breaks and you need to know exactly what to do. That is what this playbook is for.

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| Deployment passes CI but pods not updating in cluster | OIDC credentials expired or wrong cluster selected | `kubectl config current-context` | Verify kubeconfig targets the right cluster; re-run `aws eks update-kubeconfig` |
| `helm diff` shows no changes but behaviour changed | ConfigMap data changed but not tracked by diff | `helm get manifest <release> -n <ns>` and compare manually | `helm diff` only shows structured YAML changes; add ConfigMap checksums as pod annotations |
| Velero backup fails with `AccessDenied` | S3 bucket policy or IRSA permissions wrong | `kubectl logs -n velero deployment/velero` | Check S3 bucket policy allows velero SA; verify IRSA annotation on velero SA |
| OPA Gatekeeper blocks a pod but you can't find why | Multiple constraints active, violation message unclear | `kubectl describe constraint <n>` for each active constraint | Run audit: `kubectl get constraints -A` and check `TOTAL-VIOLATIONS` count |
| PSA `restricted` rejects a third-party chart | Chart not designed for restricted security contexts | `kubectl apply --dry-run=server -f chart.yaml` to see specific violation | Set PSA to `warn` for that namespace; file issue with chart maintainer; use namespace override annotation |
| Post-deploy metrics check fails but app seems fine | Prometheus scrape lag — metrics haven't updated yet | Wait 2–3 scrape cycles (60–90s at 30s interval) | Increase sleep before metrics check in deploy script to 90s |
| `helm rollback` reverts code but not config | ConfigMaps are updated by Helm but rollback doesn't revert them | `helm get manifest <release> --revision <n>` | Apply previous ConfigMap explicitly; use the checksum annotation pattern to tie ConfigMap to deployment revision |
| Cost anomaly — unexpected node count | CA not scaling down, PDB preventing drain, or memory request inflation | `kubectl describe node <n>` — check allocated resources; `kubectl get pdb -A` | Review VPA recommendations for over-sized memory requests; ensure PDB `maxUnavailable` is not `0` |

---

## Practice Exercises

**Exercise 1 — Production readiness audit:**
Run the minimum viable production checklist against your `hello-app` from Part 1. For each item that fails, fix it and re-run the check. Document every change you made. By the end, every item should be green. This exercise surfaces all the gaps between a "working" deployment and a "production-ready" one.

**Exercise 2 — Complete values file:**
Using Chapter 2's template as a starting point, write a complete `values-prod.yaml` for your `hello-app`. Fill in every field — don't leave anything blank or commented out. Run `helm template` with it and inspect the output. Verify every security context, every probe, every annotation is rendered correctly. Fix anything that isn't.

**Exercise 3 — Full CI/CD pipeline:**
Wire up the complete GitHub Actions pipeline from Chapter 3 for your repository. Test it by: (1) pushing a change that passes — verify it deploys to staging and waits for production approval; (2) pushing a change that fails a test — verify the pipeline stops at the test stage; (3) pushing a change that passes tests but Trivy rejects — verify it stops at the scan stage. All three failure modes must work correctly.

**Exercise 4 — Incident simulation:**
Deliberately break your production deployment in a reversible way (set `image.tag` to a non-existent tag). Follow the "Service Down" runbook from Chapter 5, step by step, as if you don't know the cause. Write down every command you ran and what it told you. Measure how long it takes to identify the root cause. Then fix the issue. Write a one-page post-mortem using the template from Chapter 8. The goal is not to do it fast — it is to follow the process correctly.

**Exercise 5 — Cost baseline:**
Install Kubecost in your cluster. Let it run for 24 hours. Export the cost breakdown by namespace and deployment. Identify the three highest-cost workloads. For each one: check VPA recommendations, check whether it runs 24/7 or could be scaled to zero overnight, check whether it uses reserved/spot capacity or only on-demand. Implement at least one cost optimisation and measure the savings after 48 hours.
