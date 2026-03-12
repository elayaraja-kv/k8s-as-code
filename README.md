# k8s-as-code

GitOps repo for Kubernetes workloads on GKE, managed by ArgoCD.

Pairs with [infra-as-code](https://github.com/elayaraja-kv/infra-as-code) which provisions the GCP infrastructure (GKE clusters, VPC, DNS zones, IAM/Workload Identity).

## Structure

```text
bootstrap/                              # One-time bootstrap — apply once per cluster
  argocd/
    values.yaml                         # ArgoCD Helm values
    install.md                          # Step-by-step install guide
  <cluster>-root-app.yaml               # App-of-Apps — registers cluster with ArgoCD
    e.g. stg-iac-01-ause2-root-app.yaml
         stg-iac-02-ause2-root-app.yaml
         prd-iac-01-ause2-root-app.yaml

clusters/
  <env>/                                # stg/ or prd/
    <cluster>/                          # e.g. stg-iac-01-ause2/
      addons-prereqs.yaml               # ApplicationSet → wave -1 addons (cert-manager)
      addons.yaml                       # ApplicationSet → wave 0 addons (linkerd, external-dns)
      apps.yaml                         # ApplicationSet → user workloads

addons/                                 # Platform tools (Helm umbrella charts)
  <addon>/
    Chart.yaml                          # Wraps upstream chart
    values.yaml                         # Base defaults
    values-stg.yaml                     # Stg overrides (replicas, WI SA, zone filters)
    values-prd.yaml                     # Prd overrides (HA replicas, upsert-only policy)

apps/                                   # User workloads (Helm or Kustomize)
```

## Clusters

| Cluster | Env | Sync | Bootstrap file |
| --- | --- | --- | --- |
| stg-iac-01-ause2 | stg | Auto | `bootstrap/stg-iac-01-ause2-root-app.yaml` |
| stg-iac-02-ause2 | stg | Auto | `bootstrap/stg-iac-02-ause2-root-app.yaml` |
| prd-iac-01-ause2 | prd | Manual | `bootstrap/prd-iac-01-ause2-root-app.yaml` |

## Bootstrap

See [bootstrap/argocd/install.md](bootstrap/argocd/install.md) for step-by-step instructions.

## Adding a new cluster

1. Create `clusters/<env>/<cluster>/` — copy the 3 ApplicationSet files from an existing cluster in the same env
2. Update the `cluster:` label and `external-dns.txtOwnerId` parameter to the new cluster name
3. Create `bootstrap/<cluster>-root-app.yaml` pointing to the new cluster path
4. Apply the root-app to the cluster's ArgoCD:

   ```bash
   kubectl apply -f bootstrap/<cluster>-root-app.yaml
   ```

## Adding a new env

1. Create `clusters/<new-env>/<cluster>/` with the 3 ApplicationSet files
2. Add `values-<new-env>.yaml` to each addon under `addons/`
3. Set sync policy: auto for non-prod, remove `automated` block for prod
4. Create `bootstrap/<cluster>-root-app.yaml` for each cluster

## Adding a new addon

1. Create `addons/<name>/Chart.yaml` (Helm wrapper)
2. Add `addons/<name>/values.yaml`, `values-stg.yaml`, `values-prd.yaml`
3. Commit and push — ArgoCD Git generator picks it up automatically

## Adding a new app

1. Create `apps/<name>/` with a Helm chart or Kustomize manifests
2. Commit and push — ArgoCD Git generator picks it up automatically

## Workload Identity

GCP service accounts and KSA annotations are managed in `infra-as-code`:

```text
infra-as-code/nz3es/gcp/<env>/data-plane/<project>/
  global/iam/serviceaccounts/serviceaccounts-k8s/
    external-dns/          ← GCP SA + WI binding
    google-cas-issuer/     ← GCP SA + CA pool IAM + WI binding
```

The KSA annotation in `values-<env>.yaml` must match the GCP SA email created there.

## DNS Zones

Both zones are managed by external-dns:

| Zone | Type | Provider |
| --- | --- | --- |
| `iac.com` | Public | GCP Cloud DNS |
| `iac.internal` | Private | GCP Cloud DNS |

Annotate a `Service` or `Ingress` with `external-dns.alpha.kubernetes.io/hostname` to register a DNS record.
