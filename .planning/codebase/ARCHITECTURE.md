<!-- refreshed: 2026-08-14 -->
# Architecture

**Analysis Date:** 2026-08-14

## System Overview

```text
┌────────────────────────────────────────────────────────────────────┐
│ User surfaces: `hermes`/`cli.py` · `gateway/run.py` · `web/src/`   │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│ Shared orchestration: `run_agent.py`, `agent/`, `gateway/session.py`│
└──────────────┬──────────────────────┬─────────────────────────────┘
               ▼                      ▼
     `agent/transports/`       `tools/`, `plugins/`, `skills/`
               │                      │
               └──────────────┬───────┘
                              ▼
                 SQLite/files under Hermes home
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| CLI | Commands, setup, profiles, providers, interactive sessions | `hermes_cli/main.py`, `cli.py` |
| Agent runtime | Prompt/model/tool loop, compression, retries, usage | `run_agent.py`, `agent/conversation_loop.py` |
| Gateway | Platform lifecycle, routing, delivery, concurrency | `gateway/run.py` |
| Sessions | Source identity, persistence, recovery, transcripts | `gateway/session.py` |
| Platforms | Translate external events/messages to common gateway types | `gateway/platforms/base.py`, `gateway/platforms/*.py` |
| Providers | Normalize model APIs behind transports | `agent/transports/base.py`, `agent/transports/*.py` |
| Tools | Discover, authorize, execute, and classify model tools | `tools/`, `agent/tool_executor.py` |
| Dashboard | Web API/auth and React administration UI | `hermes_cli/web_routers/`, `hermes_cli/dashboard_auth/`, `web/src/` |
| TUI | Terminal UI and its JSON/RPC bridge | `ui-tui/src/`, `tui_gateway/` |

## Pattern Overview

**Overall:** Layered, adapter-based application with a shared agent core and two orchestration hosts: interactive CLI/TUI and long-lived messaging gateway.

**Key Characteristics:**
- Reusable runtime code lives in `agent/`; entry points compose it.
- Providers and platforms are isolated behind `agent/transports/` and `gateway/platforms/` contracts.
- Session identity and persistence are explicit through `SessionSource` and `SessionStore` in `gateway/session.py`.
- Configuration is normalized into typed dataclasses and profile-scoped files (`gateway/config.py`, `hermes_cli/config.py`).
- Optional capabilities are loaded through tools, plugins, MCP servers, and skills.

## Layers

**Presentation and entry:** `hermes`, `hermes_cli/`, `tui_gateway/`, `web/src/`, and `gateway/platforms/` accept CLI, browser, terminal, or external-platform input and render output. They depend on orchestration APIs.

**Orchestration:** `run_agent.py`, `gateway/run.py`, and `tui_gateway/server.py` coordinate turns, sessions, adapters, lifecycle, queues, and shutdown. They depend on the agent runtime, state, configuration, and adapters.

**Agent runtime:** `agent/agent_init.py`, `agent/conversation_loop.py`, `agent/prompt_builder.py`, and `agent/tool_executor.py` build prompts, call models, execute tools, compress context, retry, and emit hooks.

**Integration adapters:** `agent/transports/`, `agent/*_adapter.py`, `gateway/platforms/`, `gateway/relay/`, and `tools/` contain provider, platform, relay, MCP, browser, and media boundaries.

**Persistence/configuration:** `hermes_state.py`, `hermes_state_schema.py`, `gateway/session.py`, `gateway/config.py`, and `hermes_cli/config.py` manage SQLite, transcripts, YAML/JSON settings, and profile routing under the Hermes home.

## Data Flow

### Primary Interactive Request Path

1. `hermes` enters `hermes_cli/main.py` (with legacy compatibility in `cli.py`).
2. `hermes_cli/env_loader.py`, `hermes_cli/config.py`, and `hermes_constants.py` resolve environment, profile, and provider state.
3. `agent/agent_init.py` and `run_agent.py` construct the agent.
4. `agent/conversation_loop.py:run_conversation` calls the selected `agent/transports/*` implementation.
5. `agent/tool_executor.py` validates, authorizes, and runs tools from `tools/`.
6. Results and transcript state are persisted through `hermes_state.py` and rendered by the CLI/TUI.

### Gateway Message Path

1. `gateway/run.py:GatewayRunner` loads `GatewayConfig` from `gateway/config.py` and creates adapters via `gateway/platform_registry.py`.
2. A `gateway/platforms/` adapter converts an inbound event to common source/message context.
3. `gateway/session.py:SessionStore` resolves or creates a profile-aware session.
4. The runner binds state and invokes `agent.run_conversation` through its gateway turn runner.
5. Progress/tool events become platform-safe delivery operations; the adapter sends the result.
6. Sessions, transcripts, and lifecycle state are persisted for recovery.

### Dashboard/PTY Path

1. `hermes_cli/web_routers/` exposes APIs and `hermes_cli/dashboard_auth/` protects them.
2. `web/src/pages/` calls dashboard APIs and WebSocket clients in `web/src/lib/`.
3. Embedded chat/terminal uses `tui_gateway/` PTY/RPC dispatch.
4. `tui_gateway/event_publisher.py` can mirror TUI events to the dashboard sidecar WebSocket.

**State Management:** Runtime caches are process-local; sessions/transcripts are SQLite/files under the active Hermes home/profile; profile routing is explicit in `gateway/profile_routing.py`; browser state is React context/component state backed by APIs.

## Key Abstractions

**`GatewayRunner`:** Long-lived gateway lifecycle and message orchestration in `gateway/run.py`; uses mixins for authorization, slash commands, and kanban concerns.

**`SessionSource` / `SessionEntry` / `SessionStore`:** Stable source identity, metadata, recovery, transcript persistence, and reset policy in `gateway/session.py`; use store methods instead of direct file mutation.

**`BasePlatformAdapter` / `PlatformRegistry`:** Common messaging lifecycle and deferred/plugin registration in `gateway/platforms/base.py` and `gateway/platform_registry.py`.

**`ProviderTransport`:** Common model-provider boundary in `agent/transports/base.py` and implementations such as `agent/transports/chat_completions.py`.

**Tool execution boundary:** Model schemas/definitions in `model_tools.py`, implementations in `tools/`, and authorization/execution in `agent/tool_executor.py` and `agent/tool_guardrails.py`.

## Entry Points

**Interactive CLI:** `hermes`, `hermes_cli/main.py`, and `cli.py`; parses commands, performs setup, and runs interactive sessions.

**Messaging gateway:** `gateway/run.py` and `hermes_cli/gateway.py`; starts adapters, routes messages, runs agents, delivers responses, and manages lifecycle.

**TUI RPC gateway:** `tui_gateway/entry.py` and `tui_gateway/server.py`; provides JSON/RPC, stdio, WebSocket, MCP-discovery coordination, and shutdown finalization.

**Web dashboard:** `web/src/main.tsx`, `web/src/App.tsx`, and `hermes_cli/web_routers/`; serves configuration, sessions/files/logs, profiles, skills/plugins/MCP, and embedded chat.

## Architectural Constraints

- **Threading:** Asyncio handles gateway/platform work; worker threads/executors bridge synchronous model/tool paths. `tui_gateway` coordinates background discovery and shutdown threads.
- **Global state:** Registries/caches exist in `gateway/platform_registry.py`, `gateway/run.py`, `tui_gateway/entry.py`, and provider/tool registries; preserve cleanup and scope handling.
- **Import bootstrap:** `hermes_bootstrap.py` must be imported before application imports in entry paths such as `tui_gateway/entry.py`.
- **Profile isolation:** Sessions, MCPs, skills, and config may be profile-scoped; use Hermes home/profile helpers rather than process-global paths.
- **Portability:** Windows, macOS, Linux, Termux, Docker, and Nix are supported; keep platform-specific code behind dedicated modules/guards.

## Anti-Patterns

### Bypassing the shared session store

**What happens:** Features derive IDs or edit transcripts independently of `gateway/session.py`.
**Why it's wrong:** It breaks profile routing, recovery, locking, expiry, or search.
**Do this instead:** Use `SessionSource`, `SessionStore`, and their persistence methods.

### Putting platform behavior in the runner

**What happens:** `gateway/run.py` accumulates platform-specific parsing/delivery branches.
**Why it's wrong:** The central lifecycle becomes coupled to one adapter.
**Do this instead:** Extend `gateway/platforms/base.py` or add an adapter under `gateway/platforms/` and register it through `gateway/platform_registry.py`.

## Error Handling

**Strategy:** Classify errors at boundaries, preserve user-safe messages, log operational context, and recover long-lived processes where possible.

**Patterns:** Provider errors use `agent/error_classifier.py` and retry helpers; tool failures return structured results through `agent/tool_executor.py`; gateway failures use bounded connect/disconnect, pause/retry, drain, and restart logic in `gateway/run.py`; API/auth errors are handled in `hermes_cli/web_routers/` and `hermes_cli/dashboard_auth/`.

## Cross-Cutting Concerns

**Logging:** `hermes_logging.py`, module loggers, `agent/monitoring/`, and gateway lifecycle modules.
**Validation:** `gateway/config.py`, `hermes_cli/config.py`, `agent/tool_executor.py`, and `agent/tool_guardrails.py`.
**Authentication:** `hermes_cli/auth.py`, `agent/credential_*`, `hermes_cli/dashboard_auth/`, and `gateway/pairing.py`.

---

*Architecture analysis: 2026-08-14*
