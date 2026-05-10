# HWC Breakline Generator

Web-based tool for extracting surface breaklines from statewide LiDAR point cloud data. Built for civil engineers and surveyors who need CAD-ready breakline files without manual processing.

## What It Does

Users upload raw LiDAR files (LAS/LAZ), configure processing parameters (grid spacing, gradient threshold, coordinate system), and receive DXF and CSV outputs containing extracted breaklines — the lines where terrain changes slope (road edges, ditches, ridgelines, embankments).

```
Upload LAS/LAZ → Configure → Process → Preview → Download DXF/CSV
```

The processing pipeline:
1. Filters ground-classified points (LAS classification 2)
2. Downsamples via voxel grid (10ft, 25ft, or 50ft spacing)
3. Builds Delaunay triangulation
4. Extracts edges where elevation gradient exceeds threshold
5. Exports breakline polylines to DXF and point data to CSV

Jobs run asynchronously — upload returns immediately, processing happens in the background, and results are available for download (24-hour retention).

## Tech Stack

- **API**: Python/FastAPI, Open3D, SciPy, laspy, ezdxf
- **Frontend**: Astro + React (single-page app)
- **Storage**: Azure Blob Storage (file I/O) + Cosmos DB/MongoDB (job state)
- **Hosting**: Azure Container Apps (API) + Azure Static Web Apps (frontend)
- **CI/CD**: GitHub Actions → GHCR → Azure

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
