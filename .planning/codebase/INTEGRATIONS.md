# External Integrations

**Analysis Date:** 2026-08-14

## APIs & External Services

**Model Providers:**
- OpenAI and OpenAI-compatible endpoints - Chat/Responses model execution, embeddings/vision-related calls, and Codex routes.
  - SDK/Client: `openai` and `httpx` (`agent/codex_responses_adapter.py`, `agent/chat_completion_helpers.py`).
  - Auth: provider-specific API key/base URL configuration; names are defined in `agent/` and `plugins/model-providers/`.
- Anthropic - Native Claude adapter and provider plugin (`agent/anthropic_adapter.py`, `plugins/model-providers/anthropic/`).
- Google Gemini/Vertex AI - Native Gemini HTTP adapter and Vertex provider (`agent/gemini_native_adapter.py`, `agent/vertex_adapter.py`, `plugins/model-providers/gemini/`, `plugins/model-providers/vertex/`).
- AWS Bedrock - Bedrock provider adapter (`agent/bedrock_adapter.py`, `plugins/model-providers/bedrock/`).
- Azure Foundry/identity - Azure model and identity support (`agent/azure_identity_adapter.py`, `plugins/model-providers/azure-foundry/`).
- Aggregators and hosted providers - OpenRouter, DeepSeek, xAI, Mistral-compatible/custom endpoints, Together-like provider plugins, and others under `plugins/model-providers/`.

**Web Search and Browsing:**
- Exa, Firecrawl, Parallel Web, Tavily, Brave, SearXNG, DDGS, and xAI search providers (`plugins/web/`).
- Browserbase, Browser Use, and Firecrawl browser backends (`plugins/browser/`).
- Local Chromium through Playwright and browser/CDP websockets (`agent/browser_provider.py`, `scripts/install.sh`, `Dockerfile`).

**Messaging and Collaboration:**
- Telegram, Discord, Slack, WhatsApp (native bridge and cloud), Signal, Matrix, Google Chat, Microsoft Teams, WeCom/Weixin, DingTalk, Feishu, Mattermost, IRC, LINE, SMS, email, Ntfy, SimpleX, Home Assistant, and other platform adapters (`plugins/platforms/`, `gateway/platforms/`).
- WhatsApp native transport uses a separate Node bridge with Baileys (`scripts/whatsapp-bridge/package.json`).
- Photon/iMessage uses a Node sidecar (`plugins/platforms/photon/sidecar/package.json`).

**Automation and Agent Protocols:**
- MCP servers/clients, stdio transport, OAuth support, and tool schema caching (`agent/transports/hermes_tools_mcp_server.py`, `tools/mcp_*.py`).
- A2A peer integration and authenticated relay/gateway callbacks (`plugins/platforms/a2a/`, `agent/relay/`, `gateway/relay/`).
- Chronos/NAS-managed cron integration via authenticated callbacks (`plugins/cron_providers/chronos/`, `docs/chronos-managed-cron-contract.md`).

## Data Storage

**Databases:**
- SQLite - Local configuration, session/history-related state, plugin stores, and Matrix persistence; Python stdlib and optional `aiosqlite` (`scripts/profile-tui.py`, `plugins/platforms/matrix/`, `Dockerfile`).
- PostgreSQL - Optional Matrix backend via `asyncpg` and external/managed provider integrations where enabled (`pyproject.toml`, `plugins/platforms/matrix/`).
- Hindsight, Mem0, Supermemory, Honcho, OpenViking, RetainDB, and custom memory backends - Optional cloud or service-backed memory (`plugins/memory/`, `pyproject.toml`).

**File Storage:**
- Local filesystem under the Hermes data directory (`~/.hermes` locally, `/opt/data` in Docker), including profiles, credentials, logs, skills, browser state, and media (`docker-compose.yml`, `agent/credential_persistence.py`).
- Optional object/file operations are exposed through dashboard and tool integrations (`dashboard/`, `tools/`).

**Caching:**
- Local in-process and filesystem caches for prompts, schemas, browser state, media, and provider metadata (`agent/prompt_cache_boundary.py`, `tools/mcp_schema_cache.py`).

## Authentication & Identity

**Auth Provider:**
- Provider-native API keys, OAuth/device flows, cloud service-account credentials, bot tokens, and JWTs; dispatch is provider-specific (`agent/credential_sources.py`, `tools/mcp_oauth.py`, `plugins/platforms/google_chat/oauth.py`).
- Gateway/API authentication uses API keys, bearer tokens, OAuth tickets, allowlists, and optional JWT secrets (`gateway/`, `api_server/`, `apps/shared/src/websocket-url.ts`).
- GitHub App JWT authentication supports the Skills Hub/bot identity path (`pyproject.toml`, `skills/`).

## Monitoring & Observability

**Error Tracking:**
- Not detected as a mandatory hosted error-tracking SDK. Runtime diagnostics, traces, usage/billing accounting, and verification evidence are implemented in `agent/trace_upload.py`, `agent/stream_diag.py`, `agent/billing_usage.py`, and `agent/verification_evidence.py`.

**Logs:**
- Python logging with rotating files and platform-aware handlers (`hermes_logging.py`), container stdout/stderr, and s6-overlay supervision (`Dockerfile`, `docker/`).

## CI/CD & Deployment

**Hosting:**
- Local process, Docker Compose, and container registry/image deployments; the compose file runs gateway and dashboard services (`docker-compose.yml`).

**CI Pipeline:**
- GitHub Actions configuration is under `.github/`; dependency updates are managed by `.github/dependabot.yml`.
- Install/release automation is implemented in `scripts/install.sh` and related scripts.

## Environment Configuration

**Required env vars:**
- No single universal external-service secret is required for every install. Provider keys/tokens, gateway/API settings, messaging credentials, OAuth values, and service-account paths are selected by enabled features (`config.yaml.example`, `cli-config.yaml.example`, `gateway/`, `plugins/platforms/`).
- Representative configuration surfaces include model provider credentials in `agent/credential_sources.py`, API server settings in `api_server/`, messaging settings in `gateway/platforms/` and `plugins/platforms/`, and Docker identity variables `HERMES_UID`/`HERMES_GID` in `docker-compose.yml`.

**Secrets location:**
- User-owned Hermes data/configuration directory (`~/.hermes`) or mounted secret files in Docker; `docker-compose.yml` documents mounted Google Chat service-account JSON without exposing its contents.

## Webhooks & Callbacks

**Incoming:**
- Generic authenticated webhooks (`gateway/platforms/webhook.py`).
- Messaging platform callbacks/webhooks for Telegram, Slack, Teams, Google Chat, WeCom, Matrix, and other adapters (`gateway/platforms/`, `plugins/platforms/`).
- Chronos/NAS scheduled-job callbacks and A2A push/callback routes (`plugins/cron_providers/chronos/`, `plugins/platforms/a2a/`).

**Outgoing:**
- Platform message sends, media uploads, outbound webhook events, and callback delivery (`agent/outbound_webhooks.py`, `tools/send_message_tool.py`, `gateway/`).
- External model/search/memory requests and MCP tool calls are issued through provider adapters and transport implementations (`agent/`, `plugins/`, `tools/`).

---

*Integration audit: 2026-08-14*
