# ArgoCD Bootstrap

Install ArgoCD into the cluster. Run once per cluster.

## Prerequisites

- GKE cluster provisioned via `infra-as-code`
- `kubectl`, `helm`, `gcloud` installed and on PATH
- Workload Identity SA for external-dns applied in `infra-as-code`

Authenticate to the target cluster (replace cluster name and project as needed):

```bash
gcloud container clusters get-credentials stg-iac-01-ause2 \
  --region australia-southeast2 \
  --project iac-01
```

## Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values bootstrap/argocd/values.yaml \
  --wait
```

## Access the UI (port-forward)

Until DNS is configured, use port-forward to reach the ArgoCD UI locally:

```bash
kubectl port-forward service/argocd-server -n argocd 8080:443
```

Then open: **[https://localhost:8080](https://localhost:8080)** (accept the self-signed cert warning)

## Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

## Apply root App-of-Apps

Each cluster has its own root-app file under `bootstrap/`. Apply the one matching the target cluster:

| Cluster | Root-app file |
| --- | --- |
| stg-iac-01-ause2 | `bootstrap/stg-iac-01-ause2-root-app.yaml` |
| stg-iac-02-ause2 | `bootstrap/stg-iac-02-ause2-root-app.yaml` |
| prd-iac-01-ause2 | `bootstrap/prd-iac-01-ause2-root-app.yaml` |

Update the `repoURL` in the root-app and all ApplicationSets under `clusters/<env>/<cluster>/`
to your actual repo URL before applying.

```bash
# Example for stg-iac-01-ause2
kubectl apply -f bootstrap/stg-iac-01-ause2-root-app.yaml
```

This registers the cluster folder with ArgoCD — all ApplicationSets
in `clusters/<env>/<cluster>/` are then managed automatically.

## Verify

```bash
kubectl -n argocd get applications
kubectl -n argocd get applicationsets
```

## Update admin password

While using port-forward (no DNS yet):

```bash
argocd login localhost:8080 --insecure
argocd account update-password
```

Once DNS is available, replace `localhost:8080` with `argocd.iac.internal`.
