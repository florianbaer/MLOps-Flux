# Setup Guide: MLOps Airflow with Flux CD

This guide walks you through setting up Apache Airflow on Kubernetes using Flux CD for GitOps.

## Prerequisites

### 1. Kubernetes Cluster

**For local development with kind:**
```bash
# Install kind (if not already installed)
brew install kind  # macOS
# or download from https://kind.sigs.k8s.io/

# Create cluster with Kubernetes 1.30+
kind create cluster --image kindest/node:v1.30.13

# Verify cluster
kubectl cluster-info --context kind-kind
```

**For MicroK8s:**
```bash
# Enable required addons
microk8s enable dns
microk8s enable storage

# Get kubectl config
microk8s kubectl config view --raw > ~/.kube/config
```

**For Docker Desktop:**
- Enable Kubernetes in Docker Desktop settings
- Verify: `kubectl cluster-info`

### 2. Install Flux CLI

**macOS:**
```bash
brew install fluxcd/tap/flux
```

**Linux:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Verify installation:**
```bash
flux --version
```

### 3. GitHub Personal Access Token

Create a GitHub Personal Access Token with `repo` permissions:

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Select scopes: `repo` (all)
4. Generate and copy the token 

## Installation Steps

### Step 1: Verify Prerequisites

Check that your cluster meets Flux requirements:

```bash
flux check --pre
```

Expected output should show all prerequisites met.

### Step 2: Bootstrap Flux

Bootstrap Flux to your cluster and configure it to watch this repository:

```bash
# Set environment variables
export GITHUB_TOKEN=<your-personal-access-token>
export GITHUB_USER=<your-github-username>

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=MLOps-Flux \
  --branch=main \
  --path=./clusters/production \
  --personal
```

**What happens during bootstrap:**
- Flux components are installed in the `flux-system` namespace
- A deploy key is added to your GitHub repository (with write access)
- Flux starts monitoring the `./clusters/production` directory
- A `flux-system/` directory is created in your repository

**Verify Flux installation:**
```bash
# Check Flux components
kubectl get pods -n flux-system

# Check Flux status
flux check
```

### Step 3: Create Airflow Namespace

```bash
kubectl create namespace airflow
```

### Step 4: Apply SSH Secret for DAG Repository

The SSH secret must be applied manually before Airflow deploys:

```bash
# Apply the secret
kubectl apply -f git-ssh-secret.yaml

# Verify secret
kubectl get secret flux-git-ssh -n airflow
kubectl describe secret flux-git-ssh -n airflow
```

**Important:** This secret contains:
- `identity`: SSH private key for accessing the DAGs repository
- `known_hosts`: GitHub host keys for SSH verification

### Step 5: Wait for Flux to Deploy Airflow

Flux automatically detects and deploys manifests from `clusters/production/`:

```bash
# Watch Flux reconciliation
flux logs --follow

# Check GitRepository sync
flux get sources git

# Check HelmRepository sync
flux get sources helm

# Check HelmRelease status
flux get helmreleases -n airflow
```

**Expected resources:**
- `HelmRepository/apache-airflow` in `flux-system` namespace
- `GitRepository/mlops-airflow-config` in `flux-system` namespace (optional)
- `HelmRelease/mlops-airflow` in `airflow` namespace

### Step 6: Verify Airflow Deployment

```bash
# Check pods in airflow namespace
kubectl get pods -n airflow -w

# Check all resources
kubectl get all -n airflow

# Check HelmRelease details
kubectl describe helmrelease mlops-airflow -n airflow
```

**Wait for all pods to be Running:**
- Scheduler
- Webserver
- Workers
- PostgreSQL
- Redis

### Step 7: Verify DAG Synchronization

Check that git-sync is pulling DAGs from your repository:

```bash
# Get scheduler pod name
SCHEDULER_POD=$(kubectl get pods -n airflow -l component=scheduler -o jsonpath='{.items[0].metadata.name}')

# Check git-sync logs
kubectl logs -n airflow $SCHEDULER_POD -c git-sync

# Should show successful git sync operations
```

### Step 8: Access Airflow Web UI

```bash
# Port-forward to access locally
kubectl port-forward -n airflow svc/mlops-airflow-web 8080:8080

# Access at: http://localhost:8080
```

**Default credentials** (check values.yaml for actual credentials):
- Username: `admin`
- Password: `admin` (or as configured in your Helm values)

## Making Changes

### Update Airflow Configuration

1. Edit `clusters/production/helmrelease.yaml` or `values.yaml`
2. Commit and push to GitHub:
   ```bash
   git add clusters/production/helmrelease.yaml
   git commit -m "Update Airflow configuration"
   git push
   ```
3. Wait for Flux to reconcile (default: 10 minutes) or force sync:
   ```bash
   flux reconcile source git flux-system
   flux reconcile helmrelease mlops-airflow -n airflow
   ```

### Update Airflow Version

Edit `clusters/production/helmrelease.yaml`:

```yaml
spec:
  chart:
    spec:
      version: "1.16.0"  # Update to desired version
```

Commit, push, and Flux will automatically upgrade.

### Update DAG Repository Settings

Edit the git-sync configuration in `clusters/production/helmrelease.yaml`:

```yaml
spec:
  values:
    dags:
      gitSync:
        repo: git@github.com:HSLU-DBIZ/JobMonitor-DAGs.git
        branch: main  # Change branch if needed
```

## Monitoring and Troubleshooting

### Check Flux Status

```bash
# Overall health
flux check

# All sources
flux get sources all

# Kustomizations
flux get kustomizations

# Helm releases
flux get helmreleases -A

# Recent events
flux events
```

### View Flux Logs

```bash
# All Flux logs
flux logs --all-namespaces --follow

# Specific controller
flux logs --kind=HelmRelease --namespace=airflow
```

### Troubleshoot HelmRelease Issues

```bash
# Detailed status
kubectl describe helmrelease mlops-airflow -n airflow

# Check Helm release
helm list -n airflow

# Helm release details
helm get values mlops-airflow -n airflow
```

### Troubleshoot Git-Sync Issues

```bash
# Check git-sync logs
kubectl logs -n airflow <scheduler-pod> -c git-sync

# Verify SSH secret
kubectl get secret flux-git-ssh -n airflow -o yaml

# Test secret format
kubectl get secret flux-git-ssh -n airflow -o jsonpath='{.data.identity}' | base64 -d | head -n 1
# Should show: -----BEGIN OPENSSH PRIVATE KEY-----

kubectl get secret flux-git-ssh -n airflow -o jsonpath='{.data.known_hosts}' | base64 -d
# Should show GitHub host keys
```

### Force Reconciliation

```bash
# Force git source sync
flux reconcile source git flux-system

# Force Helm repository sync
flux reconcile source helm apache-airflow -n flux-system

# Force HelmRelease reconciliation
flux reconcile helmrelease mlops-airflow -n airflow
```

## Suspend/Resume Reconciliation

Temporarily stop Flux from reconciling (useful for manual changes):

```bash
# Suspend HelmRelease
flux suspend helmrelease mlops-airflow -n airflow

# Make manual changes...

# Resume reconciliation
flux resume helmrelease mlops-airflow -n airflow
```

## Uninstalling

### Remove Airflow Only

```bash
# Delete HelmRelease (this removes Airflow)
flux delete helmrelease mlops-airflow -n airflow --silent

# Delete namespace
kubectl delete namespace airflow
```

### Uninstall Flux Completely

```bash
# Uninstall Flux components
flux uninstall --silent

# This removes:
# - Flux controllers
# - CRDs
# - flux-system namespace
```

## Comparison: Flux vs ArgoCD

| Aspect | ArgoCD (Previous) | Flux CD (Current) |
|--------|-------------------|-------------------|
| Installation | `kubectl apply` manifest | Bootstrap command |
| UI | Web UI included | CLI-based (UI available separately) |
| Configuration | Single Application CR | Multiple CRs (HelmRepository, HelmRelease) |
| Sync | Manual or auto-sync | Automatic (interval-based) |
| Repository access | Deploy key or token | Deploy key (created during bootstrap) |
| Helm hooks | Disable with flags | Disable in values |

## Rollback to ArgoCD (If Needed)

If you need to rollback to ArgoCD:

1. Uninstall Flux:
   ```bash
   flux uninstall --silent
   ```

2. Install ArgoCD:
   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

3. Restore ArgoCD application:
   ```bash
   kubectl apply -f archive/argocd-application.yaml
   ```

4. Revert SSH secret (if needed):
   ```bash
   # The old secret format is in archive/ if you need it
   ```

## Additional Resources

- [Flux Documentation](https://fluxcd.io/flux/)
- [Flux Get Started Guide](https://fluxcd.io/flux/get-started/)
- [Airflow Helm Chart Docs](https://airflow.apache.org/docs/helm-chart/)
- [Flux Helm Controller](https://fluxcd.io/flux/components/helm/)
- [GitOps Principles](https://opengitops.dev/)

## Common Issues

### Issue: Flux bootstrap fails with permission error

**Solution:** Ensure your GitHub token has `repo` permissions and the repository exists.

### Issue: HelmRelease stuck in "NotReady" state

**Solution:**
```bash
# Check events
flux events --for HelmRelease/mlops-airflow --namespace airflow

# Check detailed status
kubectl describe helmrelease mlops-airflow -n airflow
```

### Issue: DAGs not appearing in Airflow

**Solution:**
1. Check git-sync container logs
2. Verify SSH secret is correct
3. Verify repository URL and branch in helmrelease.yaml
4. Check network connectivity to GitHub

### Issue: Flux not detecting changes

**Solution:**
```bash
# Check GitRepository status
flux get sources git

# Force reconciliation
flux reconcile source git flux-system

# Check interval settings in GitRepository spec
```

## Tips

1. **Use shorter intervals for development:**
   Edit `clusters/production/gitrepository.yaml` and set `interval: 1m`

2. **Watch reconciliation in real-time:**
   ```bash
   watch flux get helmreleases -A
   ```

3. **Export Helm values for debugging:**
   ```bash
   helm get values mlops-airflow -n airflow
   ```

4. **Check Flux version:**
   ```bash
   flux version
   ```

5. **Validate manifests before pushing:**
   ```bash
   flux check
   ```
