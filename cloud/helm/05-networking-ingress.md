# Part 5: Networking & Ingress

## What This Covers
- Service types and when to use each
- Ingress controller — one load balancer for all services
- TLS with cert-manager (automatic certificates)
- DNS with ExternalDNS (automatic DNS records)
- Network policies (zero-trust)

---

## Chapter 1: Service Types

### The Networking Layers

```
Internet
    │
    ▼
Ingress Controller     ← Layer 7: routes by hostname/path, handles TLS
    │
    ▼
Service                ← Layer 4: stable IP/DNS, load balances across pods
    │
    ▼
Pods                   ← Your application
```

### Service Types Compared

| Type | Reachable From | Use Case | Cost |
|------|---------------|----------|------|
| `ClusterIP` | Inside cluster only | Service-to-service communication | Free |
| `NodePort` | `<node-ip>:30000-32767` | Local testing, quick demos | Free |
| `LoadBalancer` | Internet via cloud LB | One service exposed externally | $$$/service |
| + Ingress | Internet via single cloud LB | Many services, single entry point | $$ total |

**The cost problem:** LoadBalancer creates one cloud load balancer per service — $20-50/month each. For 10 services, that's $200-500/month just on load balancers. Ingress uses one load balancer for all services and routes by hostname/path.

```yaml
# ClusterIP — internal only (default)
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
# Reachable as: backend-svc.namespace.svc.cluster.local:8080
# Or within same namespace: backend-svc:8080
```

```yaml
# LoadBalancer — only use when you genuinely need one service exposed directly
apiVersion: v1
kind: Service
metadata:
  name: public-api-svc
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: public-api
  ports:
  - port: 443
    targetPort: 8080
```

---

## Chapter 2: Ingress Controller

### Install nginx-ingress

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.replicaCount=2

# Verify and get external IP
kubectl get svc -n ingress-nginx
# ingress-nginx-controller   LoadBalancer   10.0.0.10   a1b2c3d4.elb.amazonaws.com
```

For Minikube:
```bash
minikube addons enable ingress
minikube ip   # Use this IP for local testing
```

### Basic Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
  - host: www.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

### Path-Based Routing for Microservices

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2  # Strip the prefix
spec:
  rules:
  - host: api.myapp.com
    http:
      paths:
      # /users/anything → users-svc receives /anything
      - path: /users(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: users-svc
            port:
              number: 8080
      # /orders/anything → orders-svc
      - path: /orders(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: orders-svc
            port:
              number: 8080
```

### Useful nginx Annotations

```yaml
metadata:
  annotations:
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"

    # Request size (default is 1MB — increase for file uploads)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"

    # Timeouts (seconds)
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # Security headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Strict-Transport-Security: max-age=31536000";
```

---

## Chapter 3: TLS with cert-manager

### What cert-manager Does

Normally: manually buy/generate a TLS certificate → base64 encode it → create K8s Secret → update Ingress → repeat every 90 days.

With cert-manager: annotate your Ingress and it handles everything — requests the cert from Let's Encrypt, stores it as a Secret, renews it automatically before expiry.

### Install

```bash
helm repo add jetstack https://charts.jetstack.io && helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Create a ClusterIssuer

A ClusterIssuer tells cert-manager where to get certificates from (Let's Encrypt in this case):

```yaml
# clusterissuer-prod.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Also create a staging issuer for testing** — Let's Encrypt production has rate limits, staging doesn't:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@myapp.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: nginx
```

### TLS-Enabled Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # ← This triggers cert-manager
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls         # cert-manager creates this Secret
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
```

```bash
# Check certificate status
kubectl get certificate -n production
kubectl describe certificate myapp-tls -n production

# Status should show:
# Conditions:
#   Type    Status  Message
#   Ready   True    Certificate is up to date and has not expired
```

---

## Chapter 4: ExternalDNS

ExternalDNS watches Ingress resources and creates DNS records at your cloud provider automatically. No more manually updating Route53/CloudDNS after every deployment.

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
helm install external-dns external-dns/external-dns \
  --namespace external-dns \
  --create-namespace \
  --set provider=aws \
  --set domainFilters[0]=myapp.com \
  --set policy=sync
```

Annotate your Ingress to opt in:
```yaml
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.myapp.com
    external-dns.alpha.kubernetes.io/ttl: "300"
```

ExternalDNS then creates: `api.myapp.com A <loadbalancer-ip>` and updates it if the IP changes.

---

## Chapter 5: Network Policies

### Default Behavior (the problem)

Without network policies, every pod can talk to every other pod. A compromised pod in dev can query your production database.

### Default Deny All (start here)

```yaml
# Apply this to every production namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # {} means "all pods"
  policyTypes:
  - Ingress
  - Egress
  # No rules = deny everything
```

Then selectively allow what you need:

```yaml
# Allow ingress-nginx to reach app pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - port: 8080

---
# Allow app to reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - port: 5432

---
# Always allow DNS (pods need this for any name resolution)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
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
    - protocol: UDP
      port: 53
```

---

## Quick Reference

### Ingress Debugging

```bash
# Check ingress is created
kubectl get ingress -n production

# Check ingress controller pods
kubectl get pods -n ingress-nginx

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check certificate
kubectl get certificate -n production
kubectl describe certificate myapp-tls -n production
```

### Common Networking Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| 404 from ingress | Path not matching | Check `pathType`, test with exact path |
| 502 Bad Gateway | Backend pod not ready | Check pod readiness probe |
| SSL cert not issuing | cert-manager can't reach Let's Encrypt | Check cert-manager logs, port 80 must be open |
| DNS not updating | ExternalDNS IAM issue | Check ExternalDNS logs, verify IAM permissions |
| Pods can't talk after network policy | Policy too restrictive | Temporarily remove policy, add rules incrementally |
