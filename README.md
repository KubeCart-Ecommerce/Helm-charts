# ArgoCD GitOps Setup

This repository is the GitOps source of truth for the KubeCart platform.

The expected deployment flow is:

1. Each microservice CI pipeline builds and pushes a new image to `ghcr.io`
2. ArgoCD Image Updater detects the new tag
3. ArgoCD Image Updater updates `values-dev.yaml` or `values-prod.yaml`
4. ArgoCD Image Updater commits the change back to this repository
5. ArgoCD detects the Git change and deploys it automatically

## Repository structure

- `Chart.yaml` is the umbrella Helm chart
- `values-dev.yaml` contains the dev environment values
- `values-prod.yaml` contains the prod environment values
- `argocd/dev-app.yaml` is the ArgoCD Application for dev
- `argocd/prod-app.yaml` is the ArgoCD Application for prod

## 1. Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

## 2. Install ArgoCD Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

Note: Git write-back requires ArgoCD v2.0 or newer.

## 3. Create Git credentials secret

ArgoCD and ArgoCD Image Updater both need access to this Helm Git repository so they can read manifests and push updated image tags.

Create a file named `repo-secret.yaml` with this content:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: helm-repo-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/<org>/helm-repo.git
  username: <github-username>
  password: <github-token>
```

Apply it:

```bash
kubectl apply -f repo-secret.yaml
```

## 4. Update placeholders before applying

Replace these placeholders in:

- `values-dev.yaml`
- `values-prod.yaml`
- `argocd/dev-app.yaml`
- `argocd/prod-app.yaml`

Update:

- `<org>` with your GitHub org or username
- `https://github.com/<org>/helm-repo.git` with your real Helm repository URL
- SMTP placeholders and any application secrets with your real values

If your GHCR images are private, configure ArgoCD Image Updater with GHCR registry credentials before expecting automatic image discovery.

## 5. Apply the ArgoCD Applications

```bash
kubectl apply -f argocd/dev-app.yaml
kubectl apply -f argocd/prod-app.yaml
```

Both applications use auto-sync and will create the destination namespace if it does not already exist.

## 6. Expected CI behavior

Each microservice repository CI pipeline should only:

1. Build the Docker image
2. Push the Docker image to `ghcr.io`

CI should not update Helm values files and should not commit to this Helm repository.

## 7. How automatic image updates work

- `kubecart-dev` tracks the `dev` branch and updates `values-dev.yaml`
- `kubecart-prod` tracks the `main` branch and updates `values-prod.yaml`
- dev uses the `latest` strategy
- prod uses the `semver` strategy and only accepts tags matching `vX.X.X`

## 8. First sync note

The environment values files start with `v0.0.0` as the seed image tag so the prod `semver` strategy has a valid semantic version to compare against.

If you already have real tags in GHCR, replace `v0.0.0` with the first tag you want each environment to start from before the first ArgoCD sync.
