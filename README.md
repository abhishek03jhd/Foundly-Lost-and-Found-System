# Intelligent Lost and Found

A web MVP for reporting lost/found belongings and surfacing likely matches using image, description, category, location, and date signals.

## Stack

- React + Vite frontend
- FastAPI backend
- SQLite for report metadata
- Pillow difference hash for lightweight image similarity

The matching engine is intentionally lightweight and runs locally. Text matching normalizes common word forms and item synonyms rather than calling a hosted language model, so reports and images stay on your machine.

## Run locally

### Backend

Requires Python 3.10+.

```powershell
cd backend
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --reload
```

The API is available at `http://localhost:8000`; interactive API documentation is at `http://localhost:8000/docs`.

### Frontend

Requires Node.js 18+.

```powershell
cd frontend
npm install
npm run dev
```

Open the URL printed by Vite. Vite proxies `/api` and `/uploads` to the backend at `http://127.0.0.1:8000`.
Do not open `frontend/index.html` through VS Code Live Server: this project uses React JSX, which needs Vite to compile it. The page shows a startup message with a link if it is opened that way.

## How matching works

Reports of opposite types (lost vs. found) are scored on the signals that are available for both reports:

- Image: perceptual difference-hash similarity
- Description: normalized word-token overlap, including common item synonyms (for example, "rucksack" and "backpack")
- Context: local weighted token and adjacent-word similarity (an explainable heuristic, not a hosted language model)
- Location: distance between coordinates when supplied, otherwise matching location text
- Date: an exponential recency score
- Category: exact category match

The UI shows the total and per-signal scores plus a plain-language explanation of shared details, category, date, and proximity clues. Location can be entered as a place name or as optional latitude/longitude coordinates; coordinates provide distance-aware matching and appear on the map. Reports without coordinates remain available in the board list. Matching is explainable local text/image heuristics, not a trained semantic AI model.

## Accounts, claims, and notifications

Visitors may continue to post reports and manage them with their private report codes. Email/password accounts (minimum 12-character passwords, salted PBKDF2 hashes, HTTP-only server-side sessions) enable private item claims, claim responses, and notification history. Admin APIs require an administrator role.

To set up the first administrator locally, set `ADMIN_BOOTSTRAP_KEY` in the backend process environment, register the chosen email in Foundly, and then run this PowerShell command with that account email:

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:8000/api/auth/bootstrap-admin `
  -Method Post `
  -Headers @{ "X-Admin-Bootstrap-Key" = $env:ADMIN_BOOTSTRAP_KEY } `
  -ContentType "application/json" `
  -Body '{"email":"admin@example.com"}'
```

Only the first administrator can be bootstrapped. Store the setup key outside source control. Configure SMTP with `SMTP_HOST`, `SMTP_FROM_EMAIL`, and, if required, `SMTP_USERNAME`, `SMTP_PASSWORD`, and `SMTP_PORT` to send email alerts. Without SMTP, in-app notifications still work; browser alerts can be enabled from the notification menu.

The map uses OpenStreetMap tiles and attribution. Public coordinates are rounded to approximately 100-meter precision for privacy; matching still uses the stored coordinates. Public tile servers have usage policies and are appropriate for demos and modest use; configure an appropriate tile provider for a high-traffic deployment.

## Advanced community-board search

On the public board, use **Filters** to narrow reports by category and event-date range, and sort by posting date or the date the item was lost/found. The board updates the result count as filters change and offers a one-click reset. Press **Ctrl+K** (or **⌘K** on macOS) to focus the report search.

## Admin dashboard

Administrator accounts can select **Admin** in the top navigation to review report counts, recent activity, category distribution, and the report table. The dashboard can filter reports, export the current filtered list to CSV, mark reports open/resolved/hidden, and permanently delete a report and its uploaded image. Only open reports appear on the public board and in match suggestions. Existing SQLite databases are upgraded automatically with an `open` status for existing reports.

Admin listings intentionally omit contact information. Before public launch, deploy behind HTTPS, set `COOKIE_SECURE=true`, use a strong unique bootstrap key, configure email verification/password recovery and operational rate limiting, and define a protected backup-retention policy. The included account flow is a project baseline, not a substitute for a security review or a managed identity provider.

## Backups

From `backend/`, create a consistent SQLite-and-image archive:

```powershell
..\.venv\Scripts\python.exe -m app.backup create
```

Restore an archive only after stopping the API server:

```powershell
..\.venv\Scripts\python.exe -m app.backup restore data\backups\foundly-YYYYMMDDTHHMMSSZ.zip
```

Backups contain private account, contact, and message data; store them encrypted and access-controlled. Keep copies off the machine and periodically verify a restore.

## Container deployment

Copy `.env.example` to `.env`, replace `ADMIN_BOOTSTRAP_KEY` with a long random value, and run:

```powershell
docker compose up --build -d
```

The web app is available at `http://localhost:8080` by default. For internet deployment, put a TLS reverse proxy in front, set `COOKIE_SECURE=true`, and persist/encrypt the named database and upload volumes with regular backups.

## Tests

Run the backend's standard-library tests from `backend/`:

```powershell
..\.venv\Scripts\python.exe -m unittest discover -s tests
```

## Project structure

```text
backend/
  app/main.py
  requirements.txt
frontend/
  src/App.jsx
  src/main.jsx
  src/index.css
  package.json
```

SQLite data and uploaded images are stored under `backend/data/` and `backend/uploads/` at runtime. These are local runtime data and are not intended to be committed.
