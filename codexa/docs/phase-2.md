# Phase 2: Project setup and architecture

**Goal:** create the project foundation.
**Deliverable:** a working skeleton with extension, backend, frontend and database, each runnable on its own.
**Status:** planned. Nothing is implemented yet.

**Contents:** 1. Starting environment · 2. Architecture · 3. Technology choices · 4. Repo structure · 5. Configuration · 6. Database · 7. Build steps · 8. Done criteria · 9. Decisions · 10. Checklist

---

## 1. Starting environment

Checked on the developer machine before planning.

| Tool | State |
|---|---|
| Node | v26.12.0 (non-LTS). Node 22 LTS recommended |
| npm | 11.20.0 (pnpm not installed; we use npm) |
| Python | 3.14.8. A Python 3.12 environment will be created with uv |
| PostgreSQL | 16.15 and pgvector installed via Homebrew |
| Ollama | 0.40.6 with `qwen2.5:3b` and `nomic-embed-text` |
| Docker | Not installed. Not needed for local development |
| `gh` CLI | Not installed. Plain git is used for pushing |
| Git | 2.50.1; remote `origin` = `Nandini1071/Codexa` (no commits yet) |
| VS Code extras | SQLTools + SQLTools PostgreSQL/Cockroach Driver (for browsing the DB) |

---

## 2. Architecture

```
 VS Code extension  --REST-->  FastAPI backend  --SQL-->  PostgreSQL + pgvector
 (TypeScript)                  (Python)                   
                                  |  \--HTTP--> Ollama (local LLM + embeddings)
 Web app (React)    --REST-->     |
```

- The extension and the web app talk to the backend only. Neither touches the database or Ollama directly.
- The model is called only from the backend, behind a swappable `LLMProvider` interface.
- All three share the same API contract and finding schema ([phase-1.md](phase-1.md) appendix).

---

## 3. Technology choices

**Repo and tooling**
- Monorepo, `main` plus short-lived feature branches, conventional commit messages.
- GitHub Actions: one workflow per app running lint and tests.

**VS Code extension (`vscode-extension/`)**
- TypeScript (strict), VS Code Extension API, esbuild
- ESLint (flat config), Prettier
- Vitest for unit tests, `@vscode/test-electron` for one minimal integration test

**Backend (`backend/`)**
- Python 3.12 managed with uv
- FastAPI, Pydantic v2, pydantic-settings
- SQLAlchemy 2.0, Alembic, psycopg 3
- Ruff (lint and format), mypy, pytest + httpx
- Ollama client behind an `LLMProvider` interface

**Web (`web/`)**
- React, TypeScript, Vite
- Tailwind CSS, shadcn/ui
- React Router, TanStack Query
- ESLint, Prettier, Vitest + React Testing Library

**Database**
- PostgreSQL 16 with the pgvector extension
- No Redis, graph database or object storage yet

---

## 4. Repo structure

```
Codexa/
  .gitignore  .editorconfig  README.md
  codexa/
    docs/                 phase-1.md, phase-2.md, ...
    vscode-extension/
      src/                extension.ts, views/, commands/
      package.json  esbuild config  eslint/prettier config  tests/
    backend/
      app/
        main.py           FastAPI app, routers
        core/             config, logging
        db/               session, models, migrations (Alembic)
        api/v1/           health, (later) investigations, findings
        llm/              LLMProvider, OllamaProvider
      tests/
      pyproject.toml  .env.example
    web/
      src/                pages/, components/, lib/api.ts
      package.json  vite/tailwind config  tests/  .env.example
  .github/workflows/      backend.yml  web.yml  extension.yml
```

---

## 5. Configuration

Each app has its own `.env.example`. Real `.env` files are git-ignored.

| Variable | App | Purpose |
|---|---|---|
| `DATABASE_URL` | backend | Postgres connection string |
| `OLLAMA_BASE_URL` | backend | Default `http://localhost:11434` |
| `OLLAMA_CHAT_MODEL` | backend | Default `qwen2.5:3b` |
| `OLLAMA_EMBED_MODEL` | backend | Default `nomic-embed-text` |
| `API_DEV_KEY` | backend | Dev API key for the extension (MVP auth) |
| `CORS_ORIGINS` | backend | Allowed web origins |
| `VITE_API_URL` | web | Backend base URL |

Extension settings (`codexa.backendUrl` and others) are defined in [phase-1.md](phase-1.md) section 3. No secrets go in the extension or the web bundle.

---

## 6. Database

- Local database `codexa` with a dedicated `codexa` user.
- Extension `vector` enabled.
- Alembic migration #1 creates the Phase 1 tables: `users`, `repositories`, `investigations`, `findings`, `verification_runs`, `feedback`, `evaluation_cases`, `audit_events`.
- Paths stored repo-relative only; no raw source files (privacy rules in [phase-1.md](phase-1.md) section 7).
- Browsed with SQLTools inside VS Code.

---

## 7. Build steps

Each step is committed separately and verified before the next begins.

1. **Repo foundation:** root `.gitignore`, `.editorconfig`, README; first commit and push.
2. **Database:** start the Postgres service; create the `codexa` database and user; enable `vector`; connect from SQLTools.
3. **Backend:** uv + Python 3.12 project; FastAPI app; config; DB session; `GET /health` with DB and Ollama checks; migration #1; ruff, mypy, pytest; `.env.example`.
4. **Web:** Vite + React + TS; Tailwind and shadcn/ui; router; API client; layout shell; Investigations placeholder; ESLint, Prettier, Vitest.
5. **Extension:** scaffold with esbuild; sidebar with Changes, Impact and Findings views; hello command; settings; ESLint, Prettier, Vitest.
6. **CI and docs:** three GitHub Actions workflows; add run instructions and any architecture changes to this file.

---

## 8. Done criteria

- Backend runs and `/health` reports DB and Ollama status.
- Alembic migration applies cleanly to a fresh database.
- Web app loads and shows backend health.
- Pressing F5 opens the extension host with the Codexa sidebar.
- Lint, format and tests pass in all three apps.
- All three apps start independently from their own folder.

---

## 9. Decisions

| # | Decision | Choice | Status |
|---|---|---|---|
| P2-1 | Database runtime | Homebrew Postgres 16 + pgvector (no Docker) | Decided |
| P2-2 | Python version | 3.12 via uv | Proposed |
| P2-3 | Node version | 22 LTS | Proposed |
| P2-4 | JS package manager | npm | Proposed |
| P2-5 | GitHub repo visibility | Private | Proposed |
| P2-6 | Docker | Deferred to the deployment phase | Decided |

---

## 10. Checklist

- [x] Environment checked
- [x] Postgres 16 + pgvector installed
- [x] DB GUI extensions installed
- [ ] Step 1: repo foundation + first push
- [ ] Step 2: database created
- [ ] Step 3: backend skeleton
- [ ] Step 4: web skeleton
- [ ] Step 5: extension skeleton
- [ ] Step 6: CI + docs
- [ ] Decisions P2-2 to P2-5 confirmed
