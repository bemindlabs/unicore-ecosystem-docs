# Kubernetes Deployment

Kubernetes deployment for UniCore is recommended for production environments requiring high availability, horizontal scaling, and automated rollouts. This guide covers the manifest structure and planned Helm chart layout.

> **Status**: Kubernetes manifests and Helm chart are in active development. This page documents the intended structure and configuration model. Docker Compose remains the primary supported deployment method — see [docker.md](./docker.md).

## Prerequisites

| Requirement | Version | Notes |
|-------------|---------|-------|
| Kubernetes | 1.28+ | EKS, GKE, AKS, or self-managed |
| kubectl | 1.28+ | Matching cluster version |
| Helm | 3.12+ | For chart-based deployment |
| Container Registry | Any | Docker Hub, ECR, GCR, ACR |
| Persistent Volumes | Required | For PostgreSQL, Redis, Qdrant |

## Architecture Overview

```
Ingress (NGINX Ingress Controller)
  ├── /              → dashboard Service (ClusterIP :3000)
  ├── /api/          → api-gateway Service (ClusterIP :4000)
  ├── /auth/         → api-gateway Service (ClusterIP :4000)
  ├── /webhooks/     → api-gateway Service (ClusterIP :4000)
  └── /ws            → openclaw Service (ClusterIP :18789)

Deployments (Stateless)
  ├── unicore-dashboard       (2+ replicas)
  ├── unicore-api-gateway     (2+ replicas)
  ├── unicore-erp             (2+ replicas)
  ├── unicore-ai-engine       (2+ replicas)
  ├── unicore-rag             (2+ replicas)
  ├── unicore-bootstrap       (1 replica)
  ├── unicore-openclaw        (2+ replicas)
  └── unicore-workflow        (2+ replicas)

StatefulSets (Stateful)
  ├── unicore-postgres        (primary + read replica)
  ├── unicore-redis           (sentinel or cluster)
  ├── unicore-qdrant          (distributed mode)
  └── unicore-kafka           (3+ broker cluster)
```

## Helm Chart Structure

The `unicore-helm` chart (planned) will follow this layout:

```
unicore-helm/
├── Chart.yaml
├── values.yaml                  ← Default values
├── values-community.yaml        ← Community edition overrides
├── values-pro.yaml              ← Pro edition overrides
├── values-enterprise.yaml       ← Enterprise overrides
├── templates/
│   ├── _helpers.tpl
│   ├── namespace.yaml
│   ├── configmap.yaml           ← Non-secret environment config
│   ├── secrets.yaml             ← Database passwords, JWT secrets
│   ├── dashboard/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml             ← Horizontal Pod Autoscaler
│   ├── api-gateway/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── erp/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── ai-engine/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── rag/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── openclaw/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── workflow/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── postgres/
│   │   ├── statefulset.yaml
│   │   ├── service.yaml
│   │   └── pvc.yaml
│   ├── redis/
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── qdrant/
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── kafka/
│   │   ├── statefulset.yaml
│   │   └── service.yaml
│   ├── ingress.yaml
│   └── jobs/
│       ├── db-migrate.yaml      ← Prisma db push Job (run once)
│       └── provision-admin.yaml ← Admin provisioning Job
```

## Key Configuration (values.yaml)

```yaml
global:
  edition: full           # community | full | enterprise
  imageRegistry: ghcr.io/bemindlabs
  imageTag: latest
  storageClass: standard  # or gp2, pd-ssd, etc.

dashboard:
  replicas: 2
  resources:
    requests: { cpu: 250m, memory: 512Mi }
    limits: { cpu: 1000m, memory: 1Gi }

apiGateway:
  replicas: 2
  resources:
    requests: { cpu: 500m, memory: 512Mi }
    limits: { cpu: 2000m, memory: 2Gi }

postgres:
  storage: 50Gi
  credentials:
    user: unicore
    # password: set via --set or external secret

redis:
  storage: 5Gi
  mode: sentinel          # standalone | sentinel | cluster

qdrant:
  storage: 20Gi

kafka:
  replicas: 3
  storage: 20Gi

ingress:
  enabled: true
  className: nginx
  host: unicore.example.com
  tls: true
  certIssuer: letsencrypt-prod  # cert-manager ClusterIssuer

license:
  key: ""                 # UC-XXXX-XXXX-XXXX-XXXX
  serverUrl: ""
```

## Deploying with Helm (Preview)

```bash
# Add the UniCore Helm repo (when published)
helm repo add unicore https://charts.bemind.tech
helm repo update

# Install with custom values
helm install unicore unicore/unicore \
  --namespace unicore \
  --create-namespace \
  --values values-pro.yaml \
  --set postgres.credentials.password=<your-db-password> \
  --set global.jwt.secret=<your-jwt-secret> \
  --set license.key=UC-XXXX-XXXX-XXXX-XXXX

# Upgrade
helm upgrade unicore unicore/unicore \
  --namespace unicore \
  --values values-pro.yaml \
  --reuse-values

# Status
helm status unicore -n unicore
```

## Database Migration Job

The Helm chart includes a Kubernetes Job for initial schema setup:

```yaml
# templates/jobs/db-migrate.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: unicore-db-migrate
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-weight": "1"
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: api-gateway-migrate
          image: ghcr.io/bemindlabs/unicore-api-gateway:latest
          command: ["npx", "prisma", "db", "push", "--accept-data-loss"]
          envFrom:
            - secretRef:
                name: unicore-secrets
```

## Ingress Configuration

Example Ingress manifest with TLS (cert-manager):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: unicore-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # WebSocket support for OpenClaw
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "Upgrade";
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [unicore.example.com]
      secretName: unicore-tls
  rules:
    - host: unicore.example.com
      http:
        paths:
          - path: /api/
            pathType: Prefix
            backend:
              service: { name: unicore-api-gateway, port: { number: 4000 } }
          - path: /auth/
            pathType: Prefix
            backend:
              service: { name: unicore-api-gateway, port: { number: 4000 } }
          - path: /ws
            pathType: Prefix
            backend:
              service: { name: unicore-openclaw, port: { number: 18789 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: unicore-dashboard, port: { number: 3000 } }
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: unicore-api-gateway-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: unicore-api-gateway
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Cloud-Specific Notes

- **AWS EKS**: See [AWS deployment guide](./cloud/aws.md)
- **GCP GKE**: See [GCP deployment guide](./cloud/gcp.md)
- **Azure AKS**: See [Azure deployment guide](./cloud/azure.md)

## Related Documentation

- [Docker Compose deployment](./docker.md) — primary deployment method
- [On-premise deployment](./on-premise.md)
- [Enterprise Edition](../editions/enterprise.md)

---

© 2026 BeMind Technology
