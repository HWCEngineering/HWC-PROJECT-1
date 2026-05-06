# Deployment Guide

This document covers how the HWC Breakline Generator is deployed and how to set up the infrastructure from scratch.

## Architecture Overview

```
┌─────────────┐       ┌──────────────────────┐       ┌─────────────────────┐
│  Astro SPA  │──────▶│  FastAPI (Container)  │──────▶│  Azure Blob Storage │
│  Static Web │       │  Azure Container Apps │       │  (LAS/DXF/CSV)      │
└─────────────┘       └──────────────────────┘       └─────────────────────┘
                              │
                              ▼
                      ┌──────────────────┐
                      │  Cosmos DB       │
                      │  (MongoDB API)   │
                      └──────────────────┘
```

**Flow**: User uploads LAS/LAZ → API stores in Blob Storage → background job processes → outputs uploaded → user downloads via SAS URL.

## Azure Resources Required

| Resource | Service | Notes |
|---|---|---|
| Container App Environment | Azure Container Apps | Hosts the API container |
| Container App | Azure Container Apps | `hwc-breakline-gen-api`, 1 CPU / 2GB RAM, 0–3 replicas |
| Static Web App | Azure Static Web Apps | Hosts the Astro frontend |
| Storage Account | Azure Blob Storage | Container named after `NAME` variable |
| Cosmos DB Account | Azure Cosmos DB (MongoDB API) | Database named after `NAME` variable |
| Service Principal | Azure AD | Used by GitHub Actions for `az login` |

## Setting Up From Scratch

### 1. Create Azure Resources

```bash
# Variables
RG="your-resource-group"
LOCATION="eastus"
NAME="hwc-breakline-generator"

# Resource group
az group create --name $RG --location $LOCATION

# Storage account
az storage account create \
  --name hwcbreaklinesa \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

# Create blob container
az storage container create \
  --name $NAME \
  --account-name hwcbreaklinesa

# Cosmos DB (MongoDB API)
az cosmosdb create \
  --name hwc-breakline-cosmos \
  --resource-group $RG \
  --kind MongoDB \
  --server-version 4.2

# Container App Environment
az containerapp env create \
  --name my-projects-env \
  --resource-group $RG \
  --location $LOCATION

# Static Web App
az staticwebapp create \
  --name $NAME \
  --resource-group $RG \
  --location $LOCATION
```

### 2. Create Service Principal

```bash
az ad sp create-for-rbac \
  --name "github-actions-hwc" \
  --role contributor \
  --scopes /subscriptions/{sub-id}/resourceGroups/$RG \
  --sdk-auth
```

Save the JSON output as the `AZURE_CREDENTIALS` secret.

### 3. Create GHCR Read Token

1. Go to GitHub → Settings → Developer settings → Personal access tokens
2. Create a token with `read:packages` scope
3. Save as `GHCR_READ_TOKEN` repository secret

This token is used by Azure Container Apps to pull images from GHCR.

### 4. Configure GitHub Secrets & Variables

**Secrets** (Settings → Secrets and variables → Actions → Secrets):

| Secret | How to get it |
|---|---|
| `AZURE_CREDENTIALS` | Service principal JSON from step 2 |
| `AZURE_RESOURCE_GROUP` | Your resource group name |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | `az staticwebapp secrets list --name $NAME` |
| `AZURE_CONNECTION_STRING` | `az storage account show-connection-string --name hwcbreaklinesa` |
| `MONGO_CONNECTION_STRING` | Azure Portal → Cosmos DB → Connection strings |
| `GHCR_READ_TOKEN` | PAT from step 3 |

**Variables** (Settings → Secrets and variables → Actions → Variables):

| Variable | Value |
|---|---|
| `NAME` | `hwc-breakline-generator` (or your chosen name) |
| `PUBLIC_MAPTILER_API_KEY` | Your MapTiler key (optional) |

## Deployment Pipeline

Two separate workflows in `.github/workflows/`:

### `dev.yml` — CI Validation (push to `dev` or PR into `dev`)

1. **Build API image** — Docker build, push to GHCR with `dev-` prefix tags (skipped on PRs)
2. **Build frontend** — Astro build with placeholder API URL (validates compilation)

No deployment. This is a build/test gate only.

### `production.yml` — Full Deploy (push to `main`)

1. **Build & push API image** — Docker build from `apps/api/`, push to GHCR with `:{sha}` and `:latest`
2. **Deploy API** — Create/update Azure Container App with new image
3. **Capture API URL** — Read the Container App's FQDN
4. **Build frontend** — Astro build with `PUBLIC_API_BASE_URL` set to live API
5. **Deploy frontend** — Push to Azure Static Web Apps

The frontend job depends on the API job (needs the live URL).

## Branching & Environments

Recommended setup for handoff:

| Branch | Deploys to | Workflow trigger |
|---|---|---|
| `dev` | Nothing (or staging if you add a second environment) | — |
| `main` | Production (Azure) | `push: branches: ["main"]` |

**Workflow**:
1. Create feature branches off `dev`
2. Open PRs into `dev`
3. When ready to release, merge `dev` → `main`
4. Push to `main` triggers the full deploy

To add a staging environment, duplicate the workflow with different Azure resource targets and trigger on `dev`.

## Container App Details

| Setting | Value |
|---|---|
| Image | `ghcr.io/{owner}/hwc-breakline-gen-api:{sha}` |
| CPU / Memory | 1.0 / 2.0 Gi |
| Min replicas | 0 (scales to zero when idle — cold starts ~10s) |
| Max replicas | 3 |
| Ingress | External, port 8000 |
| Environment | `my-projects-env` |

**Note**: `min-replicas: 0` means the first request after idle will have a cold start. If latency matters, set to 1.

## Known Considerations

- **CORS**: API currently allows all origins (`["*"]`). Tighten to the Static Web App URL for production.
- **Authentication**: No API auth is implemented. The API is publicly accessible. Consider adding API keys or Azure AD auth if exposing beyond internal use.
- **Cleanup**: Jobs and blobs are automatically deleted after 24 hours by the cleanup service.
- **File size limit**: Configured at 10GB (`max_file_size_mb: 10000` in config). Adjust in `apps/api/app/config.py`.

## Troubleshooting

| Issue | Fix |
|---|---|
| Container App can't pull image | Verify `GHCR_READ_TOKEN` has `read:packages` scope and the image is published |
| Frontend shows "API unavailable" | Check `PUBLIC_API_BASE_URL` was captured correctly in the workflow logs |
| Jobs stuck in `queued` | API container might have scaled to 0. Send a health check to wake it up |
| Blob upload fails | Verify `AZURE_CONNECTION_STRING` and that the container named `NAME` exists |
| MongoDB connection timeout | Check `MONGO_CONNECTION_STRING` and Cosmos DB firewall rules |
