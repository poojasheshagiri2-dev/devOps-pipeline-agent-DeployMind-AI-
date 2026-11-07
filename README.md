# DeployMind AI

DeployMind AI is a deployment and incident intelligence workspace. FastAPI serves the browser application and API; PostgreSQL stores structured operational records; Hindsight is the separate long-term incident-memory service; Groq and LangGraph provide evidence-cited analysis; GitHub Actions can create deployments through signed webhooks.

The previous flight-search demo has been replaced. The UI contains no mock production records or fabricated analysis. Empty states reflect an empty database, and GitHub explicitly reports demo mode until configured.

## What is implemented

- Email/password registration and login with hashed passwords and signed bearer tokens.
- Per-user deployment and incident CRUD, durable deployment logs, resolution feedback, agent executions, repository metadata, GitHub integration status.
- Dashboard counts aggregated from the authenticated user's database rows.
- SQLAlchemy 2 models, PostgreSQL connection pooling, health endpoint, Alembic initial schema migration.
- Official `hindsight-client` calls for Retain, Recall, and Reflect. An unconfigured or unavailable Hindsight service is reported as such; there is no local or PostgreSQL-vector memory fallback.
- Groq `ChatGroq` calls in a LangGraph workflow. Findings must cite evidence identifiers from the actual deployment/log or retrieved Hindsight results. Unsupported citations fail the analysis instead of being presented as facts. No confidence score is invented.
- GitHub account verification, repository listing, workflow and run listing, workflow log download, and HMAC-SHA256 validation for workflow webhooks. Completed signed workflow runs create database deployment/log records and attempt analysis.
- A browser UI for overview, deployments, incidents/resolutions, AI risk and root-cause analysis, memory exploration, agent activity, GitHub and service settings.

## Prerequisites

- Python 3.11+ (3.12 recommended), Node.js 20+ (frontend unit tests), and Docker Desktop/Engine.
- Groq API key for Groq analysis and for the optional local Hindsight service's LLM provider.
- Hindsight Cloud URL/key, or run the optional Hindsight Compose service.
- GitHub personal access token and webhook secret for live GitHub features.

## Quick start with Docker Compose

1. Copy `.env.example` to `.env`. Set a long random `JWT_SECRET_KEY`; add `GROQ_API_KEY` for AI. When using the optional Hindsight Compose service, set `HINDSIGHT_BASE_URL=http://hindsight:8888` (the Compose service name). Compose's local PostgreSQL password is for development only.
2. Start PostgreSQL and API:

   ```powershell
   docker compose up --build -d db api
   ```

   The API container runs `alembic upgrade head` before starting FastAPI at `http://localhost:8000`. PostgreSQL health is available at `http://localhost:8000/health`; API docs are at `http://localhost:8000/docs`.
3. Optional, start Hindsight locally (the Hindsight service needs a working Groq key/model configuration):

   ```powershell
   docker compose --profile ai up -d hindsight
   ```

   The Hindsight API/UI are exposed at ports `8888`/`9999`. Set `HINDSIGHT_BASE_URL=http://hindsight:8888` in `.env`, then restart the API with `docker compose up -d api`.
4. Open `http://localhost:8000` and create an account. GitHub is explicitly in demo mode until a server-side `GITHUB_TOKEN` is configured and connected.

## Local Python setup

Start the PostgreSQL service with `docker compose up -d db`, then from the project directory:

```powershell
Copy-Item .env.example .env
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python -m alembic upgrade head
python -m uvicorn backend.app.main:app --reload --host 127.0.0.1 --port 8000
```

For a locally hosted Hindsight service in this setup, use `HINDSIGHT_BASE_URL=http://localhost:8888` in `.env` before starting the API.

In another terminal, `npm test` runs the dependency-free frontend API-client tests. `npm run test:backend` runs unit/API tests.

The frontend is static with no framework compilation step. `npm run build` copies the production-ready static assets into `dist/`; the FastAPI app serves the checked-in `public/` assets directly.

## Configuration and integrations

### PostgreSQL

`DATABASE_URL` must be a PostgreSQL SQLAlchemy URL, for example `postgresql+psycopg://user:password@host:5432/deploymind`. The health response reports the selected database backend and executes `SELECT 1`; SQLite is used only by isolated unit tests. Do not treat successful SQLite tests as PostgreSQL connectivity verification.

Apply migrations with `python -m alembic upgrade head`; inspect the current migration with `python -m alembic current`. Schema is not silently created on application startup.

### Hindsight

Install/use the official Python SDK (`hindsight-client`). Configure `HINDSIGHT_BASE_URL`, optional `HINDSIGHT_API_KEY`, and `HINDSIGHT_BANK_ID`. For Hindsight Cloud, use the Cloud API base URL and key from the Hindsight dashboard. For a self-hosted server, use its API URL; API key can be omitted if that server does not require one. Retain uses synchronous processing before reporting stored. Recall returns only actual SDK results. Reflect is only used when there is retrieved context in the automated analysis flow.

Incident creation and resolution feedback are committed to PostgreSQL first. The response separately reports Hindsight retention as `stored`, `not_configured`, or `failed`, so database persistence is not confused with memory availability. Manual Memory Explorer calls fail explicitly if Hindsight is unavailable.

Official references: [Hindsight documentation](https://hindsight.vectorize.io/), [Python SDK](https://hindsight.vectorize.io/sdks/python), [API reference](https://hindsight.vectorize.io/api-reference), [GitHub](https://github.com/vectorize-io/hindsight).

### Groq and LangGraph

Set `GROQ_API_KEY` and optionally `GROQ_MODEL` (defaults to `llama-3.3-70b-versatile`). The agent uses a LangGraph state graph with an actual Hindsight recall tool node, conditional reflection on retrieved memories, and Groq structured output. Missing keys, timeouts, rate limits, invalid structured output, or unsupported evidence citations produce a visible failed execution; they are not converted into successful risk reports.

On Windows, run `.\scripts\connect-groq.ps1` from PowerShell to enter the key with hidden input. The script verifies the key and configured model against Groq before saving it to the ignored `.env` file, then restarts both API and local Hindsight. Create or retrieve the key from [Groq Console API Keys](https://console.groq.com/keys); the application cannot create a key for your Groq account. Never paste the key into chat or commit `.env`.

### GitHub

For public repositories, set `GITHUB_USERNAME` to the public profile login. The UI can connect that public profile without a token and lists public repository data only; private repos and authenticated GitHub Actions access still need a fine-grained token. Set `GITHUB_TOKEN` server-side with the repository and Actions permissions required by your organization; the app verifies the token with GitHub's `/user` endpoint. On Windows, `.\scripts\connect-github.ps1` prompts for the token with hidden input, verifies it against GitHub, saves it to the private `.env`, and restarts the API without printing the token. Then click the connect button on the GitHub page. Configure `GITHUB_WEBHOOK_SECRET` to the same secret used in the repository webhook. Set webhook content type to `application/json`, event to `workflow_run`, and endpoint to `https://<public-api-host>/api/github/webhooks`. Do not expose a local webhook endpoint publicly without a trusted tunnel/reverse proxy and HTTPS. Invalid signatures receive HTTP 401.

Repository/workflow/run/log APIs use GitHub's live REST API. Without credentials, the UI labels GitHub `demo` mode; it does not display sample repositories or workflow results. Only completed, validly signed workflow events for an account linked to a DeployMind user can create deployment records.

## API overview

All routes except health, register/login, frontend assets, and the GitHub webhook require a bearer token. Register/login via `/api/auth/register` and `/api/auth/login`.

| Area | Routes |
|---|---|
| Health | `GET /health`, `GET /api/health` |
| Dashboard | `GET /api/dashboard` |
| Repositories | `GET/POST /api/repositories`, `GET/PATCH/DELETE /api/repositories/{id}` |
| Deployments | `GET/POST /api/deployments`, `GET/PATCH/DELETE /api/deployments/{id}`, `GET/POST /api/deployments/{id}/logs` |
| Incidents | `GET/POST /api/incidents`, `PATCH/DELETE /api/incidents/{id}`, `POST /api/incidents/{id}/resolve` |
| AI | `POST /api/analysis/risk`, `POST /api/analysis/root-cause`, `GET /api/agents` |
| Hindsight | `GET /api/memory/status`, `POST /api/memory/retain`, `/recall`, `/reflect` |
| GitHub | `GET /api/github/status`, `POST /api/github/connect`, repositories/workflows/runs/logs routes, `POST /api/github/webhooks` |
| Settings | `GET /api/settings` |

Interactive OpenAPI documentation: `/docs`.

## Tests and verification

```powershell
npm test
python -m pytest -q
python -m pytest -q tests/integration
python -m alembic upgrade head
python -m uvicorn backend.app.main:app --host 127.0.0.1 --port 8000
Invoke-WebRequest http://127.0.0.1:8000/health
```

Unit tests use isolated SQLite and mocked external clients. They do not prove a PostgreSQL, Groq, Hindsight, or GitHub service is reachable. `tests/integration/test_hindsight_live.py` uses the real configured official Hindsight SDK and writes a uniquely named test incident to the configured bank; it skips unless `HINDSIGHT_BASE_URL` is set. `tests/integration/test_postgres_live.py` requires a dedicated database configured via `DEPLOYMIND_TEST_POSTGRES_URL`, applies the Alembic migration, and verifies the schema. Run integration tests explicitly with `python -m pytest -q tests/integration`. Do not point them at production or shared databases unless test data and schema changes are acceptable.

The repository currently has no checked-in deployment credentials. Live service verification therefore requires the credentials and services listed above. PostgreSQL's health state is reported by the running API and must say `connected` with backend `postgresql` before describing PostgreSQL as verified.

## Security notes

- Change `JWT_SECRET_KEY` and PostgreSQL credentials before any non-local deployment.
- Keep `.env`, GitHub tokens, Hindsight keys, and webhook secrets out of source control and browser code.
- Use HTTPS and a production-grade reverse proxy for hosted deployments; configure access controls, backups, rate limits, and log retention before production use.
- This conversion provides single-user email/password accounts and signed bearer tokens; it does not yet include password reset, email verification, OAuth, or organization/role management.
