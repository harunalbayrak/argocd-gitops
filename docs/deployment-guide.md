# Deployment Guide

This guide provides detailed instructions for deploying applications using our GitOps repository structure.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Initial Setup](#initial-setup)
3. [Environment Setup](#environment-setup)
4. [Application Deployment](#application-deployment)
5. [Secret Management](#secret-management)
6. [Monitoring Setup](#monitoring-setup)
7. [Troubleshooting](#troubleshooting)

## Prerequisites

### Required Tools

- **kubectl** (v1.24+)
- **ArgoCD CLI** (v2.4+)
- **Helm** (v3.8+)
- **Git** (v2.30+)
- **Docker** (v20.10+)

### Cluster Requirements

- Kubernetes cluster (v1.24+)
- Minimum 4 CPU cores and 8GB RAM
- Storage class configured
- LoadBalancer or Ingress controller

### Access Requirements

- Cluster admin permissions
- Git repository access
- Container registry access

## Initial Setup

### 1. Clone Repository

```bash
git clone https://github.com/your-org/argo-gitops-repository.git
cd argo-gitops-repository
```

### 2. Install ArgoCD

```bash
# Create ArgoCD namespace and install
kubectl apply -f bootstrap/argocd-install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 3. Configure ArgoCD Access

#### Option A: Port Forward (Development)
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access: https://localhost:8080
```

#### Option B: LoadBalancer (Cloud)
```bash
kubectl get svc argocd-server-lb -n argocd
# Use external IP to access ArgoCD
```

#### Option C: Ingress (Production)
```bash
# Update ingress hostname in bootstrap/argocd-install.yaml
# Access: https://argocd.your-domain.com
```

### 4. Login to ArgoCD

```bash
# Login via CLI
argocd login localhost:8080 --username admin --password <initial-password>

# Change admin password
argocd account update-password
```

## Environment Setup

### Development Environment

```bash
# Deploy development bootstrap applications
kubectl apply -f clusters/development/bootstrap/

# Verify applications are created
argocd app list --project development
```

### Staging Environment

```bash
# Deploy staging bootstrap applications  
kubectl apply -f clusters/staging/bootstrap/

# Verify applications are created
argocd app list --project staging
```

### Production Environment

```bash
# Deploy production bootstrap applications
kubectl apply -f clusters/production/bootstrap/

# Verify applications are created
argocd app list --project production
```

## Application Deployment

### Understanding the Structure

Each application follows this pattern:
```
clusters/<environment>/workloads/<category>/<app-name>/
├── kustomization.yaml          # Kustomize configuration
├── <app-name>-values.yaml     # Environment-specific values
└── patches/                   # Environment-specific patches
    ├── deployment-patch.yaml
    └── service-patch.yaml
```

### Deploying a New Application

#### Step 1: Create Base Manifests

```bash
# Create base application manifests
mkdir -p manifests/applications/backend/new-service/base

# Add your Kubernetes manifests
cat > manifests/applications/backend/new-service/base/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-service
  labels:
    app: new-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: new-service
  template:
    metadata:
      labels:
        app: new-service
    spec:
      containers:
      - name: new-service
        image: your-registry/new-service:latest
        ports:
        - containerPort: 8080
EOF

# Create kustomization.yaml
cat > manifests/applications/backend/new-service/base/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:
  app.kubernetes.io/name: new-service
  app.kubernetes.io/part-of: backend-services
EOF
```

#### Step 2: Create Environment Overlays

```bash
# Create development overlay
mkdir -p clusters/development/workloads/backend/new-service

cat > clusters/development/workloads/backend/new-service/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
  - ../../../../manifests/applications/backend/new-service/base

patchesStrategicMerge:
  - patches/deployment-patch.yaml

images:
  - name: your-registry/new-service
    newTag: dev-latest

replicas:
  - name: new-service
    count: 1
EOF

# Create development-specific patches
mkdir -p clusters/development/workloads/backend/new-service/patches

cat > clusters/development/workloads/backend/new-service/patches/deployment-patch.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-service
spec:
  template:
    spec:
      containers:
      - name: new-service
        env:
        - name: ENVIRONMENT
          value: "development"
        - name: LOG_LEVEL
          value: "debug"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"  
            cpu: "200m"
EOF
```

#### Step 3: Create ArgoCD Application

```bash
cat > clusters/development/workloads/_applications/new-service-app.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: new-service-dev
  namespace: argocd
  labels:
    environment: development
    category: backend
spec:
  project: development
  source:
    repoURL: https://github.com/your-org/argo-gitops-repository
    targetRevision: HEAD
    path: clusters/development/workloads/backend/new-service
  destination:
    server: https://kubernetes.default.svc
    namespace: development
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF
```

#### Step 4: Deploy Application

```bash
# Commit and push changes
git add .
git commit -m "Add new-service application"
git push origin main

# Apply ArgoCD application
kubectl apply -f clusters/development/workloads/_applications/new-service-app.yaml

# Check deployment status
argocd app get new-service-dev
argocd app sync new-service-dev
```

### Updating Applications

#### Image Updates
```bash
# Update image tag in kustomization.yaml
cd clusters/development/workloads/backend/new-service
sed -i 's/newTag: dev-latest/newTag: dev-v1.2.3/' kustomization.yaml

# Commit and push
git add kustomization.yaml
git commit -m "Update new-service to v1.2.3 in development"
git push origin main
```

#### Configuration Updates
```bash
# Update patches or add new ones
vim clusters/development/workloads/backend/new-service/patches/deployment-patch.yaml

# Commit and push
git add .
git commit -m "Update new-service configuration"
git push origin main
```

## Secret Management

### Using Sealed Secrets

```bash
# Create secret
kubectl create secret generic mysecret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword \
  --dry-run=client -o yaml > mysecret.yaml

# Seal the secret
kubeseal -f mysecret.yaml -w mysealedsecret.yaml

# Place sealed secret in appropriate directory
mv mysealedsecret.yaml clusters/development/workloads/backend/new-service/
```

### Using External Secrets

```bash
# Create external secret
cat > clusters/development/workloads/backend/new-service/external-secret.yaml << EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: new-service-secret
  namespace: development
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-secret-store
    kind: SecretStore
  target:
    name: new-service-secret
    creationPolicy: Owner
  data:
  - secretKey: database-password
    remoteRef:
      key: secret/new-service
      property: db_password
EOF
```

## Monitoring Setup

### Enable Application Monitoring

```bash
# Add monitoring annotations to deployment
cat > clusters/development/workloads/backend/new-service/patches/monitoring-patch.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-service
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: new-service
        ports:
        - name: metrics
          containerPort: 8080
EOF

# Update kustomization.yaml to include monitoring patch
echo "  - patches/monitoring-patch.yaml" >> clusters/development/workloads/backend/new-service/kustomization.yaml
```

### Create ServiceMonitor

```bash
cat > clusters/development/workloads/backend/new-service/servicemonitor.yaml << EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: new-service
  namespace: development
  labels:
    app: new-service
spec:
  selector:
    matchLabels:
      app: new-service
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
EOF

# Add to kustomization.yaml
echo "  - servicemonitor.yaml" >> clusters/development/workloads/backend/new-service/base/kustomization.yaml
```

## Troubleshooting

### Common Issues

#### Application Stuck in Sync

```bash
# Check application status
argocd app get <app-name>

# Check sync operation
argocd app sync <app-name> --dry-run

# Force refresh
argocd app refresh <app-name>

# Hard refresh
argocd app refresh <app-name> --hard-refresh
```

#### Resource Issues

```bash
# Check resource events
kubectl describe <resource-type> <resource-name> -n <namespace>

# Check logs
kubectl logs -f deployment/<app-name> -n <namespace>

# Check resource usage
kubectl top pods -n <namespace>
```

#### GitOps Sync Issues

```bash
# Check repository access
argocd repo list

# Test repository connection
argocd repo get <repo-url>

# Check webhook configuration
argocd repo get <repo-url> --show-details
```

### Debugging Steps

1. **Check ArgoCD Application Status**
   ```bash
   argocd app get <app-name>
   ```

2. **Verify Kubernetes Resources**
   ```bash
   kubectl get all -n <namespace>
   ```

3. **Check Resource Events**
   ```bash
   kubectl get events -n <namespace> --sort-by=.metadata.creationTimestamp
   ```

4. **Validate Manifests**
   ```bash
   kubectl apply --dry-run=client -f <manifest-file>
   ```

5. **Check Resource Logs**
   ```bash
   kubectl logs -f <pod-name> -n <namespace>
   ```

## Best Practices

### Git Workflow
- Use feature branches for changes
- Create pull requests for review
- Use semantic commit messages
- Tag releases for production deployments

### Application Configuration
- Use environment-specific overlays
- Keep secrets separate from configuration
- Use resource limits and requests
- Implement health checks

### Monitoring
- Add Prometheus metrics to applications
- Configure alerting rules
- Use distributed tracing
- Implement log aggregation

### Security
- Use least privilege RBAC
- Regularly rotate secrets
- Scan container images
- Implement network policies

---

For more specific guidance, see:
- [Secret Management Guide](secret-management.md)
- [Cluster Setup Guide](cluster-setup.md)
- [Troubleshooting Guide](troubleshooting.md)
