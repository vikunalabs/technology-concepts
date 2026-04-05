## Part 5: Networking & Ingress - Making Your App Accessible

### Prerequisites
- Completed Parts 1-4 (or equivalent experience)
- A Kubernetes cluster (EKS, GKE, AKS, or Minikube)
- Domain name (optional for production, but we'll use nip.io for testing)
- Basic understanding of DNS and TLS

### What You'll Learn
- ✅ Ingress controllers and why you need them
- ✅ Exposing services with different Service types
- ✅ Advanced routing (path-based, host-based)
- ✅ TLS/SSL certificates with cert-manager (automatic!)
- ✅ DNS management with ExternalDNS
- ✅ Network policies for security
- ✅ Load balancing strategies
- ✅ Service mesh basics (Istio)

---

## Chapter 1: Kubernetes Networking Basics

### The Three Layers of Kubernetes Networking

```
┌─────────────────────────────────────────────────────────────┐
│                    Internet (External Users)                │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Ingress Controller (Layer 7)                   │
│         Routes based on hostname, path, headers             │
│         Handles TLS termination, rate limiting              │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│              Service (Layer 4)                              │
│         Stable internal IP, load balancing                  │
│         Service types: ClusterIP, NodePort, LoadBalancer    │
└─────────────────────────┬───────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│                    Pods (Your App)                          │
│         Ephemeral IPs, auto-scaling                         │
└─────────────────────────────────────────────────────────────┘
```

### Service Types Explained

| Service Type | Use Case | Accessibility | Cost | Analogy |
|--------------|----------|---------------|------|---------|
| **ClusterIP** | Internal services | Inside cluster only | Free | Internal office phone |
| **NodePort** | Simple external access | Node IP:30000-32767 | Free | Reception desk phone |
| **LoadBalancer** | Production external | Cloud load balancer | $$$ | Company main phone line |
| **Ingress** | Advanced routing | HTTP/HTTPS routing | $ | Switchboard operator |

### Service Type Examples

**ClusterIP (Internal only):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP  # Default
  selector:
    app: backend
  ports:
    - port: 8080
      targetPort: 8080
# Access: backend-service.default.svc.cluster.local:8080
```

**NodePort (Simple external):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # 30000-32767
# Access: http://NODE_IP:30080
```

**LoadBalancer (Cloud native):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # Network LB
    service.beta.kubernetes.io/aws-load-balancer-internal: "false"
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
  # Cloud provider creates external LB automatically
# Access: http://a1b2c3d4e5f6.elb.amazonaws.com
```

---

## Chapter 2: Ingress Controllers - The Smart Router

### Why Ingress?

**The Problem without Ingress:**
```yaml
# One LoadBalancer per service = $$$$$
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer  # $20/month for each!
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer  # Another $20/month!
---
apiVersion: v1
kind: Service
metadata:
  name: admin-service
spec:
  type: LoadBalancer  # And another!
```

**The Solution with Ingress:**
```yaml
# Single LoadBalancer for all services
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: api-service
    - host: www.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: web-service
    - host: admin.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: admin-service
```

### Ingress Controllers Comparison

| Controller | Pros | Cons | Best For |
|------------|------|------|----------|
| **nginx-ingress** | Most popular, battle-tested | Basic features only | General purpose |
| **Traefik** | Automatic config, dashboard | Less mature than nginx | Microservices |
| **AWS ALB Ingress** | Native AWS integration | AWS-specific, slower | AWS shops |
| **GCE Ingress** | Native GCP integration | GCP-specific | GCP shops |
| **Contour** | Envoy-based, high performance | Complex config | High traffic |
| **Istio Gateway** | Service mesh integration | Overkill for simple apps | Service mesh users |

### Installing Ingress Controller

**Option 1: nginx-ingress (Most Common)**
```bash
# Using Helm (recommended)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install with custom values
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.enabled=true \
  --set controller.metrics.enabled=true \
  --set controller.service.type=LoadBalancer

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
# Output:
# ingress-nginx-controller   LoadBalancer   10.0.0.10   a1b2c3d4.elb.amazonaws.com   80:32258/TCP,443:30242/TCP
```

**Option 2: Minikube (Local Development)**
```bash
# Enable ingress addon
minikube addons enable ingress

# Check status
minikube addons list | grep ingress
kubectl get pods -n ingress-nginx

# Get IP for local testing
minikube ip
# Access via: http://$(minikube ip)/your-path
```

**Option 3: AWS ALB Ingress Controller**
```bash
# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
```

---

## Chapter 3: Advanced Ingress Rules

### Basic Ingress Example

```yaml
# ingress-basic.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    # nginx-specific annotations
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-service
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 8080
    - host: www.myapp.com
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

### Advanced Routing with Annotations

```yaml
# ingress-advanced.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-whitelist: "192.168.1.0/24"
    
    # Authentication
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
    
    # Proxy settings
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # WebSocket support
    nginx.ingress.kubernetes.io/websocket-services: "websocket-service"
    
    # Custom headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /api/?
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /ws/?
            pathType: Prefix
            backend:
              service:
                name: websocket-service
                port:
                  number: 8081
```

### Path-Based Routing for Microservices

```yaml
# ingress-microservices.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2  # Strip prefix
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          # Route /users/* to users-service
          - path: /users(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: users-service
                port:
                  number: 8080
          
          # Route /orders/* to orders-service
          - path: /orders(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: orders-service
                port:
                  number: 8080
          
          # Route /payments/* to payments-service
          - path: /payments(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: payments-service
                port:
                  number: 8080
          
          # Default fallback
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-gateway
                port:
                  number: 8080
```

### Header-Based Routing

```yaml
# ingress-header-routing.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    # Canary deployment with header
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"  # 10% traffic
    
    # Sticky sessions
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "route"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: api-service-v2  # Canary version
                port:
                  number: 8080
          - path: /
            backend:
              service:
                name: api-service-v1  # Stable version
                port:
                  number: 8080
```

---

## Chapter 4: TLS/SSL with cert-manager

### The Problem

```yaml
# Without cert-manager, you'd manually:
# 1. Buy SSL certificate ($50-500/year)
# 2. Convert to Kubernetes secret
# 3. Update Ingress
# 4. Repeat every 90 days (Let's Encrypt) or 1 year (paid)

# Manual secret creation:
kubectl create secret tls myapp-tls \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

### The Solution: cert-manager

**cert-manager automates:**
- Certificate issuance (Let's Encrypt, self-signed, etc.)
- Automatic renewal (90-day certificates renew at 60 days)
- Secret management (automatically updates secrets)

### Installing cert-manager

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true \
  --set prometheus.enabled=true

# Verify installation
kubectl get pods -n cert-manager
# cert-manager-xxx       1/1     Running
# cert-manager-cainjector-xxx   1/1     Running
# cert-manager-webhook-xxx      1/1     Running
```

### Setting Up Issuers

**ClusterIssuer (works across all namespaces):**
```yaml
# cluster-issuer-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Production Let's Encrypt (rate limited)
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
            podTemplate:
              spec:
                nodeSelector:
                  kubernetes.io/os: linux
```

**Staging Issuer (for testing):**
```yaml
# cluster-issuer-staging.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Staging (no rate limits, fake certificates)
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

### Requesting Certificates

**Method 1: Annotations on Ingress (Easiest)**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
spec:
  tls:
    - hosts:
        - api.myapp.com
        - www.myapp.com
      secretName: myapp-tls  # cert-manager creates this
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: api-service
                port:
                  number: 8080
    - host: www.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

**Method 2: Certificate Resource (More Control)**
```yaml
# certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-cert
  namespace: production
spec:
  secretName: myapp-tls
  duration: 2160h  # 90 days
  renewBefore: 360h  # 15 days before expiry
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: api.myapp.com
  dnsNames:
    - api.myapp.com
    - www.myapp.com
    - admin.myapp.com
  privateKey:
    algorithm: RSA
    size: 2048
```

### Complete TLS Example with Spring Boot

**`values-prod.yaml` with TLS:**
```yaml
# Helm values for production with TLS
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
  
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com
        - www.myapp.com

# For local testing without domain
# Use nip.io for dynamic DNS
# api.192.168.1.100.nip.io
```

**Deploy with TLS:**
```bash
# 1. Install cert-manager
helm install cert-manager jetstack/cert-manager --set installCRDs=true

# 2. Create ClusterIssuer
kubectl apply -f cluster-issuer-prod.yaml

# 3. Deploy app with TLS-enabled Ingress
helm upgrade --install myapp ./helm-chart \
  -f values-prod.yaml \
  --set ingress.tls.enabled=true

# 4. Check certificate status
kubectl get certificate -A
kubectl describe certificate myapp-cert

# 5. Verify TLS
curl -v https://api.myapp.com
# Should show valid certificate
```

---

## Chapter 5: DNS Management with ExternalDNS

### The Problem

Without ExternalDNS, you manually:
```bash
# 1. Get load balancer IP
kubectl get svc -n ingress-nginx

# 2. Go to DNS provider (Route53, CloudFlare, etc.)
# 3. Create/update A record
# 4. Wait for propagation
# 5. Repeat for every environment
```

### The Solution: ExternalDNS

**ExternalDNS automatically:**
- Watches Ingress resources
- Creates DNS records at your provider
- Updates records when IPs change
- Deletes old records automatically

### Installing ExternalDNS

**AWS Route53 Example:**
```yaml
# external-dns-values.yaml
provider: aws
aws:
  region: us-east-1
  zoneType: public  # or private for internal zones

domainFilters:
  - myapp.com  # Only manage this domain

policy: sync  # or upsert-only to avoid deletions

sources:
  - ingress
  - service

txtOwnerId: my-cluster
```

```bash
# Install ExternalDNS
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  -f external-dns-values.yaml

# Create IAM policy for AWS
cat > external-dns-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": ["arn:aws:route53:::hostedzone/*"]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": ["*"]
    }
  ]
}
EOF

# Attach policy to node role
aws iam put-role-policy \
  --role-name eks-node-role \
  --policy-name external-dns \
  --policy-document file://external-dns-policy.json
```

### How ExternalDNS Works

```yaml
# When you create this Ingress:
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    external-dns.alpha.kubernetes.io/ttl: "300"  # 5 minutes
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: api-service

# ExternalDNS automatically:
# 1. Gets LoadBalancer IP: 54.123.45.67
# 2. Creates Route53 record: api.myapp.com A 54.123.45.67
# 3. Updates on changes
# 4. Deletes when Ingress is removed
```

### Multiple DNS Providers

**CloudFlare:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: external-dns
type: Opaque
stringData:
  api-token: YOUR_CLOUDFLARE_API_TOKEN
---
apiVersion: externaldns.k8s.io/v1alpha1
kind: ExternalDNS
metadata:
  name: cloudflare-externaldns
spec:
  provider:
    cloudflare:
      apiTokenSecretRef:
        name: cloudflare-api-token
        key: api-token
  domains:
    - myapp.com
```

**Google Cloud DNS:**
```yaml
apiVersion: externaldns.k8s.io/v1alpha1
kind: ExternalDNS
metadata:
  name: gcloud-externaldns
spec:
  provider:
    google:
      project: my-gcp-project
      serviceAccountSecretRef:
        name: gcp-sa
        key: credentials.json
```

### Wildcard DNS with ExternalDNS

```yaml
# Support wildcard domains
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wildcard-ingress
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "*.dev.myapp.com"
spec:
  rules:
    - host: "*.dev.myapp.com"
      http:
        paths:
          - path: /
            backend:
              service:
                name: dev-environment-service
# Creates: *.dev.myapp.com → LoadBalancer IP
# Now preview.myapp.com, test.myapp.com all work!
```

---

## Chapter 6: Network Policies - Zero Trust Security

### What are Network Policies?

**Analogy - Office Security:**
- **Without policies** = Open office plan (anyone can go anywhere)
- **With policies** = Secure office with keycard access (only authorized rooms)

### Default Behavior

```yaml
# By default: ALL traffic allowed (like open Wi-Fi)
# - Pods can talk to any other pod
# - External traffic can reach any service
# - No isolation between environments
```

### Implementing Network Policies

**Basic Deny All (Start with this):**
```yaml
# networkpolicy-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}  # All pods
  policyTypes:
    - Ingress
    - Egress
  # No ingress/egress rules = deny all traffic
```

**Allow Only from Frontend to Backend:**
```yaml
# networkpolicy-allow-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend  # Targets backend pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend  # Only allow from frontend
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx  # Allow from ingress
      ports:
        - protocol: TCP
          port: 8080
```

**Allow Egress to Database Only:**
```yaml
# networkpolicy-egress-db.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: app-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
    - Egress
  egress:
    # Allow DNS queries
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
    
    # Allow to database only
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    
    # Deny all other egress (implicit)
```

**Isolate Environments:**
```yaml
# networkpolicy-isolate-env.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-dev
  namespace: development
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    # Only allow from same namespace
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: development
    
    # Allow from CI/CD system
    - from:
        - namespaceSelector:
            matchLabels:
              name: gitlab-runner
```

### Testing Network Policies

```bash
# 1. Create test pods
kubectl run test-frontend --image=nginx --labels="app=frontend"
kubectl run test-backend --image=nginx --labels="app=backend"
kubectl run test-unauthorized --image=nginx

# 2. Test connectivity
kubectl exec test-frontend -- curl backend-service:8080  # Should work
kubectl exec test-unauthorized -- curl backend-service:8080  # Should fail

# 3. Check network policy status
kubectl get networkpolicies
kubectl describe networkpolicy backend-policy
```

### Network Policy Best Practices

```yaml
# 1. Default deny all (most secure)
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

# 2. Allow DNS (necessary for egress)
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - port: 53
      protocol: UDP

# 3. Allow monitoring
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - port: 9090  # Prometheus
    - port: 9102  # Metrics
```

---

## Chapter 7: Service Mesh with Istio (Optional Advanced)

### What is a Service Mesh?

**Service Mesh adds:**
- Fine-grained traffic control (canary, A/B testing)
- Automatic retries and circuit breakers
- Distributed tracing
- mTLS encryption between services
- Detailed metrics without code changes

### Installing Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install with demo profile
istioctl install --set profile=demo -y

# Enable sidecar injection
kubectl label namespace default istio-injection=enabled

# Verify
kubectl get pods -n istio-system
```

### Istio Gateway and VirtualService

```yaml
# gateway.yaml - External traffic entry
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: myapp-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "myapp.com"
        - "*.myapp.com"
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: myapp-tls
      hosts:
        - "myapp.com"

---
# virtualservice.yaml - Advanced routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
spec:
  hosts:
    - "myapp.com"
  gateways:
    - myapp-gateway
  http:
    # Canary deployment: 10% to v2
    - match:
        - headers:
            x-canary:
              exact: "true"
        - weight: 10
      route:
        - destination:
            host: app-v2-service
            port:
              number: 8080
    # 90% to v1
    - route:
        - destination:
            host: app-v1-service
            port:
              number: 8080
    
    # Fault injection for testing
    - match:
        - headers:
            x-fail:
              exact: "true"
      fault:
        delay:
          percentage:
            value: 100
          fixedDelay: 5s
      route:
        - destination:
            host: app-v1-service
    
    # Retry policy
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
```

### DestinationRule for Traffic Policies

```yaml
# destinationrule.yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp-dr
spec:
  host: app-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        http1MaxPendingRequests: 10
        http2MaxRequests: 100
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: v1
      labels:
        version: v1
    - name: v2
      labels:
        version: v2
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL  # mTLS between services
```

---

## Chapter 8: Complete Production Example

### Putting It All Together

**Complete Ingress with TLS, DNS, and Policies:**

```yaml
# complete-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: production
  annotations:
    # TLS automation
    cert-manager.io/cluster-issuer: letsencrypt-prod
    
    # DNS automation
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    external-dns.alpha.kubernetes.io/ttl: "300"
    
    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "X-XSS-Protection: 1; mode=block";
      more_set_headers "Strict-Transport-Security: max-age=31536000; includeSubDomains";
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    
    # Large file uploads
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com,https://www.myapp.com"
    
    # WAF (if using ModSecurity)
    nginx.ingress.kubernetes.io/enable-modsecurity: "true"
    nginx.ingress.kubernetes.io/modsecurity-snippet: |
      SecRuleEngine On
      SecRule ARGS "@contains cat" "id:100,phase:1,deny,status:403"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
        - www.myapp.com
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-service
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 8080
          - path: /health
            pathType: Exact
            backend:
              service:
                name: health-service
                port:
                  number: 8080
    - host: www.myapp.com
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

### Helm Chart with Full Networking

**`values-prod.yaml` for production:**
```yaml
# Complete production values
environment: production

image:
  repository: myregistry/myapp
  tag: v1.2.3

# Service configuration
service:
  type: ClusterIP  # Ingress will route to this
  port: 8080

# Ingress with all features
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
  
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com
        - www.myapp.com

# Network policies
networkPolicies:
  enabled: true
  denyAll: true  # Start with deny all
  allow:
    - fromIngress: true
    - fromMonitoring: true
    - egressToDNS: true

# TLS configuration
tls:
  enabled: true
  issuer: letsencrypt-prod
  email: admin@myapp.com

# Monitoring (for metrics)
monitoring:
  enabled: true
  prometheus:
    serviceMonitor:
      enabled: true
```

### Deploy Complete Solution

```bash
# 1. Install dependencies
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm repo update

# 2. Install Ingress Controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.metrics.enabled=true

# 3. Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set installCRDs=true

# 4. Install ExternalDNS
helm install external-dns external-dns/external-dns \
  --namespace external-dns --create-namespace \
  --set provider=aws --set domainFilters[0]=myapp.com

# 5. Create ClusterIssuer
kubectl apply -f cluster-issuer-prod.yaml

# 6. Deploy app with networking
helm upgrade --install myapp ./helm-chart \
  -f values-prod.yaml \
  --namespace production \
  --create-namespace

# 7. Verify everything
kubectl get ingress -n production
kubectl get certificate -n production
kubectl get networkpolicies -n production

# 8. Test
curl -v https://api.myapp.com/health
nslookup api.myapp.com  # Should resolve to LB IP
```

---

## Summary: Networking Checklist

| Component | Purpose | Status |
|-----------|---------|--------|
| **Service Types** | Internal/external access | ✅ |
| **Ingress Controller** | Smart routing | ✅ |
| **Path-based routing** | Microservices routing | ✅ |
| **Header-based routing** | Canary deployments | ✅ |
| **cert-manager** | Automatic TLS | ✅ |
| **ExternalDNS** | Automatic DNS | ✅ |
| **Network Policies** | Zero-trust security | ✅ |
| **Service Mesh (opt)** | Advanced traffic control | ⬜ |

## Common Networking Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| **404 Not Found** | Path not routing | Check rewrite-target annotation |
| **502 Bad Gateway** | Backend not ready | Check pod status, readiness probes |
| **Certificate errors** | TLS not working | Check cert-manager logs |
| **DNS not updating** | No A record | Check ExternalDNS logs, IAM role |
| **Network policy blocking** | Connection refused | Temporarily remove policies to test |
| **Ingress not creating LB** | Pending forever | Check cloud provider permissions |

## Practice Exercises

### Exercise 1: Deploy with Ingress
Deploy your Spring Boot app with nginx-ingress and test routing.

### Exercise 2: TLS with cert-manager
Set up Let's Encrypt staging issuer and secure your app.

### Exercise 3: Path-Based Routing
Create multiple services and route /api to one, /web to another.

### Exercise 4: Network Policies
Implement default deny and selectively allow traffic.

### Exercise 5: ExternalDNS
Set up automatic DNS management (requires domain and cloud provider).

## Next Steps

After mastering networking, you're ready for:
- **Part 6: Storage & Stateful Apps** - Databases, PV/PVC, StatefulSets
- **Part 7: Security** - RBAC, PodSecurity, OPA policies
- **Part 8: Observability** - Prometheus, Grafana, distributed tracing

---