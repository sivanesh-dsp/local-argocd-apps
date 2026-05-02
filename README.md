To deploy argocd using Argocd Application is a classic chicken-and-egg bootstrap problem. 

An `Applicaiton` CR can only be processed by an existing ArgoCD installation

### Step 1 - Install Argo CD once with helm

```
kubectl create namespace argocd

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --version 9.5.11
```

### Step 2 - Apply this root-app `app-of-apps` application to deploy child applications
```
kubectl apply -f root-app.yaml
```

#### File structure

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