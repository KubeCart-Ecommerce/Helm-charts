# KubeCart Helm GitOps

This directory is the GitOps source of truth for KubeCart.

It contains:

- the umbrella Helm chart (`Chart.yaml`)
- environment values (`values-dev.yaml`, `values-prod.yaml`)
- ArgoCD Applications (`argocd/dev-app.yaml`, `argocd/prod-app.yaml`)

The deployment flow is:

1. Each microservice repository runs GitHub Actions CI.
2. CI builds and pushes a container image to `ghcr.io`.
3. ArgoCD Image Updater watches the image repository.
4. Image Updater commits the new tag into this Helm repo.
5. ArgoCD syncs the changed values into the cluster.

## Repository structure

- `Chart.yaml`: umbrella chart for frontend, gateway, services, mongodb, storage, and grafana-nodeport
- `values.yaml`: base values for manual Helm usage
- `values-dev.yaml`: dev environment values
- `values-prod.yaml`: prod environment values
- `argocd/dev-app.yaml`: ArgoCD Application for dev
- `argocd/prod-app.yaml`: ArgoCD Application for prod
- `argocd/image-updater.yaml`: ImageUpdater CR that lets modern ArgoCD Image Updater watch both Applications while still reading legacy annotations

## Current deployment contract

- Dev branch: `dev`
- Prod branch: `main`
- Dev namespace: `dev`
- Prod namespace: `prod`
- Dev sync: automatic
- Prod sync: manual by default

## Image tagging contract

This Helm repo expects CI to publish images with two different tag styles:

- Dev images: full 40-character Git commit SHA
- Prod images: semantic version tags like `v1.2.3`

That contract matters because ArgoCD Image Updater is configured like this:

- Dev uses `newest-build` and only accepts tags matching `^[0-9a-f]{40}$`
- Prod uses `semver` and only accepts tags matching `^v[0-9]+\.[0-9]+\.[0-9]+$`

This separation prevents the dev application from accidentally consuming prod semver images from the same GHCR repository.

## Important chart behavior

- Most service configuration lives under `.Values.global.*`
- Image updater writes only top-level image keys such as `frontend.image.tag`
- Templates prefer `.Values.<service>.image.tag`
- Templates fall back to `.Values.global.<service>.image.tag`

Do not remove that override pattern. It is required for ArgoCD Image Updater.

## Frontend behavior

The frontend is expected to call the backend through the same origin and nginx reverse proxy:

- `/api/auth`
- `/api/products`
- `/api/orders`
- `/api/cart`
- `/api/profiles`
- `/api/notifications`

Because of that, the Helm frontend config keeps the `REACT_APP_*_URL` values empty.
That preserves relative API paths instead of pushing cluster-local DNS names into the browser layer.

## Backend behavior

- Each backend service gets non-secret config from a ConfigMap
- Each backend service gets secrets from a Secret
- Pods load them through `envFrom`
- MongoDB is deployed as six StatefulSets with six headless Services
- Each service uses its own Mongo service DNS name through `MONGO_URI`

## Prerequisites

Install ArgoCD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Install ArgoCD Image Updater:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
```

Cluster prerequisites:

- a working `kgateway` `GatewayClass`
- a working NFS provisioner named `cluster.local/dev-nfs-provisioner-nfs-subdir-external-provisioner`
- GHCR access if the images are private

## Git repository credential secret

ArgoCD and ArgoCD Image Updater both need access to this Git repository.

Create `repo-secret.yaml`:

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

If GHCR images are private, also configure registry credentials for ArgoCD Image Updater.

## Apply the applications

```bash
kubectl apply -f argocd/dev-app.yaml
kubectl apply -f argocd/prod-app.yaml
kubectl apply -f argocd/image-updater.yaml
```

Namespace creation is enabled through `CreateNamespace=true`.

## Sync behavior

- `kubecart-dev` auto-syncs with `prune: true` and `selfHeal: true`
- `kubecart-prod` does not auto-sync by default
- `kubecart-image-updater` selects both ArgoCD Applications and tells the controller to read their existing `argocd-image-updater.argoproj.io/*` annotations

To deploy prod after Image Updater changes `values-prod.yaml`, sync it manually:

```bash
argocd app sync kubecart-prod
```

## Values that must be replaced before real deployment

Replace the placeholders in:

- `values.yaml`
- `values-dev.yaml`
- `values-prod.yaml`

Replace:

- JWT secrets
- SMTP credentials
- repo URL placeholders if you fork the Helm repo

## First sync note

The values files start with `v0.0.0` so prod semver tracking has a valid seed tag.

If your repositories already publish real tags, replace `v0.0.0` with the first release tag you want before the first sync.

## Troubleshooting

If dev is not updating:

- confirm CI pushes SHA-based tags to GHCR
- confirm the dev image tag is a full 40-character commit SHA
- confirm ArgoCD Image Updater can read GHCR and push back to Git

If prod is not updating:

- confirm CI pushes `vX.Y.Z` tags on `main`
- confirm the new tag matches the semver regex
- confirm you manually sync `kubecart-prod` unless you intentionally enable prod auto-sync

If profile APIs fail:

- verify the route prefix is `/api/profiles`
- verify both gateway and frontend are using the plural path

