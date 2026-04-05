

kubernetes-hello-world-series/
├── README.md                           # Overview, prerequisites, navigation
├── 01-spring-boot-docker-helm-basics/  # Current document (refined)
│   ├── README.md
│   ├── code/
│   │   ├── spring-boot-app/
│   │   ├── Dockerfile
│   │   └── helm-chart/
│   └── exercises.md
│
├── 02-kubernetes-core-concepts/        # NEW - Let's create this first
│   ├── README.md
│   ├── code/
│   │   ├── namespaces/
│   │   ├── configmaps/
│   │   ├── secrets/
│   │   └── probes/
│   ├── exercises.md
│   └── cheatsheet.md
│
├── 03-helm-deep-dive/
│   ├── README.md
│   ├── code/
│   │   ├── custom-templates/
│   │   ├── subcharts/
│   │   └── hooks/
│   └── exercises.md
│
├── 04-container-registry-and-ci/
│   ├── README.md                       # Docker Hub, ECR, GCR, ACR
│   ├── code/
│   │   ├── github-actions/
│   │   └── gitlab-ci/
│   └── exercises.md
│
├── 05-networking-ingress/
│   ├── README.md
│   ├── code/
│   │   ├── nginx-ingress/
│   │   ├── tls/
│   │   └── dns/
│   └── exercises.md
│
├── 06-storage-stateful-apps/
│   ├── README.md
│   ├── code/
│   │   ├── pv-pvc/
│   │   ├── statefulsets/
│   │   └── postgres-operator/
│   └── exercises.md
│
├── 07-security/
│   ├── README.md
│   ├── code/
│   │   ├── rbac/
│   │   ├── network-policies/
│   │   └── vault-integration/
│   └── exercises.md
│
├── 08-observability/
│   ├── README.md
│   ├── code/
│   │   ├── prometheus-grafana/
│   │   ├── loki/
│   │   └── opentelemetry/
│   └── exercises.md
│
├── 09-scaling-resilience/
│   ├── README.md
│   ├── code/
│   │   ├── hpa/
│   │   ├── pdb/
│   │   └── canary-deployments/
│   └── exercises.md
│
└── 10-production-playbook/              # Capstone
    ├── README.md                        # Production checklist
    ├── real-world-scenarios/
    └── troubleshooting-guide.md