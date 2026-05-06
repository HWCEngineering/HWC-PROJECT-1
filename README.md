# HWC Breakline Generator

Monorepo for the HWC LiDAR surface/breakline generation platform.

## Structure

| Path | Description |
|---|---|
| `apps/api` | FastAPI backend — LiDAR processing, Azure Blob Storage, Cosmos DB |
| `apps/web` | Astro + React frontend — file upload, configuration, preview, download |
| `packages/` | Shared frontend packages (`assets`, `header`, `map`, `panel`, `potree`, `photo-panel`) |

## Quick Start

```bash
# Install all workspace dependencies
npm install

# Sync shared assets into each app
npm run assets:sync

# Start both frontend and API locally
npm run dev
```

- Frontend: `http://localhost:4321`
- API: `http://localhost:8000`

See [`apps/api/README.md`](apps/api/README.md) and [`apps/web/README.md`](apps/web/README.md) for app-specific details.

## Repository Secrets

| Secret | Usage |
|---|---|
| `AZURE_CREDENTIALS` | Azure service principal JSON for CLI login |
| `AZURE_RESOURCE_GROUP` | Target resource group for Container Apps |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Deploy token for Azure Static Web Apps |
| `AZURE_CONNECTION_STRING` | Azure Blob Storage connection string |
| `MONGO_CONNECTION_STRING` | Cosmos DB (MongoDB API) connection string |
| `GHCR_READ_TOKEN` | GitHub PAT with `read:packages` scope (used by Azure Container Apps to pull images from GHCR) |

## Repository Variables

| Variable | Usage |
|---|---|
| `NAME` | Shared project name (blob container + MongoDB database + Static Web App name) |
| `PUBLIC_MAPTILER_API_KEY` | MapTiler API key for map tiles (optional — ESRI layers work without it) |

## CI/CD

| Workflow | Trigger | What it does |
|---|---|---|
| `.github/workflows/dev.yml` | Push to `dev` or PR into `dev` | Builds API image + frontend (validation only, no deploy) |
| `.github/workflows/production.yml` | Push to `main` or manual dispatch | Builds & pushes API image to GHCR → deploys to Azure Container Apps → builds & deploys frontend to Azure Static Web Apps |

**Container Registry**: GitHub Container Registry (GHCR) at `ghcr.io/{owner}/hwc-breakline-gen-api`

**Image Tags**: `:dev-{sha}` / `:dev-latest` (dev) and `:{sha}` / `:latest` (production)

## Branching Strategy

| Branch | Purpose |
|---|---|
| `dev` | Default branch. All feature work and PRs merge here. |
| `main` | Production branch. Merge from `dev` triggers deployment. |

See [`DEPLOYMENT.md`](DEPLOYMENT.md) for full deployment setup and environment details.
