# apps-deployment

GitOps repository containing Kubernetes manifests and Argo CD application definitions.

## Structure

- `base/` - reusable kustomize bases for each microservice
- `overlays/` - environment-specific overlays (staging, production)
- `argocd-apps/` - one Argo CD `Application` manifest per microservice

This repo is the target of Argo CD. Image Updater commits changes back here when new container tags appear.
