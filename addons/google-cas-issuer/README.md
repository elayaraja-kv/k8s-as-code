# google-cas-issuer

cert-manager plugin that bridges Kubernetes certificate requests to GCP Certificate Authority Service.

## What this addon does

This addon deploys a controller (Pod) inside the cluster. When cert-manager needs to issue
a certificate that references a `GoogleCASClusterIssuer`, this controller:

1. Authenticates to GCP via Workload Identity (no credentials in the cluster)
2. Calls the GCP CAS API to sign the certificate using the HSM-backed root CA
3. Returns the signed cert to cert-manager, which stores it as a Kubernetes secret

It also creates a `GoogleCASClusterIssuer` resource pointing at the GCP CA pool,
which the Linkerd addon references to issue its identity issuer certificate.

```
cert-manager (Certificate CR)
        ↓
google-cas-issuer controller    ← this addon
        ↓  (Workload Identity)
GCP Certificate Authority Service  ← infra-as-code
        ↓
cert-manager stores cert → K8s secret
        ↓
Linkerd reads secret → mTLS
```

---

## Prerequisites

**Both infra-as-code units must be applied before this addon can work.**

### 1. GCP Private CA pool and root CA

```bash
cd nz3es/gcp/<env>/data-plane/<project>/australia-southeast2/private-ca/linkerd
terragrunt apply
```

Creates:

- CA pool (e.g. `linkerd-stg-01`) in `australia-southeast2`
- Root CA: `linkerd-root-stg` (EC P256, HSM-backed, 10-year lifetime)

### 2. GCP Service Account + CA pool IAM

```bash
cd nz3es/gcp/<env>/data-plane/<project>/global/iam/serviceaccounts/serviceaccounts-k8s/google-cas-issuer
terragrunt apply
```

Creates:

- GCP SA: `google-cas-issuer@<project>.iam.gserviceaccount.com`
- Workload Identity binding: allows the `google-cas-issuer` KSA to impersonate the GCP SA
- CA pool IAM: grants `roles/privateca.certificateRequester` on the CA pool

### 3. cert-manager must be healthy

```bash
kubectl get pods -n cert-manager
# All pods Running before proceeding
```

---

## ArgoCD sync order

This addon is in **sync wave `-1`** (see `clusters/<env>/<cluster>/addons-prereqs.yaml`), meaning it deploys
before wave-0 addons (linkerd, external-dns, etc.).

```
wave -1: addon-cert-manager       ← cert-manager CRDs + controller
wave -1: addon-google-cas-issuer  ← this addon + GoogleCASClusterIssuer
         ↓ (both must be Healthy)
wave  0: addon-linkerd            ← Certificate issued → mTLS ready
```

---

## Per-env configuration

Values are layered: `values.yaml` (base) → `values-<env>.yaml` (env overrides).

Update `values-<env>.yaml` with the env-specific GCP SA, project, and CA pool:

```yaml
# addons/google-cas-issuer/values-stg.yaml
cert-manager-google-cas-issuer:
  serviceAccount:
    annotations:
      iam.gke.io/gcp-service-account: google-cas-issuer@<stg-project>.iam.gserviceaccount.com

clusterIssuer:
  project: <stg-project>
  location: australia-southeast2
  caPoolId: <stg-ca-pool-name>
```

---

## What is deployed

| Resource | Kind | Purpose |
| --- | --- | --- |
| `google-cas-issuer` | Deployment | Controller that handles GCP CAS certificate requests |
| `google-cas-issuer` | ServiceAccount | KSA annotated with GCP SA for Workload Identity |
| `linkerd-cas-issuer` | GoogleCASClusterIssuer | Cluster-scoped issuer pointing at the CA pool |

---

## Verify

```bash
# Controller is running
kubectl get pods -n google-cas-issuer

# ClusterIssuer is registered
kubectl get googlecasclusterissuer linkerd-cas-issuer

# After linkerd deploys — cert-manager issued the identity issuer cert
kubectl get certificate -n linkerd
# Expected: linkerd-identity-issuer   True
```

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Controller pod `CrashLoopBackOff` | Workload Identity not set up | Verify terragrunt unit applied and KSA annotation is correct |
| `GoogleCASClusterIssuer` not found | Controller not running | Check `kubectl get pods -n google-cas-issuer` |
| `Certificate` stays `False` | IAM binding missing or CA pool wrong | Check `kubectl describe certificate linkerd-identity-issuer -n linkerd` for error |
| `certificateRequester` denied | WI unit not applied | Run `terragrunt apply` in `serviceaccounts-k8s/google-cas-issuer` |
