# Paved Road Service - Configuration

Environment-specific configuration values for the paved-road-service deployment across all environments and flavors.

## Repository Purpose

This repository contains **ONLY** environment-specific Helm values. It does NOT contain:
- Helm chart templates (see `helm-charts-paved-road` repository)
- ArgoCD/Kargo GitOps configuration (see `gitops-platform` repository)

## Directory Structure

```
environments/
├── dev/
│   ├── dev.yaml          # Default dev environment
│   └── devprod.yaml      # Shadow production (follows prod)
├── lab/
│   ├── qa1.yaml          # QA environment 1
│   ├── qa2.yaml          # QA environment 2
│   ├── qa3.yaml          # QA environment 3
│   └── qaprod.yaml       # Shadow production (follows prod)
└── prod/
    ├── preview.yaml      # Pre-production (deploys before prod)
    ├── prod.yaml         # Production default
    ├── pp1.yaml          # Production variant 1
    ├── apptest1.yaml     # Application testing 1
    └── apptest2.yaml     # Application testing 2
```

## Deployment Flavors

### Development Environment
- **dev**: Default development environment with auto-sync enabled
- **devprod**: Shadow production - mirrors production configuration for testing

### Lab Environment (QA)
- **qa1-qa3**: Three QA environments for parallel testing
- **qaprod**: Shadow production - mirrors production configuration

### Production Environment
- **preview**: Pre-production validation (sync wave 0 - deploys first)
- **prod**: Production default (sync wave 1 - deploys after preview)
- **pp1**: Production variant for A/B testing
- **apptest1-apptest2**: Application-level testing in production

## How It Works

### Kargo Updates Image Tags

Kargo watches the container registry for new images and automatically updates the `image.tag` field:

```yaml
# Before Kargo promotion
image:
  tag: ""

# After Kargo promotion
image:
  tag: "sha-abc123"  # or "v1.2.3"
```

**Important**: Do NOT manually edit `image.tag` - Kargo manages this field.

### ArgoCD Reads Values

ArgoCD ApplicationSet references this repository for environment-specific configuration:

```yaml
sources:
  - chart: paved-road-service
    repoURL: oci://registry.cloudwalkersinc.com/helm-charts
    targetRevision: 0.1.0
    helm:
      valueFiles:
        - $values/environments/prod/prod.yaml  # References this repo
  - repoURL: https://github.com/cloudwalkersinc/paved-road-service-config.git
    targetRevision: main
    ref: values
```

## Example Configuration

### Dev Environment (environments/dev/dev.yaml)

```yaml
replicaCount: 1

image:
  repository: cloudwalkersinc/paved-road-service
  tag: ""  # Kargo fills this

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: paved-road-dev.cloudwalkersinc.com
```

### Production Environment (environments/prod/prod.yaml)

```yaml
replicaCount: 3

image:
  repository: cloudwalkersinc/paved-road-service
  tag: ""  # Kargo fills this

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 2000m
    memory: 2Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: paved-road.cloudwalkersinc.com
  tls:
    - secretName: paved-road-tls
      hosts:
        - paved-road.cloudwalkersinc.com
```

## Updating Configuration

### Developer Workflow

1. **Edit environment values**:
   ```bash
   git checkout -b config/update-resources
   vim environments/prod/prod.yaml
   ```

2. **Commit and push**:
   ```bash
   git add environments/prod/prod.yaml
   git commit -m "Increase production memory limits"
   git push origin config/update-resources
   ```

3. **Create Pull Request** for review

4. **Merge to main** - ArgoCD auto-syncs changes

### Kargo Workflow (Automated)

1. **New image pushed** to container registry
2. **Kargo Warehouse detects** new version
3. **Kargo updates** `image.tag` in dev.yaml
4. **Kargo commits** to this repository
5. **ArgoCD syncs** and deploys to dev environment
6. **Manual promotion** (or auto) to QA → preview → prod

## Configuration Guidelines

### DO ✅

- Override resource requests/limits per environment
- Configure environment-specific ingress hosts
- Set replica counts appropriate for environment
- Use ConfigMaps/Secrets for environment variables
- Include pod annotations for monitoring/logging

### DON'T ❌

- Don't manually edit `image.tag` (Kargo manages this)
- Don't commit secrets in plain text (use SealedSecrets or ExternalSecrets)
- Don't create feature branches (Kargo writes to main)
- Don't duplicate values across environments (use defaults in chart)

## Secret Management

This repository should NOT contain secrets. Use one of these patterns:

### Option 1: SealedSecrets

```yaml
# Reference sealed secret
envFrom:
  - secretRef:
      name: paved-road-secrets-sealed
```

Create sealed secret separately:
```bash
kubeseal -f secret.yaml -w sealed-secret.yaml
kubectl apply -f sealed-secret.yaml
```

### Option 2: ExternalSecrets

```yaml
# Reference external secret
envFrom:
  - secretRef:
      name: paved-road-secrets
```

Create ExternalSecret in cluster:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: paved-road-secrets
spec:
  secretStoreRef:
    name: aws-secrets-manager
  target:
    name: paved-road-secrets
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: paved-road/database-url
```

## Kargo Write Access

This repository requires Kargo bot write access via SSH deploy key:

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "kargo-bot@cloudwalkersinc.com" -f kargo-deploy-key

# Add public key to GitHub repository deploy keys (with write access)
# Add private key to Kargo as secret
kubectl create secret generic kargo-git-credentials \
  --namespace kargo-project-paved-road-service \
  --from-file=sshPrivateKey=kargo-deploy-key
```

## Branching Strategy

**Main Branch Only** - Kargo commits directly to main.

- ✅ DO use feature branches for manual configuration changes
- ❌ DON'T use feature branches for image tag updates (Kargo writes to main)
- ✅ DO require PR reviews for human changes
- ❌ DON'T block Kargo commits (configure branch protection carefully)

### Branch Protection Settings

```
Branch: main
- Require pull request reviews: ✅ (for human commits)
- Allow specific actors to bypass: ✅ kargo-bot
- Require status checks: ✅ (optional)
- Require signed commits: ❌ (Kargo may not support)
```

## Monitoring Changes

Track Kargo commits vs human commits:

```bash
# Show recent commits
git log --oneline

# Filter Kargo commits
git log --author="kargo-bot"

# Filter human commits
git log --author="kargo-bot" --invert-grep
```

## Related Repositories

- **Helm Chart**: https://github.com/cloudwalkersinc/helm-charts-paved-road
- **GitOps Platform**: https://github.com/cloudwalkersinc/gitops-platform
- **Container Registry**: registry.cloudwalkersinc.com/cloudwalkersinc/paved-road-service

## Support

For configuration issues:
- Check ArgoCD Application status
- Verify Helm values syntax with `helm template`
- Review Kargo stage logs for image tag updates
- Ensure config changes are synced by ArgoCD
