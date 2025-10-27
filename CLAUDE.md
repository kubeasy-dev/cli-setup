# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

The `cli-setup` repository contains Kubernetes manifests used to bootstrap the Kubeasy local development environment. It deploys the foundational components required for the Kubeasy platform to function within a Kind cluster. This repository is referenced and deployed by the kubeasy-cli during initial cluster setup.

## Architecture Overview

### App of Apps Pattern

The repository uses ArgoCD's "App of Apps" pattern, where a root Application (`app-of-apps.yaml`) manages multiple child Applications. The root application points to the `apps/` directory, which contains definitions for:

- **ArgoCD** (apps/argocd.yaml): Self-manages ArgoCD itself from the official repository
- **Kyverno** (apps/kyverno.yaml): Policy engine for validating and mutating Kubernetes resources
- **Challenge Operator** (apps/challenge-operator.yaml): Custom operator for validating challenge solutions

### Component Details

**ArgoCD Application:**
- Source: `https://github.com/argoproj/argo-cd.git` (stable branch)
- Deploys to: `argocd` namespace
- Auto-sync enabled with prune and self-heal
- Uses ServerSideApply for better resource management

**Kyverno Application:**
- Source: Kyverno Helm chart from `https://kyverno.github.io/kyverno`
- Deploys to: `kyverno` namespace
- Auto-sync enabled with namespace creation
- Used for policy enforcement on challenge resources

**Challenge Operator Application:**
- Source: This repository's `operator/` directory
- Deploys to: `challenge-operator-system` namespace
- Points to `operator.yaml` (symlink to latest version)
- Manages two CRDs: StaticValidation and DynamicValidation

## Custom Resource Definitions

The operator defines two validation types:

### StaticValidation
Validates Kubernetes resources against Rego rules stored in ConfigMaps. Suitable for declarative checks (e.g., "must have 3 replicas", "must have specific labels").

Key fields:
- `spec.target`: Defines which resources to validate (kind, apiVersion, name/labelSelector)
- `spec.rulesConfigMap`: References ConfigMap containing Rego policy rules
- `status.allPassed`: Boolean indicating if all rules passed
- `status.resources[].ruleResults[]`: Per-resource, per-rule validation results

### DynamicValidation
Performs runtime checks on Kubernetes resources. Suitable for behavioral validation (e.g., "pod logs contain X", "service account has permission Y").

Supported check types:
- **StatusCheck**: Validates resource status conditions (e.g., Job completion)
- **LogCheck**: Searches pod logs for expected strings
- **RBACCheck**: Verifies ServiceAccount permissions using SubjectAccessReview

Key fields:
- `spec.target`: Defines which resources to validate
- `spec.checks[]`: Array of check definitions (kind: StatusCheck/LogCheck/RBACCheck)
- `status.allPassed`: Boolean indicating if all checks passed
- `status.resources[].checkResults[]`: Per-resource, per-check validation results

## Operator Versioning

Operator manifests are versioned in the `operator/` directory:
- `operator-v1.1.1.yaml`: Previous version
- `operator-v1.2.3.yaml`: Current version
- `operator.yaml`: Symlink or copy pointing to the active version

When updating the operator:
1. Create new versioned file (e.g., `operator-v1.3.0.yaml`)
2. Update container image tag in new file to match version
3. Update `operator.yaml` to reference new version
4. Commit changes - ArgoCD will auto-sync the update

The operator image is hosted at: `ghcr.io/kubeasy-dev/challenge-operator:<version>`

## Deployment Flow

1. User runs `kubeasy-cli` command to initialize cluster
2. CLI applies `app-of-apps.yaml` to ArgoCD
3. ArgoCD syncs child applications (ArgoCD, Kyverno, Operator)
4. Challenge Operator becomes available for validation tasks
5. CLI can then deploy challenges which create StaticValidation/DynamicValidation resources

## Working with Manifests

### Testing Changes Locally
```bash
# Validate YAML syntax
kubectl apply --dry-run=client -f app-of-apps.yaml
kubectl apply --dry-run=client -f apps/
kubectl apply --dry-run=client -f operator/operator.yaml

# Apply to a test cluster
kubectl apply -f app-of-apps.yaml
```

### Updating Operator Version
```bash
# Create new version file
cp operator/operator.yaml operator/operator-v1.3.0.yaml

# Update image tag in new file
# Edit operator-v1.3.0.yaml, change:
#   image: ghcr.io/kubeasy-dev/challenge-operator:1.2.3
# To:
#   image: ghcr.io/kubeasy-dev/challenge-operator:1.3.0

# Update active symlink
cp operator/operator-v1.3.0.yaml operator/operator.yaml

# Commit and push
git add operator/
git commit -m "feat(operator): update challenge-operator to v1.3.0"
git push
```

### Checking ArgoCD Sync Status
```bash
# List all applications
kubectl get applications -n argocd

# Check specific application
kubectl describe application kubeasy-challenge-operator -n argocd

# View operator resources
kubectl get all -n challenge-operator-system
```

## Integration with Other Repositories

- **kubeasy-cli**: Applies `app-of-apps.yaml` during `kubeasy init`
- **challenge-operator**: Source of operator manifests (generated via kubebuilder/controller-gen)
- **challenges**: Creates StaticValidation/DynamicValidation resources that the operator reconciles

## Important Notes

- All applications use automated sync policies (prune + self-heal)
- The operator has cluster-wide read permissions but limited write permissions
- RBAC is carefully scoped to allow validation without excessive privileges
- Operator uses leader election for high availability (though typically runs single replica)
- The operator performs validation by watching CRDs and updating status fields

## Related Documentation

- Parent project overview: `/Users/paul/Workspace/kubeasy/CLAUDE.md`
- Challenge Operator source: `/Users/paul/Workspace/kubeasy/challenge-operator/`
- CLI implementation: `/Users/paul/Workspace/kubeasy/kubeasy-cli/`
- Challenge definitions: `/Users/paul/Workspace/kubeasy/challenges/`
