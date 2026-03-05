# hello-world-deployments

GitOps repository for the Hello World application. This repo is the single source of truth for the Kubernetes deployment — ArgoCD watches it and syncs any change to the cluster automatically.

The application is packaged as a **Helm chart** which makes deployment easier and more flexible. Instead of managing raw Kubernetes manifests, Helm lets you configure the entire deployment through a single `values.yaml` file — changing the image tag, replica count, resource limits, or toggling Istio and TLS on and off without touching any templates directly. It also handles upgrades and rollbacks cleanly.

---

## Repository Structure

```
hello-world-deployments/
└── charts/
    └── hello-world/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            ├── ingress.yaml
            ├── gateway.yaml
            ├── virtual-service.yaml
            └── peer-authentication.yaml
```

---

## How It Works

The CI/CD pipeline in `hello-world-app` builds a new Docker image on every push to `main` and updates the `image.tag` in `values.yaml` via SSH deploy key. ArgoCD detects the change and performs a rolling deploy automatically — no `kubectl` is ever run in CI.

```
CI pushes new tag → values.yaml updated → ArgoCD syncs → rolling deploy
```

---

## Templates

### deployment.yaml
Deploys the FastAPI Hello World app. Runs 2 replicas with liveness and readiness probes on `/health` (port 9000). Resource requests and limits are defined to ensure stable scheduling.

### service.yaml
ClusterIP service exposing port 80, forwarding to the app on port 9000. Traffic enters via the Istio IngressGateway — not directly through this service.

### ingress.yaml
Disabled (`ingress.enabled: false`). Traffic is routed through the Istio Gateway instead of a standard Kubernetes Ingress.

### gateway.yaml
Istio Gateway resource. Configures the Istio IngressGateway to accept traffic on:
- Port 80 (HTTP)
- Port 443 (HTTPS) — enabled when `istio.tls.enabled: true`, uses the `hello-world-tls` secret in `istio-system` which is managed by External Secrets Operator

### virtual-service.yaml
Routes traffic from the Gateway to the hello-world Service. Configures a 30s timeout and 3 retry attempts with a 10s per-try timeout.

### peer-authentication.yaml
Enforces mTLS STRICT mode across the `hello-world` namespace. All pod-to-pod communication must use mutual TLS — plain text traffic is rejected.

---

## Values Reference

| Value | Default | Description |
|---|---|---|
| `replicaCount` | `2` | Number of app replicas |
| `image.repository` | ECR URL | Docker image repository |
| `image.tag` | `latest` | Image tag — updated automatically by CI, always a quoted string |
| `image.pullPolicy` | `Always` | Always pull to pick up latest |
| `service.port` | `80` | Service port |
| `service.targetPort` | `9000` | App container port |
| `istio.enabled` | `true` | Deploy Istio Gateway and VirtualService |
| `istio.gateway.host` | `"*"` | Hostname to match on the Gateway |
| `istio.tls.enabled` | `true` | Enable HTTPS on port 443 |
| `istio.tls.credentialName` | `hello-world-tls` | TLS secret name in istio-system |
| `istio.mtls.mode` | `STRICT` | mTLS mode for PeerAuthentication |
| `istio.virtualService.timeout` | `30s` | Request timeout |
| `istio.virtualService.retries.attempts` | `3` | Number of retries |

---

## Manual Installation (without CI/CD)

You can install or upgrade the chart directly with Helm without going through the CI/CD pipeline.

**Install:**
```bash
git clone https://github.com/Shlomi-Lantser/hello-world-deployments.git
cd hello-world-deployments
kubectl create namespace hello-world
kubectl label namespace hello-world istio-injection=enabled

helm install hello-world charts/hello-world \
  --namespace hello-world \
  --set image.repository=<your-ecr-url>/hello-world \
  --set image.tag="<your-image-tag>"
```

**Upgrade (e.g. deploy a new image):**
```bash
helm upgrade hello-world charts/hello-world \
  --namespace hello-world \
  --set image.repository=<your-ecr-url>/hello-world \
  --set image.tag="<your-new-tag>"
```

**Uninstall:**
```bash
helm uninstall hello-world --namespace hello-world
```

**Render templates locally without installing (useful for debugging):**
```bash
helm template hello-world charts/hello-world \
  --namespace hello-world \
  --set image.tag="abc1234"
```
