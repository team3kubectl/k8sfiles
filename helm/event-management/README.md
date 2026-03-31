# Event Management Helm Chart

This Helm chart deploys the complete Event Management microservices architecture on Kubernetes.

## Chart Structure

```
event-management/
├── Chart.yaml           # Chart metadata
├── values.yaml          # Default configuration values
├── templates/           # Kubernetes manifests
│   ├── admin-service/
│   ├── ai-service/
│   ├── booking-service/
│   ├── events-service/
│   ├── frontend/
│   ├── notification-service/
│   ├── payment-service/
│   ├── ticket-service/
│   ├── users-service/
│   └── k8s/             # Infrastructure services
│       ├── gateway/
│       ├── mongodb/
│       ├── postgres/
│       ├── rabbitmq/
│       ├── redis/
│       └── storage/
```

## Quick Start

### 1. Install the chart
```bash
helm install event-management ./event-management -n team3-ns --create-namespace
```

### 2. Using custom values
```bash
helm install event-management ./event-management -f custom-values.yaml
```

### 3. Upgrade the chart
```bash
helm upgrade event-management ./event-management
```

### 4. Uninstall the chart
```bash
helm uninstall event-management
```

## Configuration

### Global Settings
- `global.namespace`: Kubernetes namespace (default: `team3-ns`)
- `global.imagePullPolicy`: Image pull policy (default: `Always`)

### Service Configuration

Each service can be independently enabled/disabled via `values.yaml`:

```yaml
services:
  admin:
    enabled: true
    replicas: 1
    image: team3kubectl/admin-service:v1.1.1
    port: 8008
    resources: {...}
```

### Infrastructure Services

Configure infrastructure components:

```yaml
infrastructure:
  mongodb:
    enabled: true
    image: mongo:6.0
    replicas: 1
    port: 27017
    storage: 10Gi
    
  postgres:
    enabled: true
    image: postgres:15
    replicas: 1
    port: 5432
    storage: 20Gi
```

## Common Tasks

### Enable/Disable a Service

Edit `values.yaml`:
```yaml
services:
  admin:
    enabled: false  # Disable admin service
```

### Scale a Service

```bash
helm upgrade event-management ./event-management \
  --set services.admin.replicas=3
```

### Change Image Version

```bash
helm upgrade event-management ./event-management \
  --set services.admin.image=team3kubectl/admin-service:v1.2.0
```

### Customize Resources

```bash
helm upgrade event-management ./event-management \
  --set services.admin.resources.requests.cpu=200m \
  --set services.admin.resources.limits.cpu=500m
```

## Verifying the Installation

### Check deployment status
```bash
kubectl get deployments -n team3-ns
kubectl get statefulsets -n team3-ns
kubectl get services -n team3-ns
```

### View logs
```bash
kubectl logs -n team3-ns deployment/admin-deployment
```

### Port-forward to access services
```bash
kubectl port-forward -n team3-ns svc/frontend-service 8080:80
```

## Troubleshooting

### Debug template rendering
```bash
helm template event-management ./event-management
```

### Dry-run installation
```bash
helm install event-management ./event-management --dry-run --debug
```

### Check values
```bash
helm values event-management
```

## Default Credentials

- **MongoDB**: admin / adminpass
- **PostgreSQL**: postgres / postgres
- **RabbitMQ**: guest / guest

⚠️ **Important**: Change these credentials in `values.yaml` before deploying to production!

## Notes

- All services are deployed in the `team3-ns` namespace
- Infrastructure components (MongoDB, PostgreSQL, RabbitMQ, Redis) are included
- API Gateway is configured with HTTP routes for all services
- Horizontal Pod Autoscaling is enabled for the frontend service
