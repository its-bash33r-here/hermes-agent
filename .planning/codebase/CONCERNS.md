# Codebase Concerns

**Analysis Date:** 2026-08-14

## Tech Debt

**Monolithic runtime modules:**
- Issue: Core behavior is concentrated in unusually large modules, making changes difficult to isolate and increasing regression risk. `gateway/run.py` is approximately 29,595 lines, `cli.py` approximately 19,292 lines, `hermes_cli/web_server.py` approximately 18,310 lines, `tui_gateway/server.py` approximately 14,488 lines, and `hermes_state.py` approximately 11,648 lines.
- Files: `gateway/run.py`, `cli.py`, `hermes_cli/web_server.py`, `tui_gateway/server.py`, `hermes_state.py`.
- Impact: Ownership boundaries, review scope, startup behavior, and test targeting are difficult to reason about; small cross-cutting changes can affect CLI, gateway, TUI, and web paths simultaneously.
- Fix approach: Extract cohesive domains behind narrow modules, starting with the largest files; preserve public entry points and add characterization tests around each extracted boundary before moving logic.

**Duplicated command-execution implementations:**
- Issue: User/configured shell commands are implemented separately in the CLI, TUI gateway, MCP catalog, bang shell, and goal tooling, each with its own timeout, environment, output, and approval behavior.
- Files: `cli.py`, `tui_gateway/methods_tools.py`, `hermes_cli/mcp_catalog.py`, `hermes_cli/bang_shell.py`, `hermes_cli/goals.py`.
- Impact: Security fixes and output/redaction rules can drift between entry points; a command safe in one surface may have different controls in another.
- Fix approach: Centralize process launching, environment sanitization, approval checks, timeout handling, and output truncation in one shared service, then retain only surface-specific authorization at callers.

**Profile-aware secret persistence is incomplete:**
- Issue: Pairing reads can use the profile secret scope, but the allowlist grant mirror still writes through root `.env` helpers.
- Files: `gateway/pairing.py`, `hermes_cli/config.py`, `agent/secret_scope.py`.
- Impact: Multiplexed profiles can persist a pairing grant to the wrong profile or root configuration, producing authorization inconsistency and potentially exposing cross-profile configuration.
- Fix approach: Add a profile-aware write/remove API, make pairing persistence use it exclusively, and test concurrent or alternating profile grants and revocations.

**Mixed dependency declaration strategy:**
- Issue: Python dependencies are heavily exact-pinned in `pyproject.toml`, but several direct requirements remain ranged (`fastapi`, `uvicorn`, `python-multipart`, `packaging`-adjacent platform dependencies), while the JavaScript workspace has its own lockfile and overrides.
- Files: `pyproject.toml`, `uv.lock`, `package.json`, `package-lock.json`, `ui-tui/package.json`.
- Impact: Reproducibility and security review differ by runtime; an update can be intentionally pinned in Python but arrive through a workspace range or override in JavaScript.
- Fix approach: Define one documented update policy per ecosystem, enforce lockfile freshness in CI, and record exceptions for intentionally ranged packages.

## Known Bugs

**Incomplete Yuanbao chat metadata:**
- Symptoms: The adapter documents a TODO to fetch real chat names and member counts, so user-facing metadata can remain placeholder or incomplete.
- Files: `gateway/platforms/yuanbao.py`.
- Trigger: Handling Yuanbao conversations where display metadata is expected from the remote API.
- Workaround: Treat adapter-provided names/counts as optional and avoid using them as authoritative identifiers.

**Compatibility fallback remains in the desktop slash path:**
- Symptoms: A TODO-marked compatibility fallback remains active in slash command handling, so behavior depends on whether newer and legacy paths are both available.
- Files: `apps/desktop/src/app/chat/composer/use-prompt-actions/slash.ts`.
- Trigger: Desktop command handling across versions or mixed client/server deployments.
- Workaround: Test slash commands against the exact desktop build and gateway version together.

## Security Considerations

**Shell execution from configuration and gateway RPC:**
- Risk: `shell=True` executes command strings through the host shell. Configured quick commands and MCP bootstrap commands are trusted only to the extent that configuration and catalog contents are trusted; gateway `shell.exec` also exposes a command execution surface.
- Files: `cli.py`, `tui_gateway/methods_tools.py`, `hermes_cli/mcp_catalog.py`, `hermes_cli/bang_shell.py`.
- Current mitigation: Some paths sanitize environment variables, redact output, apply timeouts, or call `detect_dangerous_command`/`detect_hardline_command`; TUI quick commands still rely on configured shell strings by design.
- Recommendations: Make trust boundaries explicit in API contracts, use argv execution for commands that do not require shell syntax, require authorization at the gateway boundary, audit every caller for cwd/path restrictions, and add parity tests for environment leakage, shell metacharacters, timeout, and redaction.

**MCP bootstrap supply-chain exposure:**
- Risk: Git-backed MCP installation clones repositories and runs manifest-provided bootstrap commands through a shell. A changed repository or catalog entry can execute arbitrary code with the user's account privileges.
- Files: `hermes_cli/mcp_catalog.py`, `optional-mcps/*/manifest.yaml`.
- Current mitigation: Installation is user-initiated and uses a per-user install root; commands and output are visible.
- Recommendations: Pin repository revisions, verify signatures or checksums, show a structured approval summary before execution, isolate builds, and record provenance for installed revisions.

**Pairing fallback can mask secret-scope failures:**
- Risk: Pairing catches broad exceptions around secret-scope imports and reads, then falls back to process environment values. A real scope/configuration failure can therefore become an unscoped read rather than a fail-closed denial.
- Files: `gateway/pairing.py`.
- Current mitigation: `UnscopedSecretError` is handled separately and scoped reads attempt to return an empty value.
- Recommendations: Catch only expected compatibility errors, distinguish unavailable secret scope from an empty secret, emit an audit event, and fail closed when a profile-scoped gateway cannot establish its scope.

## Performance Bottlenecks

**Large state and gateway modules on hot paths:**
- Problem: Session, gateway, and web-server responsibilities are implemented in very large files with broad imports and shared runtime state.
- Files: `hermes_state.py`, `gateway/run.py`, `hermes_cli/web_server.py`, `tui_gateway/server.py`.
- Cause: Multiple protocol, persistence, routing, and lifecycle concerns are co-located, increasing import/startup work and making it difficult to bound expensive operations.
- Improvement path: Profile startup and representative request/turn flows, then split persistence, transport, routing, and rendering paths; add latency budgets and regression benchmarks for gateway RPC and session operations.

**Committed research and media artifacts enlarge the repository:**
- Problem: Large generated/research assets and vendor material are tracked alongside runtime code.
- Files: `mcp-research-data/*.json`, `native/fts5_cjk/vendor/sqlite3.h`, `optional-skills/mlops/*/references/*`, `apps/desktop/public/ds-assets/filler-bg0.jpg`.
- Cause: Evaluation data, vendored headers, documentation snapshots, and product assets share the source tree without a clearly separated artifact policy.
- Improvement path: Identify required-at-runtime assets, move reproducible evaluation outputs to artifact storage or dedicated fixtures, and document vendor update ownership.

## Fragile Areas

**Cross-platform process compatibility:**
- Files: `hermes_bootstrap.py`, `hermes_cli/_subprocess_compat.py`, `hermes_cli/gateway_windows.py`, `tools/environments/base.py`.
- Why fragile: Windows console suppression, encoding, process groups, shell behavior, and log rotation are handled through compatibility patches spread across entry points and environment backends.
- Safe modification: Change one platform path at a time, run the OS matrix and installer tests, and preserve the existing no-console/UTF-8 behavior tests before refactoring shared helpers.
- Test coverage: Many focused tests exist under `tests/tools`, `tests/hermes_cli`, and `tests/install`, but the breadth of platform-specific behavior makes full matrix execution essential.

**Gateway/TUI protocol boundary:**
- Files: `tui_gateway/server.py`, `tui_gateway/methods_tools.py`, `ui-tui/src/gatewayClient.ts`, `tests/test_tui_gateway_server.py`.
- Why fragile: The Python RPC server and TypeScript client evolve across separate packages while the server and its test module are both very large.
- Safe modification: Update protocol schemas/handlers and client calls together, add compatibility tests for every changed method, and verify error codes and truncation behavior end-to-end.
- Test coverage: `tests/test_tui_gateway_server.py` is extensive, but its size increases maintenance and makes targeted ownership unclear; client-side coverage is distributed across `ui-tui/src` tests.

**Plugin and platform adapters:**
- Files: `plugins/platforms/telegram/adapter.py`, `plugins/platforms/discord/adapter.py`, `plugins/platforms/slack/adapter.py`, `plugins/platforms/feishu/adapter.py`, `plugins/platforms/matrix/adapter.py`.
- Why fragile: Several adapters are 5,000–10,500 lines and each integrates external protocol behavior, auth, media, retries, and formatting.
- Safe modification: Keep adapter-specific code behind the common platform interfaces, add contract tests with fake transports, and avoid copying fixes between adapters without a shared abstraction.
- Test coverage: Platform tests are present under `tests/plugins` and gateway test directories, but adapter-level parity should be checked for every behavior change.

## Scaling Limits

**Single-process, multi-surface state coordination:**
- Current capacity: The application supports CLI, gateway, TUI, desktop, web, cron, plugins, and multiple profiles from a shared codebase and filesystem state.
- Limit: Shared environment/configuration and session state can become ambiguous under multiplexed profiles and concurrent processes, as shown by the root `.env` pairing write path.
- Scaling path: Make profile identity explicit in every state and secret API, use transactional stores for shared mutations, and test concurrent gateway/cron/TUI access rather than relying on process-local assumptions.

## Dependencies at Risk

**Platform- and provider-heavy optional dependency graph:**
- Risk: Voice, wake-word, messaging, Matrix encryption, desktop, and provider extras pull native or platform-specific packages with different wheel availability and compatibility constraints.
- Impact: A feature-specific install can fail or silently degrade on a supported platform even when the core agent works.
- Migration plan: Keep extras independently testable, publish supported platform matrices, run install tests for each extra, and isolate optional imports behind capability checks with explicit user-facing errors.

## Missing Critical Features

**Complete profile-scoped grant lifecycle:**
- Problem: Pairing grants do not have a fully profile-aware persistence path.
- Blocks: Reliable authorization management for concurrent/multiplexed profiles.

**Unified command execution policy:**
- Problem: Shell command controls are repeated across surfaces rather than enforced by one policy layer.
- Blocks: A single auditable security contract for command execution, environment inheritance, and output handling.

## Test Coverage Gaps

**Cross-surface command policy parity:**
- What's not tested: The same dangerous command, environment, timeout, and redaction cases across CLI quick commands, bang commands, TUI RPC, MCP bootstrap, and goal execution.
- Files: `cli.py`, `tui_gateway/methods_tools.py`, `hermes_cli/mcp_catalog.py`, `hermes_cli/bang_shell.py`, `hermes_cli/goals.py`.
- Risk: Security and behavior regressions can be fixed in one entry point while remaining in another.
- Priority: High

**Profile-scoped pairing persistence under concurrency:**
- What's not tested: Alternating profiles, simultaneous grants/revokes, scope-unavailable failures, and root `.env` isolation.
- Files: `gateway/pairing.py`, `agent/secret_scope.py`, `hermes_cli/config.py`.
- Risk: Cross-profile authorization leakage or grants that appear successful but persist in the wrong configuration.
- Priority: High

**Provider adapter contract parity:**
- What's not tested: Consistent auth, delivery, retry, media, and metadata behavior across all platform adapters using a shared contract suite.
- Files: `plugins/platforms/*/adapter.py`, `tests/plugins`, `tests/gateway`.
- Risk: A fix or protocol change can leave one integration silently degraded.
- Priority: Medium

---

*Concerns audit: 2026-08-14*
