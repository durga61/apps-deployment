
# Commands

```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater

kubectl logs -n argocd deploy/argocd-image-updater-controller  --tail=200
```
