# Kubernetes Mastery Series — Publishing Plan

## Series Overview

**Title:** Kubernetes Mastery: From Hello World to Production  
**Audience:** Java/Spring Boot developers with no prior Kubernetes experience  
**Format:** 10 long-form articles, each self-contained but building on previous parts  
**Goal:** A reader who starts at Part 1 with zero Kubernetes knowledge should finish Part 10 capable of running production workloads confidently

**Writing principles applied throughout:**
- Every concept is introduced with **WHAT / WHY / HOW / WHO / WHEN** framing
- Analogies before technical detail — mental model first, then mechanics
- Code comments explain *why*, not just *what*
- No concept is assumed — if it's new, it's explained
- Each part ends with a troubleshooting table, command reference, and exercises

---

## Part-by-Part Plan

---

### [Part 1: Spring Boot + Docker + Helm — Your First Cloud-Ready App](articles/01-spring-boot-docker-helm-basics.md)
**Level:** Complete beginner  
**Goal:** Get a working Spring Boot app running in Kubernetes via Helm by the end

#### Chapter 1: The Big Picture
- What problem does this stack solve? (the "works on my machine" story)
- Architecture diagram: request → Ingress → Service → Pod → Container → JAR
- The restaurant chain analogy (Spring Boot = recipe, Docker = kitchen, K8s = chain manager, Helm = SOPs)
- How the parts connect — why you need all of them, not just some

#### Chapter 2: Spring Boot Application
- WHAT: A REST API with health check endpoints
- WHY Spring Boot Actuator: Kubernetes needs liveness + readiness endpoints
- Step-by-step: create project (Initializr + curl method)
- HelloController with `@RestController`, `@Value`, `@GetMapping`
- application.yml: graceful shutdown, actuator config, why `shutdown: graceful` matters
- Test locally first — always verify before Docker

#### Chapter 3: Docker — Containerization
- WHY Docker: the "works on my machine" problem solved
- WHAT is a multi-stage build and WHY it matters (build image ~1GB vs runtime ~200MB)
- WHY non-root user: container escape risk
- Docker layer caching deep dive:
  - WHY copy `pom.xml` first and run `mvn dependency:go-offline` before `COPY . .`
  - The invalidation rule: `COPY . .` invalidates every layer below it when *any* file changes — so dependencies get re-downloaded on every build if you don't cache them first
  - What `.dockerignore` prevents from busting the cache (`target/`, `.git/`, `*.md`)
- WHY `UseContainerSupport`: JVM doesn't know about container memory limits by default
  - What happens without it: JVM allocates heap based on host RAM → OOMKilled
  - What `MaxRAMPercentage=75.0` means: JVM uses 75% of container limit as max heap
- `SPRING_PROFILES_ACTIVE` as env var in ENTRYPOINT, not hardcoded — same image works across all environments; Kubernetes sets the variable via ConfigMap/env
- Build, run, test, verify health check
- Image size comparison table: single-stage vs multi-stage

#### Chapter 4: Kubernetes — Where Apps Live
- WHY Kubernetes: what you'd have to do manually without it
- The apartment building analogy (Pod/Deployment/Service/Namespace/Node/Cluster)
- WHAT a Pod is: smallest deployable unit, wraps containers
- WHAT a Deployment is: declares desired state, Kubernetes maintains it
- WHAT a Service is: WHY pods need a stable address (pod IPs change on restart)
- Service types: ClusterIP (default, internal) / NodePort (testing) / LoadBalancer (cloud, expensive)
- Resource requests vs limits: WHY both matter, what happens without them
- Health probes: liveness, readiness, startupProbe
  - WHY startupProbe exists: Spring Boot takes 20–60s to start; without it, livenessProbe kills the pod before it finishes booting → CrashLoopBackOff
- Graceful shutdown: preStop hook + terminationGracePeriodSeconds
- Start Minikube, deploy raw YAML manifests, test with port-forward

#### Chapter 5: Helm — The Package Manager
- WHY Helm: the 3-environment YAML duplication problem
- WHAT Helm is: template engine + package manager + release manager
- Helm chart structure explained file by file
- values.yaml: the one file you change per environment
- templates/: Go template syntax basics (`{{ .Values.x }}`, pipelines, `nindent`)
- `_helpers.tpl`: WHY named templates exist; `include` vs `template` (you can pipe `include`, you can't pipe `template`)
- Environment-specific values: values-dev.yaml, values-staging.yaml, values-prod.yaml
- Key Helm commands: lint, template, dry-run, install, upgrade, rollback, history, uninstall
- `--wait`: blocks until all pods are healthy (use in CI/CD so pipeline fails fast)
- `--atomic`: auto-rollback on failure (always use in production)
- Chart.yaml: version vs appVersion distinction

#### Chapter 6: Container Registry Integration
- WHY a registry: your cluster cannot see images on your laptop
- Push to Docker Hub (free, for learning)
- Image pull secrets: creating and referencing in Helm values
- Image tagging strategy: WHY never use `latest` in production

#### Chapter 7: Complete Working Example
- Full deploy script tying everything together
- Verification commands
- Part 1 checklist

#### Troubleshooting Table
- ImagePullBackOff, CrashLoopBackOff, Pending, Connection refused, Helm template errors, OOMKilled

#### Practice Exercises (5)

---

### [Part 2: Kubernetes Core Concepts — Beyond Hello World](articles/02-kubernetes-core-concepts.md)
**Level:** Beginner–Intermediate  
**Goal:** Understand the building blocks every real deployment needs

#### Chapter 1: Container Registries — The Missing Link
- Recap of WHY: cluster ≠ laptop
- Registry comparison table: Docker Hub / AWS ECR / GCR / Azure ACR / GitHub Container Registry
- Setup for each: Docker Hub, AWS ECR, Google GCR
- Image pull secrets: how to create and use
- Image tagging best practices: specific versions, SHA tags, never `latest` in prod

#### Chapter 2: Namespaces — Organizing Your Cluster
- WHY namespaces: the "50 confusing pods" problem
- WHAT they do: virtual isolation, resource boundaries, access control
- Common namespace patterns: dev/staging/prod/monitoring/ingress-nginx
- ResourceQuota: limits total CPU/memory/pods in a namespace — WHY this prevents one team starving another
- Working with namespaces: create, list, set default, delete
- Helm + namespaces pattern

#### Chapter 3: ConfigMaps — Configuration Without Rebuilding
- WHY ConfigMaps: the 5-step rebuild problem (change URL → rebuild → push → redeploy → wait)
- The whiteboard analogy
- Three creation methods: literal, from-file, declarative YAML
- Using ConfigMaps in pods: env vars vs mounted files
  - WHEN to use env vars: simple key-value, app reads from environment
  - WHEN to use mounted files: large configs, properties files, Spring Boot config
  - IMPORTANT: env vars need pod restart to update; mounted files are updated by kubelet but the app must re-read them
- ConfigMap hot-reload — three options with tradeoffs:
  1. Pod restart (`kubectl rollout restart`) — always works, brief disruption
  2. `spring.cloud.kubernetes.reload.enabled=true` — requires `spring-cloud-kubernetes` dependency, reloads `@ConfigurationProperties` beans without restart
  3. `@RefreshScope` + `/actuator/refresh` — reloads only annotated beans, manual trigger needed
- Spring Boot reading from environment variables and mounted files
- Helm + ConfigMaps: generating config from values.yaml
- `kubectl rollout restart` — HOW to apply ConfigMap changes and WHEN it's the right choice

#### Chapter 4: Secrets — Handling Sensitive Data
- WHY Secrets are different from ConfigMaps
- CRITICAL MISCONCEPTION: Secrets are NOT encrypted by default — they are base64-encoded
  - What base64 encoding is: it's encoding for safe transmission, not security
  - What this means in practice: anyone with `kubectl get secret` access can read them
  - Inspecting a secret: `kubectl get secret <name> -o yaml` shows base64 values; `echo "dmFsdWU=" | base64 -d` decodes them — readers should know how to do this
  - How to actually secure secrets: RBAC (restrict who can `get secrets`) + encryption at rest in etcd + external secrets manager
- Creating secrets: from-literal, from-file, `stringData` vs `data` (use `stringData` — no manual base64)
- Using secrets: env vars vs mounted files (mounted files are more secure — values don't appear in `kubectl describe pod` output)
- WARNING: never commit secret YAML to Git
- Production approach: External Secrets Operator (sync from AWS Secrets Manager / GCP Secret Manager / Vault)
- Sealed Secrets: encrypt for safe Git storage — how to use `kubeseal`

#### Chapter 5: Resource Management — Stop Noisy Neighbors
- WHY resource limits: one app stealing all CPU/RAM = everyone suffers
- Requests vs Limits explained clearly:
  - Request = guaranteed minimum (scheduler uses this for placement)
  - Limit = maximum allowed (CPU throttles, Memory kills)
  - CPU over limit → throttled (slowed, not killed)
  - Memory over limit → OOMKilled (hard kill, immediate)
- Finding right values: docker stats + load test
- JVM memory deep dive: why containers need `UseContainerSupport`
- Namespace LimitRange: default requests/limits if pod doesn't specify
- ResourceQuota: namespace-level caps
- Introduction to VPA for auto-recommendation (deep dive in Part 9)

#### Chapter 6: Health Probes — Self-Healing Applications
- WHY probes: Kubernetes needs to know app state to make decisions
- The three probes and their distinct purposes:
  - startupProbe: "app is still initializing — don't touch it yet"
  - readinessProbe: "app is ready for traffic / not ready (remove from Service)"
  - livenessProbe: "app is alive / stuck (restart it)"
- CRITICAL: Spring Boot startup sequence (20–60s) and what happens without startupProbe
  - Without it: livenessProbe fires during startup → pod killed → CrashLoopBackOff loop
- Complete probe configuration with all fields explained
- Spring Boot Actuator: liveness vs readiness endpoints and what they check
- Custom health indicators: making readiness depend on database connectivity
- Graceful shutdown deep dive: preStop + terminationGracePeriodSeconds + spring.shutdown=graceful
- Probe testing: how to simulate failure

#### Chapter 7: Complete Production-Ready Deployment
- values-prod.yaml combining all concepts
- Full deployment YAML with all best practices
- Deployment walkthrough

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 3: Helm Deep Dive — Beyond Basic Templating
**Level:** Intermediate  
**Goal:** Write production-grade Helm charts from scratch

#### Chapter 1: Go Template Engine
- WHAT Go templates are and WHY Helm uses them
- The four objects: `.Values`, `.Release`, `.Chart`, `.Files`
- Syntax: `{{ }}`, `{{- -}}` whitespace control (CRITICAL — many bugs come from wrong whitespace)
- Pipelines: chaining functions with `|`
- Common functions: `default`, `quote`, `upper`, `lower`, `trunc`, `replace`, `trim`
- `nindent` vs `indent`: WHY `nindent` adds a leading newline and when that matters
- `required`: fail fast if a value is missing — better than silent defaults
- Conditionals: `if`, `else if`, `else`, `end`
- Loops: `range` over lists and maps
- `toYaml`: rendering complex objects (lists, maps) as YAML

#### Chapter 2: Named Templates (`_helpers.tpl`)
- WHY named templates: DRY principle, avoid repeating labels in 10 places
- Files starting with `_` are not rendered as manifests
- `define` and `end`: creating a template
- `include` vs `template`:
  - `template` outputs directly, cannot be piped
  - `include` returns a string, CAN be piped to `nindent` — always use `include`
- The `{{- define "name" -}}` pattern: both `-` prevent blank line output
- The standard `_helpers.tpl` explained line by line:
  - `fullname`: release-chart name, truncated to 63 chars (Kubernetes DNS limit), WHY 63
  - `labels` vs `selectorLabels`: WHY selector labels must never change after first deploy
  - `serviceAccountName`: conditional based on whether SA is created
- Writing your own named templates: practical patterns

#### Chapter 3: Subcharts — Managing Dependencies
- WHY subcharts: your app needs postgres + redis + monitoring — deploy together
- Chart.yaml `dependencies` section: name, version, repository, condition
- `condition`: HOW to enable/disable a dependency with a values flag
- `helm dependency update`: downloads to `charts/` directory
- `helm dependency list`: see what's installed
- Overriding subchart values from parent: nesting under the chart name
- Global values: sharing across all subcharts with `.Values.global`
- Accessing subchart service names: `{{ .Release.Name }}-postgresql`
- `import-values`: pulling output values from subchart into parent
- Common subcharts: postgresql, redis, prometheus (bitnami charts)
- Disabling subcharts for external services (e.g., RDS instead of in-cluster postgres)

#### Chapter 4: Hooks — Lifecycle Management
- WHY hooks: database migrations must run BEFORE new pods start serving traffic
- WHAT hooks are: Jobs/Pods annotated to run at specific Helm lifecycle points
- All hook types: pre/post-install, pre/post-upgrade, pre/post-delete, pre/post-rollback, test
- Hook weights: controlling order when multiple hooks run at the same point
  - Lower numbers run first (weight "-5" runs before "0" runs before "5")
  - Default weight is 0 if not specified
  - Use negative weights for things that must run very early (e.g., environment validation)
- Hook delete policies: `hook-succeeded`, `before-hook-creation`, `hook-failed`
  - WHY `before-hook-creation`: prevents "job already exists" error on upgrades
  - WHY keep on `hook-failed`: you need the logs to debug a failed migration
- Database migration hook: complete example with Spring Boot Flyway/Liquibase
- Pre-install validation hook: check environment before deploying
- Post-deploy notification hook: Slack alert
- Helm test hooks: smoke tests with `helm test`
- `helm test` output and what it means

#### Chapter 5: Advanced Patterns
- Feature flags: conditionally including env vars and config
- Multi-environment pattern: base values.yaml + environment overlays
- `lookup` function: check if a resource already exists
  - WARNING: `lookup` doesn't work during `--dry-run`
- Generating random passwords: `randAlphaNum` + `helm.sh/resource-policy: keep`
- Conditional resource creation based on Kubernetes API availability (`Capabilities.APIVersions.Has`)
- NOTES.txt: post-install instructions rendered in terminal

#### Chapter 6: Complete Real-World Example
- E-commerce app: web + API + postgresql + redis + rabbitmq + monitoring
- Full Chart.yaml with all dependencies
- Complete values.yaml
- Migration hook + smoke test
- Multi-environment deploy commands
- `helm history` and `helm rollback`
- `helm diff` plugin: preview changes before applying (must-have tool)

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 4: Container Registry & CI/CD — Automating Your Deployments
**Level:** Intermediate  
**Goal:** Never manually deploy again — every git push triggers a pipeline

#### Chapter 1: Container Registry Deep Dive
- The current state (manual) vs the goal (fully automated)
- Registry comparison: Docker Hub, AWS ECR, GCR, Azure ACR, GitHub Container Registry, GitLab Registry
- Setup walkthrough for each registry
- Registry best practices:
  - Immutable tags (`v1.2.3`) vs mutable tags (`latest`) — WHY immutable matters for rollbacks
  - Short SHA tags for traceability: `git-abc123d`
  - `latest` tag: only useful for local dev, never production
  - Image scanning: `scanOnPush=true` on ECR, Dependabot on GitHub

#### Chapter 2: CI/CD Concepts
- WHAT CI/CD is: assembly line analogy
- CI vs CD vs Continuous Deployment — the difference
- Pipeline stages: commit → test → build → scan → push → deploy-dev → deploy-staging → approve → deploy-prod
- Tool comparison: GitHub Actions, GitLab CI, Jenkins, CircleCI, Argo CD, Tekton

#### Chapter 3: GitHub Actions
- Core concepts: workflow, event, job, step, action, runner
- Triggers: push, pull_request, schedule, workflow_dispatch
- Secrets: HOW to add them, environment-scoped secrets
- `needs`: job dependencies (don't deploy if tests fail)
- `environment`: GitHub Environments for manual approval gates
- Complete CI workflow: test → build → scan → push
- Complete deploy workflow: dev (auto) → staging (auto) → prod (manual approval)
- `docker/metadata-action`: auto-generating image tags from git refs
- `docker/build-push-action`: build with caching, push
- Trivy vulnerability scanner: fail build on CRITICAL CVEs
- `--atomic` flag: auto-rollback on failed helm upgrade
- `helm diff` plugin in CI: add a PR step that runs `helm diff upgrade` and posts the diff as a PR comment — reviewers see exactly what will change in the cluster before merging
- Kubeconfig encoding: `base64 -w 0` — the `-w 0` disables line wrapping; without it, base64 inserts newlines every 76 chars which breaks decoding in CI
- OIDC vs static credentials: WHY OIDC (no long-lived secrets stored in GitHub, tokens expire)
- Security best practices: environment secrets, OIDC, never print secrets in logs

#### Chapter 4: Semantic Versioning & Changelog
- WHAT semver is: `MAJOR.MINOR.PATCH` and when each bumps
- Conventional commits: `feat:`, `fix:`, `feat!:` and their version impact
- Automated version bumping script
- Automated changelog generation with GitHub releases

#### Chapter 5: GitLab CI
- Full `.gitlab-ci.yml` equivalent of the GitHub Actions pipeline
- GitLab-specific: `$CI_REGISTRY`, `$CI_COMMIT_SHORT_SHA`, `when: manual`
- GitLab environments and manual deployment gates

#### Chapter 6: GitOps with Argo CD
- WHAT GitOps is: Git as the single source of truth
- Argo CD: syncs cluster state to Git repo state
- Application manifest: `repoURL`, `targetRevision`, `path`, `destination`
- Sync policies: automated vs manual, prune, selfHeal
- Kustomize overlays for multi-environment GitOps
- Promotion workflow: merge to main → Argo CD syncs → approve for prod

#### Chapter 7: Complete Developer Workflow
- Day in the life with CI/CD
- Feature branch → PR → merge → auto-deploy to dev → manual to staging → manual to prod
- What happens automatically at each step

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 5: Networking & Ingress — Making Your App Accessible
**Level:** Intermediate  
**Goal:** Expose services securely with TLS, DNS, and proper routing

#### Chapter 1: Kubernetes Networking Layers
- The three layers: Pod network → Service → Ingress
- WHY this layered design: pods are ephemeral, IPs change
- ClusterIP: internal DNS name, how pod-to-pod communication works
- NodePort: WHY it exists, WHY it's not production-ready
- LoadBalancer: cloud LB per service — works but expensive
- Ingress: one LB for all services, routes by hostname/path — the production approach
- WHY you need an Ingress controller: Ingress resource is just a spec, controller implements it

#### Chapter 2: Service Types in Depth
- ClusterIP: internal DNS format `service.namespace.svc.cluster.local`
- NodePort: port range 30000–32767, WHY this range
- LoadBalancer: cloud provider annotations (AWS NLB, GCP, Azure)
- ExternalName: maps to an external DNS name (for migrating to in-cluster services)
- Headless services: `clusterIP: None`, WHY StatefulSets need them

#### Chapter 3: Ingress Controllers
- WHAT an Ingress controller is: the implementation of the Ingress spec
- Comparison and recommendation:
  - nginx-ingress: best default choice — works on every cluster, most features, huge community
  - AWS ALB Ingress Controller: EKS-only, native AWS integration (cheaper per LB), fewer features
  - Traefik: good if you want a built-in dashboard and dynamic config
  - Istio Gateway: only if you're already running Istio — don't install Istio just for ingress
  - Contour, GCE Ingress: cloud-specific alternatives
- Installing nginx-ingress with Helm (all cloud providers + Minikube)
- Installing AWS ALB Ingress Controller

#### Chapter 4: Ingress Rules
- Basic Ingress: host-based routing
- Path-based routing: `pathType` — Prefix vs Exact vs ImplementationSpecific
  - WHY pathType matters: wrong type = routes silently don't match (common gotcha)
  - Prefix: `/api` matches `/api`, `/api/users`, `/api/orders`
  - Exact: `/api` matches ONLY `/api` — not `/api/`
  - ImplementationSpecific: controller-defined, used for regex (nginx)
- Path rewriting with `rewrite-target`:
  - The problem: `/app/users` reaches the backend as `/app/users` but the backend expects `/users`
  - The solution: regex capture groups — `path: /app(/|$)(.*)` with `rewrite-target: /$2` strips the `/app` prefix
  - Complete working example with annotation
- Multiple backends: microservices routing from one Ingress
- Header-based routing: canary deployments with `canary-weight` annotation
- Sticky sessions: cookie affinity annotation
- Useful nginx annotations: rate limiting, CORS, proxy timeouts, request size, security headers, WebSocket

#### Chapter 5: TLS with cert-manager
- WHY TLS: encrypt traffic, browser trust, compliance
- Manual TLS: the 5-step nightmare (buy cert → encode → create secret → update ingress → repeat every 90 days)
- cert-manager: automates all of that
- Installing cert-manager
- ClusterIssuer vs Issuer: scope difference
- Staging issuer first: WHY — Let's Encrypt rate limits will block you if you test with prod issuer
- Automatic TLS via Ingress annotation: `cert-manager.io/cluster-issuer`
- Certificate resource: more control over duration/renewal
- Testing locally without a domain: nip.io (wildcard DNS based on IP)
- Verifying certificate status: `kubectl get certificate`

#### Chapter 6: ExternalDNS
- WHY ExternalDNS: manual DNS updates after every deploy are error-prone
- WHAT it does: watches Ingress/Service, creates DNS records automatically
- Installing for AWS Route53, CloudFlare, Google Cloud DNS
- Annotation-based control: `external-dns.alpha.kubernetes.io/hostname`
- Wildcard DNS for preview environments
- IAM policy required for Route53

#### Chapter 7: Network Policies — Zero Trust
- WHY network policies: by default all pods talk to all pods — no isolation
- Default behavior: open Wi-Fi analogy
- Default deny-all: the foundation of zero trust
- Allow DNS: CRITICAL — always add this with any deny-all, or pods can't resolve names
- Allow ingress controller to reach app pods
- Allow app to reach database
- Allow monitoring to scrape metrics
- Isolate environments: dev pods can't reach prod pods
- Testing: `nicolaka/netshoot` debug pod
- Network policy CNI requirement: policies only work if your CNI supports them (Calico, Cilium, etc.)

#### Chapter 8: Service Mesh with Istio (Advanced)
- WHAT a service mesh is and WHY you'd want one
- WHEN to use Istio: not for simple apps — operational overhead is significant
- Installing Istio: profiles (demo vs default vs minimal)
- Sidecar injection: how Istio proxies all traffic transparently
- Gateway + VirtualService: Istio's alternative to Ingress
- Traffic splitting: canary with percentage-based routing
- mTLS between services: automatic mutual TLS
- Fault injection: testing resilience by injecting delays/errors
- Retry policies and circuit breakers in VirtualService
- DestinationRule: load balancing, connection pooling, outlier detection

#### Chapter 9: Complete Production Example
- Full Ingress with TLS + ExternalDNS + security headers + rate limiting + CORS
- Network policies for a 3-tier app
- Helm values-prod.yaml for networking

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 6: Storage & Stateful Apps — Managing Data in Kubernetes
**Level:** Intermediate–Advanced  
**Goal:** Run production databases and stateful workloads in Kubernetes

#### Chapter 1: The State Problem
- Stateless vs stateful: catering staff vs regular customer analogy
- WHY Deployments don't work for databases:
  - Random pod names → can't address specific replicas
  - Any pod deleted during scale-down → lose primary
  - No stable network identity → replicas can't find each other
  - Shared or lost storage → data corruption
- Deployment vs StatefulSet comparison table

#### Chapter 2: Persistent Volumes
- Storage abstraction: developer (PVC) → Kubernetes (matching) → physical storage
- PV: the actual disk (EBS volume, GCP PD, etc.)
- PVC: the request ("I need 10GB SSD")
- StorageClass: the disk type + provisioning rules
- Access modes: ReadWriteOnce, ReadOnlyMany, ReadWriteMany — when each is appropriate
- Reclaim policies: Delete vs Retain vs Recycle
- Static provisioning: manually creating PVs (legacy, painful)
- Dynamic provisioning: StorageClass creates PV automatically (modern approach)
- `WaitForFirstConsumer`: WHY this matters for multi-AZ clusters (create disk in same AZ as pod)
- `allowVolumeExpansion`: growing a volume without downtime

#### Chapter 3: StorageClasses
- StorageClass examples for all cloud providers:
  - AWS EBS gp3: IOPS, throughput, encryption
  - GCP Persistent Disk (SSD)
  - Azure Premium Disk
  - Minikube local-path (for testing)
- Setting a default StorageClass
- What happens when you don't specify a StorageClass (uses cluster default)

#### Chapter 4: StatefulSets
- WHY StatefulSets: all the problems Deployments have with state — solved
- How StatefulSets differ:
  - Ordered pod naming: `postgres-0`, `postgres-1`, `postgres-2`
  - Ordered scaling: 0→1→2 up, 2→1→0 down
  - Stable network identity: `postgres-0.postgres-headless.namespace.svc.cluster.local`
  - `volumeClaimTemplates`: each pod gets its own PVC
- Headless Service: `clusterIP: None` — WHY StatefulSets need it
- Complete PostgreSQL StatefulSet example
- IMPORTANT: `PGDATA` subdirectory — WHY you must set `PGDATA=/var/lib/postgresql/data/pgdata`
  - PostgreSQL refuses to start if the mount point itself isn't empty (PVCs have a `lost+found` directory)
  - Subdirectory solves this: PVC mounted at `/data`, PostgreSQL writes to `/data/pgdata`
- IMPORTANT: MySQL equivalent — `--datadir` must point to a subdirectory of the mounted volume, not the mount root, for the same reason
- `podManagementPolicy: OrderedReady` vs `Parallel`:
  - `OrderedReady` (default): pods start/stop one at a time in order — required for databases with leader election
  - `Parallel`: all pods start/stop simultaneously — use for stateless workloads that happen to use StatefulSet for stable identity
- Scaling StatefulSets: scale-up adds pods, scale-down removes from highest ordinal
- Rolling updates: reverse order (2→1→0), each waits for previous to be ready
- `kubectl rollout status statefulset/postgres`

#### Chapter 5: Production Database Patterns
- Pattern 1: Primary-replica replication via StatefulSet
- Pattern 2: Read-write split with two Services (primary + replicas)
- Pattern 3: Backup CronJob to S3
- Pattern 4: Restore Job from S3
- Connecting Spring Boot: separate datasource URLs for read vs write

#### Chapter 6: Database Operators
- WHAT operators are: Kubernetes controllers for day-2 operations
- WHY operators: backup scheduling, failover, PITR, schema migrations — all automated
- CrunchyData PGO (PostgreSQL Operator): install, create cluster, access connection info
- Zalando Postgres Operator: Patroni-based HA
- Redis Operator: sentinel mode, cluster mode
- MySQL Operator: Oracle-official
- MongoDB Enterprise Operator
- WHEN to use an operator vs plain StatefulSet

#### Chapter 7: Backup Strategies
- Application-level backup: `pg_dump` CronJob
- Cluster-level backup: Velero
  - Installing Velero with AWS S3
  - Scheduled backups: daily full, hourly incremental
  - Manual backup before risky operations
  - Restore procedure
- Volume snapshots: faster than pg_dump for large databases

#### Chapter 8: Advanced Storage Patterns
- Volume snapshots: VolumeSnapshotClass, VolumeSnapshot, restore from snapshot
- ReadWriteMany with NFS: shared storage for multiple pods
- Local SSDs: NVMe for high-performance databases with node affinity

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 7: Security — Zero-Trust Kubernetes
**Level:** Advanced  
**Goal:** Harden a cluster against internal and external threats

#### Chapter 1: Security Layers (Defense in Depth)
- WHY defense in depth: one layer failing shouldn't compromise everything
- The 8-layer model: physical → cluster → node → container → network → auth → workload → policy
- Common mistakes and their impact: running as root, wildcard RBAC, no network policies, secrets in env vars

#### Chapter 2: RBAC — Who Can Do What
- WHAT RBAC is: Role-Based Access Control
- The 4 objects: Role, ClusterRole, RoleBinding, ClusterRoleBinding
- Scope: Role+RoleBinding = namespace-only; ClusterRole+ClusterRoleBinding = cluster-wide
- Verbs: get/list/watch (read), create/update/patch (write), delete (dangerous)
- Principle of least privilege: only grant what is actually needed
- Good examples: read-only monitoring, CI/CD deployer (no delete)
- Bad examples: wildcard permissions
- Testing RBAC: `kubectl auth can-i`
- Creating test users with certificates (for local clusters)

#### Chapter 3: Service Accounts — Pod Identity
- WHAT service accounts are: identity for pods to talk to the Kubernetes API
- WHY custom service accounts: the `default` SA often has too many permissions
- `automountServiceAccountToken: false`: disables the *default* token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`
  - WHY disable it: if your app doesn't call the Kubernetes API, it doesn't need this token — removing it reduces the blast radius if the pod is compromised
  - IMPORTANT: this does NOT break IRSA — IRSA uses a *projected* service account token mounted at a separate path (`/var/run/secrets/eks.amazonaws.com/serviceaccount/token`) via a separate volume injected by the EKS pod identity webhook — a completely different mechanism from the default token
- Creating and binding roles to service accounts
- AWS IRSA (IAM Roles for Service Accounts): pods get AWS credentials without access keys
  - WHY IRSA over access keys: no long-lived credentials, automatic rotation, scoped to a specific SA
  - HOW it works step by step: OIDC trust policy → pod authenticates with projected token → STS `AssumeRoleWithWebIdentity` → temporary credentials injected as env vars
  - Full setup: OIDC provider, IAM role trust policy, SA annotation
- GCP Workload Identity: equivalent for GKE
- Azure AD Pod Identity (now: Azure Workload Identity)

#### Chapter 4: Pod Security Standards
- Pod Security Levels: Privileged / Baseline / Restricted
- Pod Security Admission (PSA): enforce at namespace level via labels
- Pod Security Context: runAsNonRoot, runAsUser, fsGroup, seccompProfile
- Container Security Context: allowPrivilegeEscalation, readOnlyRootFilesystem, capabilities
  - `readOnlyRootFilesystem: true` — IMPORTANT: apps that write temp files need `emptyDir` volumes
  - `capabilities: drop: ALL` + add only what's needed
- Testing PSA: try to run privileged pod in restricted namespace

#### Chapter 5: Network Policies (Deep Dive)
- Revisit from Part 5 with more advanced patterns
- All selector types: podSelector, namespaceSelector, ipBlock
- `ipBlock` with `except`: allow internet but block internal ranges
- Multi-tier application policies: complete example (web → API → DB)
- Egress to internet (HTTPS only)
- Allow GitLab CI/CD runners
- Debugging with `nicolaka/netshoot`

#### Chapter 6: OPA Gatekeeper — Fine-Grained Policy
- WHY OPA: RBAC can't enforce "only trusted registry images" or "all pods must have team label"
- WHAT OPA Gatekeeper is: admission webhook that evaluates Rego policies
- Installing Gatekeeper
- ConstraintTemplate: defines a policy type with Rego logic
- Constraint: applies the policy to specific resources/namespaces
- Four real-world templates:
  1. Required labels: all pods must have `team` and `environment` labels
  2. No privileged containers
  3. Allowed image registries: block images from untrusted registries
  4. Deployment window: block deployments outside business hours
- Audit mode: check existing resources for violations
- Testing: what happens when a constraint is violated

#### Chapter 7: Secrets Management with HashiCorp Vault
- WHY Vault: Kubernetes Secrets are base64, not encrypted; Vault provides real encryption + audit log
- Installing Vault in dev mode
- Vault secrets engine: KV-v2
- Kubernetes auth method: pods authenticate with their service account token
- Vault policies: grant read access to specific paths
- Vault Agent Injector: sidecar that writes secrets to files before your app starts
- Annotations: `vault.hashicorp.com/agent-inject`, `vault.hashicorp.com/role`
- Reading secrets in Spring Boot from mounted files

#### Chapter 8: Image Security
- Image signing with Cosign: sign → verify → enforce in cluster
- Vulnerability scanning with Trivy: manual scan, automated CronJob
- kube-bench: CIS benchmark compliance checking
- Falco: runtime security (detecting shell in container, reading /etc/shadow, etc.)

#### Chapter 9: Security Hardening Checklist
- Complete security-hardened values.yaml
- Deployment script with all security checks
- Security audit commands

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 8: Observability — Monitoring, Logging, and Tracing
**Level:** Advanced  
**Goal:** Know what's happening in your cluster at all times

#### Chapter 1: The Three Pillars
- WHY observability: you can't fix what you can't see
- Metrics vs Logs vs Traces:
  - Metrics: "Is the system healthy? How fast is it?" (numbers over time)
  - Logs: "What happened? What was the error?" (events)
  - Traces: "Which service caused the slowdown?" (request path + timing)
- The car dashboard analogy (metrics = dashboard, logs = trip computer, traces = GPS route)
- WHY you need all three: metrics tell you *something is wrong*, logs tell you *what happened*, traces tell you *where*
- The observability stack: Prometheus + Grafana + Loki + Tempo/Jaeger + Alertmanager

#### Chapter 2: Prometheus — Metrics Collection
- Pull-based architecture: WHY Prometheus pulls rather than apps pushing
- TSDB: time-series database, what it stores
- PromQL: the query language
- Installing kube-prometheus-stack (includes Prometheus + Grafana + Alertmanager + pre-built dashboards)
- Spring Boot integration: `micrometer-registry-prometheus` dependency
- application.yml: expose `/actuator/prometheus`
- ServiceMonitor: the Kubernetes object that tells Prometheus what to scrape
- Annotating services for auto-discovery
- Custom metrics in Spring Boot: Counter, Timer, DistributionSummary
- Important PromQL queries:
  - Request rate: `rate(http_server_requests_seconds_count[5m])`
  - Error rate percentage
  - P95/P99 latency: `histogram_quantile()`
  - JVM heap usage
  - Pod CPU/Memory
- HIGH CARDINALITY WARNING: do not use user IDs, request IDs, or any unbounded values as metric labels
  - Concrete example: `user_id` label with 10,000 users = 10,000 new time series created per scrape cycle; at 30-second scrape intervals that's millions of series per day → Prometheus runs out of memory and crashes
  - Rule: labels should have low cardinality — environment, status code, HTTP method, endpoint are fine; user ID, session ID, trace ID are not
- `rate()` vs `irate()`:
  - `rate()`: average rate over the window (smoothed) — use for alerting, dashboards showing trends
  - `irate()`: rate based on last two data points (instantaneous) — use for dashboards showing current spikes
- Prometheus Adapter for custom HPA metrics:
  - Installing the adapter
  - Working `ConfigMap` example mapping `http_server_requests_seconds_count` to `http_requests_per_second` custom metric
  - Verifying the metric is available: `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1`

#### Chapter 3: Grafana — Visualization
- Accessing Grafana, getting admin password
- Importing pre-built dashboards: IDs for Spring Boot, Kubernetes Cluster, JVM
- Creating custom dashboards: panels, queries, thresholds
- Dashboard variables: namespace, pod, interval selectors
- Alert rules in Grafana: PrometheusRule CRD
- Alert examples: HighErrorRate, HighLatency, PodCrashLooping, CPUThrottling

#### Chapter 4: Alertmanager — Routing Alerts
- WHAT Alertmanager does: routes, groups, deduplicates, silences alerts
- Configuration: global, route, receivers
- Grouping: why grouping by `alertname` prevents alert storms
- `group_wait`, `group_interval`, `repeat_interval` — what each controls
- Receivers: Slack, PagerDuty, email
- Routing: critical → PagerDuty, warning → Slack
- Silences: temporary suppression during maintenance
- Complete Slack + PagerDuty setup

#### Chapter 5: Loki — Log Aggregation
- WHY centralized logging: logs scattered across pods, pods come and go
- WHAT Loki is: like Prometheus but for logs (labels, not full-text index)
- Promtail: the log collector that runs as DaemonSet
- Installing loki-stack
- LogQL: query language
  - Basic stream selectors: `{namespace="production"}`
  - Filter expressions: `|= "ERROR"`, `|~ "regex"`
  - JSON parsing: `| json | level="ERROR"`
  - Rate queries: `rate({...}[5m])`
- Adding Loki as Grafana data source
- JSON structured logging in Spring Boot: WHY + how to configure Logback
- Log-to-trace correlation: click a log line → open the trace

#### Chapter 6: Distributed Tracing — Tempo and Jaeger
- WHY tracing: when a request crosses 5 services, logs don't tell you which one was slow
- WHAT a trace is: spans forming a tree, each span has timing + attributes
- Tempo (Grafana) vs Jaeger: when to use each
- Installing Tempo
- Spring Boot + OpenTelemetry: dependency, auto-instrumentation
- `@WithSpan`: manual span creation
- Custom span attributes: `Span.current().setAttribute()`
- Trace propagation: `traceparent` header between services
- Viewing traces in Grafana/Jaeger UI
- Trace → log correlation: finding logs for a specific trace ID
- Sampling: WHY 100% sampling is too expensive; trace sampling strategies

#### Chapter 7: SLI/SLO Implementation
- WHAT SLIs are: Service Level Indicators (measurable metrics)
- WHAT SLOs are: Service Level Objectives (targets for those metrics)
- Common SLIs: availability, latency, error rate
- Defining SLOs: availability 99.9%, P99 latency < 500ms
- Error budget: what you're allowed to fail
- Error budget burn rate alerting: burn rate > 14.4 = exhausted in 2 hours
- SLO dashboard in Grafana

#### Chapter 8: Complete Observability Stack
- Production values-observability.yaml
- Full deploy sequence
- Linking the pillars: metrics alert → logs investigation → trace root cause

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 9: Scaling & Resilience — Building Self-Healing Systems
**Level:** Advanced  
**Goal:** Systems that handle traffic spikes, failures, and maintenance without manual intervention

#### Chapter 1: Scaling Overview
- Types of scaling: Horizontal (more pods), Vertical (bigger pods), Cluster (more nodes)
- WHY all three levels: HPA fills nodes, Cluster Autoscaler adds nodes when HPA can't schedule
- Speed comparison: HPA (seconds–minutes), VPA (minutes, requires restart), CA (3–5 minutes)

#### Chapter 2: Horizontal Pod Autoscaler (HPA)
- HOW HPA works: Metrics Server → HPA controller → adjust replicas
- WHY pods need `resources.requests.cpu` set: HPA calculates utilization as `actual/requested`
- Basic CPU-based HPA: complete example
- `behavior`: controlling scale-up and scale-down speed
  - `stabilizationWindowSeconds`: WHY 300s for scale-down (prevents flapping)
  - `selectPolicy: Max`: use whichever policy scales up fastest
- Advanced HPA: multiple metrics (CPU + memory + custom)
- Custom metrics with Prometheus Adapter:
  - Installing Prometheus Adapter
  - HPA on requests-per-second
  - HPA on queue depth (RabbitMQ)
- Testing HPA: load generator + `kubectl get hpa -w`
- Spring Boot JVM tuning for HPA: `UseContainerSupport`, graceful shutdown for scale-down

#### Chapter 3: Vertical Pod Autoscaler (VPA)
- WHY VPA: finding the right resource requests is hard
- WHAT VPA does: observes actual usage, recommends (or applies) new requests
- Update modes: Off / Initial / Recreate / Auto — start with Off
- VPA recommendations output explained
- `minAllowed` / `maxAllowed`: guard rails
- VPA + HPA conflict: WHY they fight on the same metric
  - SOLUTION: HPA on custom metric (RPS), VPA on memory only — or use one, not both

#### Chapter 4: Cluster Autoscaler
- WHAT CA does: adds/removes nodes based on pending pods
- WHY pending pods happen: HPA wants more pods but no node has free capacity
- AWS EKS: IAM policy, ASG tags, Helm install
- GKE: built-in autoscaling, `gcloud` command
- AKS: `az aks update`
- CA configuration: scale-down delays, unneeded time, provision time
- Testing: generate load → HPA scales pods → pods go Pending → CA adds node

#### Chapter 5: Pod Disruption Budgets (PDB)
- WHY PDBs: node drains during upgrades/maintenance can remove too many pods → outage
- `minAvailable` vs `maxUnavailable`: when to use each
- PDB for databases: `minAvailable: 2` maintains quorum
- PDB for critical services
- Testing PDB: `kubectl drain` → observe it being blocked
- `ALLOWED DISRUPTIONS` in `kubectl get pdb` output — what it means

#### Chapter 6: Chaos Engineering
- WHY chaos engineering: fire drills for infrastructure
- Principles: steady state → hypothesis → experiment → verify → improve
- Installing Chaos Mesh + dashboard
- Pod chaos: pod-kill, pod-failure
- Network chaos: delay, loss, partition
- Stress chaos: CPU stress, memory stress
- IO chaos: disk latency
- Running experiments: apply, watch, verify app recovers
- Safe chaos practices: PDB first, monitoring during, start in staging
- Chaos scheduling: only during business hours, exclude holidays

#### Chapter 7: Advanced Deployment Strategies
- Rolling update: `maxSurge` + `maxUnavailable` — default Kubernetes strategy, no extra tooling needed
- Blue-green deployment with Argo Rollouts:
  - WHAT: run old (blue) and new (green) versions simultaneously, switch all traffic at once
  - `autoPromotionEnabled: false`: manual gate before switching traffic
  - Instant rollback: switch back to blue — new pods are still running, no redeploy needed
- Canary deployment with Argo Rollouts:
  - WHAT: gradually shift traffic from old to new (10% → 25% → 50% → 100%)
  - Requires two Services: a `stable` Service (selects old pods) and a `canary` Service (selects new pods) — Argo Rollouts controls the selector labels and traffic weight between them; this Service setup is non-obvious and not required for blue-green
  - Automated rollback via Analysis: if error rate > 5%, roll back automatically
  - AnalysisTemplate: define success criteria as Prometheus query — complete example
- When to use each:
  - Rolling update: standard day-to-day deployments
  - Blue-green: zero-risk cutover when you need instant full rollback
  - Canary: gradual rollout where you want real traffic data before committing

#### Chapter 8: Disaster Recovery
- RTO vs RPO: what they mean, how to set targets
- Velero for cluster backup: install, schedule daily + hourly, manual pre-upgrade backup
- Velero restore procedure
- DR planning: single pod failure (self-heal), node failure (reschedule), namespace deletion (restore), regional outage (activate DR region)
- DNS failover with Route53

#### Troubleshooting Table
#### Practice Exercises (5)

---

### Part 10: Production Playbook — From Development to Production
**Level:** Expert  
**Goal:** A complete reference for running production Kubernetes workloads

#### Chapter 1: Production Readiness Framework
- The minimum viable production checklist (20 items) — the things that *must* be in place before going live
- The full production checklist organized by category: application, container, Kubernetes, security, observability, scalability, CI/CD, DR, cost, documentation
- Pre-production validation script
- `kubectl diff`: compare what's in cluster with what you're about to apply — run before every `kubectl apply`
- `helm diff` plugin: same for Helm upgrades — shows field-level changes, not just "something changed"

#### Chapter 2: Complete Production Helm Values
- The definitive values-prod.yaml: every field explained
- replicaCount, image, resources, autoscaling, PDB, update strategy
- Probes (all three), lifecycle, terminationGracePeriodSeconds
- Security contexts (pod + container)
- Service, Ingress with all annotations
- Network policies
- Service account + IRSA annotation
- ConfigMap, External Secrets
- Monitoring (ServiceMonitor, PrometheusRules, Grafana dashboard)
- Tracing, logging labels
- Backup configuration
- Pod anti-affinity + topology spread constraints
- Priority classes
- Node selector + tolerations

#### Chapter 3: Complete CI/CD Pipeline
- Full GitHub Actions production pipeline: validate → build-test → security-scan → build-push → deploy-staging → deploy-production (manual)
- Version validation: semantic versioning regex check
- Pre-deploy: Velero backup
- Post-deploy: health check loop with auto-rollback
- Post-deployment metrics validation via Prometheus API
- Slack notifications with rich formatting

#### Chapter 4: Security Hardening
- Namespace security labels (PSA enforce: restricted)
- Default deny-all network policy
- OPA required labels constraint
- ResourceQuota + LimitRange
- Automated security scanning CronJob

#### Chapter 5: Monitoring & Alerting Runbooks
- Production alert rules: service down, high error rate, high latency, crash loop, memory pressure, certificate expiry
- Alertmanager routing: critical → PagerDuty, warning → Slack
- Incident severity levels: SEV0–SEV3
- Incident response steps: detection → triage → mitigation → resolution → post-mortem
- Runbooks for common incidents:
  - High error rate: check changes, rollback, scale up
  - Database connection issues: check pods, logs, connectivity
  - Performance degradation: HPA, resource throttling, node pressure
- Communication templates: initial alert, update, resolution

#### Chapter 6: Cost Optimization
- Right-sizing: VPA recommendations
- Cluster autoscaler: scale to zero overnight
- Spot/preemptible instances for non-critical workloads
- Storage tiering: SSD for hot data, HDD for cold
- Deleting unused resources: Released PVCs, idle load balancers, old images
- Kubecost: installation and usage
- Cost allocation: labels for team/environment chargeback

#### Chapter 7: Disaster Recovery Runbook
- RTO/RPO targets
- Velero backup strategy: daily full + hourly incremental
- Scenario playbooks: pod failure, node failure, namespace deletion, regional outage
- DR testing schedule: monthly restore test, quarterly full simulation, annual cross-region
- Emergency contacts template

#### Chapter 8: Service Documentation Template
- Service README: overview, architecture, URLs, runbooks, dashboards, alerts, dependencies, SLOs
- Post-mortem template: incident timeline, root cause, contributing factors, action items

#### Chapter 9: One-Command Production Deployment Script
- Full deploy-to-production.sh: test → build → scan → staging → approval gate → prod → verify → notify

#### Production Checklists (summary tables)

---

## File Naming & Structure

```
kubernetes-mastery-series/
├── README.md                          ← This file (the plan)
├── 01-spring-boot-docker-helm-basics.md
├── 02-kubernetes-core-concepts.md
├── 03-helm-deep-dive.md
├── 04-container-registry-and-cicd.md
├── 05-networking-ingress.md
├── 06-storage-stateful-apps.md
├── 07-security.md
├── 08-observability.md
├── 09-scaling-resilience.md
└── 10-production-playbook.md
```

---

## Writing Standards Applied to Every Article

### Structure of every chapter
1. **Concept introduction** — WHAT is this, WHY does it exist, WHEN do you use it
2. **Analogy** — relatable real-world comparison before technical detail
3. **Technical explanation** — the mechanics
4. **Code/YAML examples** — with inline comments explaining WHY each line exists
5. **Verification** — how to confirm it worked

### Misconceptions — handled in-prose, not callout boxes
Common misconceptions (Secrets are encrypted, `latest` tag is fine in prod, etc.) are corrected *within the relevant section* with context — not isolated into a separate box. A correction is only useful alongside the explanation of why the misconception is wrong and what the right mental model is. Pulling it into a box loses that context.

### Code comment style
```yaml
# WHY: Without this, Kubernetes sends traffic before the app is ready → 500 errors
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10   # Wait 10s before first check (app needs time to start)
  periodSeconds: 5           # Check every 5 seconds
  failureThreshold: 3        # Remove from Service after 3 consecutive failures (15s)
```

### Long YAML blocks
Any YAML block over ~30 lines is split into smaller chunks with explanatory prose between sections — not all comments inline. This keeps explanation visible and scannable rather than buried in a wall of commented code.

### Troubleshooting table format (end of every article)
| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|

### Practice exercises (end of every article)
5 exercises progressing from "follow the steps" to "figure it out yourself"

---
