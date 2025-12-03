# MLOps Airflow with Flux CD

GitOps-based deployment of Apache Airflow for MLOps workflows using Flux CD.

## Overview

This repository contains Flux CD manifests to deploy Apache Airflow on Kubernetes with automated DAG synchronization from a Git repository.

### Architecture

- **GitOps Tool**: Flux CD
- **Workload**: Apache Airflow (Helm Chart)
- **DAG Source**: Git repository with SSH authentication
- **Namespace**: `airflow`

## Prerequisites

1. **Kubernetes Cluster** (v1.30+ recommended)
   ```bash
   # For local development with kind:
   kind create cluster --image kindest/node:v1.30.13
   kubectl cluster-info --context kind-kind
   ```

2. **Flux CLI** installed
   ```bash
   # macOS
   brew install fluxcd/tap/flux

   # Verify installation
   flux --version
   ```

3. **GitHub Personal Access Token** with repository permissions
   - Go to GitHub Settings → Developer settings → Personal access tokens
   - Create a token with `repo` permissions

## Installation

### Step 1: Bootstrap Flux

Set up Flux on your cluster and configure it to watch this repository:

```bash
# Set environment variables
export GITHUB_TOKEN=<your-personal-access-token>
export GITHUB_USER=florianbaer

# Verify prerequisites
flux check --pre

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=MLOps-ArgoCD \
  --branch=main \
  --path=./clusters/production \
  --personal
```

**What happens during bootstrap:**
- Flux components are installed in the `flux-system` namespace
- A deploy key is added to your GitHub repository
- Flux starts monitoring the `./clusters/production` directory
- Automatic synchronization is enabled

### Step 2: Apply the SSH Secret for DAG Sync

The Airflow deployment needs SSH access to pull DAGs from a private Git repository.

```bash
# Apply the SSH secret
kubectl apply -f git-ssh-secret.yaml
```

**Verify the secret:**
```bash
kubectl get secret flux-git-ssh -n airflow
```

### Step 3: Wait for Flux to Deploy Airflow

Flux will automatically detect the manifests in `clusters/production/` and deploy them:

```bash
# Watch Flux reconciliation
flux logs --follow

# Check Helm releases
flux get helmreleases -n airflow

# Check all Flux sources
flux get sources all
```

### Step 4: Verify Airflow Deployment

```bash
# Check pods in airflow namespace
kubectl get pods -n airflow

# Check HelmRelease status
kubectl describe helmrelease mlops-airflow -n airflow

# Check git-sync logs (DAG synchronization)
kubectl logs -n airflow <scheduler-pod-name> -c git-sync
```

## Repository Structure

```
.
├── clusters/
│   └── production/
│       ├── helmrepository.yaml    # Airflow Helm repository source
│       ├── gitrepository.yaml     # Config repository source (optional)
│       └── helmrelease.yaml       # Airflow deployment
├── archive/
│   └── argocd-application.yaml    # Previous ArgoCD configuration
├── git-ssh-secret.yaml            # SSH key for DAG repository
├── values.yaml                    # Airflow configuration values
├── Chart.yaml                     # Helm chart metadata (optional)
└── README.md                      # This file
```

## Configuration

### Airflow Configuration

Main configuration is in `clusters/production/helmrelease.yaml`:

```yaml
spec:
  values:
    dags:
      gitSync:
        enabled: true
        repo: git@github.com:HSLU-DBIZ/JobMonitor-DAGs.git
        branch: main
        sshKeySecret: flux-git-ssh
```

### Modifying Configuration

1. Edit `clusters/production/helmrelease.yaml` or `values.yaml`
2. Commit and push changes to GitHub
3. Flux automatically detects changes and reconciles within the interval period (default: 10 minutes)

**Force immediate reconciliation:**
```bash
flux reconcile source git flux-system
flux reconcile helmrelease mlops-airflow -n airflow
```

## Monitoring

### Check Flux Status

```bash
# Overall Flux health
flux check

# Git repository sync status
flux get sources git -n flux-system

# Helm repository sync status
flux get sources helm -n flux-system

# HelmRelease status
flux get helmreleases -n airflow

# Continuous logs
flux logs --follow
```

### Check Airflow Status

```bash
# All resources in airflow namespace
kubectl get all -n airflow

# Specific pods
kubectl get pods -n airflow -w

# Scheduler logs
kubectl logs -n airflow <scheduler-pod> -f

# Git-sync status (DAG synchronization)
kubectl logs -n airflow <scheduler-pod> -c git-sync -f
```

### Access Airflow Web UI

```bash
# Port-forward to access locally
kubectl port-forward -n airflow svc/mlops-airflow-web 8080:8080

# Access at: http://localhost:8080
```

## Troubleshooting

### Flux Not Syncing

```bash
# Check Flux system pods
kubectl get pods -n flux-system

# Check for errors in Flux logs
flux logs --all-namespaces

# Force reconciliation
flux reconcile source git flux-system
```

### HelmRelease Failed

```bash
# Detailed status
kubectl describe helmrelease mlops-airflow -n airflow

# Check Helm release history
helm list -n airflow

# Flux events
flux events --for HelmRelease/mlops-airflow --namespace airflow
```

### DAGs Not Syncing

```bash
# Check git-sync logs
kubectl logs -n airflow <scheduler-pod> -c git-sync

# Verify SSH secret exists
kubectl get secret flux-git-ssh -n airflow -o yaml

# Check git-sync container status
kubectl describe pod <scheduler-pod> -n airflow
```

### SSH Authentication Issues

```bash
# Verify secret has both identity and known_hosts
kubectl get secret flux-git-ssh -n airflow -o jsonpath='{.data}' | jq 'keys'

# Should show: ["identity", "known_hosts"]

# Test SSH connection from a debug pod
kubectl run -it --rm debug --image=alpine/git --restart=Never -n airflow -- sh
```

## Flux vs ArgoCD Comparison

| Feature | ArgoCD | Flux |
|---------|---------|------|
| Application Definition | Single `Application` CR | Multiple CRs (HelmRepository, HelmRelease, etc.) |
| Auto-sync | Configured per application | Always on (interval-based) |
| UI | Web UI available | CLI-based (UI available via Flux UI extension) |
| Prune | Explicit configuration | Default behavior |
| Multi-tenancy | Built-in | Via Kustomization and RBAC |

## Updating Airflow Version

Edit `clusters/production/helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      version: "1.16.0"  # Update version here
```

Commit and push. Flux will automatically upgrade Airflow.

## Uninstalling

### Remove Airflow

```bash
flux delete helmrelease mlops-airflow -n airflow
```

### Uninstall Flux

```bash
flux uninstall --silent
```

## Rollback to ArgoCD

If needed, you can rollback to ArgoCD:

1. Uninstall Flux:
   ```bash
   flux uninstall --silent
   ```

2. Restore ArgoCD application:
   ```bash
   kubectl apply -f archive/argocd-application.yaml
   ```

3. Revert secret changes:
   ```bash
   git checkout archive/git-ssh-secret.yaml
   kubectl apply -f archive/git-ssh-secret.yaml
   ```

## Additional Resources

- [Flux Documentation](https://fluxcd.io/flux/)
- [Airflow Helm Chart](https://airflow.apache.org/docs/helm-chart/)
- [Flux Helm Guide](https://fluxcd.io/flux/guides/helmreleases/)
- [GitOps Principles](https://opengitops.dev/)

## Support

For issues related to:
- **Flux**: [Flux GitHub Issues](https://github.com/fluxcd/flux2/issues)
- **Airflow Helm Chart**: [Airflow Helm Chart Issues](https://github.com/apache/airflow/issues)
- **This Repository**: [Open an issue](https://github.com/florianbaer/MLOps-ArgoCD/issues)
