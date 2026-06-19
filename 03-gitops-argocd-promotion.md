# GitOps with ArgoCD — Environment Promotion Patterns

> How to promote from dev → staging → production safely, with real ArgoCD config included.

---

## Table of Contents

- [Why GitOps Promotion Fails in Practice](#why-gitops-promotion-fails-in-practice)
- [Repository Structure — Two Patterns](#repository-structure--two-patterns)
- [ArgoCD App of Apps Pattern](#argocd-app-of-apps-pattern)
- [Environment Promotion Flow](#environment-promotion-flow)
- [Promotion Strategy 1 — Branch-per-Environment](#promotion-strategy-1--branch-per-environment)
- [Promotion Strategy 2 — Directory-per-Environment (Recommended)](#promotion-strategy-2--directory-per-environment-recommended)
- [Automating Promotion with GitHub Actions](#automating-promotion-with-github-actions)
- [Rollback Patterns](#rollback-patterns)
- [Secrets Management Across Environments](#secrets-management-across-environments)
- [Production Checklist Before Promotion](#production-checklist-before-promotion)

---

## Why GitOps Promotion Fails in Practice

GitOps sounds simple: commit to Git → ArgoCD syncs to cluster. But production teams hit these walls:

```
❌ Dev, staging, prod drift over time — no one knows what's actually deployed where
❌ Hotfixes go to prod without going through staging
❌ Image tags get manually updated and the commit gets lost
❌ Secrets are hardcoded in manifests (git history exposure)
❌ No automated gate between environments — staging is just "prod but less scary"
❌ Rollback is unclear — which commit? which image tag?
```

This guide solves all of these with opinionated patterns that work at scale.

---

## Repository Structure — Two Patterns

### Option A — Monorepo (app code + manifests together)

```
my-service/
├── src/                        # Application code
├── Dockerfile
├── deploy/
│   ├── base/                   # Kustomize base
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   └── overlays/
│       ├── dev/
│       │   ├── kustomization.yaml
│       │   └── patch-replicas.yaml
│       ├── staging/
│       │   ├── kustomization.yaml
│       │   └── patch-replicas.yaml
│       └── production/
│           ├── kustomization.yaml
│           └── patch-replicas.yaml
```

### Option B — Separate GitOps repo (recommended for teams)

```
infrastructure-gitops/           # Separate repo
├── apps/
│   ├── my-service/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── production/
│   └── another-service/
├── argocd/
│   ├── app-of-apps-dev.yaml
│   ├── app-of-apps-staging.yaml
│   └── app-of-apps-production.yaml
└── clusters/
    ├── dev/
    ├── staging/
    └── production/
```

> **Recommendation:** Use Option B. Separating app code from deployment config means:
> - CI pipelines in the app repo build and test
> - CD pipelines in the GitOps repo deploy
> - No accidental production deploys from a feature branch

---

## ArgoCD App of Apps Pattern

Instead of managing ArgoCD Applications one by one, use a parent Application that manages all others.

```yaml
# argocd/app-of-apps-production.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production
  source:
    repoURL: https://github.com/your-org/infrastructure-gitops
    targetRevision: main
    path: clusters/production
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# clusters/production/my-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service-production
  namespace: argocd
spec:
  project: production
  source:
    repoURL: https://github.com/your-org/infrastructure-gitops
    targetRevision: main
    path: apps/my-service/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: false          # Never auto-prune in production
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
```

---

## Environment Promotion Flow

```
Developer pushes code
        ↓
CI builds & tests (GitHub Actions)
        ↓
CI pushes image to registry (tag: sha-abc1234)
        ↓
CI updates dev overlay (kustomization.yaml image tag)
        ↓
ArgoCD detects change → syncs to dev cluster
        ↓
Automated smoke tests run against dev
        ↓
Manual approval OR automated gate (e2e tests pass)
        ↓
Promotion script updates staging overlay (same image tag)
        ↓
ArgoCD syncs to staging cluster
        ↓
QA sign-off / automated integration tests
        ↓
Production promotion (PR → review → merge → ArgoCD syncs)
```

---

## Promotion Strategy 1 — Branch-per-Environment

```
main branch    → deploys to production
staging branch → deploys to staging
dev branch     → deploys to dev
```

```yaml
# ArgoCD app for staging watches staging branch
spec:
  source:
    targetRevision: staging   # ← branch name
    path: apps/my-service/overlays/staging
```

**Promotion = merge:**
```bash
# Promote staging → production
git checkout main
git merge staging --no-ff -m "chore: promote staging to production [my-service v1.2.3]"
git push origin main
```

**Downsides:**
- Branch conflicts accumulate
- Hard to track what version is where
- Cherry-picks get messy for hotfixes

---

## Promotion Strategy 2 — Directory-per-Environment (Recommended)

All environments live on `main`. Image tags are pinned per environment in Kustomize overlays.

```yaml
# apps/my-service/overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: your-registry/my-service
    newTag: sha-abc1234   # ← CI updates this automatically
patches:
  - path: patch-replicas.yaml
```

```yaml
# apps/my-service/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: your-registry/my-service
    newTag: sha-def5678   # ← only updated after staging validation
patches:
  - path: patch-replicas.yaml
  - path: patch-hpa.yaml
```

**Promotion = updating the image tag in the overlay via a PR.**

---

## Automating Promotion with GitHub Actions

### CI — Build, test, and deploy to dev automatically:

```yaml
# .github/workflows/ci-deploy-dev.yml
name: CI → Deploy to Dev

on:
  push:
    branches: [main]
    paths: ['src/**', 'Dockerfile']

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        run: |
          IMAGE_TAG=sha-${GITHUB_SHA::7}
          docker build -t your-registry/my-service:${IMAGE_TAG} .
          docker push your-registry/my-service:${IMAGE_TAG}
          echo "IMAGE_TAG=${IMAGE_TAG}" >> $GITHUB_ENV

      - name: Checkout GitOps repo
        uses: actions/checkout@v4
        with:
          repository: your-org/infrastructure-gitops
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      - name: Update dev image tag
        run: |
          cd gitops
          # Update the image tag in dev overlay
          sed -i "s|newTag:.*|newTag: ${IMAGE_TAG}|" \
            apps/my-service/overlays/dev/kustomization.yaml
          git config user.email "ci@yourorg.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore(dev): update my-service to ${IMAGE_TAG}"
          git push
```

### Promotion — Staging with manual approval:

```yaml
# .github/workflows/promote-to-staging.yml
name: Promote to Staging

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to promote (e.g. sha-abc1234)'
        required: true

jobs:
  promote:
    runs-on: ubuntu-latest
    environment: staging   # ← requires GitHub Environment approval
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/infrastructure-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Update staging image tag
        run: |
          sed -i "s|newTag:.*|newTag: ${{ github.event.inputs.image_tag }}|" \
            apps/my-service/overlays/staging/kustomization.yaml
          git config user.email "ci@yourorg.com"
          git config user.name "CI Bot"
          git add .
          git commit -m "chore(staging): promote my-service to ${{ github.event.inputs.image_tag }}"
          git push

      - name: Wait for ArgoCD sync
        run: |
          # Install argocd CLI
          curl -sSL -o /usr/local/bin/argocd \
            https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
          chmod +x /usr/local/bin/argocd
          argocd login ${{ secrets.ARGOCD_SERVER }} \
            --auth-token ${{ secrets.ARGOCD_TOKEN }} --insecure
          argocd app wait my-service-staging --timeout 300
```

### Production promotion via Pull Request:

```yaml
# .github/workflows/promote-to-production.yml
name: Promote to Production

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: 'Image tag to promote to production'
        required: true

jobs:
  create-pr:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          repository: your-org/infrastructure-gitops
          token: ${{ secrets.GITOPS_TOKEN }}

      - name: Create promotion branch and PR
        run: |
          TAG=${{ github.event.inputs.image_tag }}
          BRANCH="promote/my-service-${TAG}-to-prod"
          git checkout -b ${BRANCH}
          sed -i "s|newTag:.*|newTag: ${TAG}|" \
            apps/my-service/overlays/production/kustomization.yaml
          git add .
          git commit -m "chore(prod): promote my-service to ${TAG}"
          git push origin ${BRANCH}

      - name: Open Pull Request
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITOPS_TOKEN }}
          script: |
            const pr = await github.rest.pulls.create({
              owner: 'your-org',
              repo: 'infrastructure-gitops',
              title: 'chore(prod): promote my-service to ${{ github.event.inputs.image_tag }}',
              head: `promote/my-service-${{ github.event.inputs.image_tag }}-to-prod`,
              base: 'main',
              body: `## Production Promotion\n\n- **Service:** my-service\n- **Image:** ${{ github.event.inputs.image_tag }}\n- **Validated in staging:** ✅\n\nReview and merge to deploy to production.`
            });
            console.log('PR created:', pr.data.html_url);
```

---

## Rollback Patterns

### Option 1 — Revert the GitOps commit (recommended)

```bash
# Find the last good commit
git log --oneline apps/my-service/overlays/production/

# Revert the bad promotion commit
git revert <bad-commit-sha> --no-edit
git push origin main

# ArgoCD detects the change and rolls back automatically
```

### Option 2 — ArgoCD rollback to previous sync

```bash
# List sync history
argocd app history my-service-production

# Roll back to specific revision
argocd app rollback my-service-production <revision-id>

# ⚠️ This puts the app out of sync with Git
# You must also update the Git overlay to match, or ArgoCD will re-sync forward
```

### Option 3 — Emergency rollback script

```bash
#!/bin/bash
# rollback.sh <service> <environment> <image-tag>
SERVICE=$1
ENV=$2
TAG=$3

cd infrastructure-gitops
git pull origin main

OVERLAY="apps/${SERVICE}/overlays/${ENV}/kustomization.yaml"
sed -i "s|newTag:.*|newTag: ${TAG}|" ${OVERLAY}

git add .
git commit -m "chore(${ENV}): ROLLBACK ${SERVICE} to ${TAG}"
git push origin main

echo "Rollback committed. ArgoCD will sync in ~30 seconds."
echo "Monitor: argocd app watch ${SERVICE}-${ENV}"
```

---

## Secrets Management Across Environments

**Never put secrets in GitOps manifests.** Use one of these patterns:

### Option A — External Secrets Operator + AWS Secrets Manager

```yaml
# apps/my-service/base/external-secret.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-service-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: my-service-secrets    # Creates a K8s Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: /myapp/production/database
        property: password
    - secretKey: API_KEY
      remoteRef:
        key: /myapp/production/api
        property: key
```

### Option B — Sealed Secrets (encrypt in Git)

```bash
# Install kubeseal
kubeseal --fetch-cert --controller-name=sealed-secrets \
  --controller-namespace=kube-system > pub-cert.pem

# Seal a secret
kubectl create secret generic my-secret \
  --from-literal=password=mysecretpassword \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-cert.pem --format yaml > sealed-secret.yaml

# This sealed-secret.yaml is safe to commit to Git
# Only your cluster can decrypt it
```

---

## Production Checklist Before Promotion

```
✅ Image tag is pinned to a specific SHA (not :latest or :main)
✅ The same image ran in staging for at least 30 minutes without errors
✅ Automated tests passed in staging
✅ Resource requests and limits are set on all containers
✅ Liveness and readiness probes are configured
✅ HPA (HorizontalPodAutoscaler) is configured for production
✅ PodDisruptionBudget is in place
✅ Rollback procedure is documented and tested
✅ On-call engineer is aware of the deployment
✅ Monitoring dashboards are open during rollout
✅ ArgoCD sync policy is set to manual for production (no auto-sync)
```

---

## When GitOps Gets Complex at Scale

Managing ArgoCD across multiple clusters, teams, and environments — with RBAC, ApplicationSets, and multi-tenant isolation — is a full platform engineering effort.

> **Sygitech's [DevOps & Automation Services](https://www.sygitech.com/devops-and-automation-services.html)** design and implement production-grade GitOps pipelines for growing engineering teams — from single-cluster setups to multi-region, multi-tenant platforms.
>
> 👉 [Talk to our DevOps engineers](https://www.sygitech.com/devops-and-automation-services.html)


