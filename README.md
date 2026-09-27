# Nexa Mind

Nexa Mind is a private, responsive Flask application for exploring uploaded datasets. It profiles the **actual uploaded file**, offers deterministic pandas-based answers, and generates downloadable PDF reports.

## Requirements and setup

Python 3.11+ is required (the pinned pandas 3.x release does not support Python 3.10).

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
flask --app app run
```

For local development, the application creates its database tables on startup. The SQLite database and private uploaded files are stored under `instance/` by default.

## Production deployment

Use a persistent random signing key; do not use the development key or commit `.env`:

```powershell
py -c "import secrets; print(secrets.token_urlsafe(48))"
```

Set `APP_ENV=production`, `SECRET_KEY` (at least 32 characters), `AUTO_CREATE_SCHEMA=false`, `SESSION_COOKIE_SECURE=true`, and `HSTS_ENABLED=true` in the deployment environment. Production startup rejects a missing/weak secret, insecure session cookies, and the process-local `memory://` rate-limit store. Configure `RATELIMIT_STORAGE_URI` to a shared Redis service, for example `redis://localhost:6379/0`; protect Redis with network controls and credentials in deployed environments. Rate limits are keyed by client IP, so only enable trusted-proxy handling in infrastructure that strips untrusted forwarded headers.

Back up both the database and uploaded files before applying migrations. For a new database:

```powershell
$env:APP_ENV = "production"
$env:AUTO_CREATE_SCHEMA = "false"
flask --app app db upgrade
```

The migrations are also the schema source of truth; production does not run `create_all()` on startup. For a pre-existing database created by an earlier Nexa Mind version, first take a verified backup, then mark the initial schema as applied and upgrade to add subsequent changes:

```powershell
flask --app app db stamp c25d323bb752
flask --app app db upgrade
```

Only stamp a database whose schema matches the original Nexa Mind tables. Review and adapt the migration plan if local schema changes were made outside the app. Do not run `stamp head` on a database that still needs the session-version migration.

Install production-only server and shared rate-limit dependencies with:

```powershell
pip install -r requirements-prod.txt
```

Run the WSGI application with Waitress (HTTPS should terminate at a trusted reverse proxy):

```powershell
waitress-serve --host=127.0.0.1 --port=8000 --threads=8 wsgi:app
```

Keep the server bound to a private interface behind the proxy, restrict upload/log/database file permissions, and configure the proxy for request-size limits and TLS. HSTS is sent only when enabled; enable it only after HTTPS is correctly configured. Set `LOG_FILE` to a private writable path to enable rotating JSON logs, or leave it blank for console logging. Configure log retention and access controls outside the app. `/healthz` returns application/database readiness and can be used by a process supervisor or load balancer.

Report and dashboard QR codes use authenticated, owner-scoped routes. Set `PUBLIC_BASE_URL` to the externally reachable HTTPS origin in production; QR generation reports an unavailable state if it is not configured. Local development falls back to the current request origin. Scanning a code still requires signing in as an account authorized for that report or dataset.

`OPENAI_API_KEY` and `ENABLE_OPENAI_ANALYSIS=true` are optional. The default analysis is local; when enabled, only the question and computed result are sent for wording, not uploaded source rows.

## Features

- Email/password registration and login with Werkzeug password hashing and signed, HTTP-only sessions.
- Dark chat-first workspace with collapsible navigation, dataset projects, persisted pinned chats, grounded dashboard chat, file attachments, optional voice input, and spoken responses when the browser supports them.
- Single-page workspace navigation for Chat, Dashboard, Library, Data Explorer, Reports, and Settings; data previews and analysis stay inside the workspace. The BI dashboard supports dataset selection, detected-date filtering, fullscreen, and CSV export of its calculated KPIs and chart data.
- Multi-file data workspace for uploading, previewing, renaming, re-analyzing, downloading, and deleting CSV, Excel, and JSON datasets. Dataset profiles are computed from uploaded files and include data quality, column classifications, domains, and primary-key candidates.
- Relationship discovery and confirmation for normalized column keys; cross-file questions only join datasets through confirmed relationships. Saved dashboards are private and support rename, duplicate, delete, regeneration, and CSV export.
- Searchable, server-paginated dataset rows, actual-data dashboard KPIs and trends, inline dataset profiling, saved report records, and persisted account preferences.
- Per-user dataset access control and CSRF protection on browser forms and analysis requests.
- Upload validation and parsing for CSV, XLSX, XLS, JSON, PDF, DOCX, and TXT (25 MB default limit).
- Data profiles with row/column counts, missing values, duplicates, numeric summaries, and category frequencies.
- Paginated source-data preview, Chart.js column completeness chart, deterministic pandas question answering, and PDF report downloads.
- AI Analyst workspace with dataset-scoped conversation persistence, verified ranking/trend/anomaly/correlation calculations, chart explanations, and follow-up prompts.
- Column intelligence, calculated data-quality score, descriptive statistics, IQR outlier checks, numeric correlations, searchable/sortable server-paginated rows, and filtered CSV export.
- Included sample data: `samples/sales.csv` and `samples/feedback.csv`.

PDF, DOCX, and TXT sources are represented as rows of extracted non-empty text lines. JSON accepts records, objects, and column-oriented arrays. Excel analysis reads the first worksheet.

## API

All API routes require an authenticated session. The browser interface obtains the session CSRF token from the page metadata.

- `GET /api/datasets/<id>/profile` — computed profile.
- `GET /api/datasets/<id>/data?offset=0&limit=50` — source rows (maximum 200 per request).
- `POST /api/datasets/<id>/ask` — JSON body: `{"question":"What is the average revenue?"}`.
- `GET /api/datasets/<id>/report.pdf` — downloadable ReportLab PDF.
- `GET /api/explorer/<id>/overview|columns|quality|statistics` — computed dataset details and correlation pairs.
- `GET /api/explorer/<id>/dashboard?start_date=YYYY-MM-DD&end_date=YYYY-MM-DD` — calculated KPI, grouping, and monthly trend data from the uploaded dataset, optionally filtered using its detected date column.
- `GET /api/explorer/<id>/conversation` — persisted messages for this user's dataset analysis.
- `POST /api/explorer/<id>/chat` — JSON body: `{"question":"Show the top 5 regions by sales"}`; returns a calculated answer, chart specification, context, and follow-ups.
- `DELETE /api/explorer/<id>/conversation` — clear this user's conversation for the dataset.
- `POST /api/explorer/<id>/pin` — toggle the authenticated user's dataset conversation in Pinned Chats.
- `GET /api/explorer/<id>/export.csv` — download source rows with the current table filters and selected columns.
- `GET|POST /api/workspace/chats` and `GET|PATCH|DELETE /api/workspace/chats/<id>` — private workspace chat history, pinning, and lifecycle.
- `POST /api/workspace/chats/<id>/messages` — persist a question and its dataset-grounded analysis response.
- `GET /api/workspace/datasets` — list the signed-in user's uploaded files.
- `PATCH|DELETE /api/workspace/datasets/<id>` — rename or delete an owned dataset; deleting removes its dependent analyses and stored source.
- `POST /api/workspace/datasets/<id>/reanalyze` and `GET /api/workspace/datasets/<id>/download` — recompute a dataset profile or download its source.
- `POST /upload` — accepts repeated `file` fields for multi-file CSV, XLSX, XLS, and JSON uploads. JSON clients receive the created records in `datasets`.
- `GET /api/workspace/relationships`, `POST /api/workspace/relationships/scan`, and `POST|PATCH|DELETE /api/workspace/relationships[/<id>]` — inspect suggestions and manage confirmed dataset relationships.
- `GET /api/workspace/dashboard?dataset_ids=1,2&type=executive` — calculate summary KPIs, numeric summaries, category frequencies, and quality signals from selected owned files.
- `POST /api/workspace/analyze` — ask a cross-file question with `{"question":"...","dataset_ids":[1,2],"chat_id":null}`; joins require confirmed relationships and exchanges are persisted when `chat_id` is supplied.
- `GET|POST /api/workspace/dashboards` and `GET|PATCH|DELETE /api/workspace/dashboards/<id>` — manage private saved dashboards. Send `{"duplicate_from":id}` to `POST` to create an owned copy. Public/link sharing is not enabled.
- `GET|PUT /api/user/settings` — read or save private workspace preferences.
- `POST /api/user/password` — change the signed-in user's password and revoke other sessions.
- `POST /api/user/revoke-sessions` — revoke existing sessions for the account.
- `GET /api/user/export` — export the signed-in user's account data as JSON.
- `DELETE /api/user/datasets` — delete all of the signed-in user's datasets.
- `GET|POST /api/reports` and `DELETE /api/reports/<id>` — manage saved report entries; the PDF is generated from its owned dataset on download.

The data-quality score is a reproducible 0–100 indicator: 100 minus a weighted missing-cell rate (70%) and exact-duplicate-row rate (30%), rounded to an integer. It is a screening score, not a compliance or correctness certification. Top-N questions with multiple available measures ask the user to select a measure rather than silently choosing one. The analysis engine uses constrained pandas operations; it never executes user-provided or model-generated Python.

The `answer` field is deterministic and calculated locally with pandas. The default analysis does not transmit uploaded data to an external service. If both `OPENAI_API_KEY` and `ENABLE_OPENAI_ANALYSIS=true` are configured, the user's question and already-computed pandas answer are sent to OpenAI for optional wording in a separate `explanation` field; source rows are never sent, and the AI layer cannot replace the calculation. Disable the setting to keep all processing local.

Source-data CSV exports neutralize spreadsheet formula prefixes in textual cells and headers. Numeric values remain numeric. Email verification, account deletion, active-session listing, scheduled analytics, and predictive forecasting are not currently implemented; settings do not claim those capabilities.

The `/api/workspace/...` endpoints are the canonical paths for workspace dataset, relationship, dashboard, and analysis operations. Older `/api/datasets/...` paths remain direct compatibility aliases to the same view functions where an equivalent workspace route exists; they are not separate implementations. Aggregate relationship and dashboard responses report unreadable or missing source files in `unavailable_datasets` and continue processing available datasets.

## Tests

```powershell
pytest -q
```
"# app" 
