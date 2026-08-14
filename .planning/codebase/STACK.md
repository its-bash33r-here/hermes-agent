# Technology Stack

**Analysis Date:** 2026-08-14

## Languages

**Primary:**
- Python 3.11–3.13 - Agent runtime, CLI, gateway, provider adapters, tools, plugins, cron, and dashboard API (`agent/`, `gateway/`, `tools/`, `plugins/`, `cron/`, `hermes_cli/`).

**Secondary:**
- TypeScript/JavaScript - Web dashboard, TUI, desktop, shared packages, tests, documentation site, and WhatsApp/iMessage sidecars (`web/`, `ui-tui/`, `apps/`, `scripts/whatsapp-bridge/`, `plugins/platforms/photon/sidecar/`).
- Shell - Installation, container entrypoints, lifecycle hooks, and operational scripts (`scripts/install.sh`, `docker/`).

## Runtime

**Environment:**
- Python `>=3.11,<3.14`, managed primarily with `uv` (`pyproject.toml`, `uv.lock`).
- Node.js `>=22.22.0` for local development; the container builds with Node 26 (`package.json`, `Dockerfile`).

**Package Manager:**
- `uv` for Python dependencies; lockfile: `uv.lock`.
- `npm` for the JavaScript workspaces; lockfiles: `package-lock.json` plus workspace/sidecar lockfiles.

## Frameworks

**Core:**
- OpenAI Python SDK `2.24.0` - OpenAI-compatible model calls and Responses API (`pyproject.toml`, `agent/`).
- FastAPI and Uvicorn - Local API server and dashboard HTTP services (`pyproject.toml`, `api_server/`, `dashboard/`).
- React/Vite/TypeScript - Web dashboard and desktop UI (`web/package.json`, `apps/desktop/package.json`).
- Ink/React - Terminal UI (`ui-tui/package.json`).
- Docusaurus - Documentation website (`website/package.json`, `website/docusaurus.config.ts`).

**Testing:**
- Pytest and pytest-asyncio (`pyproject.toml`, `tests/`).
- Vitest (`ui-tui/vitest.config.ts`, `web/vitest.config.ts`, `tests-js/vitest.config.ts`).
- Playwright (`apps/desktop/playwright.config.ts`, browser tooling and Docker build).

**Build/Dev:**
- Ruff, ty, ESLint, Prettier, and TypeScript (`pyproject.toml`, `eslint.config.shared.mjs`, `.prettierrc`, workspace configs).
- Docker multi-stage build with Debian 13, s6-overlay, fixed SQLite, and Playwright Chromium (`Dockerfile`).

## Key Dependencies

**Critical:**
- `httpx`, `requests`, `tenacity`, `pydantic`, `PyYAML`, `ruamel.yaml`, `Jinja2`, and `rich` - HTTP, retries, typed provider payloads, configuration, templating, and CLI output (`pyproject.toml`).
- `prompt_toolkit`, `websockets`, `ptyprocess`/`pywinpty`, and `psutil` - Interactive CLI, browser/CDP communication, terminal sessions, and process management (`pyproject.toml`, `cli.py`, `agent/`).
- `python-dotenv`, `PyJWT[crypto]`, `cryptography`, and `certifi` - Environment loading, JWT/OAuth-style identity, cryptography, and TLS trust (`pyproject.toml`, `agent/`, `tools/`).

**Infrastructure:**
- Optional provider packages include `anthropic`, `exa-py`, `firecrawl-py`, `parallel-web`, `fal-client`, `edge-tts`, `elevenlabs`, `faster-whisper`, `mautrix`, `asyncpg`, `aiosqlite`, messaging SDKs, and cloud memory clients (`pyproject.toml`).
- `nemo-relay` provides the managed/in-process relay and shared metrics boundary (`pyproject.toml`, `agent/relay/`).

## Configuration

**Environment:**
- Configuration is loaded from YAML examples and runtime environment variables (`config.yaml.example`, `cli-config.yaml.example`, `agent/config.py`, `gateway/`).
- `.envrc` and environment files may exist; their contents are intentionally not part of this map. Secret values belong in the user configuration/data directory, typically `~/.hermes` or `/opt/data` in Docker (`docker-compose.yml`, `Dockerfile`).

**Build:**
- Python metadata and optional extras: `pyproject.toml`.
- Python resolution: `uv.lock`.
- JavaScript workspaces: `package.json`, `package-lock.json`, and package manifests under `apps/`, `web/`, `ui-tui/`, and `tests-js/`.
- Formatting/linting/typechecking: `.prettierrc`, `eslint.config.shared.mjs`, workspace ESLint configs, and `tsconfig.json` files.
- Container/runtime: `Dockerfile`, `docker-compose.yml`, `docker-compose.windows.yml`, and `docker/`.

## Platform Requirements

**Development:**
- Python 3.11–3.13, `uv`, Node/npm, Git, and optional system tools such as Docker, FFmpeg, OpenSSH, and Chromium/Playwright (`README.md`, `scripts/install.sh`).
- Platform extras support Linux, macOS, Windows, Termux/Android, and desktop packaging through conditional dependencies (`pyproject.toml`, `Dockerfile`).

**Production:**
- Supported deployment is a local host or Docker container using Debian 13, non-root `hermes`, `/opt/data` persistent state, and s6-overlay supervision (`Dockerfile`, `docker-compose.yml`).

---

*Stack analysis: 2026-08-14*
