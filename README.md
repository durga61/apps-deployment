# apps-deployment

GitOps repository containing Kubernetes manifests and Argo CD application definitions.

## Structure

- `base/` - reusable kustomize bases for each microservice
- `overlays/` - environment-specific overlays (staging, production)
- `argocd-apps/staging/` - Argo CD `Application` manifests for staging
- `argocd-apps/production/` - Argo CD `Application` manifests for production

This repo is the target of Argo CD. Image Updater commits changes back here when new container tags appear.
