# local-argocd-apps

Argo CD "app-of-apps" setup for a local Kubernetes cluster. It works with either:

- a **kubeadm** cluster running on local VMs, or
- a **kind** cluster (Kubernetes in Docker).

A single root Application deploys:

| App        | Chart                                         | Namespace    | Hostname               |
|------------|-----------------------------------------------|--------------|------------------------|
| Argo CD    | `argo/argo-cd`                                | `argocd`     | `argocd.localhost`     |
| Traefik    | `traefik/traefik` (Gateway API controller)    | `traefik`    | -                      |
| Prometheus | `prometheus-community/prometheus`             | `monitoring` | `prometheus.localhost` |
| Grafana    | `grafana-community/grafana`                   | `monitoring` | `grafana.localhost`    |

Traefik runs as a `hostNetwork` DaemonSet on every node (including control-plane nodes) and listens on ports 80/443 of each node. It exposes a `traefik-gateway` Gateway (listener `web`, port 80), and each app attaches to it with an `HTTPRoute` on its hostname.

## Why bootstrap with Helm?

Deploying Argo CD with an Argo CD `Application` is a classic chicken-and-egg bootstrap problem: an `Application` CR can only be processed by an existing Argo CD installation.

So Argo CD is installed once with Helm, and from then on the `argo-cd` Application in `apps.yaml` takes over and manages Argo CD itself. The Helm release name (`argocd`) must match `releaseName` in `apps.yaml` so Argo CD adopts the existing resources instead of creating duplicates.

## Step 0 - Prepare the cluster

Pick the section that matches your cluster.

### Option A - kubeadm cluster on VMs

> TODO: add kubeadm cluster setup and access instructions.

### Option B - kind cluster

Map ports 80/443 from the control-plane node to your machine in the kind config, so traffic reaches Traefik:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - {containerPort: 80, hostPort: 80}
  - {containerPort: 443, hostPort: 443}
  - {containerPort: 30080, hostPort: 30080}
- role: worker
- role: worker
```

```sh
kind create cluster --name local --config config.yaml
```

> `extraPortMappings` can't be added to an existing cluster; the cluster must be recreated.
>
> Traefik binds ports 80/443 without root. This works because kind node containers have `net.ipv4.ip_unprivileged_port_start=0` (Docker's default for containers).

After the apps are deployed (Steps 1-2), open them at http://argocd.localhost, http://grafana.localhost and http://prometheus.localhost.

No `/etc/hosts` entries are needed: `*.localhost` names resolve to `127.0.0.1` automatically (OS resolver, browsers and curl), and the port mappings forward that to Traefik. `/etc/hosts` does not support wildcards anyway.

## Step 1 - Install Argo CD once with Helm

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd -n argocd --create-namespace
```

## Step 2 - Apply the root `app-of-apps` Application

```sh
kubectl apply -f root-app.yaml
```

`root-app.yaml` renders the [`argocd-apps`](https://github.com/argoproj/argo-helm/tree/main/charts/argocd-apps) chart with `apps.yaml` as its values file, which creates the `internal-tools` AppProject and the child Applications. Each child Application pulls its chart from the upstream Helm repo and its values from `values/` in this repo (multi-source with `ref: repo`).

```sh
kubectl get applications -n argocd
```

## Logins

| App        | HostNames                     | Login                                                  |
|------------|-------------------------------|--------------------------------------------------------|
| Argo CD    | `http://argocd.localhost`     | `admin` / see below                                    |
| Grafana    | `http://grafana.localhost`    | `admin` / `admin` (change after first login)           |
| Prometheus | `http://prometheus.localhost` | -                                                      |

Argo CD initial admin password:

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Argo CD runs with `server.insecure: true` (plain HTTP), so use `http://`, not `https://`. Port-forwarding works too, but only with HTTP: `kubectl -n argocd port-forward svc/argocd-server 8080:80` then open http://localhost:8080. Sending HTTPS to the plain-HTTP server resets the connection and ends the port-forward with `lost connection to pod`.

## File structure

```
local-argocd-apps/
├── root-app.yaml          # Bootstrap (kubectl apply once)
├── apps.yaml              # AppProject + Application definitions (no inline values)
└── values/
    ├── argo-cd.yaml       # Argo CD helm values (insecure mode, HTTPRoute)
    ├── traefik.yaml       # Traefik helm values (hostNetwork DaemonSet, Gateway API)
    ├── prometheus.yaml    # Prometheus helm values (HTTPRoute)
    └── grafana.yaml       # Grafana helm values (datasource, dashboards, HTTPRoute)
```

## Adding a new app

1. Add a values file under `values/` with an `HTTPRoute` attached to `traefik-gateway` (namespace `traefik`, section `web`) and a `<app>.localhost` hostname.
2. Add an entry under `applications:` in `apps.yaml`, following the existing ones.
3. Commit and push to `main`; the root app syncs it automatically.
