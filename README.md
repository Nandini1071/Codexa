# Codexa

AI-powered release-risk investigation for code changes. Codexa shows what a change might break, why, and the evidence behind it, then helps you verify it.

It has three parts:

- **VS Code extension** (`codexa/vscode-extension/`): detects changes, maps their impact, shows findings, and runs approved verification.
- **Backend** (`codexa/backend/`): FastAPI service with PostgreSQL + pgvector and a local Ollama model.
- **Web app** (`codexa/web/`): dashboard for investigation history and results.

## Documentation

Planning and progress are tracked phase by phase in [codexa/docs](codexa/docs/README.md).

## Status

Early development. Phase 2 (project setup) is in progress.
