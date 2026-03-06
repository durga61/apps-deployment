# GitOps Setup Export

**Last updated:** March 6, 2026

This document reflects the current state of the code in this workspace.

## 1. Repositories

- `cart-service`: Python service + Dockerfile + GitHub Actions CI.
- `product-service`: Python service + Dockerfile + GitHub Actions CI.
- `order-service`: Python service + Dockerfile + GitHub Actions CI.
- `apps-deployment`: Kubernetes manifests (Kustomize) + Argo CD Application manifests.

## 2. Current Deployment Repository Layout

```text
apps-deployment/
  base/
    cart/
      deployment.yaml
      kustomization.yaml
    product/
      deployment.yaml
      kustomization.yaml
    order/
      deployment.yaml
      kustomization.yaml
  overlays/
    staging/
      cart/kustomization.yaml
      product/kustomization.yaml
      order/kustomization.yaml
    production/
      cart/kustomization.yaml
      product/kustomization.yaml
      order/kustomization.yaml
  argocd-apps/
    staging/
      cart-app.yaml
      product-app.yaml
      order-app.yaml
    production/
      cart-app.yaml
      product-app.yaml
      order-app.yaml
```

## 3. CI Behavior (All Three Services)

Current workflow path in each service repo:

- `.github/workflows/ci.yml`

Current behavior:

1. Trigger on push to `main`.
2. Build Docker image with Buildx.
3. Push to Docker Hub (`docker.io`) using:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN`
4. Tag format: `durga61/<service-name>:<7-char-SHA>`.

Examples from current code:

- `durga61/cart-service:${SHORT_SHA}`
- `durga61/product-service:${SHORT_SHA}`
- `durga61/order-service:${SHORT_SHA}`

## 4. Kustomize Base State

Each `base/<service>/kustomization.yaml` currently defines:

- `resources: [deployment.yaml]`
- `images` entry with `newTag: latest` as the initial/default tag.

Each `base/<service>/deployment.yaml` currently uses:

- `replicas: 1`
- image `durga61/<service-name>:latest`
- container port `8080`

## 5. Argo CD Applications (Current)

### Staging Applications

Paths:

- `argocd-apps/staging/cart-app.yaml`
- `argocd-apps/staging/product-app.yaml`
- `argocd-apps/staging/order-app.yaml`

Key settings in current code:

- `metadata.namespace: argocd-staging`
- `spec.source.path: overlays/staging/<service>`
- `spec.destination.namespace: staging`
- automated sync enabled (`prune: true`, `selfHeal: true`)

Image updater annotations are present on staging apps only:

- `argocd-image-updater.argoproj.io/image-list`
- `argocd-image-updater.argoproj.io/update-strategy: newest-build`
- `argocd-image-updater.argoproj.io/allow-tags: "regexp:^[a-f0-9]{7}$"`
- `argocd-image-updater.argoproj.io/write-back-method: git`
- `argocd-image-updater.argoproj.io/git-branch: main`

### Production Applications

Paths:

- `argocd-apps/production/cart-app.yaml`
- `argocd-apps/production/product-app.yaml`
- `argocd-apps/production/order-app.yaml`

Key settings in current code:

- `metadata.namespace: argocd-production`
- `spec.source.path: overlays/production/<service>`
- `spec.destination.namespace: production`
- automated sync enabled (`prune: true`, `selfHeal: true`)

Production app manifests currently do **not** include Image Updater annotations.

## 6. Effective Promotion Model in Current Code

- `main` push in a service repo publishes a new SHA-tagged image.
- Staging Argo CD app + Image Updater can write back new tags to `apps-deployment` (`main` branch).
- Production uses separate overlays/app manifests and can be promoted independently.

## 7. Commands Used Most Often

Apply staging applications:

```bash
kubectl apply -f apps-deployment/argocd-apps/staging/
```

Apply production applications:

```bash
kubectl apply -f apps-deployment/argocd-apps/production/
```

List apps in both Argo CD namespaces:

```bash
kubectl get applications -n argocd-staging
kubectl get applications -n argocd-production
```

## 8. Notes

- App `repoURL` values are still placeholders (`https://github.com/your-org/apps-deployment.git`) in current manifests.
- If you use a different Argo CD control-plane namespace than `argocd-staging` / `argocd-production`, update `metadata.namespace` in app manifests accordingly.
