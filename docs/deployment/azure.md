---
description: "Deploy FastSvelte on Azure - Container Apps for backend, PostgreSQL Flexible Server for database, and Static Web Apps for frontend."
keywords: "azure deployment, azure container apps, azure postgresql, azure static web apps, fastsvelte azure"
---

# Deploy to Azure

Deploy FastSvelte using Azure's modern container and serverless services. This guide covers the essential Azure resources you'll need.

## What You'll Use

- **Azure Container Apps** - Run your FastAPI backend container
- **PostgreSQL Flexible Server** - Managed PostgreSQL database
- **Azure Static Web Apps** - Host your SvelteKit frontend and landing page
- **Azure Container Registry** - Store your Docker images (built-in)

## Prerequisites

- Azure account with active subscription
- Azure CLI installed ([install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli))
- Docker installed locally
- Your FastSvelte codebase ready

## Cost Estimate

For a small production app:
- Container Apps: ~$15-30/month
- PostgreSQL Flexible Server: ~$50-100/month (can use [Neon](https://neon.tech) for ~$20/month instead)
- Static Web Apps: Free tier available, ~$10/month for standard
- Container Registry: ~$5/month

**Total:** ~$70-145/month (or ~$50/month with Neon for database)

## Architecture

```
Internet
  ↓
Azure Front Door (optional CDN)
  ↓
Static Web Apps (Frontend) → Container Apps (Backend) → PostgreSQL
```

## Deployment Steps

### 1. Login to Azure

```bash
az login
az account set --subscription "Your Subscription Name"
```

### 2. Create Resource Group

```bash
# Create resource group in your preferred region
az group create \
  --name fastsvelte-prod-rg \
  --location eastus
```

### 3. Create PostgreSQL Database

```bash
# Create PostgreSQL Flexible Server
az postgres flexible-server create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-db \
  --location eastus \
  --admin-user fsadmin \
  --admin-password <STRONG-PASSWORD> \
  --sku-name Standard_B1ms \
  --storage-size 32 \
  --version 16

# Allow Azure services to connect
az postgres flexible-server firewall-rule create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-db \
  --rule-name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Create database
az postgres flexible-server db create \
  --resource-group fastsvelte-prod-rg \
  --server-name fastsvelte-prod-db \
  --database-name fastsvelte
```

**Alternative:** Use [Neon](https://neon.tech) or [Supabase](https://supabase.com) for a more affordable managed Postgres option.

### 4. Run Database Migrations

```bash
cd db

# Add production database URL to .env
echo 'DATABASE_URL_PROD="postgres://fsadmin:<PASSWORD>@fastsvelte-prod-db.postgres.database.azure.com/fastsvelte?sslmode=require"' >> .env

# Deploy migrations
./sqitch.sh prod deploy

# Verify
./sqitch.sh prod status
```

### 5. Create Admin User

```bash
cd ../backend

# Create production admin account
uv run scripts/create_admin.py \
  --env prod \
  --password <ADMIN-PASSWORD> \
  --domain yourdomain.com
```

### 6. Create Container Registry

```bash
# Create Azure Container Registry
az acr create \
  --resource-group fastsvelte-prod-rg \
  --name faststvelteregistry \
  --sku Basic \
  --admin-enabled true

# Get registry credentials
az acr credential show --name faststvelteregistry
```

### 7. Build and Push Backend Container

```bash
# Login to registry
az acr login --name faststvelteregistry

# Build and push backend image
cd backend
docker build -t faststvelteregistry.azurecr.io/fastsvelte-api:latest .
docker push faststvelteregistry.azurecr.io/fastsvelte-api:latest
```

### 8. Create Container Apps Environment

```bash
# Create Container Apps Environment
az containerapp env create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-env \
  --location eastus
```

### 9. Deploy Backend API

```bash
# Get registry credentials
REGISTRY_USERNAME=$(az acr credential show --name faststvelteregistry --query username -o tsv)
REGISTRY_PASSWORD=$(az acr credential show --name faststvelteregistry --query passwords[0].value -o tsv)

# Create Container App
az containerapp create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --environment fastsvelte-prod-env \
  --image faststvelteregistry.azurecr.io/fastsvelte-api:latest \
  --target-port 3100 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --cpu 1.0 \
  --memory 2Gi \
  --registry-server faststvelteregistry.azurecr.io \
  --registry-username $REGISTRY_USERNAME \
  --registry-password $REGISTRY_PASSWORD
```

### 10. Configure Environment Variables

```bash
# Set environment variables for the API
az containerapp update \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --set-env-vars \
    FS_ENVIRONMENT=prod \
    FS_DB_URL="postgres://fsadmin:<PASSWORD>@fastsvelte-prod-db.postgres.database.azure.com/fastsvelte?sslmode=require" \
    FS_JWT_SECRET_KEY="<RANDOM-SECRET-KEY>" \
    FS_BASE_API_URL="https://<YOUR-CONTAINER-APP-URL>" \
    FS_BASE_WEB_URL="https://app.yourdomain.com" \
    FS_CORS_ORIGINS="https://app.yourdomain.com,https://yourdomain.com" \
    FS_STRIPE_API_KEY="<YOUR-STRIPE-KEY>" \
    FS_STRIPE_WEBHOOK_SECRET="<YOUR-STRIPE-WEBHOOK-SECRET>" \
    FS_SENDGRID_API_KEY="<YOUR-SENDGRID-KEY>"
```

Get your Container App URL:
```bash
az containerapp show \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --query properties.configuration.ingress.fqdn \
  --output tsv
```

### 11. Deploy Frontend to Static Web Apps

```bash
# Create Static Web App for frontend
az staticwebapp create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-app \
  --location eastus \
  --source https://github.com/yourusername/your-fastsvelte-fork \
  --branch main \
  --app-location "frontend" \
  --output-location "build" \
  --login-with-github
```

This will:
1. Connect to your GitHub repository
2. Set up GitHub Actions for automatic deployments
3. Build and deploy your frontend

**Configure environment variables** in Azure Portal:
1. Go to Static Web App → Configuration
2. Add application settings:
   - `PUBLIC_API_BASE_URL`: Your Container App URL
   - `PUBLIC_APP_NAME`: Your app name

### 12. Deploy Landing Page

```bash
# Create Static Web App for landing
az staticwebapp create \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-landing \
  --location eastus \
  --source https://github.com/yourusername/your-fastsvelte-fork \
  --branch main \
  --app-location "landing" \
  --output-location "build" \
  --login-with-github
```

## Custom Domains (Optional)

### For Container App (API)

```bash
az containerapp hostname bind \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --hostname api.yourdomain.com
```

### For Static Web Apps

```bash
az staticwebapp hostname set \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-app \
  --hostname app.yourdomain.com
```

## Scaling Configuration

Container Apps auto-scale based on HTTP requests by default. Configure custom scaling:

```bash
az containerapp update \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --min-replicas 2 \
  --max-replicas 10 \
  --scale-rule-name "http-scale" \
  --scale-rule-type "http" \
  --scale-rule-metadata "concurrentRequests=50"
```

## Monitoring & Logs

View Container App logs:

```bash
# Stream live logs
az containerapp logs show \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --follow

# Recent logs
az containerapp logs show \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --tail 100
```

## Updating Your Application

### Update Backend

```bash
# Build new image
cd backend
docker build -t faststvelteregistry.azurecr.io/fastsvelte-api:v2 .
docker push faststvelteregistry.azurecr.io/fastsvelte-api:v2

# Update Container App
az containerapp update \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --image faststvelteregistry.azurecr.io/fastsvelte-api:v2
```

### Update Frontend

Frontend updates automatically via GitHub Actions when you push to your repository.

## Security Best Practices

1. **Use Azure Key Vault** for secrets instead of environment variables:
   ```bash
   az keyvault create \
     --resource-group fastsvelte-prod-rg \
     --name fastsvelte-vault
   ```

2. **Enable managed identity** for Container Apps to access Key Vault

3. **Restrict database access** to only Container Apps IP range

4. **Enable Azure Front Door** for DDoS protection and CDN

5. **Set up Azure Monitor** for alerts on errors and performance

## Troubleshooting

**Container App won't start:**
```bash
# Check logs
az containerapp logs show \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --follow
```

**Database connection issues:**
- Verify firewall rules allow Container Apps
- Check connection string format
- Ensure SSL mode is set to `require`

**Static Web App build fails:**
- Check GitHub Actions logs in your repository
- Verify `app-location` and `output-location` paths
- Ensure Node.js version compatibility

## Cost Optimization Tips

1. **Use Neon/Supabase for database** - Save ~$30-80/month compared to Azure PostgreSQL
2. **Start with 1 replica** - Scale up only when needed
3. **Use consumption-based Container Apps** - Pay only for actual usage
4. **Combine frontend and landing** - Host both on same Static Web App if possible
5. **Use Azure Reservations** - Save up to 38% with 1-year commitment for predictable workloads

## Next Steps

### Security headers

Add security headers to the app by creating `frontend/staticwebapp.config.json`:

```json
{
  "globalHeaders": {
    "Content-Security-Policy": "default-src 'self'; connect-src 'self' https://api.yourdomain.com; img-src 'self' data:; style-src 'self' 'unsafe-inline'; script-src 'self'; font-src 'self'; object-src 'none'; base-uri 'self'; form-action 'self'; frame-ancestors 'none'",
    "X-Content-Type-Options": "nosniff",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()"
  }
}
```

Replace `https://api.yourdomain.com` in `connect-src` with your real API URL, or the browser will block the app from calling it. See [Security](../features/security.md#frontend-add-at-your-host).

### Other

- Set up monitoring and alerts
- Configure backups for database
- Set up CI/CD pipeline for automated deployments
- Configure custom domains
- Set up Azure Front Door for CDN

## Useful Commands

```bash
# View all resources
az resource list --resource-group fastsvelte-prod-rg --output table

# Get Container App URL
az containerapp show \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --query properties.configuration.ingress.fqdn

# Scale Container App
az containerapp update \
  --resource-group fastsvelte-prod-rg \
  --name fastsvelte-prod-api \
  --min-replicas 2 \
  --max-replicas 10

# Delete everything (careful!)
az group delete --name fastsvelte-prod-rg
```
