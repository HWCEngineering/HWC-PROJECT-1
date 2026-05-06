# Contributing

## Branching

- `dev` — default branch, all work happens here
- `main` — production, deploys automatically on merge

## Workflow

1. Create a feature branch off `dev`
2. Make your changes
3. Open a PR into `dev`
4. Once merged and verified, merge `dev` → `main` to deploy

## Commit Messages

Use clear, concise messages. No strict format enforced, but prefer:

```
short summary (imperative mood)

Optional longer description if the change is non-obvious.
```

Examples:
- `fix upload timeout for large LAS files`
- `add voxel size validation to config panel`
- `update Container App memory to 4Gi`

## Local Setup

### Frontend (Astro + React)

```bash
npm install
cp apps/web/env.example apps/web/.env
npm run dev:web
```

Runs at `http://localhost:4321`.

### API (Python/FastAPI)

Requires Python 3.11+.

```bash
cd apps/api
python -m venv .venv
source .venv/bin/activate
pip install -e .
cp env.example .env
# Fill in AZURE_CONNECTION_STRING and MONGO_CONNECTION_STRING
uvicorn app.main:app --reload
```

Runs at `http://localhost:8000`. API docs at `http://localhost:8000/docs`.

### Both together

```bash
npm run dev
```

This starts the frontend and API concurrently (requires Python deps installed globally or in a venv that's active).

## Code Style

- Frontend: TypeScript, no Tailwind, CSS custom properties for theming
- API: Python 3.11+, type hints, async/await, Pydantic models
- Shared packages: React JSX, no app-specific logic, props/callbacks for behavior

## Adding a Shared Package

1. Create `packages/your-package/` with a `package.json` and `src/`
2. Export from `src/index.js`
3. Add as a dependency in the consuming app's `package.json` using `"@hwc/your-package": "*"`
4. Run `npm install` from the monorepo root
