# SATQUERY

Satellite image analysis demo built with React/Vite and FastAPI. The local baseline analyzes user supplied imagery and labels its source honestly; it does not claim model training or live satellite retrieval.

## Features

- Registration and login with hashed passwords and bearer tokens; admin-only user listing and account state management.
- Land cover color/index baseline, paired-image pixel change comparison, and possible-water screening or before/after comparison.
- PostgreSQL-backed users, analysis history, notifications, saved result imagery, and PDF report generation in production. SQLite remains the local-development default.
- Original uploads are stored as PostgreSQL binary data and can be downloaded by the authenticated owner. Image data is limited to 15 MB per uploaded file. Database backups and capacity should account for the original files and generated image previews.
- Synthetic demo imagery can be loaded from the frontend. These images are illustrative, not real satellite acquisitions.

Classification uses an unsupervised pixel clustering/color-index heuristic (not a trained model). Flood mode identifies possible water; it cannot confirm flooding. Pixel area is reported only when supported by verified georeferenced rasters. Copernicus credentials and provider retrieval are not configured. Never label arbitrary uploads as live satellite observations.

## Local development

Requirements: Node.js 18+ and Python 3.10+.

Backend (PowerShell):

```powershell
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
Copy-Item .env.example .env
# Edit .env and set a private SECRET_KEY and ADMIN_PASSWORD.
uvicorn app.main:app --reload --port 8002
```

Frontend, in a second terminal from the repository root:

```powershell
npm install
npm run dev
```

Open http://localhost:5173. The frontend uses `VITE_API_BASE_URL` when set; otherwise it uses `http://127.0.0.1:8002/api` for local development. The backend `.env.example` uses a local SQLite URL. Keep real `.env` files private.

## Deploy with Vercel Services

The root `vercel.json` defines two services in one Vercel project: `app` builds the root React/Vite frontend, and `backend` runs FastAPI from `backend`. Ordered public rewrites send `/api/*` to the backend and all other paths to the frontend. Select **Services** as the Vercel project framework and confirm Services access is enabled for the account. Services is a beta capability and may require account access.

The frontend uses the same-domain `/api` path in production. During local Vite development it uses `VITE_API_BASE_URL` when set, otherwise `http://127.0.0.1:8002/api`. Do not set a production API URL to localhost or expose secrets through any `VITE_` variable.

Set these backend variables in Vercel's Environment settings, not in GitHub:

- `APP_ENV=production`
- `DATABASE_URL` — PostgreSQL connection URL; SQLite is rejected in production.
- `SECRET_KEY` — a newly generated private signing key.
- `ADMIN_EMAIL` and `ADMIN_PASSWORD` — private administrator credentials.
- `FRONTEND_ORIGINS` — exact HTTPS Vercel origins, comma-separated and without paths or wildcards.
- `ADMIN_BOOTSTRAP_ENABLED` — leave false while migrating existing users; enable only to create an administrator in an empty database.

The existing root `render.yaml` is an alternate Render deployment description and is not used by this Vercel Services configuration.

## Migrate the existing SQLite database

The active local database is normally `backend/satquery.db` (the local `.env` may override the path). The migration script never writes to the SQLite source. It defaults to a dry run, requires an empty destination, preserves database IDs and timestamps, and refuses to merge or overwrite existing target records. It copies users (including password hashes and roles), analyses with their saved image previews, and notifications. Generated PDFs are not separately stored; the existing endpoint regenerates them from migrated analysis records. Original upload files were not stored by the old app, so they cannot be recovered for pre-deployment analyses; newly uploaded originals are stored durably in PostgreSQL.

Run this from PowerShell on a trusted machine after PostgreSQL is created and verified empty. When using Render Postgres with a Vercel-hosted backend, use its external connection URL and require TLS. The URL includes credentials, so do not paste it into source files or commit it.

```powershell
cd backend
$env:DATABASE_URL = Read-Host "Paste the Render Postgres external URL"
python scripts/migrate_sqlite_to_postgres.py --source satquery.db
python scripts/migrate_sqlite_to_postgres.py --source satquery.db --apply
Remove-Item Env:DATABASE_URL
```

Review the dry-run counts before applying. The migration aborts if any SATQUERY table in PostgreSQL already contains rows. It does not delete or modify the SQLite database. Take a Render Postgres backup before migration and verify the migrated user, analysis, and notification counts afterward. Users should sign in with their existing accounts; existing bearer tokens may need renewal if the production `SECRET_KEY` differs from the local one.

## Persistent original uploads

There is no object-storage service configured in this project. New original uploads are stored as PostgreSQL `BYTEA` data in `original_images`. The authenticated endpoint is `/api/analysis/{analysis_id}/images/{role}/original`; access is restricted to the analysis owner. Land and single-image flood uploads use the `original` role, change detection uses `earlier`/`later`, and flood comparison uses `before`/`after`.

Image binaries increase PostgreSQL storage and backup size. For larger usage, move image bytes to private object storage and retain only keys and metadata in PostgreSQL. Vercel Functions impose request-body limits: 4.5 MB by default, with a 20 MiB public beta option requiring Fluid Compute, Performance (4 GB) memory, and `VERCEL_SUPPORT_LARGE_REQUEST_BODY=1`. This application permits 15 MB per image, and paired-image requests can exceed 20 MiB. Non-streamed responses are limited to 4.5 MB; streamed reports and original-image downloads are exempt. Therefore, the current Vercel backend cannot guarantee all maximum-size upload and analysis responses. Preserving the current upload limits requires direct uploads to persistent object storage before analysis or a backend host without these Function body limits. Do not silently reduce the application's upload-size setting.

## Environment variables

Backend variables belong in the Vercel backend service's Environment settings, not in GitHub:

- `APP_ENV=production`
- `DATABASE_URL` — PostgreSQL connection URL; for Render Postgres accessed from Vercel, use the external URL with TLS required
- `SECRET_KEY` — private production signing key; do not reuse a development key
- `ADMIN_EMAIL`, `ADMIN_PASSWORD` — private production admin settings
- `ADMIN_BOOTSTRAP_ENABLED=false` when migrating existing users; enable only when intentionally bootstrapping an admin into an empty database
- `FRONTEND_ORIGINS` — exact allowed Vercel origins, comma-separated

No production `VITE_API_BASE_URL` is required; the frontend uses same-domain `/api`. Never put database URLs, admin passwords, or signing keys in a `VITE_` variable.

## Tests

From the repository root:

```powershell
npm run build
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
python -m pytest
```

The backend tests use an isolated temporary SQLite database; they do not modify `backend/satquery.db`.
