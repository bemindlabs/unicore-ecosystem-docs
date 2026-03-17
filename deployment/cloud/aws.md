# Deploying UniCore on AWS

This guide covers deploying the UniCore platform on Amazon Web Services using EC2 (Docker Compose) or EKS (Kubernetes).

## Recommended Architecture (Production)

```
Route 53 (DNS)
  → CloudFront (optional CDN)
    → Application Load Balancer (ALB)
      → ECS / EKS Cluster
          ├── unicore-dashboard
          ├── unicore-api-gateway
          ├── unicore-erp
          └── ... (all services)

Managed Services
  ├── Amazon RDS (PostgreSQL 16)        ← Replaces unicore-postgres
  ├── Amazon ElastiCache (Redis 7)      ← Replaces unicore-redis
  ├── Amazon MSK (Kafka)                ← Replaces unicore-kafka (optional)
  └── Amazon EFS / S3                  ← Persistent storage for Qdrant
```

For simpler deployments, a single EC2 instance running Docker Compose is sufficient for small to medium workloads.

## Option A: EC2 + Docker Compose

### Instance Sizing

| Workload | Instance Type | RAM | vCPU |
|---------|--------------|-----|------|
| Development / Small | t3.xlarge | 16 GB | 4 |
| Medium production | m6i.2xlarge | 32 GB | 8 |
| Large production | m6i.4xlarge | 64 GB | 16 |

### Setup Steps

#### 1. Launch EC2 Instance

```bash
# Use Amazon Linux 2023 or Ubuntu 22.04 LTS
# Attach a security group with:
# - Port 22 (SSH) from your IP
# - Port 80 (HTTP) from 0.0.0.0/0
# - Port 443 (HTTPS) from 0.0.0.0/0

# Attach an IAM role with SSM access (optional, replaces SSH key)
```

#### 2. Install Docker

```bash
# Ubuntu 22.04
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli \
  containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

#### 3. Deploy UniCore

Follow the standard [Docker Compose deployment guide](../docker.md).

#### 4. Configure SSL with ACM + ALB

- Request a certificate in AWS Certificate Manager (ACM) for your domain
- Create an Application Load Balancer (ALB) in front of the EC2 instance
- Add HTTPS listener (port 443) with the ACM certificate
- Add HTTP listener (port 80) with a redirect to HTTPS
- Target group: forward to EC2 port 80

#### 5. Configure Route 53

```
A record (or ALIAS): unicore.example.com → ALB DNS name
```

### EBS Volume for Data

Attach a separate EBS volume for Docker volumes (PostgreSQL, Redis, Qdrant data):

```bash
# Format and mount EBS volume
sudo mkfs.ext4 /dev/xvdf
sudo mkdir -p /data
sudo mount /dev/xvdf /data

# Move Docker data root to EBS
sudo systemctl stop docker
sudo mv /var/lib/docker /data/docker
sudo ln -s /data/docker /var/lib/docker
sudo systemctl start docker
```

## Option B: EKS (Kubernetes)

### Cluster Setup

```bash
# Install eksctl
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create cluster
eksctl create cluster \
  --name unicore \
  --region us-east-1 \
  --nodegroup-name unicore-nodes \
  --node-type m6i.2xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed

# Update kubeconfig
aws eks update-kubeconfig --name unicore --region us-east-1
```

### Managed Services Integration

**RDS PostgreSQL**

```bash
# Create RDS instance (PostgreSQL 16)
aws rds create-db-instance \
  --db-instance-identifier unicore-postgres \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16 \
  --master-username unicore \
  --master-user-password your-password \
  --allocated-storage 100 \
  --storage-type gp3 \
  --multi-az \
  --vpc-security-group-ids sg-xxxx \
  --db-subnet-group-name unicore-db-subnet-group
```

Set `DATABASE_URL` to the RDS endpoint in your Kubernetes secret.

**ElastiCache Redis**

```bash
aws elasticache create-replication-group \
  --replication-group-id unicore-redis \
  --description "UniCore Redis" \
  --engine redis \
  --engine-version 7.0 \
  --cache-node-type cache.t3.medium \
  --num-cache-clusters 2 \
  --automatic-failover-enabled
```

Set `REDIS_URL` to the ElastiCache primary endpoint.

### Install NGINX Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"=nlb
```

### Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

### Deploy UniCore via Helm

```bash
helm install unicore unicore/unicore \
  --namespace unicore \
  --create-namespace \
  --set global.edition=full \
  --set postgres.external.enabled=true \
  --set postgres.external.host=unicore.cluster.us-east-1.rds.amazonaws.com \
  --set redis.external.enabled=true \
  --set redis.external.host=unicore.cache.us-east-1.amazonaws.com \
  --set ingress.host=unicore.example.com \
  --set license.key=UC-XXXX-XXXX-XXXX-XXXX
```

## Storage (S3 for Backups)

```bash
# Create S3 bucket for backups
aws s3 mb s3://unicore-backups-yourorg --region us-east-1

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket unicore-backups-yourorg \
  --versioning-configuration Status=Enabled

# Automated PostgreSQL backup (cron job)
0 2 * * * docker exec unicores-unicore-postgres-1 \
  pg_dump -U unicore unicore | gzip | \
  aws s3 cp - s3://unicore-backups-yourorg/postgres/unicore-$(date +%Y%m%d).sql.gz
```

## Security Groups

| Group | Inbound Rules |
|-------|--------------|
| ALB SG | 80 (HTTP), 443 (HTTPS) from 0.0.0.0/0 |
| EC2/EKS SG | 80 from ALB SG only |
| RDS SG | 5432 from EC2/EKS SG only |
| ElastiCache SG | 6379 from EC2/EKS SG only |

## Cost Estimation (Monthly, us-east-1)

| Component | Type | Estimated Cost |
|-----------|------|---------------|
| EC2 (m6i.xlarge) | On-Demand | ~$150 |
| RDS (db.t3.medium, Multi-AZ) | On-Demand | ~$100 |
| ElastiCache (cache.t3.medium) | On-Demand | ~$50 |
| ALB | Per LCU | ~$20 |
| EBS (200 GB gp3) | Storage | ~$16 |
| Data transfer | Outbound | ~$20 |
| **Total estimate** | | **~$356/month** |

Costs vary by region and usage. Use [AWS Pricing Calculator](https://calculator.aws) for accurate estimates.

## Related Documentation

- [Docker Compose deployment](../docker.md)
- [Kubernetes deployment](../kubernetes.md)
- [On-premise deployment](../on-premise.md)
- [GCP deployment](./gcp.md)
- [Azure deployment](./azure.md)

---

© 2026 BeMind Technology
