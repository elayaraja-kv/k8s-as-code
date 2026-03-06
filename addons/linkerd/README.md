# Linkerd Service Mesh

## Prerequisites

**cert-manager must be healthy before syncing Linkerd.**

```bash
kubectl get pods -n cert-manager
# Expected: cert-manager-*, cert-manager-cainjector-*, cert-manager-webhook-* all Running
```

---

## How certificates work

Linkerd uses a three-tier certificate model regardless of which CA option you choose:

```
Trust anchor (root CA)
        |
        └── Issuer cert          — cert-manager issues and auto-rotates (every 48h)
                |
                └── Workload certs — Linkerd issues per-pod, fully automatic (every 24h)
```

| Layer | Who manages it | Stored in Git? |
|---|---|---|
| Trust anchor cert | CA option (see below) | Public cert only — in `values-stg.yaml` |
| Trust anchor key | CA option (see below) | Never |
| Issuer cert + key | cert-manager (automatic) | Never |
| Workload mTLS certs | Linkerd (automatic) | Never |

---

## CA Options

### Option A — GCP Private CA (current setup)

> Used in this repo. The trust anchor key is HSM-backed in GCP — no manual key management.

**Prerequisites (infra-as-code):**

```bash
# 1. Provision the CA pool + root CA
cd nz3es/gcp/stg/data-plane/iac-01/australia-southeast2/private-ca/linkerd
terragrunt apply

# 2. Provision GCP SA + CA pool IAM for google-cas-issuer
cd nz3es/gcp/stg/data-plane/iac-01/global/iam/serviceaccounts/serviceaccounts-k8s/google-cas-issuer
terragrunt apply
```

**Get the trust anchor cert** (already done — committed in `values-stg.yaml`):

```bash
terragrunt output -raw ca_cert_pem
# Paste into identityTrustAnchorsPEM in values-stg.yaml
```

**ArgoCD sync order:**

```bash
argocd app sync addon-cert-manager       # 1. wait until healthy
argocd app sync addon-google-cas-issuer  # 2. wait until healthy
argocd app sync addon-linkerd            # 3.
```

No Kubernetes secret bootstrap required — cert-manager authenticates to GCP via Workload Identity.

---

### Option B — Self-managed CA (manual, no GCP dependency)

> Use this if GCP Private CA is not available or for local development.

**Step 1 — Install tools:**

```bash
brew install step    # smallstep CLI
brew install linkerd # Linkerd CLI
```

**Step 2 — Generate trust anchor (one-time):**

```bash
step certificate create root.linkerd.iac.internal ca.crt ca.key \
  --profile root-ca \
  --no-password \
  --insecure \
  --not-after 87600h
```

Store the key safely — it never goes in Git:

```bash
gcloud secrets create linkerd-trust-anchor-key \
  --project iac-01 \
  --data-file=ca.key
```

**Step 3 — Paste the trust anchor cert into values-stg.yaml:**

```bash
cat ca.crt
# Copy output → replace identityTrustAnchorsPEM in values-stg.yaml
```

**Step 4 — Load the trust anchor secret into the cluster:**

cert-manager needs the key to sign the issuer cert via the `ca` Issuer in
`templates/linkerd-identity-issuer.yaml`:

```bash
kubectl create namespace linkerd --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret tls linkerd-trust-anchor \
  --namespace linkerd \
  --cert=ca.crt \
  --key=ca.key

rm ca.key ca.crt   # safe to delete — key is in GCP Secret Manager
```

**Step 5 — Sync via ArgoCD:**

```bash
argocd app sync addon-cert-manager  # wait until healthy
argocd app sync addon-linkerd
```

---

## Verify (both options)

```bash
# cert-manager issued the issuer cert
kubectl get certificate -n linkerd
# Expected: linkerd-identity-issuer   True

# Issuer secret exists
kubectl get secret linkerd-identity-issuer -n linkerd

# Full Linkerd health check
linkerd check
```

All checks should pass. The issuer cert renews automatically every 48h.

---

## Mesh a workload

Annotate a namespace to inject Linkerd proxies into all pods:

```bash
kubectl annotate namespace <your-namespace> linkerd.io/inject=enabled
```

Or per pod/deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        linkerd.io/inject: enabled
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `linkerd check` fails on identity | Issuer cert not issued | Check `kubectl get certificate -n linkerd` |
| `Certificate` stays `False` (Option A) | `google-cas-issuer` not running or WI not set up | Check `kubectl get pods -n google-cas-issuer` |
| `Certificate` stays `False` (Option B) | `linkerd-trust-anchor` secret missing | Run Option B Step 4 |
| Pods not getting proxies | Namespace not annotated | Run `kubectl annotate namespace` above |
| `linkerd check` TLS errors | Trust anchor cert mismatch | Re-paste cert into `identityTrustAnchorsPEM` |
