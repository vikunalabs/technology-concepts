# Part 4: Container Registry & CI/CD

## What This Covers
- Why registries matter and which to use
- GitHub Actions pipeline: test → build → scan → deploy
- Semantic versioning
- Multi-environment promotion (dev → staging → prod)

---

## Chapter 1: Container Registries

### Why They Exist

Your Kubernetes cluster can't see Docker images on your laptop. Every image your cluster uses must be pulled from a registry — a server that stores and serves Docker images.

```
Your laptop
    docker build → image exists locally only
    docker push  → image lives in registry

Kubernetes cluster
    Pod spec: image: yourusername/myapp:1.0.0
    Kubelet pulls from registry → runs container
```

### Which Registry to Use

| Registry | Free Tier | Best For |
|----------|-----------|----------|
| **Docker Hub** | 1 private repo, unlimited public | Learning |
| **GitHub Container Registry (ghcr.io)** | 500MB free, integrates with Actions | GitHub workflows |
| **AWS ECR** | 500MB free, pay for storage | AWS clusters |
| **Google Artifact Registry** | 500MB free | GCP clusters |
| **Azure ACR** | Basic tier ~$0.10/GB | Azure clusters |

For local learning: Docker Hub. For production on AWS: ECR. For production on GCP: Artifact Registry.

### Docker Hub Workflow

```bash
# One-time setup
docker login

# Every release
docker build -t myapp:1.0.0 .
docker tag myapp:1.0.0 yourusername/myapp:1.0.0
docker tag myapp:1.0.0 yourusername/myapp:latest   # Convenience tag, don't use in prod

docker push yourusername/myapp:1.0.0
docker push yourusername/myapp:latest
```

### AWS ECR Workflow

```bash
# Create repository (once)
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# Login (token expires every 12 hours)
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag myapp:1.0.0 ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
```

### Image Tagging Strategy

```yaml
# ❌ Avoid in production — "latest" tells you nothing
image: yourusername/myapp:latest

# ✅ Specific semantic version — immutable, auditable
image: yourusername/myapp:1.2.3

# ✅ Git SHA — great for CI/CD traceability
image: yourusername/myapp:git-abc123d

# ✅ Both — human-readable + traceable
image: yourusername/myapp:1.2.3  # Tag points to same SHA as the git tag
```

### Pull Secret for Private Registries

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=yourpassword \
  --docker-email=your@email.com \
  -n production
```

Reference in Helm values:
```yaml
imagePullSecrets:
  - name: regcred
```

---

## Chapter 2: CI/CD Concepts

### What the Pipeline Does

```
Developer pushes code
    │
    ▼
[Test]        Run unit + integration tests. Fail fast.
    │ pass
    ▼
[Build]       Compile JAR, build Docker image
    │
    ▼
[Scan]        Check image for known CVEs (vulnerabilities)
    │ no criticals
    ▼
[Push]        Upload image to registry with version tag
    │
    ▼
[Deploy Dev]  Auto-deploy to dev environment
    │
    ▼
[Deploy Staging]  Auto-deploy to staging, run integration tests
    │
    ▼
[Approve]     Manual gate — human reviews before production
    │ approved
    ▼
[Deploy Prod] Deploy to production with automatic rollback on failure
```

### Semantic Versioning

```
v1.2.3
  │ │ └── patch: backwards-compatible bug fix
  │ └──── minor: new feature, backwards compatible
  └────── major: breaking change
```

Conventional commits drive this automatically:
```
feat: add payment endpoint       → bumps minor (1.2.3 → 1.3.0)
fix: correct tax calculation     → bumps patch (1.2.3 → 1.2.4)
feat!: redesign API (breaking)   → bumps major (1.2.3 → 2.0.0)
docs: update README              → no version bump
chore: update dependencies       → no version bump
```

---

## Chapter 3: GitHub Actions Pipeline

### Project Structure

```
.github/
└── workflows/
    ├── ci.yml          # Runs on every PR: test + build + scan
    └── deploy.yml      # Runs on main branch: deploy to dev/staging + prod gate
```

### CI Workflow (PRs and pushes)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}  # yourusername/myapp

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Set up JDK 17
      uses: actions/setup-java@v4
      with:
        java-version: '17'
        distribution: 'temurin'
        cache: maven              # Cache ~/.m2 between runs

    - name: Run tests
      run: ./mvnw test

    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()                # Upload even when tests fail
      with:
        name: test-results
        path: target/surefire-reports/

  build-and-push:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: test                   # Only runs if test job passes
    if: github.event_name == 'push'   # Skip on pull requests
    outputs:
      image_tag: ${{ steps.meta.outputs.version }}

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0            # Full history needed for version calculation

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          # On main branch: latest + git SHA
          type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
          type=sha,format=short
          # On version tags (v1.2.3): semantic version tags
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}

    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha          # Use GitHub Actions cache
        cache-to: type=gha,mode=max

    - name: Scan with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
        format: table
        exit-code: '1'              # Fail the build on critical CVEs
        severity: 'CRITICAL'
```

### Deploy Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:              # Allow manual triggering
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [dev, staging, production]

jobs:
  deploy-dev:
    name: Deploy to Dev
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.myapp.com
    steps:
    - uses: actions/checkout@v4

    - name: Set up Helm
      uses: azure/setup-helm@v4

    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        echo "${{ secrets.DEV_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config

    - name: Deploy to dev
      run: |
        helm dependency update ./helm-chart
        helm upgrade --install myapp-dev ./helm-chart \
          -f values-dev.yaml \
          --namespace development \
          --create-namespace \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 5m

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp-dev -n development --timeout=3m

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.myapp.com
    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4

    - name: Configure kubectl
      run: echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config

    - name: Deploy to staging
      run: |
        helm upgrade --install myapp-staging ./helm-chart \
          -f values-staging.yaml \
          --namespace staging \
          --create-namespace \
          --set image.tag=${{ github.sha }} \
          --wait \
          --timeout 10m

    - name: Run integration tests
      run: helm test myapp-staging -n staging

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    # environment with required reviewers = manual approval gate
    environment:
      name: production
      url: https://api.myapp.com
    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4

    - name: Configure kubectl
      run: echo "${{ secrets.PROD_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config

    - name: Deploy to production
      run: |
        helm upgrade --install myapp-prod ./helm-chart \
          -f values-prod.yaml \
          --namespace production \
          --create-namespace \
          --set image.tag=${{ github.sha }} \
          --atomic \          # Automatically rollback if deployment fails
          --wait \
          --timeout 15m

    - name: Notify team
      if: always()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "${{ job.status == 'success' && '✅' || '❌' }} Production deploy ${{ job.status }}: ${{ github.sha }}"
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Setting Up GitHub Secrets

In your GitHub repo: Settings → Secrets and variables → Actions

Required secrets:
```
DOCKER_USERNAME       — Docker Hub username
DOCKER_PASSWORD       — Docker Hub access token (not password)
DEV_KUBECONFIG        — base64-encoded kubeconfig for dev cluster
STAGING_KUBECONFIG    — base64-encoded kubeconfig for staging cluster
PROD_KUBECONFIG       — base64-encoded kubeconfig for prod cluster
SLACK_WEBHOOK         — Slack webhook URL for notifications
```

Encode your kubeconfig:
```bash
cat ~/.kube/config | base64 | pbcopy   # macOS — copies to clipboard
cat ~/.kube/config | base64 | xclip    # Linux
```

### Setting Up Environments with Manual Approval

In GitHub: Settings → Environments → New environment → "production"
- Enable "Required reviewers" and add yourself or your team
- The `deploy-production` job will pause and wait for approval

---

## Chapter 4: GitLab CI Alternative

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script:
    - mvn test
  cache:
    paths:
      - .m2/repository/

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG .
    - docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG

deploy-dev:
  stage: deploy-dev
  image: alpine/helm:3.12
  script:
    - helm upgrade --install myapp-dev ./helm-chart
        -f values-dev.yaml
        --set image.repository=$CI_REGISTRY_IMAGE
        --set image.tag=$IMAGE_TAG
        --namespace dev

deploy-staging:
  stage: deploy-staging
  image: alpine/helm:3.12
  when: manual               # Manual trigger in GitLab UI
  script:
    - helm upgrade --install myapp-staging ./helm-chart
        -f values-staging.yaml
        --set image.tag=$IMAGE_TAG
        --namespace staging

deploy-prod:
  stage: deploy-prod
  image: alpine/helm:3.12
  when: manual
  environment:
    name: production
    url: https://api.myapp.com
  script:
    - helm upgrade --install myapp-prod ./helm-chart
        -f values-prod.yaml
        --set image.tag=$IMAGE_TAG
        --namespace production
        --atomic
```

---

## Quick Reference

### Pipeline Checklist

| Stage | What to verify |
|-------|---------------|
| Test | All tests green, coverage not regressing |
| Build | Image builds successfully, size reasonable |
| Scan | No CRITICAL CVEs (HIGH acceptable with justification) |
| Dev deploy | Pods running, health check passing |
| Staging | Integration tests pass, no error spike in logs |
| Prod | Rollback plan ready, monitoring watching |

### `--atomic` Flag

`helm upgrade --atomic` enables automatic rollback: if the deployment doesn't reach healthy within the timeout, Helm rolls back to the previous release. Use this for production.

```bash
helm upgrade myapp ./chart \
  --atomic \
  --timeout 10m     # Give 10 minutes for rollout before rolling back
```

### Common CI Issues

| Issue | Fix |
|-------|-----|
| Docker rate limit (429) | Authenticate pulls: `docker login` in CI, or use GitHub Container Registry |
| Helm times out | Increase `--timeout`, check pod logs for startup errors |
| Kubeconfig invalid | Re-encode: `cat kubeconfig | base64 -w 0` (the `-w 0` prevents line wrapping) |
| Image not found after push | Wait a few seconds — registry propagation isn't always instant |
| `--atomic` rolls back unexpectedly | Check pod events: `kubectl describe pod <name>` |
