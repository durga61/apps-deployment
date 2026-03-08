# GitOps Microservices Promotion Pipeline - Setup Documentation

**Date:** March 3-4, 2026  
**Scenario:** Automated e-commerce microservices deployment with Argo CD and Image Updater

---

## Executive Summary

This document outlines a complete GitOps pipeline for a microservices architecture using:
- **Source Repos:** `cart-service`, `product-service`, `order-service` (application code)
- **Deployment Repo:** `apps-deployment` (Kubernetes manifests)
- **CI/CD:** GitHub Actions (build & push Docker images with short SHA)
- **GitOps:** Argo CD with Image Updater (automated image tag updates)
- **IaC:** Kustomize (base + overlays)
- **Registry:** Docker Hub (`durga61` namespace)

---

## Architecture Overview

### Repositories

| Repo | Purpose | Key Files |
|------|---------|-----------|
| `cart-service` | Microservice source code | `app.py`, `Dockerfile`, `.github/workflows/ci.yml` |
| `product-service` | Microservice source code | `app.py`, `Dockerfile`, `.github/workflows/ci.yml` |
| `order-service` | Microservice source code | `app.py`, `Dockerfile`, `.github/workflows/ci.yml` |
| `apps-deployment` | K8s manifests & Argo CD configs | `base/`, `overlays/`, `argocd-apps/` |

### Workflow Flow
```
┌─────────────────────────────────────────────────────────────────┐
│ Push to cart-service/main                                       │
└──────────────────────────┬──────────────────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │ GitHub Actions CI (cart-service)     │
    │ - Build Dockerfile                   │
    │ - Push to Docker Hub                 │
    │ - Tag: durga61/cart-service:<SHA7>   │
    └──────────────────────┬───────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────────┐
    │ Argo CD Image Updater detects new image      │
    │ - Monitors durga61/cart-service registry     │
    │ - newest-build strategy + allow-tags   │
    └──────────────────────┬──────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────────┐
    │ Image Updater commits to apps-deployment     │
    │ - Updates base/cart/kustomization.yaml       │
    │ - New image tag replaces old tag             │
    │ - Git commit to main branch                  │
    └──────────────────────┬──────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────────┐
    │ Argo CD detects apps-deployment change       │
    │ - Regenerates K8s manifests via kustomize    │
    │ - Syncs overlays/staging/cart to cluster     │
    │ - Cart service pod(s) updated                │
    └──────────────────────────────────────────────┘
```

---

## Repository Structure

### Microservice Repos (cart-service, product-service, order-service)

```
cart-service/
├── README.md
├── app.py                          # Simple Python application
├── Dockerfile                      # Python 3.11 image
└── .github/
    └── workflows/
        └── ci.yml                  # GitHub Actions workflow
```

#### GitHub Actions Workflow (`ci.yml`)

```yaml
name: CI

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to DockerHub
        uses: docker/login-action@v2
        with:
          registry: docker.io
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set short SHA
        run: echo "SHORT_SHA=${GITHUB_SHA::7}" >> $GITHUB_ENV

      - name: Build and push image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: durga61/<service-name>:${{ env.SHORT_SHA }}
```

**Secrets Required (GitHub repo settings):**
- `DOCKERHUB_USERNAME` - Docker Hub username
- `DOCKERHUB_TOKEN` - Docker Hub personal access token

**Image Tag Format:** `durga61/<service>:<7-char-git-sha>`  
Example: `durga61/cart-service:abc1234`

---

### Deployment Repo (apps-deployment)

```
apps-deployment/
├── README.md
├── base/                           # Kustomize bases
│   ├── cart/
│   │   ├── deployment.yaml
│   │   └── kustomization.yaml
│   ├── product/
│   │   ├── deployment.yaml
│   │   └── kustomization.yaml
│   └── order/
│       ├── deployment.yaml
│       └── kustomization.yaml
├── overlays/                       # Environment-specific overlays
│   ├── staging/
│   │   ├── cart/
│   │   │   └── kustomization.yaml
│   │   ├── product/
│   │   │   └── kustomization.yaml
│   │   └── order/
│   │       └── kustomization.yaml
│   └── production/
│       ├── cart/
│       │   └── kustomization.yaml
│       ├── product/
│       │   └── kustomization.yaml
│       └── order/
│           └── kustomization.yaml
└── argocd-apps/                    # One Argo CD Application per service
    ├── cart-app.yaml
    ├── product-app.yaml
    └── order-app.yaml
```

#### Base Kustomization Example (base/cart/kustomization.yaml)

```yaml
resources:
  - deployment.yaml

images:
  - name: durga61/cart-service
    newTag: latest
```

The `images:` section is where **Argo CD Image Updater** automatically updates the tag.

#### Base Deployment Example (base/cart/deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cart
  template:
    metadata:
      labels:
        app: cart
    spec:
      containers:
        - name: cart
          image: durga61/cart-service:latest
          ports:
            - containerPort: 8080
```

#### Overlay Example (overlays/staging/cart/kustomization.yaml)

```yaml
resources:
  - ../../../base/cart

# Staging-specific customizations can go here:
# replicas, resource limits, config maps, etc.
```

#### Argo CD Application Example (argocd-apps/cart-app.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cart
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: durga61/cart-service
    argocd-image-updater.argoproj.io/update-strategy: newest-build
    argocd-image-updater.argoproj.io/allow-tags: "true"
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/apps-deployment.git'
    targetRevision: main
    path: overlays/staging/cart
    plugin:
      name: kustomize
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: cart
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Key Annotations (Image Updater Configuration):**
- `argocd-image-updater.argoproj.io/image-list`: List of images to monitor (space-separated)
- `argocd-image-updater.argoproj.io/update-strategy`: Set to `newest-build` to use newest tag by build time
- `argocd-image-updater.argoproj.io/allow-tags`: Set to `"true"` to allow tag-based updates (e.g., SHA tags)

---

## Deployment & Configuration

### Prerequisites

1. **Kubernetes Cluster** with Argo CD installed
2. **GitHub Repositories** for each service and deployment
3. **Docker Hub Account** (durga61 namespace)
4. **GitHub Secrets** configured in each microservice repo:
   - `DOCKERHUB_USERNAME`
   - `DOCKERHUB_TOKEN`

### Step 1: Install Argo CD

```bash
kubectl create namespace argocd
#This is a known kubectl apply client-side issue with large CRDs (it tries to store a huge last-applied-configuration annotation).
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd

kubectl port-forward svc/argocd-server -n argocd 8080:443

You can now access the Argo CD web interface in your browser at https://localhost:8080.


Retrieve the initial admin password: The default password for the admin user is stored in a Kubernetes secret. Retrieve it using this command:

#Use git bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo


```

### Step 2: Install Argo CD Image Updater

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml

kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/config/install.yaml
```

### Step 3: Configure Image Updater Git Credentials

Image Updater needs permission to commit back to `apps-deployment`. Create a secret:

```bash
kubectl create secret generic argocd-image-updater-git-creds \
  -n argocd \
  --from-literal=username=<GITHUB_USERNAME> \
  --from-literal=password=<GITHUB_TOKEN>
```

Update the Image Updater config to use these credentials:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  git.user: <GITHUB_USERNAME>
  git.email: <GITHUB_EMAIL>
  registries.conf: |
    registries:
    - name: docker
      api_url: https://registry-1.docker.io
      ping: yes
      credentials: secret:default/docker-credentials
      insecure: no
      default: yes
```

### Step 4: Apply Argo CD Applications

```bash
kubectl apply -f apps-deployment/argocd-apps/staging/cart-app.yaml
kubectl apply -f apps-deployment/argocd-apps/staging/product-app.yaml
kubectl apply -f apps-deployment/argocd-apps/staging/order-app.yaml

kubectl delete -f apps-deployment/argocd-apps/staging/cart-app.yaml
kubectl delete -f apps-deployment/argocd-apps/staging/product-app.yaml
kubectl delete -f apps-deployment/argocd-apps/staging/order-app.yaml

kubectl apply -f apps-deployment/argocd-apps/staging/image-updater.yaml

```

Verify:

```bash
kubectl get applications -n argocd
```

---

## Testing the Pipeline

### Test Scenario: Update Cart Service

1. **Clone and modify cart-service:**
   ```bash
   git clone https://github.com/durga61/cart-service.git
   cd cart-service
   echo "# v2" >> app.py
   git add app.py
   git commit -m "test: add version marker"
   git push origin main
   ```

2. **Monitor GitHub Actions:**
   - Check `cart-service/.github/workflows/ci.yml` execution
   - Confirm Docker image pushed to `durga61/cart-service:<SHA7>`

3. **Monitor Argo CD Image Updater:**
   - Watch logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater -f`
   - Expect: Image Updater detects new tag, creates commit in `apps-deployment`
   - Check `apps-deployment` repo for updated `base/cart/kustomization.yaml`

4. **Monitor Argo CD:**
   - Watch Argo CD UI (port-forward: `kubectl port-forward -n argocd svc/argocd-server 8080:443`)
   - Cart application should sync automatically
   - New pod(s) launched with updated image tag

---

## Production Best Practices

For GitOps environments:

✔ Use GitHub App or PAT with minimal permissions
✔ Limit to argocd repo
✔ Use separate branch per environment

Example:

dev branch     → auto updated by ImageUpdater
staging branch → PR promotion
prod branch    → PR promotion

This prevents automatic production deployments.

## Scaling to Additional Microservices

To add a new microservice (e.g., `user-service`):

1. **Create app repo** from template: `user-service/`
   - Add `app.py`, `Dockerfile`, `.github/workflows/ci.yml`
   - Update image name in workflow: `durga61/user-service`

2. **Add to apps-deployment:**
   ```
   base/user/
   ├── deployment.yaml
   └── kustomization.yaml
   overlays/staging/user/kustomization.yaml
   overlays/production/user/kustomization.yaml
   argocd-apps/user-app.yaml
   ```

3. **Apply:** `kubectl apply -f apps-deployment/argocd-apps/user-app.yaml`

---

## Key Features & Benefits

✅ **Fully Automated Promotion:**  
Code push → Docker build/push → Image Updater detects → Git commit → Argo CD sync → Pod update

✅ **Git-Centric:**  
All infrastructure changes are version-controlled. Every deployment is traceable to a Git commit.

✅ **Isolated Services:**  
One Argo CD Application per microservice allows independent rollback and lifecycle management.

✅ **Kustomize Flexibility:**  
Reuse base manifests across environments with easy overlays for staging/production divergences.

✅ **Immutable Tags:**  
Short SHA tags provide traceability and prevent accidental tag reuse.

✅ **Autosync:**  
Argo CD automatically syncs when manifests change, ensuring cluster matches Git.

---

## Troubleshooting

### Image Updater Not Detecting Tags

**Check:**
- Allowed tags regex in annotations: `argocd-image-updater.argoproj.io/allow-tags: "true"`
- Update strategy: `argocd-image-updater.argoproj.io/update-strategy: newest-build`
- Image list is correct: `argocd-image-updater.argoproj.io/image-list: durga61/service`

**Debug:**
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-image-updater
```

### Image Updater Cannot Commit to Git

**Check:**
- Git credentials secret exists: `kubectl get secret -n argocd argocd-image-updater-git-creds`
- GitHub token has `repo` scope

**Verify permissions:**
```bash
kubectl describe secret -n argocd argocd-image-updater-git-creds
```

### Argo CD Application Not Syncing

**Check:**
- Application status: `kubectl describe app <app-name> -n argocd`
- Repository URL is accessible
- Path exists in repo (e.g., `overlays/staging/cart/`)

**Force sync:**
```bash
kubectl patch app <app-name> -n argocd -p '{"spec":{"syncPolicy":{}}}' --type merge
```

---

## Files Generated

### Microservice Repos (each has)

- `README.md` - Documentation
- `app.py` - Application code
- `Dockerfile` - Container image definition
- `.github/workflows/ci.yml` - GitHub Actions pipeline

### Deployment Repo (apps-deployment)

- `README.md` - Documentation
- `base/{cart,product,order}/*` - Kustomize bases with image definitions
- `overlays/{staging,production}/{cart,product,order}/*` - Environment overlays
- `argocd-apps/{cart,product,order}-app.yaml` - Argo CD Application manifests with Image Updater annotations

---

## Future Scope & Enhancements

### 1. **Multi-Environment Promotion Strategy**

**Current State:** Staging overlay in use; production overlay created but not actively deployed.

**Future Enhancements:**
- **Automated Promotion Pipeline:** Create a workflow that promotes from staging → production after successful testing
  - Use Argo CD ApplicationSet to manage multiple environment instances
  - Implement approval gates via GitHub pull requests or Argo CD notifications
  
- **Canary/Blue-Green Deployments:**
  - Integrate Flagger with Argo CD for canary rollouts
  - Progressive traffic shifting on production services
  
- **Example Implementation:**
  ```yaml
  # argocd-apps/cart-app-production.yaml
  apiVersion: argoproj.io/v1alpha1
  kind: Application
  metadata:
    name: cart-production
    namespace: argocd
  spec:
    source:
      path: overlays/production/cart
    destination:
      namespace: production
  # Promotion triggered via PR merge from staging manifest
  ```

---

### 2. **Advanced Image Updater Strategies**

**Current State:** `newest-build` with allow-tags enabled for SHA-based tagging.

**Future Enhancements:**
- **Semantic Versioning Support:**
  - Switch to `semver` update strategy for version-tagged releases
  - Enable filtering: `major.minor.patch` patterns (e.g., `v1.2.x`)
  - Maintain separate channels (stable, beta, RC)

- **Custom Registry Configuration:**
  - Support multiple registries (ECR, GCR, private registries)
  - Registry-specific credentials per microservice

- **Automated Dependency Updates:**
  - Monitor base image versions (Python 3.11 → 3.12)
  - Auto-update Dockerfile base images with Image Updater

- **Example Configuration:**
  ```yaml
  annotations:
    argocd-image-updater.argoproj.io/image-list: durga61/cart-service
    argocd-image-updater.argoproj.io/update-strategy: semver
    argocd-image-updater.argoproj.io/image-list.semver.range: "~1.2.0"  # Allow 1.2.x
  ```

---

### 3. **GitOps Security Enhancements**

**Current State:** Basic GitHub Actions secrets; minimal RBAC.

**Future Enhancements:**
- **Secret Management:**
  - Sealed Secrets or External Secrets Operator for managing K8s secrets
  - HashiCorp Vault integration for centralized secret rotation
  - Encrypted secrets in Git repos (sops + kustomize)

- **RBAC & Access Control:**
  - Argo CD projects per team/environment
  - AppProject with restrictions on namespaces/resources
  - GitHub SAML/OIDC for Argo CD UI access

- **Audit & Compliance:**
  - Argo CD audit logs shipped to centralized logging (ELK, Loki)
  - Git commit verification (GPG signed commits)
  - Compliance scanning integration (Falco, OPA/Gatekeeper)

- **Example AppProject:**
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: AppProject
  metadata:
    name: ecommerce
    namespace: argocd
  spec:
    destinations:
      - namespace: "staging"
        server: "https://kubernetes.default.svc"
      - namespace: "production"
        server: "https://kubernetes.default.svc"
    sourceRepos:
      - "https://github.com/durga61/*"
  ```

---

### 4. **CI/CD Pipeline Enhancements**

**Current State:** GitHub Actions builds Docker image and pushes to hub.docker.com.

**Future Enhancements:**
- **Multi-Stage Docker Builds:**
  - Reduce image size with builder patterns
  - Scan images for vulnerabilities (Trivy, Grype)
  - Sign images (Cosign, Notary)

- **Unit & Integration Testing:**
  - Run tests before Docker build
  - Push to registry only on passing tests
  - Report coverage metrics

- **Release Automation:**
  - Semantic versioning with `commitlint` + `semantic-release`
  - Generate changelogs automatically
  - Draft GitHub releases

- **Container Registry Security:**
  - Image vulnerability scanning before push
  - Enforce signed images in Kubernetes (Connaisseur, Portieris)
  - Image lifecycle policies (auto-delete old tags)

- **Example Enhanced Workflow:**
  ```yaml
  - name: Run tests
    run: |
      python -m pytest tests/
      python -m coverage report
  
  - name: Scan image with Trivy
    uses: aquasecurity/trivy-action@master
    with:
      image-ref: ${{ env.IMAGE_TAG }}
      exit-code: '1'  # Fail if vulnerabilities found
  
  - name: Sign image with Cosign
    run: |
      cosign sign --key cosign.key ${{ env.IMAGE_TAG }}
  ```

---

### 5. **Observability & Monitoring**

**Current State:** Logs checked manually via kubectl logs.

**Future Enhancements:**
- **Distributed Tracing:**
  - Integrate Jaeger or Tempo for end-to-end request tracing
  - Trace deployments and rollbacks

- **Metrics & Dashboards:**
  - Prometheus for Kubernetes & Argo CD metrics
  - Grafana dashboards showing:
    - Deployment frequency, lead time, failure rate, MTTR (DORA metrics)
    - Argo CD application sync status & health
    - Image Updater activity (commits, errors)

- **Alerting:**
  - Alert on failed syncs, high error rates, slow deployments
  - Slack/PagerDuty notifications for incidents

- **Log Aggregation:**
  - ELK Stack or Loki for centralized logs
  - Filter logs by service, environment, or tag

- **Example Prometheus ServiceMonitor:**
  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: argocd
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/name: argocd-metrics
    endpoints:
      - port: metrics
  ```

---

### 6. **Service Mesh Integration**

**Current State:** Deployments exposed via standard Kubernetes Services.

**Future Enhancements:**
- **Istio or Linkerd:**
  - Virtual Services for traffic management
  - Kustomize overlays for mesh-specific configs
  - mTLS by default

- **Traffic Management:**
  - Canary deployments with automatic rollback
  - Circuit breaking, retries, timeouts
  - A/B testing via header-based routing

- **Observability:**
  - Built-in metrics and tracing (Jaeger integration)
  - Service topology visualization (Kiali)

- **Example Istio VirtualService:**
  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: VirtualService
  metadata:
    name: cart
  spec:
    hosts:
      - cart
    http:
      - match:
          - uri:
              prefix: "/v2"
        route:
          - destination:
              host: cart
              subset: v2
            weight: 10
          - destination:
              host: cart
              subset: v1
            weight: 90
  ```

---

### 7. **Infrastructure as Code (IaC) Expansion**

**Current State:** Kubernetes manifests managed manually; cluster setup assumed.

**Future Enhancements:**
- **Cluster Provisioning:**
  - Terraform for AWS EKS, GKE, or AKS provisioning
  - Helm for Argo CD, Image Updater, ingress controller installation
  - GitOps bootstrap (Flux or Argo App-of-Apps pattern)

- **Networking:**
  - Ingress Controller (NGINX, Envoy) managed via Kustomize
  - DNS & TLS certificate management (cert-manager)
  - Network policies for service-to-service communication

- **Example Terraform:**
  ```hcl
  resource "aws_eks_cluster" "ecommerce" {
    name            = "ecommerce-cluster"
    role_arn       = aws_iam_role.eks.arn
    vpc_config {
      subnet_ids = [aws_subnet.private.*.id]
    }
  }
  ```

---

### 8. **Cost Optimization & Resource Management**

**Current State:** Static replica counts; no resource limits defined.

**Future Enhancements:**
- **Autoscaling:**
  - Horizontal Pod Autoscaler (HPA) based on CPU/memory
  - Vertical Pod Autoscaler (VPA) for right-sizing
  - Workload-specific scaling policies via kustomize overlays

- **Cost Analysis:**
  - Kubecost or OpenCost for cost allocation per service/env
  - Budget alerts and chargeback models

- **Right-Sizing:**
  - Container resource limits in kustomize bases
  - Dev/staging resource limits lower than production

- **Example HPA in Overlay:**
  ```yaml
  # overlays/production/cart/hpa.yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: cart-hpa
  spec:
    scaleTargetRef:
      kind: Deployment
      name: cart
    minReplicas: 3
    maxReplicas: 10
    metrics:
      - type: Resource
        resource:
          name: cpu
          target:
            type: Utilization
            averageUtilization: 70
  ```

---

### 9. **Disaster Recovery & Backup**

**Current State:** No backup/restore strategy; assumes Git is source of truth.

**Future Enhancements:**
- **Cluster Backup:**
  - Velero for persistent volume snapshots, cluster state backup
  - Automated backup schedule and retention policies

- **GitOps Recovery:**
  - Disaster recovery procedures (re-bootstrap from Git)
  - Testing backups regularly (DR drills)

- **Secrets Backup:**
  - Separate encrypted backup for secrets (not in Git)
  - Key rotation strategies

---

### 10. **Multi-Cluster & High Availability**

**Current State:** Single cluster; single Argo CD instance.

**Future Enhancements:**
- **Multi-Cluster Management:**
  - Argo CD managing multiple clusters (staging, prod, disaster-recovery)
  - ApplicationSet with cluster-specific parameters

- **High Availability:**
  - Multiple Argo CD replicas with PostgreSQL backend
  - Load-balanced Argo CD server

- **GitOps at Scale:**
  - Helm for templating complex deployments
  - Argo CD notifications to multiple Slack channels/teams
  - Custom Argo CD plugins for enterprise tools integration

- **Example ApplicationSet (multi-cluster):**
  ```yaml
  apiVersion: argoproj.io/v1alpha1
  kind: ApplicationSet
  metadata:
    name: cart-all-clusters
  spec:
    generators:
      - clusters: {}
    template:
      metadata:
        name: 'cart-{{name}}'
      spec:
        source:
          repoURL: https://github.com/durga61/apps-deployment
          path: overlays/{{name}}/cart
        destination:
          server: '{{server}}'
  ```

---

### 11. **Policy Enforcement & Governance**

**Current State:** No policy validation; trust-based deployment.

**Future Enhancements:**
- **Policy as Code:**
  - OPA/Gatekeeper for admission control (enforce image registries, labels, etc.)
  - Kyverno for simplified Kubernetes-native policies
  - ValidatingAdmissionPolicy (K8s 1.26+)

- **Compliance Scanning:**
  - Sonobuoy for conformance testing
  - CIS benchmarks for security hardening

- **Policy Example (OPA/Gatekeeper):**
  ```rego
  # Enforce image comes from durga61 registry
  deny[msg] {
    image := input.review.object.spec.containers[_].image
    not startswith(image, "durga61/")
    msg := sprintf("Image %v must come from durga61 registry", [image])
  }
  ```

---

### 12. **Documentation & Knowledge Base**

**Current State:** README files in repos; minimal runbooks.

**Future Enhancements:**
- **Runbooks & Playbooks:**
  - Emergency rollback procedures
  - Performance troubleshooting guides
  - Disaster recovery procedures

- **Architecture Decision Records (ADRs):**
  - Document why certain tools/patterns chosen
  - Evolve decisions as needs change

- **Developer Onboarding:**
  - Automated setup scripts for new developers
  - CI/CD best practices guide

- **Automated Documentation:**
  - Generate docs from code (docstrings, comments)
  - Architecture diagrams from manifests (diagrams-as-code)

---

## Implementation Roadmap (Suggested Phases)

| Phase | Timeline | Focus | Benefits |
|-------|----------|-------|----------|
| **Phase 1** | Weeks 1-4 | Multi-env promotion, RBAC, Sealed Secrets | Production-ready security; automated staging→prod |
| **Phase 2** | Weeks 5-8 | Image scanning, SAST/DAST, code signing | Vulnerability detection; supply chain security |
| **Phase 3** | Weeks 9-12 | Observability (Prometheus/Grafana), Jaeger | Visibility into deployments & application health |
| **Phase 4** | Weeks 13-16 | Service Mesh (Istio), Canary deployments | Traffic management; safer rollouts |
| **Phase 5** | Weeks 17-20 | Multi-cluster setup, AppSets, Helm | Global scale; resilience |
| **Phase 6** | Weeks 21+ | Cost optimization, policy enforcement, DR | Long-term sustainability; compliance |

---

## References

- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [GitHub Actions Docker Build](https://github.com/docker/build-push-action)
- [OPA/Gatekeeper](https://open-policy-agent.github.io/gatekeeper/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Flagger](https://flagger.app/)
- [Velero](https://velero.io/)
- [Istio Service Mesh](https://istio.io/)
- [DORA Metrics](https://cloud.google.com/architecture/devops-measurement-caste)

---

**Document Generated:** March 4, 2026  
**Status:** Complete setup exported for reference and reuse  
**Future Scope Added:** Comprehensive roadmap with 12 enhancement areas