# Codebase Structure

**Analysis Date:** 2026-08-14

## Directory Layout

```text
hermes-agent/
├── agent/                 # Shared agent runtime and provider integrations
├── gateway/               # Messaging gateway, sessions, platforms, relay
├── hermes_cli/            # CLI, config, auth, profiles, dashboard backend
├── tools/                 # Model-callable tools and registries
├── tui_gateway/           # JSON/RPC and PTY bridge
├── ui-tui/                # TypeScript/Ink terminal UI
├── web/                   # React/Vite dashboard frontend
├── plugins/               # Built-in plugin packages
├── skills/                # Built-in skill bundles and indexes
├── optional-skills/       # Optional skill groups
├── optional-mcps/         # Optional MCP integrations
├── tests/                 # Python tests
├── tests-js/              # JavaScript tests
├── scripts/               # CI, packaging, sandbox, bridge, diagnostics
├── docs/                  # ADRs, RFCs, architecture and operations docs
├── native/                # C extension and vendored headers
├── docker/                # Container entrypoints and s6 services
├── nix/                   # Nix packages/modules/dev shell/checks
├── locales/               # Translation YAML files
├── apps/                  # Desktop/bootstrap/shared workspaces
├── providers/             # Provider package boundary/docs
├── cli.py                 # Legacy/monolithic CLI implementation
├── run_agent.py           # Direct agent runtime entry
├── hermes_state.py        # Durable state implementation
├── mcp_serve.py           # MCP serving entrypoint
├── hermes                 # Executable launcher
├── pyproject.toml         # Python package/dependencies/tool config
└── package.json           # Node workspaces/scripts
```

## Directory Purposes

**`agent/`:** Shared model-agent runtime: conversation loop, prompt/context, transports/adapters, tool execution, memory, security, lifecycle, and monitoring. Key files: `agent/agent_init.py`, `agent/conversation_loop.py`, `agent/tool_executor.py`, `agent/transports/base.py`.

**`gateway/`:** Long-running multi-platform host: runner, config, sessions, delivery, lifecycle, adapters, relay, media, and profile routing. Key files: `gateway/run.py`, `gateway/config.py`, `gateway/session.py`, `gateway/platforms/base.py`, `gateway/platform_registry.py`.

**`hermes_cli/`:** CLI command implementation and local management: setup/auth/config, profiles, gateway, MCP/skills/plugins, dashboard APIs, and dashboard auth. Key files: `hermes_cli/main.py`, `hermes_cli/config.py`, `hermes_cli/gateway.py`, `hermes_cli/profiles.py`, `hermes_cli/web_routers/`, `hermes_cli/dashboard_auth/`.

**`tools/`:** Model-callable terminal/filesystem/browser/web/search/media/MCP and specialized tools. Key files: `tools/terminal_tool.py`, `tools/browser_tool.py`, `tools/mcp_tool.py`, `tools/lazy_deps.py`.

**`tui_gateway/`:** JSON/RPC, stdio, WebSocket, PTY, rendering, host supervision, and session/event helpers. Key files: `tui_gateway/entry.py`, `tui_gateway/server.py`, `tui_gateway/transport.py`, `tui_gateway/ws.py`.

**`ui-tui/`:** TypeScript/Ink UI. `ui-tui/src/app/` owns orchestration/state, `src/components/` presentation, `src/domain/` logic/types, `src/hooks/` reusable behavior, and `src/__tests__/` tests. Key files: `ui-tui/src/entry.tsx`, `ui-tui/src/app.tsx`, `ui-tui/src/gatewayClient.ts`.

**`web/`:** React/Vite dashboard. `web/src/pages/`, `src/components/`, `src/contexts/`, `src/hooks/`, `src/lib/`, `src/plugins/`, and `src/themes/` separate screens, UI, state, behavior, APIs, extensibility, and themes. Key files: `web/src/main.tsx`, `web/src/App.tsx`, `web/package.json`.

**`plugins/`, `skills/`, `optional-skills/`, `optional-mcps/`:** Extensibility bundles. Use `plugins/` for in-process plugin packages; `skills/` or `optional-skills/` for skill bundles; `optional-mcps/` for optional MCP implementations.

**`tests/`, `tests-js/`, `ui-tui/src/__tests__/`:** Python tests, JS integration tests, and colocated TUI unit/component tests.

**`docs/`, `scripts/`, `docker/`, `nix/`, `native/`:** Operational documentation, automation, deployment, packaging, and native extension support. Key files: `docs/ADR.md`, `docs/session-lifecycle.md`, `scripts/run_tests.sh`, `docker/entrypoint.sh`, `nix/hermes-agent.nix`, `native/fts5_cjk/`.

## Key File Locations

**Entry Points:** `hermes`; `hermes_cli/main.py`; `cli.py`; `run_agent.py`; `gateway/run.py`; `tui_gateway/entry.py`; `web/src/main.tsx`; `ui-tui/src/entry.tsx`; `mcp_serve.py`.

**Configuration:** `pyproject.toml`; `package.json`; `gateway/config.py`; `hermes_cli/config.py`; `cli-config.yaml.example`; `web/vite.config.ts`; `web/tsconfig*.json`; `ui-tui/package.json`; `ui-tui/tsconfig.json`.

**Core Logic:** `agent/conversation_loop.py`; `agent/agent_init.py`; `agent/tool_executor.py`; `gateway/run.py`; `gateway/session.py`; `hermes_state.py`.

**Testing:** `tests/`; `tests-js/`; `ui-tui/src/__tests__/`; `scripts/tests/`.

## Naming Conventions

**Files:** Python modules use lowercase `snake_case` (`gateway/profile_routing.py`); TypeScript components use PascalCase (`web/src/components/ProfileSwitcher.tsx`) while clients/helpers use camelCase (`ui-tui/src/gatewayClient.ts`). Tests use `test_*.py`, `*_test.py`, `*.test.ts`, or `*.test.tsx`. Documentation is lowercase descriptive text except generated GSD maps.

**Directories:** Python packages are lowercase subsystem names; frontend code is grouped into `components/`, `pages/`, `hooks/`, `lib/`, and `domain/`. Platform adapters belong directly under `gateway/platforms/`; provider transports under `agent/transports/`.

## Where to Add New Code

**New Feature:** Put reusable runtime behavior in `agent/`, messaging routing/lifecycle in `gateway/`, CLI behavior in `hermes_cli/`, dashboard screens in `web/src/pages/`, shared dashboard UI in `web/src/components/`, and TUI state/logic in `ui-tui/src/app/` or `ui-tui/src/domain/`. Add tests under the nearest existing `tests/` grouping or frontend package test directory.

**New Component/Module:** For a platform, implement `gateway/platforms/base.py` contract, document in `gateway/platforms/ADDING_A_PLATFORM.md`, and register in `gateway/platform_registry.py`. For a provider, use `agent/transports/` or `agent/*_adapter.py` and connect selection through `agent/agent_init.py`. For a tool, use `tools/` plus existing `model_tools.py`, `agent/tool_executor.py`, and `agent/tool_guardrails.py` boundaries. For plugins/skills, preserve the conventions in `plugins/`, `skills/`, or `optional-skills/`.

**Utilities:** Shared runtime helpers belong in `utils.py` or focused `agent/`/`hermes_cli/` modules; gateway-only helpers belong under `gateway/`; frontend helpers belong in `web/src/lib/` or `ui-tui/src/domain/`; non-runtime automation belongs in `scripts/`.

## Special Directories

**`.planning/codebase/`:** GSD-generated maps; generated by mapper workflows and intended for orchestrator-managed commits.

**`skills/index-cache/`:** Generated skill metadata; do not hand-edit indexes.

**`native/fts5_cjk/vendor/`:** Vendored SQLite/FTS5 headers; committed source support for the native extension.

**`web/dist/`, `ui-tui/dist/`, and dependency directories:** Generated build output/installed dependencies; normally uncommitted and governed by `.gitignore`.

---

*Structure analysis: 2026-08-14*
