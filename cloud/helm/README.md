# Kubernetes + Helm Reference

A personal reference for deploying Spring Boot applications to Kubernetes using Helm.

## Files

| File | Content |
|------|---------|
| `01-spring-boot-docker-helm-basics.md` | Spring Boot → Docker → K8s → Helm end-to-end |
| `02-kubernetes-core-concepts.md` | Namespaces, ConfigMaps, Secrets, Resource Limits, Probes |
| `03-helm-deep-dive.md` | Go templates, named templates, subcharts, hooks |
| `04-container-registry-and-ci.md` | Registries, GitHub Actions pipeline, GitLab CI |
| `05-networking-ingress.md` | Services, Ingress, TLS (cert-manager), DNS, Network Policies |
| `06-storage-stateful-apps.md` | PV/PVC, StorageClasses, StatefulSets, Backups |
| `07-10-security-observability-scaling-production.md` | RBAC, Security, Prometheus/Grafana/Loki, HPA/VPA, Production playbook |

---

## Core Mental Model

```
Code (Spring Boot)
    → packaged as JAR
    → baked into Docker image
    → pushed to container registry
    → deployed to Kubernetes via Helm chart
    → exposed via Ingress
    → monitored by Prometheus/Grafana
    → scales automatically via HPA
```

---

## Most-Used Commands

### Helm

```bash
helm lint ./chart                                    # Check syntax
helm template release ./chart -f values-prod.yaml   # Preview rendered YAML
helm upgrade --install myapp ./chart -f values.yaml -n prod --atomic
helm history myapp -n prod
helm rollback myapp -n prod                          # Roll back to previous
```

### Kubernetes

```bash
kubectl get all -n production                        # Everything in namespace
kubectl logs -f deployment/myapp -n production       # Stream logs
kubectl describe pod <n> -n production              # Details + events
kubectl exec -it <pod> -n production -- /bin/sh     # Shell into pod
kubectl top pods -n production                       # CPU/memory usage
kubectl rollout status deployment/myapp -n prod      # Wait for rollout
kubectl rollout undo deployment/myapp -n prod        # Emergency rollback
kubectl port-forward svc/myapp 8080:8080 -n prod    # Local access
```

### Debugging

```bash
# Pod not starting
kubectl describe pod <n> -n production   # Check Events section
kubectl logs <n> -n production --previous  # Previous container crash logs

# Service not routing
kubectl get endpoints <svc> -n production  # Should show pod IPs
kubectl get pods -n production --show-labels  # Check labels match selector

# Image not pulling
kubectl describe pod <n> -n production   # Check "Failed to pull image"
kubectl get secret regcred -n production  # Check pull secret exists

# Certificate not issuing
kubectl describe certificate myapp-tls -n production
kubectl logs -n cert-manager deploy/cert-manager
```

---

## Key Decisions Reference

### Which Service Type?

- Internal only (pod to pod) → `ClusterIP`
- Multiple services on one domain → `ClusterIP` + `Ingress`
- Single service needs direct external access → `LoadBalancer`
- Local testing → `NodePort` or `port-forward`

### Which Storage?

- Stateless app (API, web) → No storage needed
- Single database instance → `StatefulSet` + `PVC`
- Multiple pods need shared files → `PVC` with `ReadWriteMany` (needs NFS)
- Temporary scratch space → `emptyDir` volume

### HPA vs VPA?

- Variable request load → `HPA` (scales replicas)
- Unsure of right resource requests → `VPA` in "Off" mode (recommendations only)
- Both → HPA on custom metric (not CPU), VPA on memory only

### Secrets Management?

- Local/dev → `kubectl create secret` is fine
- Production → External Secrets Operator (AWS SM) or Sealed Secrets (GitOps)
- Never → Plain Secret YAML in Git
