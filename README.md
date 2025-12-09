# threads-infra

Infrastructure as Code for Threads project using Kubernetes, Kustomize, and ArgoCD.

## 🏗️ Architecture

### Services

- **API** - Backend service (NestJS)
- **Web** - Frontend application (React)
- **PostgreSQL** - Primary database
- **Redis** - Cache and session storage
- **MinIO** - Object storage for files
- **ELK Stack** - Centralized logging (Elasticsearch, Filebeat, Kibana)

### Infrastructure Tools

- **ArgoCD** - GitOps continuous deployment
- **Drone CI** - CI/CD pipeline
- **Grafana + Prometheus** - Monitoring (shared infrastructure)

## 📁 Project Structure

```
threads-infra/
├── manifests/
│   ├── base/                    # Base Kubernetes manifests
│   │   ├── api/                 # Backend service
│   │   ├── web/                 # Frontend service
│   │   ├── postgres/            # Database
│   │   ├── redis/               # Cache
│   │   ├── minio/               # Object storage
│   │   ├── elasticsearch/       # Log storage
│   │   ├── filebeat/            # Log collector
│   │   └── kibana/              # Log visualization
│   └── overlays/
│       └── dev/                 # Development environment overlays
└── docs/
    ├── ELK_SETUP.md            # ELK deployment guide
    └── LOGGING_GUIDE.md        # Developer logging guide
```

## 🚀 Quick Start

### Prerequisites

- Kubernetes cluster
- kubectl configured
- ArgoCD installed

### Deploy All Services

```bash
# Deploy infrastructure services
kubectl apply -k manifests/overlays/dev

# Or deploy individually
kubectl apply -k manifests/overlays/dev/postgres
kubectl apply -k manifests/overlays/dev/redis
kubectl apply -k manifests/overlays/dev/minio
kubectl apply -k manifests/overlays/dev/api
kubectl apply -k manifests/overlays/dev/web

# Deploy ELK stack
kubectl apply -k manifests/overlays/dev/elasticsearch
kubectl apply -k manifests/overlays/dev/filebeat
kubectl apply -k manifests/overlays/dev/kibana
```

## 🔐 Credentials

### ArgoCD

```bash
# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d && echo

# Password: ibhLYtMlTn7mdpag

# Login
argocd login localhost:8080 --username admin --password ibhLYtMlTn7mdpag
```

### PostgreSQL

```
POSTGRES_USER: namth
POSTGRES_PASSWORD: 01664157092aA
POSTGRES_DB: threads_db
```

### MinIO

```
MINIO_ROOT_USER: namth
MINIO_ROOT_PASSWORD: 01664157092aA
```

## 🌐 Service Ports

- **ArgoCD**: 8080
- **Drone CI**: 8081
- **Kibana**: http://kibana.namth.online (or port-forward 5601)

## 📊 ELK Stack (Logging)

### Overview

Centralized logging for all services using Elasticsearch, Filebeat, and Kibana.

### Quick Access

```bash
# Port forward Kibana
kubectl port-forward svc/kibana 5601:5601

# Open browser: http://localhost:5601
```

### Log Indices

- `filebeat-api-*` - API service logs (30 days retention)
- `filebeat-postgres-*` - PostgreSQL logs (7 days retention)
- `filebeat-redis-*` - Redis logs (3 days retention)
- `filebeat-minio-*` - MinIO logs (14 days retention)
- `filebeat-web-*` - Web/Nginx logs (7 days retention)

### Documentation

- **Setup Guide**: [docs/ELK_SETUP.md](docs/ELK_SETUP.md)
- **Logging Best Practices**: [docs/LOGGING_GUIDE.md](docs/LOGGING_GUIDE.md)

## 🔧 Development

### Update Ingress

```bash
kubectl edit application ingress-nginx -n argocd
# Remove old ingress configuration
```

### View Logs

```bash
# Application logs (now in Kibana)
kubectl logs -f deployment/api
kubectl logs -f deployment/web

# Or query in Kibana:
# service: "api" AND level: "error"
```

### Scale Services

```bash
kubectl scale deployment/api --replicas=3
kubectl scale deployment/web --replicas=2
```

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Kustomize Documentation](https://kustomize.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ELK Stack Documentation](https://www.elastic.co/guide/index.html)

## 🤝 Contributing

1. Create feature branch
2. Make changes to manifests
3. Test with `kubectl kustomize manifests/overlays/dev`
4. Commit and push
5. ArgoCD will auto-sync (if enabled)

## 📝 Notes

- Grafana and Prometheus are installed at infrastructure level (shared across projects)
- ELK stack is project-specific for isolated log management
- All secrets should be managed via SealedSecrets in production
- Storage sizes can be adjusted in overlay configurations
