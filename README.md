# Mosaic Life GitOps Repository

This repository contains **environment-specific configuration values** for Mosaic Life deployments.

## Architecture

This repo works in conjunction with the [mosaic-life](https://github.com/mosaic-stories/mosaic-life) application repository using **ArgoCD Multi-Source Applications**.

### Repository Separation

- **Application Repo** (`mosaic-life`): Helm charts, application code
- **GitOps Repo** (`gitops`): Environment-specific values, configuration

### Structure

```
gitops/
├── base/
│   └── values.yaml              # Common values across all environments
└── environments/
    ├── prod/
    │   └── values.yaml          # Production overrides
    └── staging/
        └── values.yaml          # Staging overrides
```

## How It Works

ArgoCD combines:
1. **Helm Chart** from `mosaic-life/infra/helm/mosaic-life/`
2. **Base Values** from `gitops/base/values.yaml`
3. **Environment Values** from `gitops/environments/{env}/values.yaml`

### Value Precedence (last wins)

1. Chart defaults (`mosaic-life` repo)
2. Base values (this repo - `base/values.yaml`)
3. Environment values (this repo - `environments/{env}/values.yaml`)

## Making Changes

### Scaling Applications

```bash
# Edit environment-specific values
vim environments/prod/values.yaml

# Example: Scale web service
web:
  replicaCount: 5
  autoscaling:
    minReplicas: 5
    maxReplicas: 20

# Commit and push
git add environments/prod/values.yaml
git commit -m "ops: scale prod web to 5 replicas"
git push
```

ArgoCD will automatically detect changes and sync (if auto-sync is enabled).

### Changing Image Tags

```bash
# Update image tag for an environment
vim environments/staging/values.yaml

global:
  imageTag: v1.2.3-rc1

git add environments/staging/values.yaml
git commit -m "ops: deploy v1.2.3-rc1 to staging"
git push
```

### Adding Common Configuration

```bash
# Edit base values (applies to all environments)
vim base/values.yaml

# Add new common config
global:
  newFeatureFlag: true

git add base/values.yaml
git commit -m "ops: enable new feature across all environments"
git push
```

## Environment Details

### Production (`environments/prod/`)

- **Namespace**: `mosaic-prod`
- **Domain**: `mosaiclife.me`
- **High Availability**: Yes
- **Auto-scaling**: Enabled
- **Git Branch**: `main`

### Staging (`environments/staging/`)

- **Namespace**: `mosaic-staging`
- **Domain**: `staging.mosaiclife.me`
- **High Availability**: Limited
- **Auto-scaling**: Disabled
- **Git Branch**: `main`

## ArgoCD Integration

Applications in ArgoCD reference this repository for values:

```yaml
sources:
  # Helm chart from app repo
  - repoURL: https://github.com/mosaic-stories/mosaic-life
    path: infra/helm/mosaic-life
    helm:
      valueFiles:
        - $values/environments/prod/values.yaml
        - $values/base/values.yaml
  
  # Values from this GitOps repo
  - repoURL: https://github.com/mosaic-stories/gitops.git
    ref: values
```

## Quick Commands

```bash
# View current production configuration
cat environments/prod/values.yaml

# Compare staging vs production
diff environments/staging/values.yaml environments/prod/values.yaml

# Check what's deployed
argocd app get mosaic-life-prod
argocd app get mosaic-life-staging

# Sync manually (if auto-sync disabled)
argocd app sync mosaic-life-prod
```

## Best Practices

1. **Always commit changes** - Never edit live resources directly
2. **Use descriptive commit messages** - Ops team needs clear audit trail
3. **Test in staging first** - Validate changes before prod
4. **Keep secrets in AWS Secrets Manager** - Never commit secrets
5. **Review PRs** - Get peer review for production changes

## References

- [Main Application Repo](https://github.com/mosaic-stories/mosaic-life)
- [GitOps Setup Guide](https://github.com/mosaic-stories/mosaic-life/blob/main/docs/ops/GITOPS-SETUP.md)
- [ArgoCD Multi-Source Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)

## SHA-Based Deployments

### How It Works

Images are tagged with the **git commit SHA** (7 characters):

```
033691785857.dkr.ecr.us-east-1.amazonaws.com/mosaic-life/web:abc1234
```

### Automatic Deployment

When code is merged to `main` or `develop`:

1. ✅ GitHub Actions builds images with SHA tag
2. ✅ Automatically updates `global.imageTag` in this repo
3. ✅ ArgoCD detects change and deploys

### Manual Deployment

Deploy a specific SHA to production:

```bash
# In the mosaic-life repo
cd /apps/mosaic-life
just deploy-sha abc1234 prod
```

### Rollback

Find and deploy a previous SHA:

```bash
# View deployment history
git log environments/prod/values.yaml

# Deploy previous version
cd /apps/mosaic-life
just deploy-sha xyz5678 prod
```

### Current Deployments

Check what's currently deployed:

```bash
# Production
cat environments/prod/values.yaml | grep imageTag

# Staging
cat environments/staging/values.yaml | grep imageTag
```

## Image Tag Updates

The `global.imageTag` value is automatically updated by CI/CD:

```yaml
global:
  imageTag: "abc1234"  # ← Updated by GitHub Actions
```

**Do not manually edit** unless performing a manual deployment or rollback.

## Deployment Verification

After a deployment commit is pushed to this repo:

```bash
# Watch ArgoCD sync (from mosaic-life repo)
just argocd-watch mosaic-life-prod

# Check pod status
kubectl get pods -n mosaic-prod

# Verify image tags
kubectl get pods -n mosaic-prod -o jsonpath='{.items[*].spec.containers[*].image}'
```

## Troubleshooting

### Image Not Found

If pods show `ImagePullBackOff`:

1. Check if image exists in ECR
2. Verify GitHub Actions build completed
3. Ensure SHA tag matches exactly

### Deployment Not Triggering

If ArgoCD isn't syncing after update:

1. Check ArgoCD application status
2. Verify auto-sync is enabled
3. Manually trigger sync if needed

See [SHA-Based Deployments Guide](https://github.com/mosaic-stories/mosaic-life/blob/main/docs/cicd/SHA-BASED-DEPLOYMENTS.md) for details.
