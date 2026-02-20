# ArgoCD Bootstrap

Install ArgoCD into the cluster. Run once per cluster.

## Prerequisites

```bash
gcloud container clusters get-credentials stg-iac-01-ause2 \
  --region australia-southeast2 \
  --project iac-01
```

## Install

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values bootstrap/argocd/values.yaml \
  --wait
```

## Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

## Apply root App-of-Apps

```bash
kubectl apply -f bootstrap/root-app.yaml
```

This registers the cluster folder with ArgoCD — all ApplicationSets
in `clusters/stg-iac-01-ause2/` are then managed automatically.

## Update admin password

```bash
argocd login argocd.iac.com
argocd account update-password
```
