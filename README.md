# GitOps Repository - Kubernetes Multi-Environment Platform

This repository contains a GitOps structure designed for multi-environment application deployment on Kubernetes using ArgoCD.

## 📁 Repository Structure

```
argo-gitops-repository/
├── docs/                    # Detailed documentation
├── bootstrap/               # ArgoCD installation files
├── manifests/               # Base Kubernetes manifests
├── clusters/                # Environment-specific configurations
├── secrets/                 # Secret management
├── infrastructure/          # Infrastructure as Code (Terraform)
├── build/                   # Dockerfiles and build configs
├── scripts/                 # Automation scripts
└── tools/                   # Helper tools
```

## 🚀 QuP0+r\P0+r\ick Start

### Prerequisites
- Kubernetes cluster (v1.24+)
- kubectl configured
- ArgoCD CLI installed
- Helm v3+

### 1. Install ArgoCD
```bash
# Install ArgoCD to cluster
kubectl apply -f bootstrap/argocd-install.yaml

# Port-forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 2. Deploy Bootstrap Applications
```bash
# For development cluster
kubectl apply -f clusters/development/bootstrap/

# For staging cluster  
kubectl apply -f clusters/staging/bootstrap/

# For production cluster
kubectl apply -f clusters/production/bootstrap/
```

### 3. Setup Secret Management
```bash
# Install secret management tools
./scripts/setup/setup-secrets.sh <environment>
```

## 🌍 Environments

| Environment | Cluster | Description |
|-------------|---------|-------------|
| Development | development | Development and testing environment |
| Staging | staging | Pre-production environment for final testing |
| Production | production | Live production environment |

## 🏗️ Architecture

### Application Categories

- **Database**: PostgreSQL, MongoDB
- **Auth**: Keycloak, OAuth2 Proxy
- **Cache**: Redis, Memcached
- **Messaging**: RabbitMQ, Kafka, NATS
- **Storage**: MinIO, Longhorn
- **Search**: Elasticsearch, OpenSearch
- **Backend Services**: Microservices
- **Frontend**: React applications, Admin dashboards
- **Monitoring**: Prometheus, Grafana, Jaeger, Loki
- **Security**: Vault, External Secrets, Cert-Manager
- **Networking**: Ingress NGINX, Istio, MetalLB
- **Tools**: Jenkins, GitLab Runner, SonarQube, Harbor

## 🔐 Secret Management Strategy

This repository supports multiple secret management approaches:

1. **Sealed Secrets** - Git-safe encrypted secrets
2. **External Secrets Operator** - Integration with external secret stores
3. **HashiCorp Vault** - Enterprise-grade secret management

See [Secret Management Documentation](docs/secret-management.md) for detailed setup.

## 📚 Documentation

- [Deployment Guide](docs/deployment-guide.md)
- [Cluster Setup](docs/cluster-setup.md)
- [Secret Management](docs/secret-management.md)
- [Troubleshooting](docs/troubleshooting.md)

## 🔧 Development Workflow

1. **Make Changes**: Update manifests in appropriate environment folder
2. **Commit & Push**: Git commit triggers ArgoCD sync
3. **Monitor**: Check ArgoCD UI for deployment status
4. **Validate**: Run health checks and tests

## 📦 Adding New Applications

1. Create base manifests in `manifests/applications/`
2. Add environment-specific overlays in `clusters/<env>/workloads/`
3. Create ArgoCD application in `clusters/<env>/workloads/_applications/`
4. Update this README with new application info

## 🛠️ Useful Commands

```bash
# Check ArgoCD applications status
argocd app list

# Sync specific application
argocd app sync <app-name>

# Get application details
argocd app get <app-name>

# Setup new environment
./scripts/setup/bootstrap-cluster.sh <environment>

# Deploy specific service
./scripts/deploy/deploy-app.sh <service-name> <environment>
```

## 🚨 Emergency Procedures

### Rollback Application
```bash
# Rollback to previous version
argocd app rollback <app-name>

# Or use helper script
./scripts/deploy/rollback-app.sh <app-name> <environment>
```

### Secret Rotation
```bash
# Rotate secrets for environment
./scripts/maintenance/rotate-secrets.sh <environment>
```

## 📋 Monitoring & Alerts

- **Grafana Dashboard**: http://grafana.<domain>
- **Prometheus**: http://prometheus.<domain>
- **ArgoCD UI**: http://argocd.<domain>
- **Jaeger Tracing**: http://jaeger.<domain>

## 🤝 Contributing

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support

- Create an issue for bug reports or feature requests
- Check [Troubleshooting Guide](docs/troubleshooting.md)
- Contact platform team: platform-team@company.com

## 🏷️ Version

Current Version: v1.0.0

---

**Note**: This repository follows GitOps principles. All changes should be made through Git commits, not direct kubectl commands on the cluster.
