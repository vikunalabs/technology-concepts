# Part 5: Networking & Ingress — Making Your App Accessible

> **Series:** Kubernetes Mastery — From Hello World to Production
> **Level:** Intermediate
> **Prerequisites:** Completed Parts 1–4, or comfortable with Kubernetes Deployments, Services, and Helm
> **Time to complete:** 5–6 hours
> **What you'll learn:** Kubernetes networking layers, all Service types, Ingress controllers, TLS automation with cert-manager, automatic DNS with ExternalDNS, zero-trust network policies, and a complete Istio service mesh introduction

---

## What This Part Covers

Your app runs in Kubernetes. The cluster knows it's healthy. But right now, nobody outside the cluster can reach it. A `kubectl port-forward` works on your laptop — it does not work for users.

Getting traffic from the outside world to the right pod, securely, at scale, with automatic TLS, automatic DNS, and controlled internal traffic between pods: that is what this part covers. It starts with the fundamentals of how Kubernetes routes traffic and builds to a complete production networking configuration.

---

## Chapter 1: Kubernetes Networking Layers

### Why Networking in Kubernetes Is Layered

Kubernetes networking is not one thing — it is three distinct layers, each solving a different problem. Understanding why each layer exists makes the whole system make sense instead of feeling like arbitrary complexity.

```
Internet
   │
   ▼
┌─────────────────────────────────────────────┐
│  Layer 3: Ingress                           │
│  Routes by hostname/path, handles TLS       │
│  "api.myapp.com/users → users-service"      │
│  One load balancer for ALL services         │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│  Layer 2: Service                           │
│  Stable IP and DNS name for a set of pods   │
│  Load balances across healthy pods          │
│  "users-service:8080 → pod-1, pod-2, pod-3" │
└──────────────────────┬──────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   Pod 1      │ │   Pod 2      │ │   Pod 3      │
│  10.244.1.5  │ │  10.244.2.7  │ │  10.244.3.2  │
└──────────────┘ └──────────────┘ └──────────────┘
Layer 1: Pod network — flat, every pod can reach every other pod
```

**Layer 1 — Pod network:** Every pod gets an IP address from the cluster's internal IP range. These IPs are not exposed outside the cluster. Pods can communicate directly with each other by IP — but pod IPs are ephemeral. When a pod restarts, it gets a new IP. You cannot rely on a pod's IP to stay the same.

**Layer 2 — Service:** A Service gives a stable, permanent IP and DNS name to a set of pods. The Service continuously updates its list of target pod IPs (called endpoints) as pods come and go. Other services in the cluster communicate with each other via Service names, not pod IPs. This layer solves the ephemeral IP problem.

**Layer 3 — Ingress:** A Service can expose itself outside the cluster, but each Service doing this gets its own cloud load balancer — which costs money and doesn't support path-based routing. Ingress sits in front of all Services, receives all external traffic on a single load balancer, and routes requests to the right Service based on hostname and path. This layer solves the cost and routing problems.

### The Ephemeral IP Problem — Why Services Exist

Imagine your API service trying to call the database. Without Services, the API pod hardcodes the database pod's IP — say `10.244.1.5`. The database pod is restarted during a rolling update. It comes back with IP `10.244.3.9`. Every API pod that cached the old IP now gets connection errors. You need to update every caller with the new IP. This is unmanageable.

Services solve this by giving the database a stable DNS name — `postgres-service.production.svc.cluster.local` — that always resolves to the current healthy pods. The API calls that name forever, regardless of how many times the underlying pods restart or how their IPs change.

### The Cost Problem — Why Ingress Exists

Each LoadBalancer Service creates one cloud load balancer. At $15–25/month each on AWS or GCP, an application with 10 services costs $150–250/month in load balancers alone — before a single byte of traffic.

Ingress uses one cloud load balancer for everything. The Ingress controller inside the cluster receives all traffic and routes it internally to the right Service. Ten services, one load balancer, one monthly cost.

---

## Chapter 2: Service Types in Depth

### ClusterIP — Internal Communication

ClusterIP is the default Service type. It creates a stable internal IP address and DNS name reachable only from within the cluster. Nothing outside the cluster can reach a ClusterIP Service directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: users-service
  namespace: production
spec:
  type: ClusterIP   # The default — can be omitted
  selector:
    app: users      # Routes to pods with this label
  ports:
  - name: http
    port: 8080        # Port the Service listens on
    targetPort: 8080  # Port on the pod to forward to
```

Every Service gets a DNS name following the pattern:
```
<service-name>.<namespace>.svc.cluster.local

# Full form:
users-service.production.svc.cluster.local:8080

# Short form (within the same namespace):
users-service:8080

# Short form (cross-namespace):
users-service.production:8080
```

When the orders service needs to call the users service, it uses `http://users-service:8080/users/123`. Kubernetes DNS resolves this to the Service's ClusterIP. The Service load-balances to healthy pods. The caller never knows or cares about pod IPs.

### NodePort — Direct Node Access

NodePort opens a port on every node in the cluster and forwards traffic on that port to the Service. It exists for situations where you need external access without a cloud load balancer — primarily local development and bare-metal clusters.

```yaml
spec:
  type: NodePort
  ports:
  - port: 8080        # Service port (internal)
    targetPort: 8080  # Pod port
    nodePort: 30080   # Port opened on every node (30000–32767)
    # If nodePort is omitted, Kubernetes assigns one randomly in the valid range
```

The port range 30000–32767 is reserved specifically for NodePort Services by Kubernetes. Ports below 30000 are used by the OS and other processes — Kubernetes avoids them to prevent conflicts.

NodePort is not production-ready for two reasons: you expose node IPs directly (security concern), and you need to know which node is running the pod (no DNS name, just `<node-ip>:30080`). For local testing with kind, node IPs aren't reachable from your laptop at all — kind's nodes are Docker containers on a private Docker network — so the simplest way to reach a NodePort Service is `kubectl port-forward service/users-service 8080:8080`, which we already used in Part 1.

### LoadBalancer — Cloud Load Balancer per Service

LoadBalancer extends NodePort by additionally provisioning a cloud load balancer (AWS ELB/NLB, GCP LB, Azure LB) and routing its traffic to the NodePort. You get a stable external IP/hostname from the cloud provider.

```yaml
spec:
  type: LoadBalancer
  ports:
  - port: 443
    targetPort: 8080
  # Cloud-provider-specific annotations control the load balancer type
```

**AWS — Network Load Balancer (preferred over classic ELB):**
```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
```

**GCP — Internal Load Balancer:**
```yaml
metadata:
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
```

**Azure — Standard Load Balancer:**
```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

Use LoadBalancer directly only when one specific service genuinely needs its own dedicated external IP — for example, a TCP service that Ingress cannot handle, or a service requiring static IP allocation. For HTTP/HTTPS services, use Ingress instead.

### ExternalName — DNS Alias for External Services

ExternalName maps a Service name to an external DNS name. No proxying happens — the Service simply returns a CNAME:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  type: ExternalName
  externalName: myapp.cluster-xyz.us-east-1.rds.amazonaws.com
```

Now `database.production.svc.cluster.local` resolves to the RDS hostname. Your application code points to `database:5432` without knowing whether the database is in-cluster or external. Useful during migrations: start with an external service, later replace with an in-cluster StatefulSet — update only the Service, not the application config.

### Headless Services — Direct Pod DNS

A headless Service has `clusterIP: None`. Instead of routing traffic through a stable ClusterIP, DNS returns the IPs of all matching pods directly.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None     # Makes it headless — no stable ClusterIP assigned
  selector:
    app: postgres
  ports:
  - port: 5432
```

With a headless Service, each pod gets an individual DNS record:
```
postgres-0.postgres-headless.production.svc.cluster.local → 10.244.1.5
postgres-1.postgres-headless.production.svc.cluster.local → 10.244.2.7
postgres-2.postgres-headless.production.svc.cluster.local → 10.244.3.2
```

StatefulSets require a headless Service to give each pod a stable, addressable DNS name. A primary-replica database cluster uses this to let replicas find and connect to the primary by name (`postgres-0`), regardless of IP changes. We cover StatefulSets in depth in Part 6.

---

## Chapter 3: Ingress Controllers

### What an Ingress Controller Is — and Why It's Separate

Kubernetes defines an **Ingress resource** — a YAML spec that says "route requests for this hostname to this Service." But Kubernetes does not implement that routing itself. The Ingress resource is just a configuration object.

An **Ingress controller** is a separately deployed component that reads Ingress resources and actually implements the routing. It is typically an Nginx, HAProxy, or Envoy proxy running inside the cluster, watching for Ingress resource changes and reconfiguring itself accordingly.

This separation means you choose which controller to run based on your needs. The same Ingress YAML works with any compliant controller, though most controllers have their own annotations for advanced features.

### Choosing a Controller

| Controller | Best for | Notes |
|-----------|----------|-------|
| **nginx-ingress** | Most users — default choice | Works on every cluster, huge feature set, large community, excellent docs |
| **AWS ALB Controller** | EKS users wanting native AWS integration | Each Ingress creates an ALB; native WAF/Shield integration; EKS-only |
| **Traefik** | Teams wanting a built-in dashboard and automatic service discovery | Good for dynamic environments; middleware model for auth/rate limiting |
| **Istio Gateway** | Teams already running Istio | Don't install Istio just for ingress — the operational overhead is significant |
| **GCE Ingress** | GKE users who want native Google Cloud LB | GKE-only; integrates with Cloud Armor |
| **Contour** | Teams wanting HTTP/2 and gRPC support | Built on Envoy; good for gRPC-heavy workloads |

**The recommendation:** start with nginx-ingress. It works everywhere, the documentation is comprehensive, and every Stack Overflow answer about Ingress almost certainly applies to it. Switch to a cloud-native controller later if you have a specific integration requirement.

### Installing nginx-ingress

**kind:** kind ships a manifest variant of ingress-nginx specifically patched to work with its Docker-based nodes — it targets nodes labeled `ingress-ready=true` and binds to the host ports we mapped when creating the cluster back in Part 1 (`kind-config.yaml`'s `extraPortMappings`). If you didn't set those up, go back and re-create the cluster with that config first — they can't be added to a running cluster.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for the controller pod to be ready before continuing
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

Once this is running, anything you send to `localhost:80` or `localhost:443` on your laptop reaches the ingress controller inside the kind cluster — that's the `extraPortMappings` from Part 1 doing their job.

**All cloud providers via Helm:**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"=true \
  --set controller.podAnnotations."prometheus\.io/port"=10254

# Wait for external IP to be assigned (takes 1–3 minutes on cloud providers)
kubectl get svc ingress-nginx-controller -n ingress-nginx -w
# NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
# ingress-nginx-controller   LoadBalancer   10.100.50.100   a1b2c3.elb.aws... 80:31234,443:30456
```

**AWS EKS — with NLB:**
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-scheme"=internet-facing
```

**GKE:**
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."cloud\.google\.com/load-balancer-type"=External
```

**AKS:**
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

### Installing AWS ALB Ingress Controller (EKS Alternative)

The AWS Load Balancer Controller creates an ALB per Ingress resource — better AWS integration, WAF support, but EKS-only:

```bash
# Install via Helm (requires IRSA setup first — IAM role for the controller)
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

With the ALB controller, each Ingress resource creates a dedicated ALB. The tradeoff vs nginx-ingress: you pay per ALB ($0.008/hour on AWS), but you get native integration with AWS WAF, AWS Shield, and ACM certificate management.

---

## Chapter 4: Ingress Rules

### Basic Host-Based Routing

The simplest Ingress routes all traffic for a hostname to one Service:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx   # Which controller handles this Ingress
spec:
  rules:
  - host: api.myapp.com         # Requests with Host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service   # Forward to this Service
            port:
              number: 8080
  - host: www.myapp.com         # A second hostname on the same Ingress
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

One Ingress resource, two hostnames, one load balancer. This is the core cost saving vs two LoadBalancer Services.

### Path-Based Routing and `pathType` — A Common Gotcha

When routing by path, `pathType` controls how the path is matched. Getting it wrong causes requests to silently not match — you get a 404 with no obvious error:

| pathType | Behaviour | Example |
|----------|-----------|---------|
| `Prefix` | Matches the path and all sub-paths | `/api` matches `/api`, `/api/`, `/api/users`, `/api/orders/123` |
| `Exact` | Matches only that exact path | `/api` matches ONLY `/api` — NOT `/api/` or `/api/users` |
| `ImplementationSpecific` | Controller-defined — nginx uses this for regex | Use when you need regex patterns |

```yaml
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      # All /api/* requests → api-service
      - path: /api
        pathType: Prefix      # /api, /api/users, /api/orders/123 all match
        backend:
          service:
            name: api-service
            port:
              number: 8080

      # Exactly /health → health-service (not /health/ready)
      - path: /health
        pathType: Exact
        backend:
          service:
            name: health-service
            port:
              number: 8081

      # Everything else → web-service (catch-all must be last)
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**The silent failure:** If you use `pathType: Exact` but expect sub-paths to match, requests to `/api/users` return 404 from the Ingress. There is no error log entry saying "pathType mismatch" — the request simply falls through to a lower-priority rule or returns 404. Always verify with `curl -v` after deploying.

### Path Rewriting — Stripping URL Prefixes

A common microservices pattern: route `/api/users/*` to the users service, but the users service only knows about `/users/*`. The `/api` prefix is meaningful to the gateway but not to the backend service.

Without rewriting, the users service receives requests like `GET /api/users/123` and returns 404 because it has no route matching `/api/users/123`.

The `rewrite-target` annotation strips the prefix:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # ↑ Replace the matched path with capture group $2
    # ↑ $1 captures the separator (/ or end), $2 captures the rest

spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      # pathType must be ImplementationSpecific for regex paths
      - path: /api/users(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-service
            port:
              number: 8080
        # Request: GET /api/users/123
        # Capture groups: $1 = "/", $2 = "123"
        # Rewritten as:   GET /123  ← backend receives this
        # ✓ users-service has a route for /{id}

      - path: /api/orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-service
            port:
              number: 8080
        # Request: GET /api/orders/456/items
        # Rewritten as:   GET /456/items
```

The regex `(/|$)(.*)` means: match a `/` or end-of-string (captured as `$1`), then match everything after (captured as `$2`). The `rewrite-target: /$2` produces a path starting with `/` followed by whatever was after the prefix.

**Testing the rewrite before deploying:**
```bash
# After deploying, test with verbose output to see the actual request path
curl -v https://api.myapp.com/api/users/123

# Check nginx logs to see what the backend received
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | grep "users"
```

### Multiple Backends — Full Microservices Routing

A single Ingress can route to many services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /users(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-service
            port:
              number: 8080
      - path: /orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-service
            port:
              number: 8080
      - path: /products(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: products-service
            port:
              number: 8080
      - path: /payments(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: payments-service
            port:
              number: 8080
```

### Useful nginx Annotations

nginx-ingress exposes its full feature set through annotations. Here are the ones you'll use most often:

**Rate limiting — protect against abuse and DDoS:**
```yaml
annotations:
  # Allow 100 requests per minute per IP
  nginx.ingress.kubernetes.io/limit-rps: "10"          # Per second
  nginx.ingress.kubernetes.io/limit-connections: "20"  # Concurrent connections per IP
  nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"  # Allow bursts up to 5×
```

**CORS — allow browser requests from other origins:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://www.myapp.com,https://app.myapp.com"
  nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
  nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
  nginx.ingress.kubernetes.io/cors-max-age: "86400"
```

**Timeouts — prevent long requests from holding connections:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"   # Seconds to connect to backend
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"      # Seconds to wait for response
  nginx.ingress.kubernetes.io/proxy-send-timeout: "60"      # Seconds to send request body
```

**Request size — for file uploads:**
```yaml
annotations:
  # Default is 1MB — increases for file upload endpoints
  nginx.ingress.kubernetes.io/proxy-body-size: "50m"
```

**Security headers:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/configuration-snippet: |
    more_set_headers "X-Frame-Options: DENY";
    more_set_headers "X-Content-Type-Options: nosniff";
    more_set_headers "X-XSS-Protection: 1; mode=block";
    more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
    more_set_headers "Permissions-Policy: camera=(), microphone=(), geolocation=()";
```

**WebSocket support:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"   # Keep WS connections open
  nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  nginx.ingress.kubernetes.io/websocket-services: "ws-service"
```

**Sticky sessions — route a user to the same pod:**
```yaml
annotations:
  nginx.ingress.kubernetes.io/affinity: "cookie"
  nginx.ingress.kubernetes.io/session-cookie-name: "SERVERID"
  nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
  nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
```

**Canary routing — send a percentage of traffic to a new version:**
```yaml
# Main Ingress (existing, unmodified)
# ...

# Canary Ingress — sends 10% of traffic to the canary service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-canary
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"   # 10% to canary
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service-v2    # New version receives 10%
            port:
              number: 8080
```


## Chapter 5: TLS with cert-manager

### Why TLS Is Non-Negotiable

TLS (Transport Layer Security) encrypts traffic between clients and your application. Without it, any network device between the user and your server can read request bodies, headers, cookies, and tokens in plaintext. Beyond security, browsers mark non-HTTPS sites as "Not Secure," and many APIs refuse to function without it.

The traditional certificate management process is deeply tedious:

1. Generate a Certificate Signing Request (CSR)
2. Purchase a certificate from a CA or use Let's Encrypt (free)
3. Complete a domain validation challenge
4. Download the certificate files
5. `base64` encode the certificate and private key
6. Create a Kubernetes Secret with the encoded values
7. Reference the Secret in your Ingress `tls:` section
8. Set a calendar reminder to repeat this in 90 days (Let's Encrypt certs expire every 90 days)

That's eight steps every 90 days, per domain. With cert-manager, it is one annotation on your Ingress and zero subsequent steps. cert-manager handles validation, issuance, storage in a Secret, and renewal — automatically, forever.

### Installing cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --version v1.13.0

# Verify all three cert-manager pods are running
kubectl get pods -n cert-manager
# NAME                                      READY   STATUS
# cert-manager-5c6866597-zw7kh             1/1     Running
# cert-manager-cainjector-bd5f9c764-z7bh8  1/1     Running
# cert-manager-webhook-5f57f59fbc-m7kp8    1/1     Running
```

### ClusterIssuer vs Issuer — Scope Matters

cert-manager has two issuer types:

- **Issuer:** Scoped to one namespace. Can only issue certificates for resources in that namespace.
- **ClusterIssuer:** Cluster-wide. Can issue certificates for Ingress resources in any namespace.

For most setups, create a `ClusterIssuer` — you don't want to create a separate Issuer in every namespace that needs TLS.

### Create Staging Issuer First

Let's Encrypt has two environments: production and staging. The production environment issues trusted certificates — but it has strict rate limits (50 certificates per domain per week). If you misconfigure and retry repeatedly, you'll hit the rate limit and be blocked for a week.

The staging environment has no rate limits. It issues certificates signed by a fake CA ("Fake LE Intermediate X1") that browsers don't trust — but cert-manager will successfully issue and renew them. Use staging to verify your configuration works before switching to production.

```yaml
# clusterissuer-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Let's Encrypt staging server — no rate limits, untrusted certificates
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@myapp.com     # Your email — used for expiry notifications

    # cert-manager stores the ACME account key in this Secret
    privateKeySecretRef:
      name: letsencrypt-staging-account-key

    solvers:
    - http01:
        ingress:
          class: nginx    # Must match your Ingress controller class
          # HOW http01 validation works:
          # 1. Let's Encrypt sends a challenge: "prove you control api.myapp.com"
          # 2. cert-manager creates a temporary Ingress serving a token at:
          #    http://api.myapp.com/.well-known/acme-challenge/<token>
          # 3. Let's Encrypt fetches that URL — if it gets the right token, validation passes
          # 4. cert-manager receives the signed certificate
          # Requirement: port 80 must be publicly reachable from the internet
```

```yaml
# clusterissuer-prod.yaml — identical except for the server URL and name
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

```bash
kubectl apply -f clusterissuer-staging.yaml
kubectl apply -f clusterissuer-prod.yaml

# Verify issuers are ready
kubectl get clusterissuer
# NAME                   READY   AGE
# letsencrypt-staging    True    30s
# letsencrypt-prod       True    30s
```

### Automatic TLS via Ingress Annotation

Add one annotation to your Ingress — cert-manager handles the rest:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    # Start with staging to test, switch to letsencrypt-prod once confirmed working
    cert-manager.io/cluster-issuer: letsencrypt-staging

    # Force all HTTP traffic to HTTPS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

spec:
  ingressClassName: nginx

  # The tls section triggers cert-manager to request a certificate.
  # secretName is where cert-manager stores the issued certificate.
  # cert-manager creates this Secret automatically — don't create it yourself.
  tls:
  - hosts:
    - api.myapp.com
    secretName: api-myapp-com-tls    # cert-manager creates and manages this

  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

**What happens after applying this:**

```
1. cert-manager sees Ingress with cert-manager.io/cluster-issuer annotation
2. cert-manager creates a Certificate resource targeting api-myapp-com-tls Secret
3. cert-manager contacts Let's Encrypt: "I need a cert for api.myapp.com"
4. Let's Encrypt responds with an HTTP-01 challenge token
5. cert-manager creates a temporary Ingress serving that token
6. Let's Encrypt fetches http://api.myapp.com/.well-known/acme-challenge/<token>
7. Validation passes — Let's Encrypt issues the certificate
8. cert-manager stores it in the api-myapp-com-tls Secret
9. nginx-ingress reads the Secret and starts serving HTTPS with the certificate
10. cert-manager monitors the certificate — renews automatically 30 days before expiry
```

**Verifying certificate status:**
```bash
# Watch the certificate being issued
kubectl get certificate -n production -w
# NAME                 READY   SECRET               AGE
# api-myapp-com-tls    False   api-myapp-com-tls    10s   ← validating
# api-myapp-com-tls    True    api-myapp-com-tls    45s   ← issued ✓

# Detailed status including any errors
kubectl describe certificate api-myapp-com-tls -n production
# Look for: "Certificate issued successfully" in Events

# Inspect the actual certificate
kubectl get secret api-myapp-com-tls -n production -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -text -noout | grep -A2 "Validity"
# Validity
#     Not Before: Jan  1 00:00:00 2024 GMT
#     Not After : Apr  1 00:00:00 2024 GMT   ← 90 days from issuance
```

### Testing Locally Without a Real Domain — nip.io

When testing on kind or a cluster without a real domain, you need a URL that resolves to your cluster IP but looks like a real hostname (required for HTTP-01 validation and for Ingress host matching).

[nip.io](https://nip.io) provides wildcard DNS based on IP address: `anything.192.168.1.100.nip.io` always resolves to `192.168.1.100`. No configuration needed.

With kind, this is actually simpler than on most local clusters: because of the `extraPortMappings` we configured in Part 1, the ingress controller is already reachable at `127.0.0.1` on your laptop — there's no separate cluster IP to look up like there would be with a VM-based tool.

```bash
# No "get cluster IP" step needed — kind's ingress is already on localhost,
# thanks to the extraPortMappings configured when the cluster was created.

# Your test domain (no registration needed — nip.io handles DNS)
# api.127.0.0.1.nip.io → 127.0.0.1

# Use this as your Ingress host
```

```yaml
spec:
  tls:
  - hosts:
    - api.127.0.0.1.nip.io
    secretName: nip-tls
  rules:
  - host: api.127.0.0.1.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

Note: Let's Encrypt staging works with nip.io, but production does not — it blocks wildcard DNS services. For local testing, the staging issuer is sufficient to verify cert-manager is working correctly.

---

## Chapter 6: ExternalDNS — Automatic DNS Records

### The Manual DNS Problem

Every time you create a new Ingress, you need to update your DNS provider:
1. Look up the Ingress controller's external IP or hostname
2. Log into Route53 / Cloudflare / Google Cloud DNS
3. Create an A or CNAME record pointing to that IP
4. Wait for DNS propagation (up to 48 hours, though usually minutes)
5. Repeat for every new environment, every preview deployment, every service

With 20 services across 3 environments, that's 60 DNS records to maintain manually. ExternalDNS automates this entirely.

### What ExternalDNS Does

ExternalDNS runs inside the cluster and watches Ingress and Service resources. When it sees an Ingress with a hostname, it creates the corresponding DNS record in your cloud provider's DNS service. When the Ingress is deleted, the DNS record is removed. When the load balancer IP changes, the record is updated.

```
You create Ingress with host: api.myapp.com
          ↓
ExternalDNS detects it
          ↓
ExternalDNS calls Route53 API: "Create A record: api.myapp.com → 1.2.3.4"
          ↓
DNS resolves. No manual steps.
```

### Installing ExternalDNS

**AWS Route53:**
```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=aws \
  --set aws.region=us-east-1 \
  --set domainFilters[0]=myapp.com \    # Only manage records for this domain
  --set policy=sync \                    # sync: create AND delete records; upsert-only: only create/update
  --set registry=txt \                   # Use TXT records to track ownership
  --set txtOwnerId=my-cluster
```

ExternalDNS needs IAM permissions to create and update Route53 records. The minimal policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets",
        "route53:ListTagsForResource"
      ],
      "Resource": ["*"]
    }
  ]
}
```

Attach this to the ExternalDNS service account via IRSA (Part 7).

**Google Cloud DNS:**
```bash
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=google \
  --set google.project=my-gcp-project \
  --set domainFilters[0]=myapp.com \
  --set policy=sync
```

**Cloudflare:**
```bash
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=cloudflare \
  --set cloudflare.apiToken=your-cloudflare-api-token \
  --set domainFilters[0]=myapp.com \
  --set policy=sync
```

### Annotation-Based Control

ExternalDNS by default picks up hostnames from Ingress `spec.rules[].host`. You can also use annotations to control behaviour:

```yaml
metadata:
  annotations:
    # Explicit hostname (overrides or supplements Ingress host)
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com

    # Custom TTL (in seconds) — default is 300
    external-dns.alpha.kubernetes.io/ttl: "60"

    # Multiple hostnames (comma-separated)
    external-dns.alpha.kubernetes.io/hostname: "api.myapp.com,api-v2.myapp.com"
```

### Wildcard DNS for Preview Environments

CI/CD pipelines often create ephemeral preview environments for each pull request — a live deployment of the PR's code, accessible at a unique URL. ExternalDNS combined with wildcard DNS makes this zero-configuration:

```yaml
# Create a wildcard DNS record pointing to the Ingress controller
# *.preview.myapp.com → <ingress-controller-ip>

# Each PR deploys to preview.pr-123.myapp.com automatically
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "pr-{{ PR_NUMBER }}.preview.myapp.com"
```

The wildcard record handles all `*.preview.myapp.com` without ExternalDNS needing to create individual records per PR. ExternalDNS creates one record for the wildcard during cluster setup; each PR Ingress reuses it.

---

## Chapter 7: Network Policies — Zero-Trust Pod Communication

### The Default: An Open Network

By default, every pod in a Kubernetes cluster can communicate with every other pod — regardless of namespace, team, or environment. The dev database is reachable from the production API. A compromised pod can probe every other service in the cluster. There is no segmentation.

Think of it as a corporate office where every desk, server room, and filing cabinet is accessible to every employee with no locks, no keycard readers, and no audit log. A single compromised laptop or negligent employee has access to everything.

Network Policies are Kubernetes firewall rules. They control which pods can communicate with which other pods, on which ports, in which direction (ingress/egress). With a default-deny policy in place, communication is blocked unless explicitly permitted — zero trust.

### CNI Requirement — Policies Only Work If Your CNI Supports Them

Network Policies are enforced by the CNI (Container Network Interface) plugin — the component that handles pod networking. **Policies have no effect if your CNI doesn't support them.** They will be accepted by Kubernetes without error but silently ignored.

CNIs that enforce Network Policies: **Calico** (most common), **Cilium** (eBPF-based, excellent performance), **Weave Net**, **Antrea**.

CNIs that do NOT enforce Network Policies: **Flannel**, **kubenet**, and **kindnet** — kind's own default CNI, which handles basic pod networking but does not enforce Network Policies at all.

Check your CNI:
```bash
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel|kindnet"
```

For kind with Network Policy support, you need to disable the default CNI at cluster-creation time (this can't be changed on a running cluster, same as the ingress port mappings in Part 1) and install a policy-enforcing CNI yourself:

```yaml
# kind-config-netpol.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: hello-app-netpol
networking:
  disableDefaultCNI: true   # Skip kindnet — we're installing Calico instead
```

```bash
kind create cluster --config kind-config-netpol.yaml

# Install Calico (Tigera's operator-based install works well with kind)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml

# Wait for Calico to be ready before applying any Network Policies
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n calico-system --timeout=120s
```

This is a separate cluster from the one we created in Part 1, since disabling the default CNI is only meaningful at creation time and isn't something the ingress-focused cluster from Part 1 was configured for. In a real project you'd decide on your CNI and ingress requirements together, upfront, in one `kind-config.yaml`.

### Default Deny-All — The Foundation

Start by blocking all traffic in the namespace, then explicitly allow only what is needed:

```yaml
# default-deny.yaml
# Apply this to every namespace that runs application workloads
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # {} matches ALL pods in this namespace
  policyTypes:
  - Ingress          # Block all incoming traffic to pods
  - Egress           # Block all outgoing traffic from pods
  # No ingress/egress rules = deny everything
```

After applying this, all pods in `production` are completely isolated. Nothing reaches them and they reach nothing. This is the baseline you build on.

### Allow DNS — Always Required

The first thing to add back after a default-deny is DNS. Without DNS, pods cannot resolve any hostname — including Service names. Everything breaks immediately.

```yaml
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}    # All pods need DNS
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53       # DNS uses UDP port 53
    - protocol: TCP
      port: 53       # Some DNS queries use TCP (large responses)
    # No 'to:' selector = allow DNS to anywhere
    # In practice this reaches kube-dns in kube-system namespace
```

This is the one policy you must always apply alongside default-deny. Forgetting it is the most common mistake when setting up network policies for the first time — pods appear to be running but fail to make any connections.

### Allow Ingress Controller to Reach Application Pods

```yaml
# allow-ingress-controller.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp     # Apply this policy to myapp pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx   # From ingress-nginx namespace
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx         # Specifically the controller pods
    ports:
    - protocol: TCP
      port: 8080
```

### Allow Application to Reach Database

```yaml
# allow-app-to-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-postgres
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres    # Applied to postgres pods (controls who reaches them)
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp   # Only myapp pods can reach postgres
    ports:
    - protocol: TCP
      port: 5432
```

### Allow Monitoring to Scrape Metrics

```yaml
# allow-monitoring.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}    # All pods — Prometheus scrapes everything
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - protocol: TCP
      port: 8080     # Your app's metrics port (usually same as app port for /actuator/prometheus)
```

### Allow App Egress to External HTTPS APIs

```yaml
# allow-external-https.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-https
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: TCP
      port: 443      # Allow HTTPS to anywhere
  - ports:
    - protocol: TCP
      port: 80       # Allow HTTP (for Let's Encrypt HTTP-01 challenges)
```

### Environment Isolation — Dev Cannot Reach Production

Without this, a pod in the `dev` namespace can directly query `postgres.production.svc.cluster.local`:

```yaml
# allow-same-namespace-only.yaml
# Applied to the production namespace — only pods within production can talk to each other
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}    # Any pod in the same namespace (no namespaceSelector = same namespace)
```

Combined with the default-deny policy, this ensures only pods within `production` can reach other pods in `production`.

### Testing Network Policies — netshoot

`netshoot` is a Swiss Army knife container with networking tools pre-installed (curl, dig, nmap, netcat, tcpdump, etc.) — perfect for debugging network policies:

```bash
# Spin up a debug pod in the production namespace
kubectl run netshoot --image=nicolaka/netshoot -it --rm \
  --restart=Never -n production -- /bin/bash

# Test if DNS works
dig kubernetes.default.svc.cluster.local

# Test if the app is reachable from another production pod
curl -s http://myapp-service:8080/actuator/health

# Test if production can reach dev (should fail with default-deny + namespace isolation)
curl -s --connect-timeout 5 http://myapp-service.dev.svc.cluster.local:8080/health
# Expected: connection timed out — policy is working

# Test if app can reach database
nc -zv postgres-service 5432
# Expected: open — policy allows it

# Test if app can reach external HTTPS
curl -s https://api.stripe.com
# Expected: response — policy allows external HTTPS
```


## Chapter 8: Service Mesh with Istio (Advanced)

### What a Service Mesh Is — and What Problem It Solves

As microservice counts grow, a new class of problems emerges. Every service-to-service call now needs:

- **Mutual TLS:** Every call between services should be encrypted. But managing TLS certificates for 50 services and rotating them regularly is operationally painful.
- **Retries and timeouts:** If the orders service calls the inventory service and gets a 503, should it retry? How many times? With what backoff? Putting this logic in every service is duplication.
- **Circuit breaking:** If the payment service is slow, should the checkout service keep waiting? Or cut the connection after 2 seconds and return a degraded response?
- **Observability:** Which service-to-service calls are slow? Which are failing? You need traces that span multiple services.
- **Traffic splitting:** Route 10% of calls to `inventory-service-v2` for canary testing — across all callers, not just one.

These are cross-cutting concerns. A service mesh handles all of them uniformly, at the infrastructure level, without requiring changes to application code.

### When to Use Istio — and When Not To

Istio is powerful but carries significant operational overhead. Be honest about your situation before installing it:

| Use Istio when | Don't use Istio when |
|---------------|---------------------|
| You have 10+ microservices with complex inter-service communication | You have a monolith or 2–3 services |
| You need uniform mTLS enforcement across all services | Basic TLS at the edge (cert-manager) is sufficient |
| You need traffic splitting across all callers simultaneously | A single Ingress canary annotation is enough |
| Your team has bandwidth to operate it | Your team is already stretched |
| You have dedicated platform/SRE engineers | Everyone is a developer, nobody owns the mesh |

Installing Istio to solve a problem you don't have yet is a common mistake. It adds complexity that affects every team working on the cluster.

### Installing Istio

```bash
# Download istioctl
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-*/bin:$PATH

# Install with the demo profile (for learning/testing — not for production)
# Profiles: demo (full features, not production-tuned), default (production baseline), minimal (control plane only)
istioctl install --set profile=demo -y

# Verify installation
kubectl get pods -n istio-system
# NAME                                   READY   STATUS
# istiod-7d9c4b8f6d-xj8mk               1/1     Running   ← control plane
# istio-ingressgateway-5c7b8d9f6-2qmpz  1/1     Running   ← ingress gateway
# istio-egressgateway-6b9c4f8d7-mklpq   1/1     Running   ← egress gateway

istioctl verify-install
```

### Sidecar Injection — Transparent Traffic Interception

Istio works by injecting a sidecar proxy container (Envoy) into every pod. The sidecar intercepts all network traffic to and from the pod — enforcing policies, collecting metrics, and enabling Istio features — without any changes to the application code.

Enable sidecar injection per namespace:

```bash
# Label the namespace — Istio automatically injects the sidecar into new pods
kubectl label namespace production istio-injection=enabled

# Restart existing pods to pick up the sidecar
kubectl rollout restart deployment -n production

# Verify: pods now show 2/2 containers (app + sidecar)
kubectl get pods -n production
# NAME                    READY   STATUS
# myapp-7d9b4f8-xj8mk    2/2     Running   ← app + istio-proxy sidecar
```

### Gateway + VirtualService — Istio's Routing

With Istio, you use `Gateway` and `VirtualService` instead of Kubernetes Ingress:

```yaml
# Gateway: the entry point — binds to the Istio ingress gateway
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: myapp-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway   # Target the Istio ingress gateway pods
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-myapp-com-tls   # Kubernetes Secret with TLS cert
    hosts:
    - api.myapp.com
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - api.myapp.com
    tls:
      httpsRedirect: true   # Redirect HTTP to HTTPS
```

```yaml
# VirtualService: routing rules — how traffic is distributed
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: myapp-vs
  namespace: production
spec:
  hosts:
  - api.myapp.com
  gateways:
  - myapp-gateway
  http:
  - match:
    - uri:
        prefix: /api/users
    route:
    - destination:
        host: users-service
        port:
          number: 8080
  - match:
    - uri:
        prefix: /api/orders
    route:
    - destination:
        host: orders-service
        port:
          number: 8080
```

### Traffic Splitting for Canary Deployments

The killer feature: shift traffic by percentage across all callers simultaneously, without any caller knowing or caring:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: orders-vs
  namespace: production
spec:
  hosts:
  - orders-service    # All callers of orders-service are affected
  http:
  - route:
    - destination:
        host: orders-service
        subset: v1    # Stable version
      weight: 90      # 90% of traffic
    - destination:
        host: orders-service
        subset: v2    # Canary version
      weight: 10      # 10% of traffic
```

```yaml
# DestinationRule defines the v1 and v2 subsets
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: orders-dr
  namespace: production
spec:
  host: orders-service
  subsets:
  - name: v1
    labels:
      version: v1   # Pods with this label receive v1 traffic
  - name: v2
    labels:
      version: v2   # Pods with this label receive v2 traffic
```

### mTLS — Automatic Mutual TLS Between Services

With Istio, you can require that all service-to-service communication uses mutual TLS — both sides authenticate each other. No certificate management needed; Istio handles it automatically via its own CA:

```yaml
# Enforce STRICT mTLS for all services in the namespace
# STRICT: reject plaintext connections — all callers must use mTLS
# PERMISSIVE: accept both mTLS and plaintext (for migration)
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
```

With this in place, any service not in the mesh (not injected with a sidecar) cannot communicate with mesh services. This is powerful for zero-trust environments and eliminates the need for application-level authentication between services.

### Fault Injection — Testing Resilience

Inject artificial failures into service calls to test how your application behaves under adverse conditions — without needing the actual downstream service to fail:

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: inventory-fault-test
spec:
  hosts:
  - inventory-service
  http:
  - fault:
      delay:
        percentage:
          value: 50.0     # Inject 2-second delay into 50% of calls
        fixedDelay: 2s
      abort:
        percentage:
          value: 10.0     # Return HTTP 503 for 10% of calls
        httpStatus: 503
    route:
    - destination:
        host: inventory-service
        port:
          number: 8080
```

Apply this during load testing and observe whether your checkout service handles slow inventory responses gracefully (timeouts, circuit breaking) or cascades failures.

### Retry Policies and Circuit Breakers

```yaml
# Retry policy in VirtualService
http:
- route:
  - destination:
      host: payment-service
  retries:
    attempts: 3           # Retry up to 3 times
    perTryTimeout: 2s     # Each attempt has a 2-second timeout
    retryOn: 5xx,gateway-error,reset   # Retry conditions
```

```yaml
# Circuit breaker in DestinationRule
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-cb
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 5       # Open circuit after 5 consecutive errors
      interval: 30s                 # Evaluation window
      baseEjectionTime: 30s         # Eject unhealthy host for 30 seconds
      maxEjectionPercent: 100       # Eject up to 100% of instances if needed
```

---

## Chapter 9: Complete Production Example

### The Goal

A three-tier application: frontend, API, and database. Fully TLS-terminated, automatic DNS, security headers, rate limiting, network policies enforcing zero-trust between tiers.

### Complete Production Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production-ingress
  namespace: production
  annotations:
    # TLS automation
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains; preload";
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
      more_set_headers "Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'";

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/limit-connections: "50"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://www.myapp.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type, X-Request-ID"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"

    # File uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # ExternalDNS
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    external-dns.alpha.kubernetes.io/ttl: "300"

spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: api-myapp-com-tls
  - hosts:
    - www.myapp.com
    secretName: www-myapp-com-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: www.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Complete Network Policy Set for a 3-Tier App

```yaml
# 1. Default deny — applies to all pods in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]
---
# 2. Allow DNS for all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
---
# 3. Frontend receives traffic from Ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - protocol: TCP
      port: 80
---
# 4. API receives traffic from Ingress controller AND frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
---
# 5. API can call the database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - protocol: TCP
      port: 5432
---
# 6. API egress: to database + Redis + external HTTPS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes: [Egress]
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          tier: cache
    ports:
    - protocol: TCP
      port: 6379
  - ports:   # External HTTPS (payment APIs, email services, etc.)
    - protocol: TCP
      port: 443
---
# 7. Allow monitoring to scrape all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app.kubernetes.io/name: prometheus
    ports:
    - protocol: TCP
      port: 8080
```

### Helm Values for Networking

```yaml
# values-prod.yaml (networking section)
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "20"
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

networkPolicy:
  enabled: true
  # Templates in templates/networkpolicy.yaml read these values
  allowedNamespaces:
    - ingress-nginx
    - monitoring
```

---

## Troubleshooting

| Symptom | Likely Cause | Diagnostic Command | Fix |
|---------|-------------|-------------------|-----|
| Ingress returns 404 for all paths | No Ingress controller installed, or wrong `ingressClassName` | `kubectl get ingressclass` — if empty, no controller | Install nginx-ingress; set `ingressClassName: nginx` |
| Ingress returns 404 for specific path only | Wrong `pathType` — using `Exact` when `Prefix` needed | `kubectl describe ingress <n>` — check rules | Change `pathType: Exact` to `pathType: Prefix`; test with `curl -v` |
| Path rewriting strips too much / too little | Regex capture groups misconfigured | Test with `curl -v` and check nginx access logs | Run `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` and trace the rewritten path |
| Certificate stuck in `False` / never issues | Port 80 not reachable from internet, or wrong `ingressClassName` on ClusterIssuer | `kubectl describe certificate <n>` → Events; `kubectl describe challenge <n>` | Ensure port 80 is open; security groups/firewall must allow port 80 from internet |
| Certificate issued by "Fake LE" (not trusted) | Using staging issuer in production | `kubectl get certificate -n <ns> -o yaml` → check issuer ref | Change annotation to `letsencrypt-prod`; delete and re-create the certificate Secret |
| ExternalDNS not creating records | Missing IAM permissions, wrong domain filter, or wrong owner ID | `kubectl logs -n external-dns deploy/external-dns` | Check IAM policy allows `route53:ChangeResourceRecordSets`; verify `domainFilters` matches your zone |
| Network policy blocks traffic unexpectedly | Missing DNS allow policy, or overly restrictive selector | `kubectl exec -it netshoot -- dig <svc>` — if DNS fails, DNS policy is missing | Add `allow-dns` policy first; verify selector labels match pod labels exactly |
| Network policy has no effect | CNI doesn't support Network Policies | `kubectl get pods -n kube-system \| grep -E "calico\|cilium"` | Reinstall cluster with Calico or Cilium CNI; Flannel does not enforce policies |
| Istio sidecar not injecting | Namespace not labeled for injection | `kubectl get ns production --show-labels` | `kubectl label namespace production istio-injection=enabled` then `kubectl rollout restart deployment -n production` |
| mTLS STRICT mode breaks connections | A service outside the mesh is trying to call a mesh service | Check Istio access logs: `kubectl logs <pod> -c istio-proxy` | Either inject the calling service into the mesh, or set `PeerAuthentication` to `PERMISSIVE` temporarily |

---

## Practice Exercises

**Exercise 1 — End-to-end Ingress with TLS:**
Deploy two services in your kind cluster — `api-service` and `web-service`. Create an Ingress that routes `api.127.0.0.1.nip.io/api` to `api-service` and `www.127.0.0.1.nip.io` to `web-service`. Install cert-manager and create a staging ClusterIssuer. Add the `cert-manager.io/cluster-issuer` annotation and watch the certificate issue. Verify with `kubectl get certificate` and `curl -k https://api.127.0.0.1.nip.io/api/hello` (`-k` to skip verification for staging cert).

**Exercise 2 — Path rewriting:**
Deploy a simple service that only knows about paths starting with `/users` (no `/api` prefix). Create an Ingress that accepts requests at `/api/users/.*` and rewrites them to `/users/.*` before forwarding. Test with `curl api.myapp.com/api/users/123` and verify the backend receives `GET /users/123`. Check the nginx controller logs to see the rewritten path.

**Exercise 3 — Network policies from scratch:**
In a test namespace, apply a default-deny-all policy. Then deploy two pods: `pod-a` and `pod-b`. Verify they can't reach each other (`kubectl exec pod-a -- curl pod-b`). Add a NetworkPolicy that allows `pod-a` to call `pod-b` on port 8080 but not the reverse. Verify unidirectional communication. Add the DNS allow policy and verify that Service names resolve. Use `netshoot` to debug at each step.

**Exercise 4 — ExternalDNS with a real domain:**
If you have a domain in Route53 or Cloudflare, install ExternalDNS in your cluster. Create an Ingress with a hostname in your domain. Watch `kubectl logs -n external-dns deploy/external-dns` as it creates the DNS record. Verify with `dig api.yourdomain.com`. Delete the Ingress and observe the record being cleaned up (`policy=sync` required).

**Exercise 5 — Istio traffic splitting:**
Install Istio in your kind cluster. Deploy two versions of the same service — `v1` and `v2` with different response text. Create a `VirtualService` that sends 90% of traffic to `v1` and 10% to `v2`. Run `for i in $(seq 1 100); do curl -s http://api.myapp.com/hello; done | sort | uniq -c` and verify the ratio is approximately 90/10. Gradually shift weight to 50/50, then 0/100. Practice the Istio canary promotion workflow.
