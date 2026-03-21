# Deploying UniCore on Microsoft Azure

This guide covers deploying the UniCore platform on Microsoft Azure using Azure Virtual Machines (Docker Compose) or AKS (Kubernetes).

## Recommended Architecture (Production)

```
Azure DNS
  → Azure Front Door or Application Gateway (WAF + SSL)
    → AKS Cluster
        ├── unicore-dashboard
        ├── unicore-api-gateway
        ├── unicore-erp
        └── ... (all services)

Managed Services
  ├── Azure Database for PostgreSQL (Flexible Server)  ← Replaces unicore-postgres
  ├── Azure Cache for Redis                            ← Replaces unicore-redis
  └── Azure Blob Storage                               ← Backups & persistent data
```

## Option A: Azure VM + Docker Compose

### VM Size Selection

| Workload | VM Size | RAM | vCPU |
|---------|---------|-----|------|
| Development | Standard_D4s_v3 | 16 GB | 4 |
| Medium production | Standard_D8s_v3 | 32 GB | 8 |
| Large production | Standard_D16s_v3 | 64 GB | 16 |

### Setup Steps

#### 1. Create Resource Group and VM

```bash
# Create resource group
az group create \
  --name unicore-rg \
  --location eastus

# Create VM (Ubuntu 22.04)
az vm create \
  --resource-group unicore-rg \
  --name unicore-prod \
  --image Ubuntu2204 \
  --size Standard_D8s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --public-ip-sku Standard

# Open HTTP/HTTPS ports
az vm open-port \
  --resource-group unicore-rg \
  --name unicore-prod \
  --port 80,443 \
  --priority 100
```

#### 2. Install Docker

```bash
ssh azureuser@<VM_PUBLIC_IP>

# On the VM:
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
sudo usermod -aG docker azureuser
```

#### 3. Deploy UniCore

Follow the standard [Docker Compose deployment guide](../docker.md).

#### 4. Configure SSL with Application Gateway

```bash
# Create Application Gateway with SSL termination
az network application-gateway create \
  --resource-group unicore-rg \
  --name unicore-appgw \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name unicore-vnet \
  --subnet appgw-subnet \
  --public-ip-address unicore-pip \
  --frontend-port 443 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic

# Upload SSL certificate
az network application-gateway ssl-cert create \
  --resource-group unicore-rg \
  --gateway-name unicore-appgw \
  --name unicore-ssl \
  --cert-file ./unicore.pfx \
  --cert-password "<cert-password>"
```

#### 5. Configure Azure DNS

```bash
# Create DNS zone (if not exists)
az network dns zone create \
  --resource-group unicore-rg \
  --name example.com

# Create A record pointing to Application Gateway public IP
az network dns record-set a add-record \
  --resource-group unicore-rg \
  --zone-name example.com \
  --record-set-name unicore \
  --ipv4-address <APPGW_PUBLIC_IP>
```

### Managed Disk for Data

```bash
# Create and attach a managed disk
az disk create \
  --resource-group unicore-rg \
  --name unicore-data \
  --size-gb 200 \
  --sku Premium_LRS

az vm disk attach \
  --resource-group unicore-rg \
  --vm-name unicore-prod \
  --name unicore-data \
  --new

# On the VM: format and mount
sudo fdisk -l  # find the new disk (e.g., /dev/sdc)
sudo mkfs.ext4 /dev/sdc
sudo mkdir -p /data
sudo mount /dev/sdc /data
echo "/dev/sdc /data ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

## Option B: AKS (Kubernetes)

### Create AKS Cluster

```bash
# Create AKS cluster
az aks create \
  --resource-group unicore-rg \
  --name unicore-aks \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10 \
  --network-plugin azure \
  --network-policy azure \
  --generate-ssh-keys

# Get credentials
az aks get-credentials \
  --resource-group unicore-rg \
  --name unicore-aks
```

### Managed Services Integration

**Azure Database for PostgreSQL (Flexible Server)**

```bash
az postgres flexible-server create \
  --resource-group unicore-rg \
  --name unicore-postgres \
  --location eastus \
  --admin-user unicore \
  --admin-password your-password \
  --sku-name Standard_D2s_v3 \
  --tier GeneralPurpose \
  --version 16 \
  --storage-size 128 \
  --high-availability Enabled

# Create databases
az postgres flexible-server db create \
  --resource-group unicore-rg \
  --server-name unicore-postgres \
  --database-name unicore

az postgres flexible-server db create \
  --resource-group unicore-rg \
  --server-name unicore-postgres \
  --database-name unicore_erp
```

**Azure Cache for Redis**

```bash
az redis create \
  --resource-group unicore-rg \
  --name unicore-redis \
  --location eastus \
  --sku Standard \
  --vm-size C1 \
  --redis-version 7.0 \
  --enable-non-ssl-port false
```

Get the Redis connection string:

```bash
az redis list-keys \
  --resource-group unicore-rg \
  --name unicore-redis \
  --query primaryKey
```

### Install NGINX Ingress Controller on AKS

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

### Install cert-manager with Let's Encrypt

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Create ClusterIssuer for Let's Encrypt
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <your-email>@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

### Deploy UniCore via Helm

```bash
helm install unicore unicore/unicore \
  --namespace unicore \
  --create-namespace \
  --set global.edition=full \
  --set postgres.external.enabled=true \
  --set postgres.external.host=unicore-postgres.postgres.database.azure.com \
  --set redis.external.enabled=true \
  --set redis.external.host=unicore-redis.redis.cache.windows.net \
  --set redis.external.tls=true \
  --set ingress.host=unicore.example.com \
  --set ingress.certIssuer=letsencrypt-prod \
  --set license.key=UC-XXXX-XXXX-XXXX-XXXX
```

## Azure Entra ID (SSO Integration)

UniCore Pro/Enterprise can integrate with Azure Entra ID (formerly Azure AD) for SSO:

### Register Application

```bash
# Register UniCore as an Entra ID application
az ad app create \
  --display-name "UniCore" \
  --sign-in-audience AzureADMyOrg \
  --web-redirect-uris "https://unicore.example.com/auth/sso/callback"

# Get the app ID and tenant ID
az ad app list --display-name "UniCore" \
  --query "[].{appId:appId,tenantId:publisherDomain}"
```

### Configure SSO in UniCore

```bash
# Set environment variables in your compose or Kubernetes secret:
SSO_PROVIDER=azure
SSO_TENANT_ID=<azure-tenant-id>
SSO_CLIENT_ID=<azure-app-client-id>
SSO_CLIENT_SECRET=<azure-app-client-secret>
SSO_REDIRECT_URI=https://unicore.example.com/auth/sso/callback
```

## Azure Blob Storage (Backups)

```bash
# Create storage account
az storage account create \
  --resource-group unicore-rg \
  --name unicorebackupsyourorg \
  --location eastus \
  --sku Standard_LRS

# Create container for backups
az storage container create \
  --name postgres-backups \
  --account-name unicorebackupsyourorg

# Automated backup (cron job using azcopy)
# Replace <postgres-container> with your actual container name
0 2 * * * docker exec <postgres-container> \
  pg_dump -U unicore unicore | gzip | \
  azcopy copy - \
  "https://unicorebackupsyourorg.blob.core.windows.net/postgres-backups/unicore-$(date +%Y%m%d).sql.gz"
```

## Azure Monitor & Logging

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group unicore-rg \
  --workspace-name unicore-logs

# Enable container insights on AKS
az aks enable-addons \
  --resource-group unicore-rg \
  --name unicore-aks \
  --addons monitoring \
  --workspace-resource-id $(az monitor log-analytics workspace show \
    --resource-group unicore-rg \
    --workspace-name unicore-logs \
    --query id -o tsv)
```

## Cost Estimation (Monthly, East US)

| Component | Type | Estimated Cost |
|-----------|------|---------------|
| VM (Standard_D8s_v3) | Pay-as-you-go | ~$280 |
| Azure DB PostgreSQL (D2s v3, HA) | Pay-as-you-go | ~$180 |
| Azure Cache Redis (C1 Standard) | Pay-as-you-go | ~$55 |
| Application Gateway (WAF_v2) | Pay-as-you-go | ~$125 |
| Managed Disk (200 GB Premium) | Storage | ~$30 |
| Blob Storage (50 GB LRS) | Storage | ~$1 |
| **Total estimate** | | **~$671/month** |

Use the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/) for accurate estimates.

## Related Documentation

- [Docker Compose deployment](../docker.md)
- [Kubernetes deployment](../kubernetes.md)
- [AWS deployment](./aws.md)
- [GCP deployment](./gcp.md)
- [On-premise deployment](../on-premise.md)

---

© 2026 BeMind Technology
