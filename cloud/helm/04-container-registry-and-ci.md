## Part 4: Container Registry & CI/CD Pipeline - Automating Your Deployments

### Prerequisites
- Completed Parts 1-3 (Spring Boot app + Docker + Helm)
- GitHub account (or GitLab)
- Docker Hub account (free tier works)
- Basic understanding of Git

### What You'll Learn
- ✅ Container registry deep dive (ECR, GCR, ACR, Docker Hub)
- ✅ CI/CD concepts and pipeline architecture
- ✅ GitHub Actions workflows from scratch
- ✅ Automated building and pushing to registries
- ✅ Semantic versioning and changelog generation
- ✅ Security scanning (vulnerability detection)
- ✅ Multi-environment promotion (dev → staging → prod)

---

## Chapter 1: Container Registry Deep Dive

### The Big Picture: Where We Are vs. Where We're Going

**Where we are (manual):**
```bash
# Developer laptop - manual, error-prone, slow
./mvnw test
./mvnw package
docker build -t myapp .
docker tag myapp myregistry/myapp:latest
docker push myregistry/myapp:latest
helm upgrade myapp ./chart --set image.tag=latest
# Did we push the right version? Is it safe? Who knows!
```

**Where we're going (automated):**
```bash
# Developer only does this:
git commit -m "feat: add new feature"
git push

# Everything else happens automatically:
# → Tests run
# → Image builds
# → Security scan passes
# → Deploys to dev
# → Waits for approval
# → Deploys to production
```

### Registry Types Compared

| Registry | Free Tier | Best For | Authentication | Image Scanning |
|----------|-----------|----------|----------------|----------------|
| **Docker Hub** | 1 private repo, unlimited public | Learning, open source | Username/password | Basic |
| **AWS ECR** | 500MB storage | AWS users | IAM roles | Enhanced (Inspector) |
| **Google GCR** | 5GB/month | GCP users | GCP IAM | Container Analysis |
| **Azure ACR** | Basic tier ($0.10/GB) | Azure users | AAD | Defender for Cloud |
| **GitHub Container Registry** | 500MB free | GitHub users | GitHub token | GitHub Advanced Security |
| **GitLab Container Registry** | 10GB free | GitLab users | GitLab token | Basic |

### Setting Up Each Registry

**Docker Hub (Easiest for Learning):**
```bash
# 1. Create account at hub.docker.com
# 2. Login locally
docker login

# 3. Create repository (via UI or CLI)
# 4. Tag and push
docker tag myapp:latest username/myapp:1.0.0
docker push username/myapp:1.0.0

# 5. Create pull secret for Kubernetes
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=username \
  --docker-password=password \
  --docker-email=email@example.com
```

**AWS ECR (Production Standard):**
```bash
# 1. Install AWS CLI
# 2. Configure credentials
aws configure

# 3. Create repository
aws ecr create-repository \
  --repository-name myapp \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true

# 4. Get login token and authenticate
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com

# 5. Tag and push
docker tag myapp:latest \
  ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0.0

# 6. Create pull secret (using IAM role is better!)
kubectl create secret docker-registry ecr-regcred \
  --docker-server=${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password)
```

**Google GCR:**
```bash
# 1. Install gcloud CLI
gcloud auth configure-docker

# 2. Tag and push (uses your GCP project)
docker tag myapp gcr.io/your-project/myapp:1.0.0
docker push gcr.io/your-project/myapp:1.0.0

# 3. Pull secret (using GKE's workload identity is better)
kubectl create secret docker-registry gcr-regcred \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat ~/gcp-key.json)"
```

### Registry Best Practices

```yaml
# ✅ DO: Use immutable tags for production
image: myregistry/myapp:v1.2.3  # Specific version

# ❌ DON'T: Use latest in production
image: myregistry/myapp:latest  # What version is this?

# ✅ DO: Use short SHA for development
image: myregistry/myapp:git-sha-abc123def

# ✅ DO: Include metadata in tags
image: myregistry/myapp:v1.2.3-prod  # Environment
image: myregistry/myapp:v1.2.3-arm64  # Architecture

# ✅ DO: Enable image scanning
# AWS ECR: scanOnPush=true
# Docker Hub: Automatic scanning
# GitHub: Dependabot alerts
```

---

## Chapter 2: CI/CD Pipeline Architecture

### What is CI/CD?

**Analogy - Car Manufacturing:**
- **Continuous Integration (CI)** = Automated assembly line (every part is tested as it's added)
- **Continuous Delivery (CD)** = Automated quality checks before shipping
- **Continuous Deployment** = Automatic shipping to customers

### Pipeline Stages

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│    Commit    │───▶│    Build     │───▶│    Test      │───▶│    Scan      │
│   (git push) │    │  (compile)   │    │  (unit/integ)│    │ (security)   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                    │
                                                                    ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Deploy     │◀───│   Approve    │◀───│    Push      │◀───│    Package   │
│   (to prod)  │    │  (manual)    │    │  (registry)  │    │   (docker)   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Pipeline Tools Comparison

| Tool | Pros | Cons | Best For |
|------|------|------|----------|
| **GitHub Actions** | Integrated with GitHub, free for public | YAML can get complex | GitHub users |
| **GitLab CI** | Built-in registry, great docs | Self-hosted requires resources | GitLab users |
| **Jenkins** | Extremely flexible, plugins | Hard to maintain, legacy | Complex enterprise |
| **CircleCI** | Fast, good UX | Expensive for private | Performance-focused |
| **Argo CD** | GitOps native | Learning curve | Kubernetes native |
| **Tekton** | Kubernetes native | Verbose YAML | Cloud-native teams |

---

## Chapter 3: GitHub Actions Deep Dive

### Understanding GitHub Actions

**Core Concepts:**
- **Workflow** = Automated process defined in YAML
- **Event** = Trigger (push, PR, schedule, manual)
- **Job** = Set of steps running on same runner
- **Step** = Individual task (run command, action)
- **Action** = Reusable unit of code
- **Runner** = Machine that executes jobs

### Complete Production Pipeline

Create `.github/workflows/deploy.yml`:

```yaml
name: Build, Test, and Deploy

# Trigger on pushes to main, PRs, and manual
on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'  # Don't run on doc changes
      - 'docs/**'
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

# Environment variables available to all jobs
env:
  REGISTRY: docker.io  # Change to your registry
  IMAGE_NAME: ${{ github.repository }}
  HELM_CHART_PATH: ./helm-chart

# Permissions (security)
permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  # JOB 1: Test (fast, fails early)
  test:
    name: Test Application
    runs-on: ubuntu-latest
    timeout-minutes: 10
    
    steps:
      # Step 1: Checkout code
      - name: Checkout code
        uses: actions/checkout@v3
        with:
          fetch-depth: 0  # Full history for versioning
      
      # Step 2: Setup Java
      - name: Setup JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      
      # Step 3: Run unit tests
      - name: Run unit tests
        run: ./mvnw test
      
      # Step 4: Run integration tests (if needed)
      - name: Run integration tests
        run: ./mvnw verify -DskipUnitTests
        env:
          TEST_DB_URL: ${{ secrets.TEST_DB_URL }}
      
      # Step 5: Upload test results (for visibility)
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()  # Even if tests fail
        with:
          name: test-results
          path: target/surefire-reports/
  
  # JOB 2: Build and Push Docker Image
  build:
    name: Build and Push Image
    runs-on: ubuntu-latest
    needs: [test]  # Only if tests pass
    if: github.event_name != 'pull_request'  # Skip on PRs
    
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
      image_digest: ${{ steps.digest.outputs.digest }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      # Step: Get version from Git tag or generate
      - name: Generate version
        id: version
        run: |
          if [[ $GITHUB_REF == refs/tags/v* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
          else
            # Use semantic versioning with commit SHA
            VERSION=$(git describe --tags --always --dirty)
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "version_short=${VERSION:0:12}" >> $GITHUB_OUTPUT
      
      # Step: Setup Docker Buildx (for multi-arch)
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      # Step: Login to container registry
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      # Step: Extract metadata for Docker
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,format=short
            type=raw,value=${{ steps.version.outputs.version }}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      # Step: Build and push Docker image
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ steps.version.outputs.version }}
            BUILD_TIME=${{ github.event.head_commit.timestamp }}
      
      # Step: Get image digest for provenance
      - name: Get image digest
        id: digest
        run: |
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ${{ steps.meta.outputs.tags }} | cut -d'@' -f2)
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
      
      # Step: Security scan with Trivy
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.meta.outputs.tags }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      # Step: Upload scan results
      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
  
  # JOB 3: Deploy to Development
  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: [build]
    environment: 
      name: development
      url: https://dev.myapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.12.0
      
      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'latest'
      
      - name: Setup kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.DEV_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config
      
      - name: Helm dependency update
        run: helm dependency update ${{ env.HELM_CHART_PATH }}
      
      - name: Deploy to development
        run: |
          helm upgrade --install myapp-dev ${{ env.HELM_CHART_PATH }} \
            --namespace development \
            --create-namespace \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ needs.build.outputs.image_tag }} \
            --set image.digest=${{ needs.build.outputs.image_digest }} \
            --set environment=dev \
            --wait \
            --timeout 5m
      
      - name: Post-deployment tests
        run: |
          # Wait for rollout
          kubectl rollout status deployment/myapp-dev -n development --timeout=5m
          
          # Run smoke tests
          kubectl run smoke-test --image=curlimages/curl --rm -it --restart=Never -n development -- \
            curl -f http://myapp-dev:8080/actuator/health
  
  # JOB 4: Deploy to Staging (Auto)
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [deploy-dev]
    if: github.ref == 'refs/heads/main'  # Only main branch
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Helm and kubectl
        uses: azure/setup-helm@v3
      
      - name: Configure staging cluster
        run: |
          echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config
      
      - name: Deploy to staging
        run: |
          helm upgrade --install myapp-staging ${{ env.HELM_CHART_PATH }} \
            --namespace staging \
            --create-namespace \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ needs.build.outputs.image_tag }} \
            --set environment=staging \
            --set replicaCount=2 \
            --wait \
            --timeout 5m
      
      - name: Run integration tests on staging
        run: |
          # Run full test suite
          helm test myapp-staging -n staging
          
          # Performance test (if applicable)
          # ./scripts/performance-test.sh
  
  # JOB 5: Deploy to Production (Manual Approval)
  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    if: github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production'
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup Helm
        uses: azure/setup-helm@v3
      
      - name: Configure production cluster
        run: |
          echo "${{ secrets.PROD_KUBECONFIG }}" | base64 --decode > $HOME/.kube/config
      
      - name: Backup current state
        run: |
          # Backup current deployment for rollback
          helm get values myapp-prod -n production > backup-values-$(date +%Y%m%d-%H%M%S).yaml
      
      - name: Deploy to production
        run: |
          helm upgrade --install myapp-prod ${{ env.HELM_CHART_PATH }} \
            --namespace production \
            --create-namespace \
            --set image.repository=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }} \
            --set image.tag=${{ needs.build.outputs.image_tag }} \
            --set environment=prod \
            --set replicaCount=5 \
            --set autoscaling.enabled=true \
            --atomic \
            --wait \
            --timeout 10m
      
      - name: Canary verification
        run: |
          # Monitor for 5 minutes
          echo "Monitoring canary deployment..."
          sleep 300
          
          # Check error rate
          ERROR_RATE=$(kubectl get pods -n production -l app=myapp -o json | jq '.items[].metadata.name' | xargs -I {} kubectl logs -n production {} --tail=100 | grep -c ERROR)
          if [ $ERROR_RATE -gt 5 ]; then
            echo "Error rate too high, rolling back..."
            helm rollback myapp-prod -n production
            exit 1
          fi
      
      - name: Notify team
        uses: slackapi/slack-github-action@v1.24
        with:
          payload: |
            {
              "text": "✅ Deployment to production completed! Version: ${{ needs.build.outputs.image_tag }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "✅ *Deployment Successful*\nVersion: ${{ needs.build.outputs.image_tag }}\nEnvironment: Production\nDigest: ${{ needs.build.outputs.image_digest }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### GitHub Actions Security Best Practices

```yaml
# ✅ USE: Environment-specific secrets
# Environment: production
secrets:
  PROD_KUBECONFIG: ${{ secrets.PROD_KUBECONFIG }}

# ✅ USE: OIDC instead of long-lived credentials
permissions:
  id-token: write
  contents: read

# ✅ USE: OpenID Connect for AWS
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-role
    aws-region: us-east-1

# ❌ DON'T: Print secrets
- run: echo ${{ secrets.PASSWORD }}  # Never!

# ✅ DO: Use environment for sensitive steps
- name: Deploy to production
  environment: production
  run: ./deploy.sh
```

---

## Chapter 4: Semantic Versioning and Changelog

### Automating Version Management

**`scripts/version.sh`:**
```bash
#!/bin/bash
# Automated version bumping based on commits

# Get current version from git tag
CURRENT_VERSION=$(git describe --tags --abbrev=0 2>/dev/null || echo "0.0.0")

# Parse commit messages to determine version bump
if git log -1 --pretty=%B | grep -q "^feat:"; then
    # Minor bump for new features
    NEW_VERSION=$(echo $CURRENT_VERSION | awk -F. '{$2++; $3=0; print}' OFS=.)
elif git log -1 --pretty=%B | grep -q "^fix:"; then
    # Patch bump for bug fixes
    NEW_VERSION=$(echo $CURRENT_VERSION | awk -F. '{$3++; print}' OFS=.)
elif git log -1 --pretty=%B | grep -q "!:"; then
    # Major bump for breaking changes
    NEW_VERSION=$(echo $CURRENT_VERSION | awk -F. '{$1++; $2=0; $3=0; print}' OFS=.)
else
    # No version bump
    NEW_VERSION=$CURRENT_VERSION
fi

echo "Current: $CURRENT_VERSION"
echo "New: $NEW_VERSION"

# Create and push tag
git tag -a "v$NEW_VERSION" -m "Release v$NEW_VERSION"
git push origin "v$NEW_VERSION"
```

**Automated Changelog Generation:**
```yaml
# .github/workflows/release.yml
name: Create Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0
      
      - name: Generate changelog
        id: changelog
        uses: metcalfc/changelog-generator@v4.0.1
        with:
          myToken: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Create Release
        uses: softprops/action-gh-release@v1
        with:
          body: ${{ steps.changelog.outputs.changelog }}
          prerelease: false
          draft: false
          files: |
            target/*.jar
            helm-chart-*.tgz
```

---

## Chapter 5: GitLab CI Alternative

For GitLab users, here's the equivalent pipeline:

**`.gitlab-ci.yml`:**
```yaml
stages:
  - test
  - build
  - scan
  - deploy-dev
  - deploy-staging
  - deploy-prod

variables:
  REGISTRY: $CI_REGISTRY
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  HELM_CHART_PATH: ./helm-chart

cache:
  paths:
    - .m2/repository/

# Job 1: Test
test:
  stage: test
  image: maven:3.8-openjdk-17
  script:
    - mvn test
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths:
      - target/surefire-reports/
    expire_in: 1 week

# Job 2: Build Docker image
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$IMAGE_TAG .
    - docker push $CI_REGISTRY_IMAGE:$IMAGE_TAG
    - docker tag $CI_REGISTRY_IMAGE:$IMAGE_TAG $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:latest

# Job 3: Security scan
security-scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image --severity HIGH,CRITICAL --exit-code 1 $CI_REGISTRY_IMAGE:$IMAGE_TAG

# Job 4: Deploy to development
deploy-dev:
  stage: deploy-dev
  image: alpine/helm:latest
  environment:
    name: development
    url: https://dev.myapp.com
  only:
    - main
    - develop
  script:
    - helm upgrade --install myapp-dev $HELM_CHART_PATH \
        --namespace development \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$IMAGE_TAG

# Job 5: Deploy to staging (manual)
deploy-staging:
  stage: deploy-staging
  image: alpine/helm:latest
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - main
  when: manual
  script:
    - helm upgrade --install myapp-staging $HELM_CHART_PATH \
        --namespace staging \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$IMAGE_TAG \
        --set replicaCount=2

# Job 6: Deploy to production (manual + approval)
deploy-prod:
  stage: deploy-prod
  image: alpine/helm:latest
  environment:
    name: production
    url: https://myapp.com
  only:
    - main
  when: manual
  script:
    - helm upgrade --install myapp-prod $HELM_CHART_PATH \
        --namespace production \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --set image.tag=$IMAGE_TAG \
        --set replicaCount=5 \
        --atomic
```

---

## Chapter 6: Multi-Environment Promotion Strategy

### GitOps with Argo CD

**What is GitOps?** Git repository as single source of truth. Argo CD syncs cluster to Git.

**`environments/` directory structure:**
```
environments/
├── base/                    # Base configuration
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── deployment-patch.yaml
│   │   └── kustomization.yaml
│   ├── staging/
│   │   ├── deployment-patch.yaml
│   │   └── kustomization.yaml
│   └── prod/
│       ├── deployment-patch.yaml
│       └── kustomization.yaml
└── argocd/
    ├── applications/
    │   ├── dev-app.yaml
    │   ├── staging-app.yaml
    │   └── prod-app.yaml
    └── projects/
        └── myapp-project.yaml
```

**Argo CD Application:**
```yaml
# argocd/applications/prod-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-prod
  namespace: argocd
spec:
  project: myapp-project
  
  source:
    repoURL: https://github.com/yourorg/myapp
    targetRevision: main
    path: environments/overlays/prod
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true      # Delete resources not in Git
      selfHeal: true   # Fix drift
    syncOptions:
      - CreateNamespace=true
    
    # Manual approval for production
    automated: {}
    # Uncomment for automatic sync:
    # automated:
    #   prune: true
    #   selfHeal: true
```

**Promotion Workflow:**
```bash
# Developer workflow
git checkout -b feature/new-feature
# ... make changes ...
git commit -m "feat: add new feature"
git push origin feature/new-feature

# Create PR
# → CI runs tests
# → Preview environment created
# → Merge to main

# After merge to main:
# → Argo CD auto-syncs to dev (if configured)
# → Manual promotion to staging:
kubectl patch application myapp-staging -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"main"}}}'

# → Manual promotion to production:
kubectl patch application myapp-prod -n argocd --type merge -p '{"spec":{"source":{"targetRevision":"main"}}}'
```

---

## Chapter 7: Complete Workflow Example

### Developer's Day with CI/CD

**Morning: Feature Development**
```bash
# 1. Create feature branch
git checkout -b feature/add-metrics

# 2. Write code
# ... edit Spring Boot app ...

# 3. Test locally
./mvnw test
docker build -t myapp:test .
docker run -p 8080:8080 myapp:test

# 4. Commit with conventional commit message
git add .
git commit -m "feat: add prometheus metrics endpoint"

# 5. Push and create PR
git push origin feature/add-metrics
gh pr create --title "feat: add metrics" --body "Adds /metrics endpoint"
```

**Automatically Happens:**
```yaml
# GitHub Actions runs:
1. ✅ Unit tests pass (45 seconds)
2. ✅ Integration tests pass (2 minutes)
3. ✅ Security scan - 0 vulnerabilities
4. 🏗️ Docker image built: myapp:abc123def
5. 🚀 Deployed to preview environment
6. 🔗 Preview URL: https://pr-123.dev.myapp.com
```

**After PR Approval:**
```yaml
# Merge to main triggers:
1. ✅ All tests pass
2. 🏷️ Version calculated: v1.2.4 (from feat: commit)
3. 📦 Image built: myapp:v1.2.4
4. 🔍 Deep security scan
5. 🚀 Auto-deploy to dev
6. 📧 Team notified
```

**Manual Promotion to Staging:**
```bash
# Developer or DevOps clicks "Approve" in GitHub Environments
# → Deploys to staging
# → Runs integration tests
# → Performance tests
```

**Manual Promotion to Production:**
```bash
# After staging verification
# → Click "Deploy to Production"
# → 10-minute canary period
# → Auto-rollback if errors detected
# → Slack notification
```

---

## Summary: CI/CD Checklist

| Component | Status | Tool |
|-----------|--------|------|
| Source Control | ✅ | GitHub/GitLab |
| CI Pipeline | ✅ | GitHub Actions |
| Container Registry | ✅ | Docker Hub/ECR |
| Image Scanning | ✅ | Trivy |
| Version Management | ✅ | Semantic versioning |
| Multi-environment | ✅ | Dev/Staging/Prod |
| Manual Approval | ✅ | Environments |
| Automatic Rollback | ✅ | --atomic flag |
| Notifications | ✅ | Slack |
| GitOps (optional) | ⬜ | Argo CD |

## Common CI/CD Pitfalls

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| **Long build times** | Pipeline takes 20+ min | Cache dependencies, parallel jobs |
| **Flaky tests** | Random failures | Retry logic, fix flaky tests |
| **Secret leakage** | Secrets in logs | Use environment, mask secrets |
| **Registry rate limits** | 429 errors | Use authenticated pulls, cache |
| **Concurrent deployments** | Race conditions | Use environment locks |
| **No rollback plan** | Broken production | Always have rollback strategy |

## Practice Exercises

### Exercise 1: Set Up Your First Pipeline
Create GitHub Actions workflow that builds and pushes your Part 1 app to Docker Hub.

### Exercise 2: Add Security Scanning
Integrate Trivy scanner and fail build on CRITICAL vulnerabilities.

### Exercise 3: Multi-Environment Deployment
Set up dev/staging/prod environments with manual approval for prod.

### Exercise 4: Automatic Versioning
Implement semantic versioning based on commit messages.

### Exercise 5: GitOps with Argo CD
Install Argo CD locally and sync your app from Git.

## Next Steps

After mastering CI/CD, you're ready for:
- **Part 5: Networking & Ingress** - TLS, DNS, advanced routing
- **Part 6: Storage & Stateful Apps** - Databases in Kubernetes
- **Part 7: Security** - RBAC, network policies, PodSecurity

---

**Ready for Part 5?** Let me know and I'll create **Networking & Ingress** covering:
- Ingress controllers (nginx, traefik, AWS ALB)
- TLS/SSL certificates with cert-manager
- DNS management (ExternalDNS)
- Service meshes (Istio, Linkerd)
- Network policies for security

Would you also like me to:
1. Provide a starter GitHub repository with all these workflows?
2. Create a Terraform module for setting up EKS + ECR + GitHub OIDC?
3. Add blue-green deployment strategies?
4. Create a troubleshooting guide for common pipeline failures?