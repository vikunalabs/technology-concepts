# Part 1: Spring Boot + Docker + Helm — Your First Cloud-Ready App

> **Series:** Kubernetes Mastery — From Hello World to Production

> **Level:** Complete Beginner — no prior Kubernetes or Docker experience needed

> **Prerequisites:** Java 17+, Maven, a terminal

> **Time to complete:** 3–4 hours

> **What you'll build:** A Spring Boot REST API, containerized with Docker, deployed to Kubernetes using Helm

---

## Chapter 1: The Big Picture — Why Does This Stack Exist?

### The Problem

You've written a Spring Boot application. It runs perfectly on your laptop. Now your manager asks you to deploy it.

Immediately, questions pile up:

- How do you make sure the same Java version is on the server?
- What happens when the app crashes at 3 AM — does someone have to restart it manually?
- How do you deploy a new version without users seeing downtime?
- How do you run 5 copies of it for high availability?
- How do you deploy to dev with 1 copy and prod with 5 copies, using different database URLs, without maintaining 5 nearly-identical config files?

These are not exotic problems. They hit every team that moves from "it works on my laptop" to "it runs in production." This series exists to solve them — systematically, with the tools the industry settled on.

### The Stack and What Each Piece Does

| Tool | What it solves | Analogy |
|------|---------------|---------|
| **Spring Boot** | Your business logic | The recipe |
| **Docker** | "Works on my machine" — packages the app with its entire environment | The standardized kitchen |
| **Kubernetes** | Keeps your app running, scales it, restarts it on failure | The restaurant chain manager |
| **Helm** | Repeatable, configurable deployments without copy-pasting YAML | The Standard Operating Procedure manual |

Think of it like opening a restaurant chain. The recipe (Spring Boot) is what makes the food good. But a single great recipe isn't enough to open 50 restaurants. You need standardized kitchens so the recipe produces the same result everywhere (Docker). You need a management system to open new locations, close underperforming ones, and handle emergencies (Kubernetes). And you need an SOP so every new location is set up consistently, not improvised from memory (Helm).

Without any one of these pieces, you have a gap:

- **No Docker:** The app works in your kitchen but breaks in the server's kitchen because of a different Java version, different OS library, or a missing environment variable.
- **No Kubernetes:** You manually SSH into servers to restart crashed apps, manually spin up new instances during traffic spikes, and manually roll back bad deployments.
- **No Helm:** You maintain separate YAML files for dev, staging, and production — 95% identical — and every shared change has to be applied in three places.

### How a Request Reaches Your App

Before writing any code, it helps to see the complete picture of what you're building toward:

```
User's browser
      │
      │  GET https://api.myapp.com/hello
      ▼
┌─────────────────────┐
│   Ingress Controller │  ← Receives the request, handles TLS, routes by hostname/path
│  (nginx, one per     │    (Covered in Part 5)
│   cluster)           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   Kubernetes Service │  ← Stable DNS name + load balances across pods
│   (ClusterIP)        │    "hello-app-service:8080"
└──────────┬──────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
┌─────────┐  ┌─────────┐     ← Pods: one or more copies of your app
│  Pod 1  │  │  Pod 2  │       Kubernetes keeps these running,
│ ┌─────┐ │  │ ┌─────┐ │       restarts them on failure,
│ │ JAR │ │  │ │ JAR │ │       and manages their lifecycle
│ └─────┘ │  │ └─────┘ │
└─────────┘  └─────────┘
```

By the end of Part 1, you'll have the pods running via Helm. Parts 2 through 10 add everything else — the Ingress, TLS, observability, autoscaling, and production hardening.

---

## Chapter 2: Spring Boot — Building the Application

### What We're Building

A REST API with three endpoints plus Spring Boot Actuator health endpoints. The health endpoints are not just nice to have — Kubernetes uses them to decide whether to send traffic to your app and whether to restart it. We'll come back to why that matters in Chapter 4.

**Endpoints:**

| Path | What it returns |
|------|----------------|
| `GET /api/hello` | `Hello, World from Spring Boot!` |
| `GET /api/info` | JSON with app name, version, timestamp |
| `GET /api/greet?name=Alice` | `Hello, Alice!` |
| `GET /actuator/health/liveness` | `{"status":"UP"}` — Kubernetes liveness probe |
| `GET /actuator/health/readiness` | `{"status":"UP"}` — Kubernetes readiness probe |

### Step 1: Create the Project

**Option A — Spring Initializr website (visual):**
1. Go to [https://start.spring.io](https://start.spring.io)
2. Fill in: Project = Maven, Language = Java, Spring Boot = 3.2.x
3. Group = `com.example`, Artifact = `hello-app`, Java = 17
4. Click **Add Dependencies** → add **Spring Web** and **Spring Boot Actuator**
5. Click **Generate** → download the ZIP → extract it

**Option B — curl (faster, no browser needed):**
```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,actuator \
  -d name=hello-app \
  -d groupId=com.example \
  -d artifactId=hello-app \
  -d javaVersion=17 \
  -o hello-app.zip

unzip hello-app.zip -d hello-app
cd hello-app
```

Verify it runs before touching anything else:
```bash
./mvnw spring-boot:run
# Should see: Started HelloAppApplication in X.XXX seconds
```

Open a second terminal and confirm:
```bash
curl http://localhost:8080/actuator/health
# {"status":"UP"}
```

Stop the app with `Ctrl+C`. We'll now add our own code.

### Step 2: Create the REST Controller

Create the file `src/main/java/com/example/helloapp/HelloController.java`:

```java
package com.example.helloapp;

import org.springframework.web.bind.annotation.*;
import org.springframework.beans.factory.annotation.Value;
import java.time.LocalDateTime;
import java.util.Map;

@RestController
@RequestMapping("/api")
public class HelloController {

    // @Value reads from application.yml, environment variables, or Kubernetes ConfigMaps.
    // The ":Hello App" after the colon is a default — used if the property is not set.
    // In Kubernetes we'll override these with ConfigMaps without touching this code.
    // NOTE: We'll define app.name and app.version in application.yml in Step 3.
    @Value("${app.name:Hello App}")
    private String appName;

    @Value("${app.version:1.0.0}")
    private String appVersion;

    // GET /api/hello
    @GetMapping("/hello")
    public String sayHello() {
        return "Hello, World from Spring Boot!";
    }

    // GET /api/info
    // Returns a JSON object. Map.of() creates an unmodifiable map.
    // Jackson (included with Spring Web) automatically serializes Map to JSON.
    @GetMapping("/info")
    public Map<String, String> getInfo() {
        return Map.of(
            "appName",   appName,
            "version",   appVersion,
            "timestamp", LocalDateTime.now().toString(),
            "status",    "healthy"
        );
    }

    // GET /api/greet?name=Alice  → "Hello, Alice!"
    // GET /api/greet             → "Hello, World!"  (defaultValue kicks in)
    @GetMapping("/greet")
    public String greet(@RequestParam(defaultValue = "World") String name) {
        return String.format("Hello, %s!", name);
    }
}
```

### Step 3: Configure application.yml

Replace the contents of `src/main/resources/application.yml` (rename `application.properties` to `application.yml` if needed):

```yaml
spring:
  application:
    name: hello-app

  # WHY graceful shutdown: when Kubernetes needs to stop a pod (scaling down, rolling
  # update, node drain), it sends a SIGTERM signal. Without graceful shutdown, Spring Boot
  # exits immediately — dropping any in-flight requests and returning errors to users.
  # With graceful shutdown, Spring Boot stops accepting new requests but finishes
  # processing active ones before exiting. We configure the timeout below under
  # spring.lifecycle.timeout-per-shutdown-phase.
  shutdown: graceful

  lifecycle:
    # Allow up to 30 seconds for in-flight requests to complete after SIGTERM.
    # If they finish sooner, the app exits early. If they don't finish in 30s, it exits anyway.
    timeout-per-shutdown-phase: 30s

server:
  port: 8080

# Custom properties — read by @Value annotations in the controller.
# These can be overridden via environment variables or Kubernetes ConfigMaps
# without changing this file or rebuilding the image.
app:
  name: Hello Kubernetes App
  version: 1.0.0

management:
  endpoints:
    web:
      exposure:
        # WHY not expose everything (*): actuator endpoints can leak sensitive data
        # (heap dumps, env vars, thread state). Only expose what you actually need.
        include: health,info,metrics
  endpoint:
    health:
      show-details: always
      probes:
        # WHY enable probes: this creates two separate health endpoints:
        #   /actuator/health/liveness  — Is the JVM alive? (Kubernetes restarts on failure)
        #   /actuator/health/readiness — Is the app ready for traffic? (Kubernetes stops
        #                                sending requests on failure)
        # Without these separate endpoints, you'd have to point both probes at the same
        # /actuator/health endpoint, losing the ability to control them independently.
        # For example: if your database goes down, you want readiness to fail (stop traffic)
        # but liveness to stay UP (don't restart — the app itself is fine).
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true

logging:
  level:
    com.example.helloapp: INFO
```

### Step 4: Test Locally Before Anything Else

This is a habit worth building: always verify the app works as plain Java before introducing Docker or Kubernetes. Problems caught here take 10 seconds to fix. The same problems caught inside a container take 10 minutes.

```bash
./mvnw spring-boot:run
```

In a second terminal:
```bash
# Basic endpoint
curl http://localhost:8080/api/hello
# Expected: Hello, World from Spring Boot!

# JSON response
curl http://localhost:8080/api/info
# Expected: {"appName":"Hello Kubernetes App","version":"1.0.0","timestamp":"...","status":"healthy"}

# Query parameter
curl "http://localhost:8080/api/greet?name=Developer"
# Expected: Hello, Developer!

# Kubernetes liveness probe endpoint
curl http://localhost:8080/actuator/health/liveness
# Expected: {"status":"UP"}

# Kubernetes readiness probe endpoint
curl http://localhost:8080/actuator/health/readiness
# Expected: {"status":"UP"}
```

All five working? Good. Stop the app and move to Docker.

---

## Chapter 3: Docker — Packaging for Anywhere

### Why Docker? The Real Story

Consider what it takes to run a Java app on a server today:

1. Install the right JDK version (and hope it doesn't conflict with another app's version)
2. Set the right environment variables (`JAVA_HOME`, etc.)
3. Copy the JAR to the right path
4. Write a startup script
5. Configure the init system (systemd) to restart it on crash
6. Repeat this for every server, every environment, every team member's laptop

Docker flips this model. Instead of configuring the server to run your app, you package your app with everything it needs — the JDK, the filesystem layout, the startup command — into a single artifact called an **image**. Servers just run images. The image runs identically on your laptop, on CI, on staging, on production.

| Concept | Analogy | What it actually is |
|---------|---------|---------------------|
| **Dockerfile** | A recipe | Instructions to build an image |
| **Image** | A frozen meal, ready to heat | A built, immutable artifact |
| **Container** | The heated meal on your plate | A running instance of an image |
| **Registry** | A grocery store's freezer section | A server that stores and serves images |

### Multi-Stage Builds — Why Size Matters

A naive Dockerfile copies your source code in and runs `mvn package`. The resulting image contains Maven, the entire JDK, all downloaded dependencies in `~/.m2`, your source files, and your compiled JAR. That's roughly 1GB for a simple Spring Boot app.

None of that is needed at runtime. Your app only needs the JAR and a JRE to run it.

**Multi-stage builds** solve this by using two separate images in one Dockerfile:
- **Stage 1 (build):** Full JDK + Maven — compiles the JAR. This image is large but thrown away.
- **Stage 2 (runtime):** Lean JRE — only contains the JAR and what's needed to run it. This is the image that ships.

The result: a runtime image around 200MB instead of 1GB. Smaller images pull faster, start faster, and have a smaller attack surface.

### Docker Layer Caching — How to Make Builds Fast

Docker builds images in layers. Each instruction in your Dockerfile (`FROM`, `RUN`, `COPY`, etc.) creates a layer. Docker caches these layers and reuses them on subsequent builds — **unless a layer changes, in which case that layer and everything below it is rebuilt from scratch.**

This has a critical implication for Maven projects:

```dockerfile
# SLOW — rebuilds dependencies on every code change
COPY . .                    # Changes whenever ANY file changes
RUN mvn dependency:go-offline  # Re-downloads all dependencies every time
RUN mvn clean package          # Recompiles every time
```

```dockerfile
# FAST — dependencies cached as long as pom.xml doesn't change
COPY pom.xml .              # Only changes when dependencies change (rarely)
RUN mvn dependency:go-offline  # Cached — runs once, then skipped
COPY src ./src              # Changes when code changes
RUN mvn clean package       # Recompiles code, but NOT re-downloading dependencies
```

The rule: **copy files that change rarely before files that change often.** Your `pom.xml` changes when you add a dependency (uncommon). Your `src/` changes every time you write code (frequent). Separating them into two `COPY` instructions means Maven only re-downloads dependencies when `pom.xml` actually changes.

### The Dockerfile

Create a file named `Dockerfile` in the root of your project (same level as `pom.xml`):

```dockerfile
# =============================================================================
# Stage 1: Build
# Use the full JDK + Maven image to compile the application.
# This image is large (~700MB) but is NEVER shipped — it's only used to build.
# =============================================================================
FROM maven:3.9-eclipse-temurin-17 AS build

WORKDIR /app

# Copy pom.xml FIRST and download dependencies.
# WHY: Docker caches this layer. As long as pom.xml doesn't change,
# subsequent builds skip the dependency download entirely — saving minutes.
COPY pom.xml .
RUN mvn dependency:go-offline -q

# Now copy source code and build.
# WHY we copy source AFTER dependencies: if we did COPY . . first,
# every code change would invalidate the dependency cache layer above it,
# forcing a full re-download every single build.
COPY src ./src
RUN mvn clean package -DskipTests -q

# =============================================================================
# Stage 2: Runtime
# Use a minimal JRE-only image — no build tools, no source code, no Maven.
# This is the image that actually gets deployed.
# =============================================================================
FROM eclipse-temurin:17-jre-alpine

# WHY create a dedicated user: running as root inside a container is a security
# risk. If an attacker exploits a vulnerability in your app and escapes the
# container, they'd have root access to the host. A non-root user limits
# the blast radius significantly.
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only the compiled JAR from the build stage.
# The build stage with Maven, JDK, and source code is left behind entirely.
COPY --from=build /app/target/*.jar app.jar

# Fix file ownership before switching to non-root user
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 8080

# Health check for Docker itself (separate from Kubernetes probes).
# This lets docker ps show a health status for local testing.
HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# =============================================================================
# ENTRYPOINT — how the JVM starts inside the container
#
# -XX:+UseContainerSupport
#   WHY: By default, the JVM reads total RAM from the host machine, not the
#   container's memory limit. If your container has a 512Mi limit but the host
#   has 32GB, the JVM sees 32GB and tries to allocate heap proportionally.
#   Result: the JVM allocates ~8GB of heap → instantly OOMKilled by Kubernetes.
#   UseContainerSupport tells the JVM to respect the container's cgroup limits.
#   This flag is on by default in Java 11+ but being explicit avoids surprises.
#
# -XX:MaxRAMPercentage=75.0
#   WHY: With a 512Mi container limit, the JVM uses 75% of 512Mi = ~384Mi as max
#   heap. The remaining 25% (~128Mi) is left for JVM overhead: metaspace, thread
#   stacks, code cache, GC bookkeeping, and JVM internals.
#   Without this, the JVM uses its own heuristic which is often too conservative
#   (25% of available RAM) or doesn't account for the 25% overhead.
#
# SPRING_PROFILES_ACTIVE
#   WHY: We don't hardcode a profile here. The same Docker image runs in dev,
#   staging, and production — only the profile differs. Kubernetes sets this
#   environment variable via a ConfigMap, so the image works everywhere without
#   being rebuilt. If no profile is set, Spring Boot uses its defaults.
# =============================================================================
ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-jar", \
  "/app/app.jar"]
```

### .dockerignore — Keeping the Build Context Clean

Docker sends your entire project directory to the Docker daemon when building. Without a `.dockerignore`, it sends `target/` (compiled classes, test reports, the old JAR), `.git/` (full git history), IDE files, and everything else. This slows down the build and — more importantly — can bust your layer cache when unrelated files change.

Create `.dockerignore` in the project root:

```
# Build output — we compile fresh inside Docker, don't need this
target/

# Git history — not needed in the image, and changes on every commit
.git/
.gitignore

# IDE files — irrelevant to the build
.idea/
*.iml
.vscode/

# Local environment files — never copy secrets into images
.env
*.env

# Documentation — not needed in the image
*.md
docs/

# Test reports — generated during build, not needed in runtime image
```

### Build and Test the Container

First, compile the JAR:
```bash
./mvnw clean package -DskipTests
```

Build the Docker image:
```bash
docker build -t hello-app:1.0.0 .
```

On your first build you'll see every layer being executed. On the second build (try changing a Java file and rebuilding), notice that the dependency download layer is skipped — only the source copy and compile layers re-run.

Check the image size to appreciate the multi-stage difference:
```bash
docker images hello-app
# REPOSITORY   TAG     IMAGE ID       CREATED        SIZE
# hello-app    1.0.0   abc123def456   1 minute ago   ~220MB
```

If you built without multi-stage (a JDK image with Maven included), it would be around 900MB–1.1GB. The runtime image is ~200MB.

Run the container:
```bash
docker run -d \
  --name hello-test \
  -p 8080:8080 \
  hello-app:1.0.0
```

Test it:
```bash
curl http://localhost:8080/api/hello
# Hello, World from Spring Boot!

curl http://localhost:8080/actuator/health/liveness
# {"status":"UP"}

# Check health status
docker ps
# The STATUS column should show "(healthy)" after ~30 seconds
```

View logs:
```bash
docker logs hello-test
docker logs -f hello-test  # Follow (live stream)
```

Clean up:
```bash
docker stop hello-test && docker rm hello-test
```

---

## Chapter 4: Kubernetes — Where Apps Live

### Why Not Just Use Docker?

You just ran your app in a Docker container. It worked. So why add Kubernetes?

Because Docker alone only solves the packaging problem. It doesn't solve:

- **Automatic restarts:** If the container crashes, Docker doesn't restart it unless you added `--restart=always` — and even then, it's limited.
- **Multiple copies:** Running 5 containers manually across 3 servers requires scripts and coordination.
- **Zero-downtime updates:** Replacing a running container with a new version has a window of downtime.
- **Health-based traffic:** Docker doesn't know if your app is ready to serve requests or still starting up.
- **Resource management:** Without limits, one container can starve every other process on the host.
- **Self-healing:** If a server goes down, Docker doesn't move your containers to a healthy server.

Kubernetes does all of this — automatically, continuously, without human intervention.

### The Apartment Building Analogy

Kubernetes concepts map cleanly to an apartment building:

| Kubernetes | Building |
|-----------|---------|
| **Cluster** | The entire apartment complex |
| **Node** | One building in the complex |
| **Namespace** | One floor of a building |
| **Pod** | One apartment |
| **Container** | One room in the apartment |
| **Deployment** | The building manager who ensures apartments stay occupied |
| **Service** | The building's address — stable even as tenants come and go |

The key insight from this analogy: tenants (pods) come and go, but the building's address (Service) stays the same. This is exactly why Services exist — pod IP addresses change every time a pod restarts or is replaced, but the Service gives a stable address that always points to healthy pods.

### Core Kubernetes Concepts

**Pod** — The smallest thing Kubernetes can schedule and manage. A pod wraps one or more containers that share a network namespace (same IP address) and can share storage. In practice, most pods run exactly one container — your app.

**Deployment** — Declares a desired state: "I want 3 pods running this container image." Kubernetes continuously reconciles reality toward that desired state. If a pod crashes, the Deployment creates a new one. If you ask for 5 replicas and only 3 are running, Kubernetes creates 2 more. You never say "start this pod" — you say "the desired state is X pods" and Kubernetes makes it happen.

**Service** — A stable network endpoint for a set of pods. Pods matching the Service's `selector` labels receive traffic from the Service. When pods restart and get new IP addresses, the Service automatically updates its list of targets. From a caller's perspective, the Service address never changes.

**Namespace** — A virtual boundary within a cluster. Resources in different namespaces are isolated from each other by default. Teams use namespaces to separate environments (dev/staging/prod) or ownership (team-a/team-b). We'll go deep on namespaces in Part 2 — for now, we'll create one namespace for our app.

**Node** — A physical or virtual machine that runs pods. A cluster has one or more nodes. The scheduler decides which node each pod runs on based on resource availability, constraints, and policies.

### Resource Requests and Limits

When you define a pod, you can (and should) specify how much CPU and memory it needs:

```yaml
resources:
  requests:
    memory: "256Mi"   # The scheduler guarantees this much is available on the node
    cpu: "250m"       # 250 millicores = 0.25 of one CPU core
  limits:
    memory: "512Mi"   # Hard ceiling — pod is killed if it exceeds this
    cpu: "500m"       # Soft ceiling — pod is throttled (slowed) if it exceeds this
```

**Requests** are what the scheduler uses to decide which node to place the pod on. If a node only has 100m CPU free, a pod requesting 250m won't be scheduled there.

**Limits** cap what the pod can consume. The behavior differs between CPU and memory:
- **CPU over limit:** The pod is *throttled* — it runs slower, but keeps running.
- **Memory over limit:** The pod is *OOMKilled* — the kernel kills it immediately with no warning.

Without resource limits, a single pod with a memory leak can exhaust all memory on a node, causing every other pod on that node to be evicted. This is called the "noisy neighbor" problem.

### Health Probes — How Kubernetes Knows Your App's State

Kubernetes uses three types of probes to make decisions about your pod:

**startupProbe:** "Is the app still initializing?" While this probe is failing, liveness and readiness probes are disabled. Once it succeeds, it's done — it never runs again.

**readinessProbe:** "Is the app ready to receive traffic?" If this fails, Kubernetes removes the pod from the Service's list of targets — no traffic is sent to it. When it passes again, traffic resumes. The pod is NOT restarted.

**livenessProbe:** "Is the app still alive?" If this fails a configured number of times, Kubernetes restarts the container.

Here's why all three matter for a Spring Boot application specifically:

Spring Boot takes 20–60 seconds to start. During this time it's loading the application context, connecting to databases, running Flyway migrations, and warming up caches. The app is not ready to serve traffic — but it's also not dead.

Without a `startupProbe`, the `livenessProbe` starts checking immediately. It fails (the app isn't responding yet). After 3 failures, Kubernetes restarts the container. The container starts booting again. The liveness probe fails again. Kubernetes restarts it again. You're in a `CrashLoopBackOff` loop, and the app never finishes starting.

The `startupProbe` exists specifically to handle this. While it's failing (app is starting up), the liveness probe is completely disabled — it won't fire at all. Once the startup probe succeeds (app is up), liveness takes over for the rest of the pod's life.

### Graceful Shutdown — Handling the End

When Kubernetes needs to stop a pod (rolling update, scale-down, node drain), it follows this sequence:

```
1. Pod marked as Terminating → removed from Service endpoints → no new traffic
2. preStop hook executes (your chance to drain in-flight requests)
3. SIGTERM sent to the container process
4. Spring Boot's graceful shutdown handles the SIGTERM (finishes active requests)
5. terminationGracePeriodSeconds countdown begins (default: 30s)
6. If still running after the grace period → SIGKILL (forced exit)
```

The `preStop` sleep and `terminationGracePeriodSeconds` together ensure in-flight requests have time to complete before the process exits.

### Installing and Starting Minikube

Minikube runs a single-node Kubernetes cluster on your laptop inside a VM or Docker container. It's the standard way to develop Kubernetes locally.

```bash
# Start Minikube with enough resources for our experiments
minikube start --cpus=4 --memory=8192 --driver=docker

# Verify the cluster is running
kubectl cluster-info
# Expected: Kubernetes control plane is running at https://127.0.0.1:PORT

kubectl get nodes
# Expected: NAME       STATUS   ROLES           AGE   VERSION
#           minikube   Ready    control-plane   Xm    v1.X.X
```

Enable the Ingress addon (we'll use it in Part 5):
```bash
minikube addons enable ingress
minikube addons enable metrics-server
```

### Writing Kubernetes Manifests

A Kubernetes manifest is a YAML file that describes the desired state of a resource. You write it, `kubectl apply` it, and Kubernetes makes reality match your description.

Create a directory for your Kubernetes files:
```bash
mkdir k8s
```

**`k8s/namespace.yaml`** — Logical boundary for our app:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello-app
  labels:
    # Labels let you query and select resources by category.
    # These are arbitrary key-value pairs you define yourself.
    environment: development
```

**`k8s/deployment.yaml`** — Declares the desired state for our pods:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-app
  namespace: hello-app
  labels:
    app: hello-app
spec:
  # WHY 3 replicas: with 3 pods, one can fail or be updated at any time
  # and the other two keep serving traffic. With 1 replica, every restart
  # or update causes a brief outage.
  replicas: 3

  # selector tells the Deployment which pods it "owns" and is responsible for.
  # It matches pods whose labels include app=hello-app.
  selector:
    matchLabels:
      app: hello-app

  # Rolling update strategy: how to replace old pods with new ones.
  # maxSurge: 1     → at most 1 extra pod above the desired count during update
  # maxUnavailable: 0 → never go below the desired count (zero downtime)
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        # These labels MUST match spec.selector.matchLabels above.
        # The Deployment uses this to track which pods belong to it.
        app: hello-app
    spec:
      # preStop: sleep before SIGTERM is sent.
      # WHY: Kubernetes removes the pod from the Service endpoints (step 1)
      # and sends SIGTERM (step 3) almost simultaneously — but load balancers
      # can take a second or two to propagate the endpoint removal. Without the
      # sleep, the pod may receive requests after SIGTERM while the LB still
      # thinks it's available. The 15s sleep bridges that gap.
      #
      # terminationGracePeriodSeconds: 60 gives the pod up to 60 seconds total to shut down.
      # Breakdown: 15s preStop sleep + up to 30s Spring Boot graceful shutdown + 15s buffer.
      # If requests don't finish in 60s, the pod is force-killed with SIGKILL.
      terminationGracePeriodSeconds: 60

      containers:
      - name: hello-app
        image: hello-app:1.0.0
        # IfNotPresent: use local image if available, otherwise pull from registry.
        # This is what allows us to use a locally built image in Minikube.
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080

        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

        env:
        # Override the Spring profile via environment variable.
        # The image doesn't hardcode a profile — Kubernetes tells it which one to use.
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"

        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

        # startupProbe: disables liveness until this passes.
        # WHY: Spring Boot takes 20-60s to start. Without this, the liveness
        # probe fires during startup, fails, and triggers a restart loop.
        # failureThreshold * periodSeconds = max startup time allowed.
        # 30 * 10s = 300 seconds (5 minutes) — generous for slow environments.
        startupProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          failureThreshold: 30
          periodSeconds: 10

        # readinessProbe: checked continuously throughout the pod's life.
        # WHY: if the database goes down, readiness fails → pod removed from Service
        # (no traffic sent) → no errors returned to users. Pod is NOT restarted —
        # restarting won't fix the database being down.
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          periodSeconds: 5
          failureThreshold: 3

        # livenessProbe: restarts the container if the app is truly stuck.
        # WHY: use a different path than readiness. The liveness endpoint should
        # only check internal JVM health — not external dependencies like databases.
        # If you link liveness to database health, a DB outage causes all pods
        # to restart in a loop, making recovery worse, not better.
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          periodSeconds: 10
          failureThreshold: 3
```

**`k8s/service.yaml`** — Stable network endpoint:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-app-service
  namespace: hello-app
spec:
  # ClusterIP: the Service is reachable only from within the cluster.
  # This is correct for internal services. We'll add an Ingress in Part 5
  # to expose it externally without using the more expensive LoadBalancer type.
  type: ClusterIP

  # selector matches pods with this label. The Service routes traffic to all
  # pods matching this selector that pass their readiness check.
  selector:
    app: hello-app  # Must match the pod labels in the Deployment

  ports:
  - name: http
    port: 8080        # Port the Service listens on
    targetPort: 8080  # Port on the pod to forward to
    protocol: TCP
```

### Deploy and Verify

For Minikube, we need to make our locally built image available to the cluster. The easiest way is to build the image directly inside Minikube's Docker daemon:

```bash
# Point your Docker CLI at Minikube's Docker daemon
eval $(minikube docker-env)

# Build inside Minikube — this image is now directly available to Kubernetes
docker build -t hello-app:1.0.0 .

# When done, reset to your normal Docker daemon:
# eval $(minikube docker-env --unset)
# TIP: Run this reset after your session — otherwise your Docker CLI will keep
# talking to Minikube's daemon and you won't be able to run normal docker commands.
```

Apply the manifests:
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
```

**Why this order?** Kubernetes applies resources in dependency order. The namespace must exist first so the Deployment and Service have a place to live. The Deployment must create pods before the Service can route traffic to them. `kubectl apply` handles this gracefully even if you apply all files at once, but understanding the dependency order helps when debugging.

Watch the pods come up:
```bash
kubectl get pods -n hello-app -w
# NAME                         READY   STATUS              RESTARTS   AGE
# hello-app-5d4b9f6c8d-2lmpq   0/1     ContainerCreating   0          2s
# hello-app-5d4b9f6c8d-2lmpq   0/1     Running             0          5s
# hello-app-5d4b9f6c8d-2lmpq   1/1     Running             0          25s  ← ready
```

The pod transitions: `ContainerCreating` → `Running` (process started, but not yet ready) → `1/1 Running` (startup probe passed, readiness probe passing, traffic being sent). The time from `Running` to `1/1` is Spring Boot's startup time.

Test the app via port-forward (temporary, for local testing only):
```bash
kubectl port-forward service/hello-app-service 8080:8080 -n hello-app
# NOTE: When deployed via Helm, the service name becomes <release-name>-<chart-name>.
# For example: service/hello-app-dev-hello-app-chart
# Don't worry about this now, we'll learn about helm later in the series
```

In a second terminal:
```bash
curl http://localhost:8080/api/hello
# Hello, World from Spring Boot!

curl http://localhost:8080/api/info
# {"appName":"Hello Kubernetes App",...}
```

Useful commands to understand what's happening:
```bash
# See all resources in the namespace at once
kubectl get all -n hello-app

# Describe a pod — shows events, conditions, resource usage, probe results
kubectl describe pod <pod-name> -n hello-app

# Stream logs
kubectl logs -f deployment/hello-app -n hello-app

# See resource usage (requires metrics-server)
kubectl top pods -n hello-app

# Get events sorted by time (first place to check when something goes wrong)
kubectl get events -n hello-app --sort-by='.lastTimestamp'
```

---

## Chapter 5: Helm — Never Copy-Paste YAML Again

### The Problem Helm Solves

You now have two YAML files that deploy your app. Now imagine you need three environments: dev, staging, and prod.

```
dev:     1 replica, image tag "dev-latest", 128Mi memory, no ingress
staging: 2 replicas, image tag "staging-latest", 256Mi memory, no ingress
prod:    5 replicas, image tag "v1.2.3", 512Mi memory, ingress enabled
```

Without Helm, you'd maintain three copies of every YAML file — six files total, 95% identical. Every time you add a label, change a probe path, or adjust a port, you edit six files. You'll eventually miss one. The environments drift apart silently.

Helm solves this with templates. You write your YAML once with `{{ .Values.x }}` placeholders, keep a `values.yaml` with your defaults, and override only what differs per environment.

```
helm install myapp ./chart -f values-prod.yaml
# Renders the template with prod values → applies to cluster
```

### What Helm Actually Is

Helm is three things in one:

1. **Template engine:** Renders Kubernetes YAML from Go templates + values files
2. **Package manager:** A "chart" is a deployable package (like a `.deb` or `.jar`)
3. **Release manager:** Tracks what's deployed where, supports rollback

### Creating a Chart

```bash
helm create hello-app-chart
```

This generates:

```
hello-app-chart/
├── Chart.yaml          ← Metadata: chart name, version, description
├── values.yaml         ← Default values — the single file you change per environment
├── templates/
│   ├── _helpers.tpl    ← Named templates (shared snippets used by other templates)
│   ├── deployment.yaml ← Template — uses {{ .Values.x }} instead of hardcoded values
│   ├── service.yaml    ← Template
│   ├── ingress.yaml    ← Template (disabled by default in values.yaml)
│   ├── hpa.yaml        ← Template (disabled by default)
│   └── NOTES.txt       ← Printed to terminal after install
└── charts/             ← Subchart dependencies (covered in Part 3)
```

### Chart.yaml

```yaml
apiVersion: v2
name: hello-app-chart

# description: shown in helm list and helm search
description: A Helm chart for the Hello App Spring Boot application

type: application

# version: the chart version — bump this when you change the chart itself
# (templates, values structure, new resources added)
version: 1.0.0

# appVersion: the version of the APPLICATION the chart deploys.
# Shown in helm list. Does NOT affect the chart — purely informational.
# Conventionally matches your Docker image tag for the current release.
appVersion: "1.0.0"
```

The `version` vs `appVersion` distinction matters: `version` is the chart's own version (bump when you change templates). `appVersion` is the application version (informational, shown in `helm list`). You'll typically bump `appVersion` when releasing new app versions and `version` when changing chart structure.

### values.yaml

```yaml
# Default values — applied to every environment unless overridden.
# These should be safe, conservative defaults.

replicaCount: 2

image:
  repository: hello-app
  tag: "1.0.0"
  # IfNotPresent: use cached image if already present on the node.
  # In production, use Always so you always get the latest push for a given tag.
  pullPolicy: IfNotPresent

imagePullSecrets: []  # For private registries — see Chapter 6 of this article

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: false  # Disabled by default; enabled in prod values

resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "500m"
    memory: "512Mi"

# Health probe configuration — same defaults for all environments,
# override in individual values files if needed.
probes:
  startup:
    path: /actuator/health/readiness
    failureThreshold: 30
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    periodSeconds: 5
    failureThreshold: 3
  liveness:
    path: /actuator/health/liveness
    periodSeconds: 10
    failureThreshold: 3

# Environment variables injected into the container
env:
  SPRING_PROFILES_ACTIVE: "kubernetes"
  APP_NAME: "Hello App"

# Graceful shutdown
terminationGracePeriodSeconds: 60
```

### Understanding Go Templates

Helm templates use Go's template language. Here's the minimal syntax you need:

```yaml
# .Values reads from values.yaml (or your -f override file)
replicas: {{ .Values.replicaCount }}

# .Release is set by Helm at install time
name: {{ .Release.Name }}        # The name you gave: helm install this-name ./chart
namespace: {{ .Release.Namespace }}

# .Chart reads from Chart.yaml
appVersion: {{ .Chart.AppVersion }}

# Pipelines: pass output from left to right
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default "latest" }}"

# quote: wraps in double quotes (required for strings that look like numbers)
value: {{ .Values.config.port | quote }}

# nindent: adds N spaces of indentation AND a leading newline.
# Critical when including a multi-line template block inside a larger YAML structure.
labels:
  {{- include "hello-app-chart.labels" . | nindent 4 }}
#  ↑ The - trims the whitespace before {{ so no blank line appears
```

### `_helpers.tpl` — Reusable Template Snippets

`_helpers.tpl` defines named templates — reusable blocks of text you can call from any template file. Files starting with `_` are not rendered as Kubernetes manifests.

Here's the important part of a standard `_helpers.tpl`:

```
{{/*
hello-app-chart.fullname
Constructs the full resource name: <release-name>-<chart-name>
Truncated to 63 characters because Kubernetes DNS labels have a 63-character limit.
*/}}
{{- define "hello-app-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{/*
hello-app-chart.labels
Common labels applied to every resource.
These are standard Kubernetes labels that tools like kubectl and Helm understand.
*/}}
{{- define "hello-app-chart.labels" -}}
helm.sh/chart: {{ printf "%s-%s" .Chart.Name .Chart.Version }}
{{ include "hello-app-chart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
hello-app-chart.selectorLabels
Labels used in Deployment.spec.selector.matchLabels and Service.spec.selector.
IMPORTANT: these MUST NOT CHANGE after the first deployment.
Kubernetes rejects updates to matchLabels on an existing Deployment.
That's why selectorLabels are kept minimal and stable (no version info),
while the broader "labels" template can include version which changes every release.
*/}}
{{- define "hello-app-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

**Why `include` instead of `template`?**

You'll see two ways to call a named template in Helm:
```
{{ template "myapp.labels" . }}   ← outputs directly, cannot be modified
{{ include "myapp.labels" . }}    ← returns a string, CAN be piped
```

Use `include` everywhere. The reason: `template` outputs directly into the document — you can't pipe its output to `nindent` for indentation. `include` returns a string, which you can then pipe:

```yaml
labels:
  {{- include "hello-app-chart.labels" . | nindent 4 }}
#                                         ↑ this only works with include, not template
```

### The Deployment Template

Replace `templates/deployment.yaml` with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "hello-app-chart.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "hello-app-chart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "hello-app-chart.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "hello-app-chart.selectorLabels" . | nindent 8 }}
    spec:
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - containerPort: 8080
          protocol: TCP
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
        env:
        {{- range $key, $val := .Values.env }}
        - name: {{ $key }}
          value: {{ $val | quote }}
        {{- end }}
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
        startupProbe:
          httpGet:
            path: {{ .Values.probes.startup.path }}
            port: 8080
          failureThreshold: {{ .Values.probes.startup.failureThreshold }}
          periodSeconds: {{ .Values.probes.startup.periodSeconds }}
        readinessProbe:
          httpGet:
            path: {{ .Values.probes.readiness.path }}
            port: 8080
          periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          failureThreshold: {{ .Values.probes.readiness.failureThreshold }}
        livenessProbe:
          httpGet:
            path: {{ .Values.probes.liveness.path }}
            port: 8080
          periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          failureThreshold: {{ .Values.probes.liveness.failureThreshold }}
```

### Environment-Specific Values Files

```yaml
# values-dev.yaml — override only what differs from defaults
replicaCount: 1

image:
  tag: "dev-latest"
  pullPolicy: Always

resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "200m"
    memory: "256Mi"

env:
  SPRING_PROFILES_ACTIVE: "dev"
  APP_NAME: "Hello App (Dev)"
```

```yaml
# values-staging.yaml
replicaCount: 2

image:
  tag: "staging-latest"

env:
  SPRING_PROFILES_ACTIVE: "staging"
  APP_NAME: "Hello App (Staging)"
```

```yaml
# values-prod.yaml
replicaCount: 5

image:
  repository: yourusername/hello-app  # Full registry path in prod
  tag: "v1.2.3"                       # Always pin to a specific version in prod
  pullPolicy: Always

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.hello-app.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "1Gi"

env:
  SPRING_PROFILES_ACTIVE: "prod"
  APP_NAME: "Hello App"
```

### Key Helm Commands

**Before deploying, always validate:**
```bash
# Check syntax — catches template errors before they hit the cluster
helm lint hello-app-chart/

# Preview rendered YAML without deploying — essential for debugging
helm template my-release ./hello-app-chart -f values-dev.yaml

# Dry run — sends to cluster API for validation but doesn't create anything
helm install my-release ./hello-app-chart -f values-dev.yaml \
  --namespace hello-app \
  --dry-run --debug
```

**Deploy:**
```bash
# First deployment
helm install hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace hello-app \
  --create-namespace

# Update (changed code or values)
helm upgrade hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace hello-app

# --wait: block until all pods are healthy or timeout
# WHY in CI/CD: without --wait, the pipeline succeeds even if pods are crashing
helm upgrade hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace hello-app \
  --wait \
  --timeout 5m

# --atomic: if the upgrade fails, automatically roll back to the previous release
# WHY in production: prevents a bad deploy from leaving the cluster in a broken state
helm upgrade hello-app-prod ./hello-app-chart \
  -f values-prod.yaml \
  --namespace production \
  --atomic \
  --wait \
  --timeout 10m
```

**Inspect and manage:**
```bash
# What's deployed
helm list -n hello-app

# What values are currently in use
helm get values hello-app-dev -n hello-app

# Full rendered YAML of the current release
helm get manifest hello-app-dev -n hello-app

# Release history
helm history hello-app-dev -n hello-app

# Roll back to a specific revision
helm rollback hello-app-dev 2 -n hello-app

# Remove everything
helm uninstall hello-app-dev -n hello-app
```

**Package for distribution:**
```bash
helm package hello-app-chart/
# Creates: hello-app-chart-1.0.0.tgz
# This file is what you'd push to a chart repository
```

---

## Chapter 6: Container Registry — Making Images Available to Kubernetes

### Why a Registry?

When you built the Docker image earlier, it existed only on your laptop. Minikube worked because we pointed Docker at Minikube's internal daemon — effectively building the image inside the cluster. But in a real environment with a multi-node cluster, there's no shared Docker daemon. Every node needs to pull the image from a central location: a **container registry**.

```
Your laptop          Registry              Kubernetes cluster
──────────           ──────────            ─────────────────
docker build    →    docker push    →    kubelet pulls image
hello-app:1.0.0      hello-app:1.0.0      on each node when
                                           pod is scheduled
```

### Push to Docker Hub

Docker Hub is free for public images and has one free private repository — enough to get started.

```bash
# Log in (one-time)
docker login

# Tag your local image with your Docker Hub username
# Format: <username>/<repository>:<tag>
docker tag hello-app:1.0.0 yourusername/hello-app:1.0.0

# Push to Docker Hub
docker push yourusername/hello-app:1.0.0
```

### Image Tagging Strategy

The tag `latest` seems convenient but causes real problems in production:

```yaml
# ❌ Avoid in production
image: yourusername/hello-app:latest
```

With `latest`, you have no idea which actual version is running. Rolling back means re-tagging and re-pushing. Two nodes can end up running different versions if one pulled before your push and one after.

```yaml
# ✅ Specific semantic version — immutable and auditable
image: yourusername/hello-app:v1.2.3

# ✅ Git SHA — absolute traceability (used in CI/CD pipelines)
image: yourusername/hello-app:git-a1b2c3d
```

In production, always pin to a specific tag. In CI/CD, build with the git commit SHA as the tag — you can always trace exactly which code is running.

### Image Pull Secrets for Private Registries

If your registry is private, Kubernetes needs credentials to pull from it:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=yourusername \
  --docker-password=youraccesstoken \
  --docker-email=you@example.com \
  --namespace hello-app
```

Reference the secret in your Helm values:
```yaml
# values-prod.yaml
imagePullSecrets:
  - name: regcred
```

This causes Helm to add `imagePullSecrets` to the pod spec, and Kubernetes uses it when pulling.

### Updating values.yaml to Use the Registry

```yaml
# values.yaml — update image.repository to include your Docker Hub username
image:
  repository: yourusername/hello-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent
```

---

## Chapter 7: Putting It All Together

### The Complete Workflow

Here's the full sequence from code to running cluster:

```bash
# 1. Build the JAR
./mvnw clean package -DskipTests

# 2. Build the Docker image inside Minikube
eval $(minikube docker-env)
docker build -t yourusername/hello-app:1.0.0 .

# 3. Validate the Helm chart
helm lint hello-app-chart/
helm template my-release ./hello-app-chart -f values-dev.yaml

# 4. Deploy with Helm
helm upgrade --install hello-app-dev ./hello-app-chart \
  -f values-dev.yaml \
  --namespace hello-app \
  --create-namespace \
  --wait \
  --timeout 5m

# 5. Verify
kubectl get pods -n hello-app
kubectl get svc -n hello-app

# 6. Test
kubectl port-forward service/hello-app-dev-hello-app-chart 8080:8080 -n hello-app
curl http://localhost:8080/api/hello
```

### Verification Checklist

After every deployment, verify these before declaring success:

```bash
# All pods are running (1/1 READY, not 0/1)
kubectl get pods -n hello-app

# No recent restarts (RESTARTS column should be 0 or very low)
kubectl get pods -n hello-app

# Readiness probe is passing
kubectl describe pod <pod-name> -n hello-app | grep -A5 "Readiness:"

# Service has endpoints (pods registered)
kubectl get endpoints hello-app-dev-hello-app-chart -n hello-app
# Should show pod IPs, not "<none>"

# App responds correctly
kubectl port-forward service/hello-app-dev-hello-app-chart 8080:8080 -n hello-app &
curl http://localhost:8080/actuator/health/readiness
```

### Part 1 Checklist

By the end of this part, you should have:

- [ ] Spring Boot app with Actuator liveness + readiness endpoints
- [ ] Multi-stage Dockerfile with non-root user, correct JVM flags
- [ ] `.dockerignore` to keep build context lean
- [ ] Kubernetes Deployment with all three probe types configured
- [ ] Kubernetes Service (ClusterIP)
- [ ] Helm chart with `values.yaml`, `values-dev.yaml`, `values-prod.yaml`
- [ ] App running in Minikube via Helm
- [ ] `helm upgrade --install` working with `--wait`

---

## Common Pitfalls

These are the issues that trip up most people on their first Kubernetes deployment:

**1. Docker context stuck on Minikube**
After running `eval $(minikube docker-env)`, your Docker CLI points at Minikube's daemon. When you're done, run `eval $(minikube docker-env --unset)` or restart your terminal. Otherwise, `docker run` commands will fail because Minikube's daemon isn't reachable outside the VM.

**2. Service selector doesn't match pod labels**
The Service uses `selector.matchLabels` to find pods. If your Deployment's pod labels don't match exactly, the Service shows `<none>` for endpoints. Fix: `kubectl get pods --show-labels` and compare to your Service's selector.

**3. Helm upgrade without `--install`**
`helm upgrade` fails if the release doesn't exist yet. Always use `helm upgrade --install` — it does both in one command.

**4. Liveness probe fires before app starts**
Without a `startupProbe`, the `livenessProbe` fires immediately. Spring Boot takes 20-60s to start, so the liveness probe fails and Kubernetes restarts the pod in a `CrashLoopBackOff` loop. The `startupProbe` disables liveness checks until the app is ready.

**5. Memory limits too low for JVM**
The JVM needs heap + metaspace + thread stacks + native memory. A 256Mi limit often causes `OOMKilled`. Start with 512Mi minimum for Spring Boot apps.

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| `ImagePullBackOff` | Image not found in registry, or wrong tag | `kubectl describe pod <n> -n hello-app` | Check image name/tag, verify push succeeded, check pull secret |
| `CrashLoopBackOff` | App crashes on startup | `kubectl logs <n> -n hello-app --previous` | Check logs for exception; common: missing env var, wrong DB URL |
| Pod stuck in `Pending` | No node has enough resources | `kubectl describe pod <n> -n hello-app` → look at Events | Reduce resource requests, or free up node capacity |
| `0/1 READY` (not progressing to 1/1) | Startup or readiness probe failing | `kubectl describe pod <n>` → Conditions section | Check probe path is correct; check app actually exposes that path |
| `OOMKilled` | Pod exceeded memory limit | `kubectl describe pod <n>` → Last State section shows OOMKilled | Increase memory limit, or check for memory leak; verify JVM flags are set |
| Service has no endpoints (`<none>`) | Pod labels don't match Service selector | `kubectl get pods --show-labels -n hello-app` | Make sure pod labels match `spec.selector` in Service |
| `helm: release not found` on upgrade | Using upgrade before install | N/A | Use `helm upgrade --install` which does both |
| Template renders with wrong indentation | Missing `nindent` after `include` | `helm template ... \| cat` | Add `\| nindent N` after the include call |
| Port-forward fails | Wrong service name | `kubectl get svc -n hello-app` | Use exact service name from `helm get manifest` or `helm list` |
| Service has no endpoints (`<none>`) | Pod labels don't match Service selector | `kubectl get pods --show-labels -n hello-app` | Make sure pod labels match `spec.selector` in Service |

---

## Practice Exercises

**Exercise 1 — Environment variable override:**
Add a new `GET /api/env` endpoint that returns the `SPRING_PROFILES_ACTIVE` environment variable. Verify that it returns `"dev"` when deployed with `values-dev.yaml`.

**Exercise 2 — Resource limits experiment:**
Set the memory limit to `64Mi` in a dev values file. Deploy and watch what happens. Find the `OOMKilled` in `kubectl describe pod`. Then increase the limit by 64Mi increments (128Mi, 192Mi, 256Mi...) until the app runs stably. At each step, document:
- The RESTARTS count in `kubectl get pods`
- The "Last State" shown in `kubectl describe pod`
- The exact memory usage from `kubectl top pods`

What's the minimum memory this app needs to run without being throttled or killed?

**Exercise 3 — Probe behaviour:**
Temporarily change the liveness probe path to `/actuator/health/INVALID` in your values file. Deploy and watch what happens to the pod (watch with `kubectl get pods -n hello-app -w`). How long before Kubernetes restarts it? Restore the correct path and redeploy.

**Exercise 4 — Multi-environment deploy:**
Deploy the same chart twice into the same namespace using different release names and values files:
```bash
helm upgrade --install hello-app-dev ./hello-app-chart -f values-dev.yaml -n hello-app
helm upgrade --install hello-app-staging ./hello-app-chart -f values-staging.yaml -n hello-app
```
Verify both releases coexist. Check `helm list -n hello-app` and `kubectl get pods -n hello-app`. Notice the pod counts match each values file.

**Exercise 5 — Rolling update:**
Update the `GET /api/info` endpoint to include a `"deployedBy": "helm"` field. Build a new image with tag `1.0.1`, push it, update `values.yaml` to use `tag: "1.0.1"`, and run `helm upgrade`. Use `kubectl get pods -n hello-app -w` to watch the rolling update — old pods terminate one at a time as new pods become ready. Verify there's no gap in service by running `curl http://localhost:8080/api/info` in a loop during the upgrade.

---

## What's Next?

You now have a working Spring Boot application running in Kubernetes via Helm. But there's more to cover:

**Part 2: Kubernetes Core Concepts** dives into the building blocks every production deployment needs:
- **ConfigMaps** — Externalize configuration so you don't rebuild images for every environment change
- **Secrets** — Handle sensitive data (database passwords, API keys) securely
- **Namespaces** — Organize your cluster into virtual environments (dev/staging/prod)
- **ResourceQuota** — Prevent one team from consuming all cluster resources
- **Network Policies** — Restrict which pods can talk to each other (zero-trust networking)

By the end of Part 2, you'll understand how to structure a multi-environment cluster and manage configuration without rebuilding Docker images.
