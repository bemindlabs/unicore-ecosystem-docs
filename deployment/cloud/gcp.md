# Deploying UniCore on Google Cloud Platform

This guide covers deploying the UniCore platform on Google Cloud Platform using Compute Engine (Docker Compose) or GKE (Kubernetes).

## Recommended Architecture (Production)

```
Cloud DNS
  → Cloud Load Balancing (HTTPS)
    → GKE Autopilot or Standard Cluster
        ├── unicore-dashboard
        ├── unicore-api-gateway
        ├── unicore-erp
        └── ... (all services)

Managed Services
  ├── Cloud SQL (PostgreSQL 16)         ← Replaces unicore-postgres
  ├── Memorystore (Redis 7)             ← Replaces unicore-redis
  └── GCS (Google Cloud Storage)       ← Persistent storage & backups
```

## Option A: Compute Engine + Docker Compose

### Machine Type Selection

| Workload | Machine Type | RAM | vCPU |
|---------|-------------|-----|------|
| Development | e2-standard-4 | 16 GB | 4 |
| Medium production | n2-standard-8 | 32 GB | 8 |
| Large production | n2-standard-16 | 64 GB | 16 |

### Setup Steps

#### 1. Create VM Instance

```bash
gcloud compute instances create unicore-prod \
  --project=<your-project-id> \
  --zone=us-central1-a \
  --machine-type=n2-standard-8 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --boot-disk-size=50GB \
  --boot-disk-type=pd-ssd \
  --tags=unicore-server

# Allow HTTP/HTTPS traffic
gcloud compute firewall-rules create allow-unicore-web \
  --allow tcp:80,tcp:443 \
  --target-tags unicore-server \
  --source-ranges 0.0.0.0/0
```

#### 2. Install Docker

```bash
gcloud compute ssh unicore-prod --zone=us-central1-a

# On the VM:
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
```

#### 3. Deploy UniCore

Follow the standard [Docker Compose deployment guide](../docker.md).

#### 4. Configure SSL

**Option 1: Nginx Proxy Manager with Let's Encrypt** (recommended for simplicity)

**Option 2: Cloud Load Balancer with Managed SSL**

```bash
# Reserve a static IP
gcloud compute addresses create unicore-ip \
  --global \
  --project=<your-project-id>

# Create a Google-managed SSL certificate
gcloud compute ssl-certificates create unicore-ssl \
  --domains=unicore.example.com \
  --global

# Create HTTPS load balancer pointing to the VM
# (Done via Cloud Console or Terraform)
```

#### 5. Configure Cloud DNS

```bash
# Get the reserved IP
gcloud compute addresses describe unicore-ip --global --format="value(address)"

# Create A record
gcloud dns record-sets create unicore.example.com. \
  --zone=<your-dns-zone> \
  --type=A \
  --ttl=300 \
  --rrdatas=YOUR_IP
```

### Persistent Disk for Data

```bash
# Create and attach a persistent disk
gcloud compute disks create unicore-data \
  --size=200GB \
  --type=pd-ssd \
  --zone=us-central1-a

gcloud compute instances attach-disk unicore-prod \
  --disk=unicore-data \
  --zone=us-central1-a

# On the VM: format and mount
sudo mkfs.ext4 /dev/sdb
sudo mkdir -p /data
sudo mount /dev/sdb /data

# Persist mount in /etc/fstab
echo "/dev/sdb /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

## Option B: GKE (Kubernetes)

### Create GKE Cluster

```bash
# GKE Autopilot (recommended — fully managed nodes)
gcloud container clusters create-auto unicore \
  --location=us-central1 \
  --project=<your-project-id>

# Or GKE Standard (more control)
gcloud container clusters create unicore \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=n2-standard-4 \
  --enable-autoscaling \
  --min-nodes=2 \
  --max-nodes=10 \
  --project=<your-project-id>

# Get credentials
gcloud container clusters get-credentials unicore \
  --zone=us-central1-a \
  --project=<your-project-id>
```

### Managed Services Integration

**Cloud SQL (PostgreSQL 16)**

```bash
gcloud sql instances create unicore-postgres \
  --database-version=POSTGRES_16 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --availability-type=REGIONAL \
  --storage-type=SSD \
  --storage-size=100GB \
  --project=<your-project-id>

gcloud sql databases create unicore \
  --instance=unicore-postgres

gcloud sql databases create unicore_erp \
  --instance=unicore-postgres

gcloud sql users create unicore \
  --instance=unicore-postgres \
  --password=<db-password>
```

Use the **Cloud SQL Auth Proxy** sidecar in GKE pods for secure connectivity:

```yaml
# In your Deployment spec:
containers:
  - name: cloud-sql-proxy
    image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2
    args:
      - "--structured-logs"
      - "--port=5432"
      - "<your-project-id>:us-central1:unicore-postgres"
```

**Memorystore (Redis)**

```bash
gcloud redis instances create unicore-redis \
  --size=1 \
  --region=us-central1 \
  --redis-version=redis_7_0 \
  --tier=STANDARD_HA \
  --project=<your-project-id>
```

Set `REDIS_URL` to the Memorystore host IP and port.

### GKE Ingress with Google-Managed SSL

```bash
# Install the GKE Ingress controller (pre-installed on GKE)
# Create a managed certificate
cat <<EOF | kubectl apply -f -
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: unicore-cert
spec:
  domains:
    - unicore.example.com
EOF

# Reference in Ingress:
# annotations:
#   networking.gke.io/managed-certificates: unicore-cert
#   kubernetes.io/ingress.class: "gce"
```

### Deploy UniCore via Helm

```bash
helm install unicore unicore/unicore \
  --namespace unicore \
  --create-namespace \
  --set global.edition=full \
  --set postgres.external.enabled=true \
  --set postgres.external.host=127.0.0.1 \
  --set postgres.external.port=5432 \
  --set redis.external.enabled=true \
  --set redis.external.host=<redis-host-ip> \
  --set ingress.host=unicore.example.com \
  --set license.key=UC-XXXX-XXXX-XXXX-XXXX
```

## Storage: Google Cloud Storage (GCS)

```bash
# Create bucket for backups
gsutil mb -p <your-project-id> \
  -c STANDARD \
  -l us-central1 \
  gs://unicore-backups-yourorg/

# Enable versioning
gsutil versioning set on gs://unicore-backups-yourorg/

# Automated PostgreSQL backup
# Replace <postgres-container> with your actual container name
0 2 * * * docker exec <postgres-container> \
  pg_dump -U unicore unicore | gzip | \
  gsutil cp - gs://unicore-backups-yourorg/postgres/unicore-$(date +%Y%m%d).sql.gz
```

## IAM & Service Accounts

```bash
# Create service account for UniCore
gcloud iam service-accounts create unicore-sa \
  --display-name="UniCore Service Account" \
  --project=<your-project-id>

# Grant Cloud SQL access
gcloud projects add-iam-policy-binding <your-project-id> \
  --member="serviceAccount:unicore-sa@<your-project-id>.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

# Grant GCS access for backups
gcloud projects add-iam-policy-binding <your-project-id> \
  --member="serviceAccount:unicore-sa@<your-project-id>.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

## Cost Estimation (Monthly, us-central1)

| Component | Type | Estimated Cost |
|-----------|------|---------------|
| Compute Engine (n2-standard-8) | On-Demand | ~$200 |
| Cloud SQL (db-n1-standard-2, HA) | On-Demand | ~$150 |
| Memorystore Redis (1 GB, HA) | On-Demand | ~$50 |
| Cloud Load Balancing | Per rule | ~$25 |
| Persistent SSD (200 GB) | Storage | ~$34 |
| Cloud Storage (50 GB) | Storage | ~$1 |
| **Total estimate** | | **~$460/month** |

Use [Google Cloud Pricing Calculator](https://cloud.google.com/products/calculator) for accurate estimates.

## Related Documentation

- [Docker Compose deployment](../docker.md)
- [Kubernetes deployment](../kubernetes.md)
- [AWS deployment](./aws.md)
- [Azure deployment](./azure.md)
- [On-premise deployment](../on-premise.md)

---

© 2026 BeMind Technology
