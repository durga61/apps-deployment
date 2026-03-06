# apps-deployment

GitOps repository containing Kubernetes manifests and Argo CD Application definitions for `cart`, `product`, and `order` services.

## Structure

- `base/` - reusable Kustomize base manifests per service.
- `overlays/staging/` - staging overlays for each service.
- `overlays/production/` - production overlays for each service.
- `argocd-apps/staging/` - staging Argo CD `Application` manifests.
- `argocd-apps/production/` - production Argo CD `Application` manifests.

## Current Behavior

- Base manifests default to `latest` image tags.
- Staging apps include Argo CD Image Updater annotations and write-back to Git (`main` branch).
- Production apps are separate and currently do not include Image Updater annotations.
- All apps have automated sync enabled (`prune` + `selfHeal`).

## Important Paths

- `base/<service>/kustomization.yaml` - image tag fields updated by Image Updater (staging flow).
- `argocd-apps/staging/*-app.yaml` - staging app definitions (`argocd-staging` namespace).
- `argocd-apps/production/*-app.yaml` - production app definitions (`argocd-production` namespace).

## Related Doc

- See `GITOPS_SETUP_EXPORT.md` for a detailed setup and flow summary aligned with current code.
