# DevFlow AI

AI-Powered Developer Productivity & Engineering Intelligence Platform.

Combines GitHub integration, project/issue management, and LLM-powered code review/analytics into one workspace.

## Status

**Working end-to-end (real code, runs with `docker compose up`):**
- Full DB schema as SQLAlchemy models (all 20 tables) + Alembic wired up
- JWT auth: register, login, refresh-token rotation, logout, bcrypt hashing
- RBAC dependency (`require_role`) enforced server-side, not just hidden UI
- Projects + Issues CRUD API with membership-based authorization
- AI abstraction layer (`AIProvider` protocol) with a working Anthropic implementation, used by real `/api/ai/code-review`, `/api/ai/issue-generator`, `/api/ai/assistant` endpoints, with token/cost tracking in `ai_usage`
- **GitHub OAuth**: full code-exchange flow, encrypted token storage (Fernet), repo listing, repo import with initial PR sync (`app/services/github_service.py`, `app/integrations/github_client.py`)
- **GitHub webhooks**: HMAC-verified endpoint stores every event, then dispatches to a real Celery handler that updates `pull_requests`/`issues` and fans out `notifications` to project members (`app/workers/github_tasks.py`) — covers `pull_request`, `issues`, and `push` events
- Notifications: table + list/mark-read/**mark-all-read** endpoints (fixed a serialization bug — the endpoint was returning raw ORM rows), **frontend bell with 15s polling, unread badge, click-to-read, click-outside-to-close**
- **Kanban board**: full Task CRUD API with per-column position ordering, frontend board with native HTML5 drag-and-drop across all 5 columns, optimistic updates (add/move/delete) that reconcile with the server on failure, inline "add task" per column
- React + TS frontend: routing, protected routes, login/register wired to the real API, dashboard shell + sidebar + topbar, GitHub connect button → OAuth callback → repository list/import UI, Tailwind dark theme
- Docker Compose (postgres, redis, backend, worker, frontend) + GitHub Actions CI (lint, test, build both images)
- Demo seed script (`scripts/seed.py`) creating admin/lead/developer users + sample project/issues
- Unit tests for webhook signature verification

**Deliberately stubbed — architecture is in place, logic is not:**
- PR dashboard UI, analytics charts UI, Projects list/create UI (Kanban board currently takes a project id straight from the URL rather than a project picker)
- WebSocket notification delivery (the bell works via polling; a real-time push would replace the 15s interval)
- Analytics aggregation job (`compute_analytics_snapshot` — still `NotImplementedError`)
- Rate limiting, Redis caching of AI responses, full audit-log coverage
- Commit-level sync (client method exists in `github_client.py`, not yet called by the import flow)

Honest reason: these each need nontrivial UI state or a scheduling layer (celery beat) that's a real multi-day build on top of what's here, not something worth faking with placeholder logic that would need to be thrown away.

See `docs/ARCHITECTURE.md`, `docs/DATABASE_SCHEMA.md`, `docs/API_ROUTES.md` for the full design all of the above follows.

## Quickstart
```bash
cp .env.example .env   # fill in JWT secrets + AI_API_KEY at minimum
docker compose up --build
# in another shell, seed demo data:
docker compose exec backend python -m scripts.seed
```
Frontend: http://localhost:5173 · API docs: http://localhost:8000/docs

## Stack
- **Frontend**: React, TypeScript, Vite, Tailwind, shadcn/ui, TanStack Query, React Hook Form + Zod, Recharts
- **Backend**: FastAPI, SQLAlchemy, Alembic, PostgreSQL, Redis, Celery, WebSockets
- **AI**: provider-agnostic layer (default: Anthropic Claude)
- **Infra**: Docker, Docker Compose, GitHub Actions CI/CD

## Repo layout
```
devflow-ai/
  frontend/src/{components,features/*,hooks,lib,services,types}
  backend/app/{api,core,models,schemas,services,repositories,integrations,ai,workers,utils,tests}
  docs/
  .env.example
```

## Roles
`admin`, `team_lead`, `developer` — enforced in both frontend routing and backend dependencies/services (never frontend-only).

## Next phase
Phase 2 — Backend foundation: FastAPI app skeleton, SQLAlchemy models from schema, Alembic migrations, JWT auth + refresh rotation, RBAC dependency, error-handling middleware.
