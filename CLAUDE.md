# k8s-as-code — Claude Context

## What this repo is

GitOps repo for deploying Kubernetes platform addons and apps across multiple GKE clusters using ArgoCD. Pairs with `infra-as-code` for GCP infrastructure (GKE clusters, IAM, Private CA, DNS).

## Directory structure

```
bootstrap/                          # One-time bootstrap per cluster (apply manually to each cluster's ArgoCD)
  <cluster>-root-app.yaml           # e.g. stg-iac-01-ause2-root-app.yaml

clusters/
  <env>/                            # stg/ or prd/
    <cluster>/                      # e.g. stg-iac-01-ause2/
      addons-prereqs.yaml           # ApplicationSet: wave -1 (cert-manager, google-cas-issuer)
      addons.yaml                   # ApplicationSet: wave 0 (linkerd, external-dns)
      apps.yaml                     # ApplicationSet: user workloads

addons/                             # Helm umbrella charts wrapping upstream charts
  cert-manager/
  external-dns/
  google-cas-issuer/
  linkerd/

apps/                               # User workload Helm charts (currently empty)
```

## Multi-env / multi-cluster pattern

- Each cluster has its own ArgoCD with a `bootstrap/<cluster>-root-app.yaml` applied to it
- The root-app points to `clusters/<env>/<cluster>/` which contains 3 ApplicationSets
- **stg**: auto-sync enabled; **prd**: manual sync only
- Helm value layering per addon: `values.yaml` (base) → `values-<env>.yaml` (env overrides)
- Cluster-specific values (e.g. `external-dns.txtOwnerId`) are injected via ApplicationSet `helm.parameters`

## Current clusters

| Cluster | Env | Bootstrap file |
|---|---|---|
| stg-iac-01-ause2 | stg | `bootstrap/stg-iac-01-ause2-root-app.yaml` |
| stg-iac-02-ause2 | stg | `bootstrap/stg-iac-02-ause2-root-app.yaml` |
| prd-iac-01-ause2 | prd | `bootstrap/prd-iac-01-ause2-root-app.yaml` |

## Addons

| Addon | Wave | Namespace | Notes |
|---|---|---|---|
| cert-manager | -1 | cert-manager | Installs CRDs via Helm |
| google-cas-issuer | -1 (commented out, manual) | cert-manager | GCP CAS ClusterIssuer for linkerd |
| linkerd | 0 | linkerd | Native sidecars (GKE Autopilot); external CA via cert-manager |
| external-dns | 0 | external-dns | Google provider; Workload Identity; txtOwnerId injected per-cluster |

## Key design decisions

- **Sync waves**: addons-prereqs (wave -1) must be Healthy before addons (wave 0) sync
- **external-dns txtOwnerId**: set to cluster name via ApplicationSet `helm.parameters` — do NOT hardcode in values.yaml
- **prd sync policy**: no `automated` block — requires manual ArgoCD sync trigger
- **GKE Gateway CRDs**: ignored in `ignoreDifferences` to avoid conflicts with GKE-managed `httproutes.gateway.networking.k8s.io`
- **Linkerd trust anchor**: cluster-specific PEM in `values-<env>.yaml`; private key must be stored in GCP Secret Manager

## To add a new cluster

1. Create `clusters/<env>/<cluster>/` and copy ApplicationSets from an existing cluster in the same env
2. Update `cluster:` label and `external-dns.txtOwnerId` parameter to the new cluster name
3. Create `bootstrap/<cluster>-root-app.yaml` pointing to the new cluster path
4. Apply the root-app to the new cluster's ArgoCD: `kubectl apply -f bootstrap/<cluster>-root-app.yaml`

## To add a new env

1. Create `clusters/<new-env>/` with at least one cluster subdir
2. Add `values-<new-env>.yaml` to each addon in `addons/`
3. Decide sync policy (auto for non-prod, manual for prod)
4. Create `bootstrap/<cluster>-root-app.yaml` for each cluster in the env

## Related repos

- `infra-as-code`: GKE cluster provisioning, IAM, Private CA, DNS zones (Terraform/Terragrunt)
- GCP project naming: `iac-01` = stg data-plane project
