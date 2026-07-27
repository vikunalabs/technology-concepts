# Part 4: Container Registry & CI/CD — Automating Your Deployments

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Intermediate
> **Prerequisites:** Completed Parts 1–3, or comfortable with Docker, Kubernetes, and Helm basics
> **Time to complete:** 4–5 hours
> **What you'll learn:** Container registry best practices, complete GitHub Actions CI/CD pipeline, semantic versioning, GitLab CI, GitOps with Argo CD, and the `helm diff` plugin wired into pull request reviews

---

## What This Part Covers

Up to now, every deployment in this series has been a manual command in a terminal. You built the image, pushed it, ran `helm upgrade`. That process works for one developer, occasionally. It doesn't scale — it's slow, error-prone, undocumented, and impossible to audit.

This part automates the entire path from `git push` to running in production. By the end, you'll have a pipeline where merging a pull request automatically tests, builds, scans, and deploys your application — with manual approval gates before production and automatic rollback if something goes wrong. You'll also understand GitOps with Argo CD, where Git becomes the single source of truth for cluster state.

---

## Chapter 1: Container Registry Deep Dive

### The Gap Between Where You Are and Where You're Going

Today's workflow:
```
Write code → mvn package (or gradle build) → docker build → docker push → helm upgrade
↑ manual      ↑ manual       ↑ manual        ↑ manual      ↑ manual
```

Target workflow:
```
git push → pipeline runs → helm upgrade to prod (with approval)
↑ one action  ↑ automatic    ↑ automatic
```

The container registry is the handoff point between these two worlds. The pipeline builds and pushes the image. The cluster pulls it. The registry must be reliable, secure, and integrated with both ends of this pipeline.

### Registry Comparison

| Registry | Free Tier | Best For | Key Feature |
|----------|-----------|----------|-------------|
| **Docker Hub** | 1 private repo, unlimited public | Learning, open source | Universal, no account needed to pull public images |
| **GitHub Container Registry (ghcr.io)** | Free for public, included in Actions | Teams on GitHub | Native OIDC with GitHub Actions — no stored credentials |
| **AWS ECR** | 500 MB free, pay per GB | EKS workloads | IAM integration, scan on push, lifecycle policies |
| **Google Artifact Registry** | 500 MB free, pay per GB | GKE workloads | Multi-format (Docker, Maven, npm), Workload Identity |
| **Azure Container Registry** | Basic ~$5/mo, no free tier | AKS workloads | Geo-replication, tasks for automated builds |
| **GitLab Registry** | Included with GitLab | Teams on GitLab | Zero config with GitLab CI, scoped to project |

**The principle for choosing:** match the registry to the cluster's cloud provider. When your cluster runs on AWS, use ECR. Authentication is handled by IAM — no stored credentials, automatic rotation, and no token expiry surprises during deployments.

### Registry Best Practices

**Image scanning on push**

Every image pushed to production should be scanned for known vulnerabilities before it's deployed. Most registries support this natively:

```bash
# AWS ECR — enable scan on push when creating the repository
aws ecr create-repository \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true \
  --region us-east-1

# Check scan results after push
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=v1.2.3 \
  --region us-east-1
```

For Docker Hub and GCR, use Trivy in the CI pipeline instead (covered in Chapter 3).

**Lifecycle policies — automatic cleanup**

Without cleanup, registries fill up with thousands of old images. Each stored gigabyte costs money. Lifecycle policies automatically delete old images:

```bash
# AWS ECR lifecycle policy — keep last 30 tagged images, delete untagged after 7 days
aws ecr put-lifecycle-policy \
  --repository-name myapp \
  --lifecycle-policy-text '{
    "rules": [
      {
        "rulePriority": 1,
        "description": "Delete untagged images after 7 days",
        "selection": {
          "tagStatus": "untagged",
          "countType": "sinceImagePushed",
          "countUnit": "days",
          "countNumber": 7
        },
        "action": { "type": "expire" }
      },
      {
        "rulePriority": 2,
        "description": "Keep only 30 tagged releases",
        "selection": {
          "tagStatus": "tagged",
          "tagPrefixList": ["v"],
          "countType": "imageCountMoreThan",
          "countNumber": 30
        },
        "action": { "type": "expire" }
      }
    ]
  }'
```

**Immutable tags**

Once you push `v1.2.3`, that tag should never be overwritten with a different image. Immutability guarantees that `v1.2.3` means the same thing everywhere, forever:

```bash
# AWS ECR — enable tag immutability
aws ecr put-image-tag-mutability \
  --repository-name myapp \
  --image-tag-mutability IMMUTABLE \
  --region us-east-1
```

With immutable tags, attempts to push a different image to an existing tag fail with an error, preventing accidental overwrites.

**The complete tagging strategy**

For a production CI/CD pipeline, every image build produces two tags:

```bash
# Git SHA tag — absolute traceability from running container back to commit
docker tag myapp:latest myregistry.com/myapp:git-${GITHUB_SHA::7}

# Semantic version tag — human-readable, used for deployments and rollbacks
docker tag myapp:latest myregistry.com/myapp:v1.2.3
```

The SHA tag answers "exactly which code is this?". The version tag answers "which release is this?" Both are needed. In CI/CD pipelines, the `docker/metadata-action` (covered in Chapter 3) generates both automatically from your git tags and commits.

---

## Chapter 2: CI/CD Concepts

### What CI/CD Actually Is

Continuous Integration/Continuous Delivery is often described abstractly. Here's the concrete version:

**CI (Continuous Integration):** Every code change is automatically tested. You cannot merge code that breaks tests. The build is always in a known-good state.

**CD (Continuous Delivery):** Every successful build produces a deployable artefact that can be released at any time. The deployment to production is triggered manually — but the artefact is always ready.

**Continuous Deployment** (different from Delivery): Every successful build is automatically deployed to production without human approval. Appropriate only for teams with excellent monitoring, fast rollback, and very high test coverage.

The assembly line analogy maps this well. A car assembly line doesn't wait for a human to check each station — components move automatically through each stage, with quality checks at every step. A problem at any station stops the line immediately. Shipping a bad car is far more expensive than stopping the line.

Your CI/CD pipeline is that assembly line. Code is the raw material. A running production deployment is the finished product. Automated checks at every stage catch problems early — when they're cheap to fix — rather than in production, where they're expensive.

### The Pipeline Stages

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  Commit  │ → │   Test   │ → │  Build   │ → │   Scan   │ → │   Push   │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
                  Fail here:     Fail here:     Fail here:
                  • Unit tests   • Compile      • Critical CVE
                  • Integration  • Dockerfile   • Secrets in image
                  • Coverage     • Lint

┌──────────────┐   ┌─────────────────┐   ┌────────────┐   ┌──────────────┐
│ Deploy Dev   │ → │ Deploy Staging  │ → │  Approve   │ → │ Deploy Prod  │
│ (automatic)  │   │ (automatic)     │   │ (manual)   │   │ (automatic)  │
└──────────────┘   └─────────────────┘   └────────────┘   └──────────────┘
  Fail here:          Fail here:            Human reviews:
  • Health check      • Smoke tests         • helm diff output
  • Probe timeout     • Integration tests   • Staging test results
```

Everything up to "Push" runs on every commit and every pull request. Deployment to dev and staging runs automatically on merges to main. Production requires explicit human approval.

### Tool Comparison

| Tool | Model | Best for | Learning curve |
|------|-------|----------|----------------|
| **GitHub Actions** | YAML workflows, runs in GitHub | Teams on GitHub | Low — declarative, huge marketplace |
| **GitLab CI** | YAML pipelines, built into GitLab | Teams on GitLab | Low — similar to Actions, native registry |
| **Jenkins** | Groovy/declarative pipelines, self-hosted | Large enterprises, custom needs | High — flexible but complex |
| **CircleCI** | YAML config, hosted service | Any git host | Medium — good Docker support |
| **Argo CD** | GitOps controller in cluster | Kubernetes-native delivery | Medium — different mental model |
| **Tekton** | Kubernetes-native CRD-based pipelines | Kubernetes-first teams | High — powerful, verbose |

This part covers GitHub Actions (Chapters 3–4), GitLab CI (Chapter 5), and Argo CD (Chapter 6). Jenkins and CircleCI follow similar patterns — the concepts translate directly.

---

## Chapter 3: GitHub Actions — A Complete Pipeline

### Core Concepts

A GitHub Actions **workflow** is a YAML file in `.github/workflows/`. It defines what runs, when it runs, and in what order.

```
Workflow (.github/workflows/ci.yml)
  └── triggered by Event (push, pull_request, schedule, manual)
       └── runs Jobs in parallel (or sequentially with needs:)
            └── each Job runs Steps in sequence
                 └── each Step is a shell command OR an Action (reusable module)
```

Every job runs on a **runner** — a fresh virtual machine (ubuntu-latest, windows-latest, macos-latest) or a self-hosted machine. Runners are ephemeral: they're created for the job and destroyed when it completes. Nothing persists between jobs unless you explicitly cache or upload artefacts.

### Repository Setup

Before writing workflows, set up the project structure:

```
.github/
└── workflows/
    ├── ci.yml          # Runs on every PR and push: test + build + scan
    └── deploy.yml      # Runs on merge to main: deploy to all environments
```

**Secrets and environments** (set up in GitHub before the pipeline runs):

*Repository secrets* (`Settings → Secrets and variables → Actions`):
```
DOCKER_USERNAME     ← Docker Hub username
DOCKER_TOKEN        ← Docker Hub access token (not password)
```

*Environment secrets* (create Environments first at `Settings → Environments`):
```
Environment: development
  Secret: DEV_KUBECONFIG     ← base64-encoded kubeconfig for dev cluster

Environment: staging
  Secret: STAGING_KUBECONFIG ← base64-encoded kubeconfig for staging cluster

Environment: production
  Secret: PROD_KUBECONFIG    ← base64-encoded kubeconfig for prod cluster
  Required reviewers: [your-team]  ← enables manual approval gate
```

**Encoding kubeconfig — the `-w 0` flag**

```bash
# Encode kubeconfig for use as a GitHub secret
cat ~/.kube/config | base64 -w 0
#                          ↑
# -w 0 disables line wrapping on Linux.
# Without it, base64 inserts a newline every 76 characters.
# When GitHub Actions decodes the secret, those newlines produce
# an invalid kubeconfig file that kubectl silently rejects.
# On macOS, base64 doesn't wrap by default — but -w 0 is ignored
# harmlessly, so it's safe to always include.
```

### The CI Workflow — Test, Build, Scan, Push

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
  IMAGE_NAME: ${{ github.repository }}   # → yourusername/myapp

jobs:
  # ─────────────────────────────────────────────────────────
  # Job 1: Test
  # Runs on every push and pull request.
  # Subsequent jobs only run if this passes.
  # ─────────────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up JDK 26
      uses: actions/setup-java@v4
      with:
        java-version: '26'
        distribution: 'temurin'
        cache: maven            # Cache ~/.m2/repository between runs — saves ~2 minutes

    - name: Run unit tests
      run: ./mvnw test

    - name: Run integration tests
      run: ./mvnw verify -P integration-tests

    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()              # Upload even if tests fail — you need the report to diagnose
      with:
        name: test-results
        path: target/surefire-reports/
        retention-days: 30

  # ─────────────────────────────────────────────────────────
  # Job 2: Build and Push image
  # Only runs on push events (not pull requests from forks).
  # needs: test — only runs if test job passes.
  # ─────────────────────────────────────────────────────────
  build-and-push:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: test                 # Gate: only run if test passes
    if: github.event_name == 'push'   # Don't push images from PRs

    # outputs: expose values to downstream jobs
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      with:
        fetch-depth: 0          # Full history needed for semantic versioning tools

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      # Buildx enables build caching and multi-platform builds

    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_TOKEN }}

    - name: Extract image metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          # On push to main: tag with "latest" and short SHA
          type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
          type=sha,format=short,prefix=git-
          # On version tags (v1.2.3): generate semver tags
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
          type=semver,pattern={{major}}
      # Result for a push to main at commit abc1234:
      #   yourusername/myapp:latest
      #   yourusername/myapp:git-abc1234
      # Result for tag v1.2.3:
      #   yourusername/myapp:1.2.3
      #   yourusername/myapp:1.2
      #   yourusername/myapp:1
      #   yourusername/myapp:git-abc1234

    - name: Build and push image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        # GitHub Actions cache — dramatically speeds up subsequent builds
        # mode=max caches all layers, not just the final stage
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Output image digest
      run: echo "Image digest: ${{ steps.build.outputs.digest }}"

  # ─────────────────────────────────────────────────────────
  # Job 3: Security scan
  # Scans the pushed image for known vulnerabilities.
  # Runs after build-and-push so it scans the actual pushed image.
  # ─────────────────────────────────────────────────────────
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-and-push

    steps:
    - name: Log in to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_TOKEN }}

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:git-${{ github.sha }}
        format: table
        exit-code: '1'          # Fail the job (and block deployment) on critical findings
        ignore-unfixed: true    # Ignore CVEs with no available fix — can't fix what doesn't exist
        severity: 'CRITICAL'    # Only fail on CRITICAL; warn on HIGH

    - name: Upload Trivy results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: trivy-results.sarif   # Appears in GitHub Security tab
```

**Gradle equivalent for the `test` job:** the rest of this pipeline (Docker build, Trivy scan, push, deploy) is identical regardless of build tool — it operates on the image, not the source. Only the `test` job's steps change:

```yaml
    - name: Set up JDK 26
      uses: actions/setup-java@v4
      with:
        java-version: '26'
        distribution: 'temurin'
        cache: gradle            # Cache ~/.gradle/caches and ~/.gradle/wrapper between runs

    - name: Run unit tests
      run: ./gradlew test

    - name: Run integration tests
      run: ./gradlew integrationTest

    - name: Upload test results
      uses: actions/upload-artifact@v4
      if: always()
      with:
        name: test-results
        path: build/reports/tests/       # Gradle's report path, vs Maven's target/surefire-reports/
        retention-days: 30
```

The `cache: gradle` shorthand in `setup-java` caches the same things `cache: maven` does for Maven — dependency downloads — just from Gradle's cache directories instead of `~/.m2`. `integrationTest` above assumes a separate source set/task configured for integration tests (a common pattern, e.g. via the `java-test-fixtures` plugin or a custom `sourceSets` block); if your project runs everything through `test`, drop that step.

### The Deploy Workflow — Dev, Staging, Production

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:              # Allow manual trigger with custom inputs
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'staging'
        type: choice
        options: [development, staging, production]
      image-tag:
        description: 'Image tag to deploy (e.g. v1.2.3 or git-abc1234)'
        required: false

env:
  REGISTRY: docker.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ─────────────────────────────────────────────────────────
  # Job 1: Deploy to Development
  # Runs automatically on every merge to main.
  # ─────────────────────────────────────────────────────────
  deploy-dev:
    name: Deploy → Development
    runs-on: ubuntu-latest
    environment:
      name: development
      url: https://dev.myapp.example.com

    steps:
    - uses: actions/checkout@v4

    - name: Set up Helm
      uses: azure/setup-helm@v4
      with:
        version: '3.13.0'       # Pin version — don't let it drift

    - name: Install helm-diff plugin
      run: helm plugin install https://github.com/databus23/helm-diff

    - name: Configure kubectl
      run: |
        mkdir -p $HOME/.kube
        # Decode the base64-encoded kubeconfig stored in the environment secret.
        # The -w 0 when encoding prevents line wrapping; | base64 -d decodes it cleanly.
        echo "${{ secrets.DEV_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    - name: Set image tag
      id: tag
      run: |
        # Use the git short SHA as the image tag for dev deployments
        echo "tag=git-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

    - name: Helm diff — preview changes
      run: |
        helm dependency update ./helm-chart
        helm diff upgrade myapp-dev ./helm-chart \
          -f values.yaml \
          -f values-dev.yaml \
          --set image.tag=${{ steps.tag.outputs.tag }} \
          --namespace development \
          --allow-unreleased         # Don't fail if release doesn't exist yet
      continue-on-error: true        # Diff is informational — don't block on diff errors

    - name: Deploy to development
      run: |
        helm upgrade --install myapp-dev ./helm-chart \
          -f values.yaml \
          -f values-dev.yaml \
          --set image.tag=${{ steps.tag.outputs.tag }} \
          --namespace development \
          --create-namespace \
          --wait \
          --timeout 10m \
          --atomic

    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp-dev \
          -n development \
          --timeout=5m
        kubectl get pods -n development

  # ─────────────────────────────────────────────────────────
  # Job 2: Deploy to Staging
  # Runs after dev succeeds. Runs smoke tests.
  # ─────────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy → Staging
    runs-on: ubuntu-latest
    needs: deploy-dev             # Only deploy to staging if dev succeeded
    environment:
      name: staging
      url: https://staging.myapp.example.com

    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4
      with:
        version: '3.13.0'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    - name: Set image tag
      id: tag
      run: echo "tag=git-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

    - name: Deploy to staging
      run: |
        helm dependency update ./helm-chart
        helm upgrade --install myapp-staging ./helm-chart \
          -f values.yaml \
          -f values-staging.yaml \
          --set image.tag=${{ steps.tag.outputs.tag }} \
          --namespace staging \
          --create-namespace \
          --wait \
          --timeout 10m \
          --atomic

    - name: Run smoke tests
      run: |
        helm test myapp-staging -n staging --timeout 5m
        echo "Smoke tests passed ✓"

    - name: Run integration tests against staging
      run: |
        BASE_URL="https://staging.myapp.example.com"
        curl -sf "$BASE_URL/actuator/health" || exit 1
        curl -sf "$BASE_URL/api/hello" | grep -q "Hello" || exit 1
        echo "Integration tests passed ✓"

  # ─────────────────────────────────────────────────────────
  # Job 3: Deploy to Production
  # Requires manual approval via GitHub Environment protection rules.
  # The "environment: production" line triggers the approval gate —
  # configured in GitHub Settings → Environments → production → Required reviewers.
  #
  # Pipeline pauses here. A reviewer sees the pending deployment,
  # reviews the staging results, and clicks Approve (or Reject).
  # ─────────────────────────────────────────────────────────
  deploy-production:
    name: Deploy → Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://api.myapp.example.com

    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4
      with:
        version: '3.13.0'

    - name: Install helm-diff plugin
      run: helm plugin install https://github.com/databus23/helm-diff

    - name: Configure kubectl
      run: |
        echo "${{ secrets.PROD_KUBECONFIG }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    - name: Set image tag
      id: tag
      run: echo "tag=git-$(echo ${{ github.sha }} | cut -c1-7)" >> $GITHUB_OUTPUT

    - name: Helm diff — show production changes
      # This runs AFTER approval, giving the approver a final preview
      # of exactly what will change before the deployment executes.
      run: |
        helm dependency update ./helm-chart
        helm diff upgrade myapp-prod ./helm-chart \
          -f values.yaml \
          -f values-prod.yaml \
          --set image.tag=${{ steps.tag.outputs.tag }} \
          --namespace production

    - name: Deploy to production
      run: |
        helm upgrade --install myapp-prod ./helm-chart \
          -f values.yaml \
          -f values-prod.yaml \
          --set image.tag=${{ steps.tag.outputs.tag }} \
          --namespace production \
          --create-namespace \
          --wait \
          --timeout 15m \
          --atomic                # Auto-rollback if pods don't become healthy

    - name: Verify production health
      run: |
        kubectl rollout status deployment/myapp-prod -n production --timeout=10m

        # Poll the health endpoint until it responds or timeout
        for i in $(seq 1 12); do
          STATUS=$(curl -sf https://api.myapp.example.com/actuator/health/readiness \
            -o /dev/null -w "%{http_code}" 2>/dev/null || echo "000")
          if [ "$STATUS" = "200" ]; then
            echo "Production health check passed ✓"
            exit 0
          fi
          echo "Attempt $i/12: status=$STATUS, waiting 10s..."
          sleep 10
        done
        echo "Production health check timed out ✗"
        exit 1

    - name: Notify Slack on success
      if: success()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "✅ *${{ github.event.repository.name }}* deployed to production",
            "attachments": [{
              "color": "good",
              "fields": [
                {"title": "Version", "value": "${{ steps.tag.outputs.tag }}", "short": true},
                {"title": "By", "value": "${{ github.actor }}", "short": true},
                {"title": "Commit", "value": "${{ github.event.head_commit.message }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

    - name: Notify Slack on failure
      if: failure()
      uses: slackapi/slack-github-action@v1
      with:
        payload: |
          {
            "text": "❌ *${{ github.event.repository.name }}* production deployment FAILED — rolled back automatically",
            "attachments": [{
              "color": "danger",
              "fields": [
                {"title": "Commit", "value": "${{ github.sha }}", "short": true},
                {"title": "Run", "value": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}", "short": false}
              ]
            }]
          }
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### `helm diff` as a Pull Request Comment

Beyond running `helm diff` in the deploy job, you can run it on every PR and post the diff as a PR comment. Reviewers see exactly what the cluster change will be alongside the code change:

```yaml
# .github/workflows/helm-diff-pr.yml
name: Helm Diff on PR

on:
  pull_request:
    branches: [main]
    paths:
      - 'helm-chart/**'
      - 'values*.yaml'

jobs:
  helm-diff:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write    # Needed to post PR comments

    steps:
    - uses: actions/checkout@v4
    - uses: azure/setup-helm@v4
      with:
        version: '3.13.0'

    - name: Install helm-diff
      run: helm plugin install https://github.com/databus23/helm-diff

    - name: Configure kubectl (staging cluster for diff)
      run: |
        echo "${{ secrets.STAGING_KUBECONFIG }}" | base64 -d > $HOME/.kube/config

    - name: Generate helm diff
      id: diff
      run: |
        helm dependency update ./helm-chart
        # Capture diff output — could be multiline
        DIFF=$(helm diff upgrade myapp-staging ./helm-chart \
          -f values.yaml \
          -f values-staging.yaml \
          --namespace staging \
          --allow-unreleased \
          --no-color 2>&1 || true)

        # Store in GitHub output (handles multiline via heredoc)
        EOF=$(dd if=/dev/urandom bs=15 count=1 status=none | base64)
        echo "diff<<$EOF" >> $GITHUB_OUTPUT
        echo "$DIFF" >> $GITHUB_OUTPUT
        echo "$EOF" >> $GITHUB_OUTPUT

    - name: Post diff as PR comment
      uses: actions/github-script@v7
      with:
        script: |
          const diff = `${{ steps.diff.outputs.diff }}`;
          const body = diff.trim()
            ? `### Helm Diff (staging)\n\`\`\`diff\n${diff}\n\`\`\``
            : '### Helm Diff (staging)\n_No changes to Kubernetes resources._';

          // Find and update existing comment, or create new one
          const { data: comments } = await github.rest.issues.listComments({
            owner: context.repo.owner,
            repo: context.repo.repo,
            issue_number: context.issue.number,
          });
          const existing = comments.find(c => c.body.startsWith('### Helm Diff'));
          if (existing) {
            await github.rest.issues.updateComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              comment_id: existing.id,
              body
            });
          } else {
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body
            });
          }
```

### OIDC — Eliminating Long-Lived Credentials

The kubeconfig approach above stores static cluster credentials in GitHub Secrets. These credentials never expire and represent a permanent secret that could be leaked.

For AWS EKS, GitHub Actions supports OIDC — short-lived tokens that GitHub generates per-job, exchanged for temporary AWS credentials. No stored access keys, no rotation needed:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # Required for OIDC token request
      contents: read

    steps:
    - name: Configure AWS credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
        aws-region: us-east-1
        # No access key or secret needed — GitHub mints a short-lived token,
        # AWS verifies it came from your repository, and issues 1-hour credentials

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Update kubeconfig for EKS
      run: aws eks update-kubeconfig --name my-cluster --region us-east-1
      # No stored kubeconfig — generated fresh per job from the OIDC credentials
```

**Setting up the AWS IAM trust policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub":
          "repo:yourusername/myapp:*"
      }
    }
  }]
}
```

The `sub` condition restricts this trust policy to only your repository — not any random GitHub Actions run.

---

## Chapter 4: Semantic Versioning and Changelog Automation

### What Semantic Versioning Is

Semantic versioning (semver) is a versioning convention: `MAJOR.MINOR.PATCH`.

| Part | Incremented when | Example |
|------|-----------------|---------|
| **PATCH** | Backwards-compatible bug fix | `1.2.3` → `1.2.4` |
| **MINOR** | New feature, backwards compatible | `1.2.3` → `1.3.0` |
| **MAJOR** | Breaking change — existing users must update | `1.2.3` → `2.0.0` |

The value of semver is the contract it implies. When users see `v1.2.4`, they know they can upgrade from `v1.2.3` safely — it's a bug fix. When they see `v2.0.0`, they know something significant changed and they need to review before upgrading.

### Conventional Commits — Human and Machine Readable

Conventional commits is a commit message format that encodes the type of change. It makes changelogs automatable and version bumps deterministic:

```
feat: add order export to CSV
 ↑
 type

fix: correct tax calculation for EU countries

docs: update API authentication guide

refactor: extract payment processing to separate service

test: add integration tests for checkout flow

chore: update Spring Boot to 3.2.1

feat!: redesign checkout API (breaking change)
      ↑
      ! means breaking change → bumps MAJOR version
```

**Version bump rules:**
- Any `fix:` commit → bump PATCH
- Any `feat:` commit → bump MINOR  
- Any commit with `!` or `BREAKING CHANGE:` in footer → bump MAJOR
- `docs:`, `chore:`, `test:`, `refactor:` → no version bump

### Automated Version Bumping and Release

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    branches: [main]

jobs:
  release:
    name: Create Release
    runs-on: ubuntu-latest
    permissions:
      contents: write   # Needed to create tags and releases

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0  # Full history needed to calculate version bump

    - name: Determine next version
      id: version
      run: |
        # Get the latest tag
        LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
        echo "Latest tag: $LATEST_TAG"

        # Analyse commits since last tag to determine bump type
        COMMITS=$(git log ${LATEST_TAG}..HEAD --pretty=format:"%s")

        MAJOR=0; MINOR=0; PATCH=0
        while IFS= read -r commit; do
          if echo "$commit" | grep -qE '^feat!:|BREAKING CHANGE'; then
            MAJOR=1
          elif echo "$commit" | grep -qE '^feat:'; then
            MINOR=1
          elif echo "$commit" | grep -qE '^fix:'; then
            PATCH=1
          fi
        done <<< "$COMMITS"

        # Parse latest tag
        VERSION=${LATEST_TAG#v}
        IFS='.' read -r MA MI PA <<< "$VERSION"

        # Apply bump
        if [ "$MAJOR" = "1" ]; then
          NEW_VERSION="$((MA+1)).0.0"
        elif [ "$MINOR" = "1" ]; then
          NEW_VERSION="${MA}.$((MI+1)).0"
        elif [ "$PATCH" = "1" ]; then
          NEW_VERSION="${MA}.${MI}.$((PA+1))"
        else
          echo "No releasable commits found — skipping release"
          echo "skip=true" >> $GITHUB_OUTPUT
          exit 0
        fi

        echo "New version: v${NEW_VERSION}"
        echo "version=v${NEW_VERSION}" >> $GITHUB_OUTPUT
        echo "skip=false" >> $GITHUB_OUTPUT

    - name: Generate changelog
      id: changelog
      if: steps.version.outputs.skip != 'true'
      run: |
        LATEST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
        if [ -n "$LATEST_TAG" ]; then
          LOG=$(git log ${LATEST_TAG}..HEAD --pretty=format:"- %s (%h)")
        else
          LOG=$(git log --pretty=format:"- %s (%h)")
        fi

        # Group by type
        FEATURES=$(echo "$LOG" | grep "^- feat" || true)
        FIXES=$(echo "$LOG" | grep "^- fix" || true)
        OTHERS=$(echo "$LOG" | grep -vE "^- (feat|fix|docs|chore|test)" || true)

        CHANGELOG=""
        [ -n "$FEATURES" ] && CHANGELOG="${CHANGELOG}### Features\n${FEATURES}\n\n"
        [ -n "$FIXES" ]    && CHANGELOG="${CHANGELOG}### Bug Fixes\n${FIXES}\n\n"
        [ -n "$OTHERS" ]   && CHANGELOG="${CHANGELOG}### Other Changes\n${OTHERS}\n"

        echo "changelog<<EOF" >> $GITHUB_OUTPUT
        printf "$CHANGELOG" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT

    - name: Create Git tag and GitHub Release
      if: steps.version.outputs.skip != 'true'
      uses: softprops/action-gh-release@v1
      with:
        tag_name: ${{ steps.version.outputs.version }}
        name: Release ${{ steps.version.outputs.version }}
        body: ${{ steps.changelog.outputs.changelog }}
        draft: false
        prerelease: false
```

This workflow runs on every merge to main, determines the version bump from commit messages, creates a git tag, and publishes a GitHub Release with an auto-generated changelog. The CI workflow then picks up the tag and builds a versioned image (`v1.2.3` instead of just `git-abc1234`).

---

## Chapter 5: GitLab CI — The Full Equivalent Pipeline

### How GitLab CI Differs from GitHub Actions

GitLab CI defines pipelines in a single `.gitlab-ci.yml` file at the repository root. The mental model maps closely to GitHub Actions:

| GitHub Actions | GitLab CI | Notes |
|---------------|-----------|-------|
| Workflow | Pipeline | The top-level YAML file |
| Job | Job | A unit of work |
| Step | Script line | Commands within a job |
| Action | Component / include | Reusable pipeline fragment |
| Environment | Environment | Protected, with approval gates |
| `needs:` | `needs:` | Job dependency |
| `if:` | `rules:` | Conditional execution |
| `secrets.MY_SECRET` | `$MY_SECRET` | Predefined + custom variables |

GitLab provides built-in variables for common values — no setup required:

```bash
$CI_REGISTRY               # Your GitLab registry: registry.gitlab.com
$CI_REGISTRY_IMAGE         # Full image path: registry.gitlab.com/group/project
$CI_COMMIT_SHORT_SHA       # Short git SHA (8 chars)
$CI_COMMIT_TAG             # Set when pipeline runs from a tag (empty otherwise)
$CI_ENVIRONMENT_NAME       # The environment name from the job
$CI_PROJECT_NAME           # Repository name
```

### Complete `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - scan
  - deploy-dev
  - deploy-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_TAG: $CI_REGISTRY_IMAGE:git-$CI_COMMIT_SHORT_SHA

# ─────────────────────────────────────────────────────────
# Stage: test
# ─────────────────────────────────────────────────────────
test:
  stage: test
  image: maven:3.9-eclipse-temurin-26
  script:
    - mvn test verify
  cache:
    key: "${CI_PROJECT_ID}-maven"
    paths:
      - .m2/repository/
  artifacts:
    when: always          # Upload test results even on failure
    paths:
      - target/surefire-reports/
    expire_in: 30 days

# Gradle equivalent of the test stage above — swap this in if your project
# uses Gradle instead of Maven. Everything downstream (build, scan, deploy)
# is unchanged, since it operates on the built image, not the source.
# test:
#   stage: test
#   image: gradle:8.10-jdk26-alpine
#   script:
#     - gradle test integrationTest --no-daemon
#   cache:
#     key: "${CI_PROJECT_ID}-gradle"
#     paths:
#       - .gradle/caches/
#       - .gradle/wrapper/
#   artifacts:
#     when: always
#     paths:
#       - build/reports/tests/
#     expire_in: 30 days

# ─────────────────────────────────────────────────────────
# Stage: build
# ─────────────────────────────────────────────────────────
build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind     # Docker-in-Docker — needed to run docker build
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
    # Also tag with semver if this is a tagged commit
    - |
      if [ -n "$CI_COMMIT_TAG" ]; then
        docker tag $IMAGE_TAG $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
        docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == "main"   # Only build on main branch
    - if: $CI_COMMIT_TAG                # Or on version tags

# ─────────────────────────────────────────────────────────
# Stage: scan
# ─────────────────────────────────────────────────────────
scan:
  stage: scan
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]
  script:
    - trivy image
        --exit-code 1
        --ignore-unfixed
        --severity CRITICAL
        $IMAGE_TAG
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_COMMIT_TAG

# ─────────────────────────────────────────────────────────
# Stage: deploy-dev
# ─────────────────────────────────────────────────────────
deploy-dev:
  stage: deploy-dev
  image: alpine/helm:3.13.0
  before_script:
    - apk add --no-cache curl
    - helm plugin install https://github.com/databus23/helm-diff || true
    - echo "$DEV_KUBECONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - helm dependency update ./helm-chart
    - helm diff upgrade myapp-dev ./helm-chart
        -f values.yaml
        -f values-dev.yaml
        --set image.tag=git-$CI_COMMIT_SHORT_SHA
        --namespace development
        --allow-unreleased || true
    - helm upgrade --install myapp-dev ./helm-chart
        -f values.yaml
        -f values-dev.yaml
        --set image.tag=git-$CI_COMMIT_SHORT_SHA
        --namespace development
        --create-namespace
        --wait
        --timeout 10m
        --atomic
  environment:
    name: development
    url: https://dev.myapp.example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─────────────────────────────────────────────────────────
# Stage: deploy-staging
# ─────────────────────────────────────────────────────────
deploy-staging:
  stage: deploy-staging
  image: alpine/helm:3.13.0
  before_script:
    - echo "$STAGING_KUBECONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - helm dependency update ./helm-chart
    - helm upgrade --install myapp-staging ./helm-chart
        -f values.yaml
        -f values-staging.yaml
        --set image.tag=git-$CI_COMMIT_SHORT_SHA
        --namespace staging
        --create-namespace
        --wait
        --timeout 10m
        --atomic
    # Run smoke tests
    - helm test myapp-staging -n staging --timeout 5m
  environment:
    name: staging
    url: https://staging.myapp.example.com
  needs:
    - deploy-dev          # Only run if dev succeeded
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ─────────────────────────────────────────────────────────
# Stage: deploy-production
# when: manual — pipeline pauses here and shows a Play button.
# A team member must click it to proceed (the approval gate).
# Protected environments add an additional approval layer in GitLab UI.
# ─────────────────────────────────────────────────────────
deploy-production:
  stage: deploy-production
  image: alpine/helm:3.13.0
  before_script:
    - apk add --no-cache curl
    - helm plugin install https://github.com/databus23/helm-diff || true
    - echo "$PROD_KUBECONFIG" | base64 -d > /tmp/kubeconfig
    - export KUBECONFIG=/tmp/kubeconfig
  script:
    - helm dependency update ./helm-chart
    # Show what will change before deploying
    - helm diff upgrade myapp-prod ./helm-chart
        -f values.yaml
        -f values-prod.yaml
        --set image.tag=git-$CI_COMMIT_SHORT_SHA
        --namespace production || true
    - helm upgrade --install myapp-prod ./helm-chart
        -f values.yaml
        -f values-prod.yaml
        --set image.tag=git-$CI_COMMIT_SHORT_SHA
        --namespace production
        --create-namespace
        --wait
        --timeout 15m
        --atomic
  environment:
    name: production
    url: https://api.myapp.example.com
  needs:
    - deploy-staging
  when: manual            # Requires a human to click "Play" in the GitLab UI
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

**Setting up GitLab environments with approval:**

1. Go to `Settings → CI/CD → Environments`
2. Click the `production` environment → Edit
3. Enable "Required approval before deployment"
4. Add approvers

When the pipeline reaches `deploy-production`, it shows as blocked in the UI. Approvers receive a notification and can review the staging results, then click Approve — which triggers the `when: manual` job.

---

## Chapter 6: GitOps with Argo CD

### What GitOps Is — and Why It Matters

In the pipelines above, the CI/CD system actively pushes changes to the cluster. It connects to the cluster API, runs `helm upgrade`, and modifies state. The cluster is a passive recipient.

GitOps inverts this model. The cluster is not a passive recipient — it is an active reconciler. You commit the desired state to a Git repository. A controller running inside the cluster continuously watches that repository. When the cluster state drifts from what Git says it should be, the controller corrects it automatically.

```
Traditional CI/CD (push model):      GitOps (pull model):

Pipeline → kubectl apply             Git repo ← controller watches
         → helm upgrade              Cluster → controller syncs
         → (modifies cluster)        (cluster corrects itself)
```

Why does this matter?

**Auditability:** Every change to cluster state is a git commit. `git log` is a complete audit trail. Who changed what, when, and why.

**Recovery:** If a cluster is destroyed (node failure, accidental deletion), recreating it is a `git clone` and `argocd app sync`. The cluster converges to the desired state automatically.

**Drift detection:** If someone runs `kubectl edit` directly on the cluster, Argo CD detects the drift and either alerts or corrects it automatically. "Works in staging, broken in prod" becomes much harder when both are driven from the same Git repo.

**Security:** CI/CD systems no longer need cluster credentials. The Argo CD controller inside the cluster pulls — nothing external pushes.

### Installing Argo CD

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl wait --for=condition=available deployment --all -n argocd --timeout=5m

# Get the admin password (auto-generated on install)
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath='{.data.password}' | base64 -d

# Access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Visit: https://localhost:8080  Username: admin  Password: from above

# Install the CLI
brew install argocd   # macOS
# Or download from: https://github.com/argoproj/argo-cd/releases
argocd login localhost:8080 --insecure
```

### The Application Resource

An Argo CD Application tells the controller what to deploy and where to get it from:

```yaml
# argocd-app-staging.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-staging
  namespace: argocd          # Application resource always lives in argocd namespace
  finalizers:
    # When this Application is deleted, also delete all its managed resources
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/yourusername/myapp.git
    targetRevision: main      # Branch, tag, or commit SHA to track
    path: helm-chart          # Path within the repo to the chart

    helm:
      valueFiles:
        - values.yaml
        - values-staging.yaml
      parameters:
        # Override specific values programmatically
        - name: image.tag
          value: git-abc1234  # CI/CD updates this value when deploying

  destination:
    server: https://kubernetes.default.svc   # The cluster Argo CD is running in
    namespace: staging

  syncPolicy:
    automated:
      prune: true       # Delete resources that are no longer in Git
      selfHeal: true    # Revert manual changes to the cluster (drift correction)
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
```

```yaml
# argocd-app-production.yaml — same structure, different settings
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/yourusername/myapp.git
    targetRevision: main
    path: helm-chart
    helm:
      valueFiles:
        - values.yaml
        - values-prod.yaml

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true
      selfHeal: false    # Don't auto-correct in prod — alert instead, human decides
    syncOptions:
      - CreateNamespace=true
```

### How CI/CD and Argo CD Work Together

With Argo CD, the CI pipeline no longer runs `helm upgrade`. Instead, it updates a value in the Git repository (the image tag) and commits. Argo CD sees the commit and syncs the cluster:

```yaml
# .github/workflows/deploy-gitops.yml
jobs:
  update-image-tag:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
    - uses: actions/checkout@v4
      with:
        token: ${{ secrets.GITHUB_TOKEN }}

    - name: Update image tag in values-staging.yaml
      run: |
        NEW_TAG="git-$(echo ${{ github.sha }} | cut -c1-7)"

        # Use sed to update the tag in the values file
        sed -i "s/  tag: .*/  tag: \"${NEW_TAG}\"/" values-staging.yaml

        git config user.name "github-actions[bot]"
        git config user.email "github-actions[bot]@users.noreply.github.com"
        git add values-staging.yaml
        git commit -m "chore: update staging image to ${NEW_TAG} [skip ci]"
        # [skip ci] prevents the commit from triggering the pipeline again
        git push
```

After this commit, Argo CD detects the change to `values-staging.yaml`, renders the Helm chart with the new tag, and applies the result to the staging cluster — automatically, within a minute of the push.

### Kustomize Overlays for Multi-Environment GitOps

An alternative to Helm for GitOps is Kustomize, which Argo CD also supports natively. Kustomize uses a base configuration with environment-specific patches, rather than templates with values:

```
gitops-repo/
├── base/
│   ├── deployment.yaml   ← base Deployment with placeholders
│   ├── service.yaml
│   └── kustomization.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml   ← patches for dev
│   │   └── patch-replicas.yaml
│   ├── staging/
│   │   └── kustomization.yaml
│   └── prod/
│       ├── kustomization.yaml
│       └── patch-resources.yaml
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

images:
  - name: myapp
    newTag: v1.2.3         # CI/CD updates this line

replicas:
  - name: myapp
    count: 5

patches:
  - path: patch-resources.yaml  # Increases CPU/memory for prod
```

Whether you use Helm or Kustomize with Argo CD is a team preference. Helm is better for parameterised, reusable charts shared across teams. Kustomize is better for environment-specific patches on a single application's manifests.

---

## Chapter 7: The Complete Developer Workflow

### A Day in the Life — With CI/CD and Argo CD

Here is what the experience looks like from a developer's perspective once the pipeline is running:

**Monday morning: start a new feature**

```bash
git checkout -b feature/order-export
# ... write code ...
git commit -m "feat: add CSV export endpoint for orders"
git push origin feature/order-export
```

**Open a pull request:**
Within 2 minutes of the push, GitHub Actions runs the `ci.yml` workflow:
- Unit and integration tests run
- If you changed `helm-chart/` or `values*.yaml`, the Helm diff workflow triggers
- A comment appears on the PR showing exactly which Kubernetes resources would change

**Merge the PR:**
```bash
# After code review and approval
# Squash merge: git history stays clean
```

The moment the merge lands on `main`, `deploy.yml` triggers automatically:

```
Merge to main
  │
  ├── [2 min] Tests run again on the merged code
  ├── [5 min] Docker image built and pushed: git-abc1234
  ├── [3 min] Trivy scan — no critical CVEs
  │
  ├── [10 min] Deployed to development
  │     ↓ helm upgrade, --atomic, health checks pass
  │
  ├── [15 min] Deployed to staging
  │     ↓ smoke tests run, integration tests pass
  │
  └── [PAUSED] Production deployment waiting for approval
        ↑ Slack notification sent to #deployments channel
```

**Production approval:**
A senior engineer receives the Slack notification, clicks the link to the GitHub Actions run, reviews:
- What changed (from the `helm diff` output in the deploy log)
- Staging test results (linked artefacts)
- The commit that triggered this

They click **Approve** in the GitHub Environment protection gate.

```
Approval granted
  │
  ├── [15 min] Helm diff output logged
  ├── [10 min] Deployed to production, --atomic
  ├── [2 min]  Health check loop — all 200 OK
  └── [1 min]  Slack: "✅ myapp deployed to production by alice"
```

Total time from merge to production: approximately 45–60 minutes, zero manual steps after the approval.

### What Happens at Each Automated Step

**On every commit to any branch:**
- Linting (if configured)
- Unit tests

**On pull request open/update:**
- Full test suite
- Helm diff comment on PR (if chart files changed)

**On merge to main:**
- Full test suite
- Docker build + push
- Trivy security scan
- Deploy to dev (auto, ~10 min)
- Deploy to staging (auto, after dev succeeds, ~15 min)
- Smoke tests against staging
- Production waits for approval

**On production approval:**
- `helm diff` logged to pipeline output
- Deploy to production (`--atomic`, auto-rollback if unhealthy)
- Production health verification
- Slack notification

**On production deployment failure:**
- Helm `--atomic` automatically rolls back to previous release
- Slack notification: deployment failed + rollback triggered
- GitHub Actions run marked as failed
- No manual intervention needed to restore service

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `Error: invalid kubeconfig` in CI | base64 line wrapping — encoded without `-w 0` | `echo "$SECRET" \| base64 -d \| head -1` | Re-encode: `cat kubeconfig \| base64 -w 0` |
| `ImagePullBackOff` after CI push | Image pushed to wrong registry path, or pull secret missing | `kubectl describe pod <n>` → Events | Verify `IMAGE_NAME` matches values.yaml `image.repository`; check pull secret |
| Trivy scan fails on `CRITICAL` with no fix available | Unfixed CVE in base image | Check CVE details in scan output | Add `--ignore-unfixed` flag; or switch to a different base image |
| `helm diff` shows no changes when changes expected | Helm diff comparing against wrong release name or namespace | `helm list -A` | Verify release name and namespace match in diff command |
| Production environment not showing approval gate | Environment not configured in GitHub Settings | Settings → Environments | Create environment named exactly `production`, add required reviewers |
| `--atomic` rollback triggered unexpectedly | Pod not reaching Ready state within timeout | `kubectl describe pod <n>` → probe failures | Increase `--timeout`; check probe configuration |
| Argo CD app stuck in `Progressing` | Deployment rolling update taking too long | `argocd app get myapp-staging` | Check pod events; may be OOMKilled or probe failure |
| Argo CD shows `OutOfSync` immediately after sync | Manual change was made to cluster (`kubectl edit`) | `argocd app diff myapp-staging` | Revert manual change or commit it to Git; `selfHeal: true` auto-corrects |
| GitOps: image tag commit triggers pipeline loop | Commit message missing `[skip ci]` | Check Actions run history | Add `[skip ci]` to the automated commit message |
| Semantic version not bumping | Commit messages don't follow conventional format | `git log --oneline` | Enforce conventional commits with `commitlint` in a pre-commit hook |
| `when: manual` job in GitLab never unblocks | Pipeline not running on protected branch, or user lacks permission | GitLab CI/CD → Pipelines | Ensure user has `developer` role minimum; check pipeline rules |

---

## Practice Exercises

**Exercise 1 — End-to-end pipeline from scratch:**
Set up the GitHub Actions CI workflow from Chapter 3 (`ci.yml`) for your `hello-app` repository. Make a code change, push it, and watch the workflow run. Verify: (1) tests run, (2) image is built and pushed to Docker Hub, (3) Trivy scan runs. Read the Trivy output — what vulnerabilities, if any, does it find in the `eclipse-temurin:26-jre-alpine` base image?

**Exercise 2 — Manual approval gate:**
Set up the deploy workflow (`deploy.yml`) with GitHub Environments. Configure `staging` and `production` environments. Add yourself as a required reviewer for `production`. Push a change, let it auto-deploy to dev and staging, then observe the pipeline pause at production. Approve it, and watch the final deployment. Check `helm history` in the production namespace after it completes.

**Exercise 3 — `helm diff` in a pull request:**
Set up the `helm-diff-pr.yml` workflow. Make a change to `values-staging.yaml` (change replica count or image tag) on a feature branch. Open a pull request. Verify a comment appears on the PR showing the diff. Make a second change to the same PR and verify the comment updates rather than creating a duplicate.

**Exercise 4 — Conventional commits and versioning:**
Set up the release workflow from Chapter 4. Make three commits using conventional commit format: one `fix:`, one `feat:`, and one `docs:`. Push to main. Verify: (1) the `fix:` and `feat:` commits trigger a version bump, (2) `docs:` is skipped, (3) a GitHub Release is created with a categorised changelog. Check that the new version tag triggers the CI workflow and produces a semantically versioned image tag.

**Exercise 5 — GitOps with Argo CD:**
Install Argo CD in your kind cluster. Create an Application manifest pointing at your `hello-app` chart in a GitHub repository. Apply the manifest and observe Argo CD sync the app. Then make a manual change with `kubectl scale deployment hello-app --replicas=10 -n hello-app`. Watch Argo CD detect the drift and correct it back to the value in values.yaml (if `selfHeal: true`) or mark the app as `OutOfSync` (if false). Switch `selfHeal` between `true` and `false` and observe the difference.
