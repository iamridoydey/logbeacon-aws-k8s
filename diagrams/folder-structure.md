logbeacon-aws-k8s/
│
├── apps/
│   └── logbeacon/
│       ├── base/
│       │   ├── namespace.yaml
│       │   ├── backend-rollout.yaml
│       │   ├── backend-service.yaml
│       │   ├── frontend-rollout.yaml
│       │   ├── frontend-service.yaml
│       │   ├── configmap.yaml
│       │   ├── externalsecret.yaml
│       │   ├── gateway.yaml
│       │   ├── httproute.yaml
│       │   └── kustomization.yaml
│       │
│       └── overlays/
│           ├── dev/
│           └── prod/
│
├── argocd/
│   └── argo-apps/
│       ├── management/
│       │   ├── external-dns.yaml
│       │   ├── external-secrets.yaml
│       │   ├── traefik.yaml
│       │   └── cert-manager.yaml
│       │
│       └── workload/
│           ├── traefik.yaml
│           ├── external-dns.yaml
│           ├── external-secrets.yaml
│           ├── postgresql.yaml
│           ├── redis.yaml
│           ├── sonarqube.yaml
│           └── logbeacon.yaml
│
├── management/
│   ├── external-dns/
│   ├── external-secrets/
│   ├── traefik/
│   └── cert-manager/
│
├── infra/
│   ├── external-dns/
│   ├── external-secrets/
│   ├── traefik/
│   ├── cert-manager/
│   ├── postgresql/
│   ├── redis/
│   ├── sonarqube/
│   └── cert-manager/
│   └── storage/
│
└── diagrams/
    └── folder-structure.md