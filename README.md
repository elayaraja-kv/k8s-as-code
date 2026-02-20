# k8s-as-code

GitOps repo for Kubernetes workloads on GKE, managed by ArgoCD.

Pairs with [infra-as-code](https://github.com/elayaraja-kv/infra-as-code) which provisions the GCP infrastructure (GKE cluster, VPC, DNS zones, IAM/Workload Identity).

## Structure

```text
bootstrap/              # One-time ArgoCD install
  argocd/
    values.yaml         # ArgoCD Helm values
    install.md          # Step-by-step install guide
  root-app.yaml         # App-of-Apps — registers cluster with ArgoCD

clusters/
  stg-iac-01-ause2/
    addons.yaml         # ApplicationSet → addons/ (platform tools)
    apps.yaml           # ApplicationSet → apps/ (workloads)

addons/                 # Platform tools (external-dns, cert-manager, ...)
  external-dns/
    Chart.yaml          # Helm wrapper (bitnami/external-dns)
    values.yaml         # Base values
    values-stg.yaml     # Stg overrides (Workload Identity SA, zone filters)

apps/                   # User workloads (Helm or Kustomize)
```

## Bootstrap Order

### Prerequisites

- GKE cluster provisioned via `infra-as-code`
- `kubectl`, `helm`, `gcloud` installed
- Workload Identity SA for external-dns applied in `infra-as-code`

### 1. Authenticate to the cluster

```bash
gcloud container clusters get-credentials stg-iac-01-ause2 \
  --region australia-southeast2 \
  --project iac-01
```

### 2. Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values bootstrap/argocd/values.yaml \
  --wait
```

### 3. Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 4. Apply root App-of-Apps

Update the `repoURL` in these files to your actual repo URL before applying:

- `bootstrap/root-app.yaml`
- `clusters/stg-iac-01-ause2/addons.yaml`
- `clusters/stg-iac-01-ause2/apps.yaml`

```bash
kubectl apply -f bootstrap/root-app.yaml
```

ArgoCD will automatically pick up all ApplicationSets under `clusters/stg-iac-01-ause2/` and deploy addons and apps.

### 5. Verify

```bash
kubectl -n argocd get applications
kubectl -n argocd get applicationsets
```

## Adding a new addon

1. Create `addons/<name>/Chart.yaml` (Helm wrapper or raw manifests)
2. Add `addons/<name>/values.yaml` and `addons/<name>/values-stg.yaml`
3. Commit and push — ArgoCD Git generator picks it up automatically

## Adding a new app

1. Create `apps/<name>/` with Helm chart or Kustomize manifests
2. Commit and push — ArgoCD Git generator picks it up automatically

## Workload Identity

GCP service accounts and KSA annotations are managed in `infra-as-code`:

```text
infra-as-code/nz3es/gcp/stg/data-plane/iac-01/
  global/iam/serviceaccounts/serviceaccounts-k8s/
    external-dns/    ← GCP SA + WI binding
```

The KSA annotation in `values-stg.yaml` must match the GCP SA email created there.

## DNS Zones

Both zones are managed by external-dns:

| Zone | Type | Provider |
| --- | --- | --- |
| `iac.com` | Public | GCP Cloud DNS |
| `iac.internal` | Private | GCP Cloud DNS |

Annotate a `Service` or `Ingress` with `external-dns.alpha.kubernetes.io/hostname` to register a DNS record.
