```
local-argocd-apps/
├── root-app.yaml          # Bootstrap (kubectl apply once)
├── apps.yaml              # App definitions (no inline values)
└── values/
    ├── argo-cd.yaml       # ArgoCD helm values
    ├── traefik.yaml       # Traefik helm values
    ├── prometheus.yaml    # Prometheus helm values
    └── grafana.yaml       # Grafana helm values
```