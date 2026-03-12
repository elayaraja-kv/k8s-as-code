# Linkerd Service Mesh

Linkerd edge channel (`2026.3.1`) with native sidecar support (`proxy.nativeSidecar: true`).
Required for GKE Autopilot and proper Job/batch workload support (K8s 1.29+).

## Prerequisites

**cert-manager must be healthy before syncing Linkerd.**

```bash
kubectl get pods -n cert-manager
# Expected: cert-manager-*, cert-manager-cainjector-*, cert-manager-webhook-* all Running
```

---

## How certificates work

Linkerd uses a three-tier certificate model:

```text
Trust anchor (root CA)
        |
        └── Issuer cert          — cert-manager issues and auto-rotates (every 48h)
                |
                └── Workload certs — Linkerd issues per-pod, fully automatic (every 24h)
```

| Layer | Who manages it | Stored in Git? |
| --- | --- | --- |
| Trust anchor cert | CA option (see below) | Public cert only — in `values-<env>.yaml` |
| Trust anchor key | CA option (see below) | Never |
| Issuer cert + key | cert-manager (automatic) | Never |
| Workload mTLS certs | Linkerd (automatic) | Never |

---

## CA Options

### Option A — Local CA via cert-manager (current setup)

> Used in this repo. Trust anchor is a self-signed ECDSA P256 root CA.
> Key is stored in GCP Secret Manager — no HSM but no ongoing cost.

**Step 1 — Generate trust anchor (one-time per cluster):**

```bash
# Generate ECDSA P256 root CA (10-year validity)
openssl ecparam -name prime256v1 -genkey -noout -out ca.key
openssl req -new -x509 -key ca.key \
  -subj "/O=cluster.local/CN=root.linkerd.cluster.local" \
  -days 3650 -extensions v3_ca -out ca.crt
```

Store the key safely — it never goes in Git:

```bash
gcloud secrets create linkerd-trust-anchor-key --project <project-id> --data-file=ca.key
gcloud secrets create linkerd-trust-anchor-cert --project <project-id> --data-file=ca.crt
```

**Step 2 — Paste the trust anchor cert into `values-<env>.yaml`:**

```bash
cat ca.crt
# Copy output → replace identityTrustAnchorsPEM in addons/linkerd/values-<env>.yaml
```

**Step 3 — Load the trust anchor secret into the cluster:**

cert-manager's `ClusterIssuer` (type `ca`) reads the key from the **cert-manager namespace**:

```bash
kubectl create secret tls linkerd-trust-anchor \
  --namespace cert-manager \
  --cert=ca.crt \
  --key=ca.key

rm ca.key ca.crt   # safe to delete — key is in GCP Secret Manager
```

**Step 4 — Sync via ArgoCD:**

```bash
argocd app sync addon-cert-manager  # wait until healthy
argocd app sync addon-linkerd
```

---

### Option B — GCP Private CA (Certificate Authority Service)

> Use this for production where HSM-backed keys are required.
> **Requires ENTERPRISE tier CA pool** (~$200/month) — DEVOPS tier cannot issue `isCA:true`
> certificates required by Linkerd's identity issuer.

**Prerequisites (infra-as-code):**

```bash
# 1. Provision the ENTERPRISE CA pool + root CA
cd nz3es/gcp/<env>/data-plane/<project>/australia-southeast2/private-ca/linkerd
terragrunt apply

# 2. Provision GCP SA + CA pool IAM for google-cas-issuer
cd nz3es/gcp/<env>/data-plane/<project>/global/iam/serviceaccounts/serviceaccounts-k8s/google-cas-issuer
terragrunt apply
```

**Get the trust anchor cert:**

```bash
terragrunt output -raw ca_cert_pem
# Paste into identityTrustAnchorsPEM in addons/linkerd/values-<env>.yaml
```

**Enable google-cas-issuer:**

1. Uncomment `addons/google-cas-issuer` in `clusters/<env>/<cluster>/addons-prereqs.yaml`
2. Update `addons/google-cas-issuer/values-<env>.yaml` with project, location, caPoolId
3. Update `issuerRef` in `templates/linkerd-identity-issuer.yaml` to use `GoogleCASClusterIssuer`

**ArgoCD sync order:**

```bash
argocd app sync addon-cert-manager       # 1. wait until healthy
argocd app sync addon-google-cas-issuer  # 2. wait until healthy
argocd app sync addon-linkerd            # 3.
```

---

## Per-env configuration

Values are layered: `values.yaml` (base) → `values-<env>.yaml` (env overrides).

Key differences between envs:

| Setting | stg | prd |
| --- | --- | --- |
| `controllerReplicas` | 1 | 3 |
| `identityTrustAnchorsPEM` | stg trust anchor cert | prd trust anchor cert (generate separately) |

Each env's trust anchor must be generated independently — they are separate root CAs.

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

**Verify native sidecar:**

```bash
# linkerd-proxy must appear as an initContainer with restartPolicy=Always
kubectl get pod -n <meshed-namespace> -o jsonpath='{range .items[0].spec.initContainers[*]}{.name}: restartPolicy={.restartPolicy}{"\n"}{end}'
# Expected:
#   linkerd-init: restartPolicy=
#   linkerd-proxy: restartPolicy=Always
```

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
| --- | --- | --- |
| `linkerd check` fails on identity | Issuer cert not issued | Check `kubectl get certificate -n linkerd` |
| `Certificate` stays `False` (Option A) | `linkerd-trust-anchor` secret missing in `cert-manager` namespace | Run Option A Step 3 |
| `Certificate` stays `False` (Option B) | `google-cas-issuer` not running or WI not set up | Check `kubectl get pods -n google-cas-issuer` |
| Pods not getting proxies | Namespace not annotated | Run `kubectl annotate namespace` above |
| `linkerd check` TLS errors | Trust anchor cert mismatch | Re-paste cert into `identityTrustAnchorsPEM` |
| Linkerd identity fatal: "must be intermediate-CA" | GCP CAS DEVOPS tier used | Switch to ENTERPRISE tier (Option B) or use Option A |
| DNS resolution fails (`svc.iac.internal`) | Wrong `clusterDomain` | Set `clusterDomain: cluster.local` in values |
