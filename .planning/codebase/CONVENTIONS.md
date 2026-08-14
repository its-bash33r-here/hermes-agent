# Coding Conventions

**Analysis Date:** 2026-08-14

## Naming Patterns

**Files:**
- Python modules use lowercase `snake_case`, for example `agent/retry_utils.py` and `acp_adapter/edit_approval.py`.
- TypeScript modules use lowercase names for libraries and hooks, with PascalCase for React components such as `web/src/pages/ChatPage.tsx`.
- Tests use `test_*.py` / `*_test.py` for Python and `.test.ts` / `.test.tsx` for TypeScript. Keep tests beside the implementation in UI workspaces, as in `ui-tui/src/lib/fuzzy.test.ts`.

**Functions:**
- Python functions use `snake_case`; private helpers start with `_`, as in `_jitter_counter` and `_hermes_home_points_at_production` in `tests/conftest.py`.
- TypeScript functions use `camelCase`; React hooks use the `useX` prefix, and exported types/interfaces use PascalCase, as in `ui-tui/src/lib/fuzzy.ts`.

**Variables:**
- Python locals and constants use `snake_case` / uppercase constants. Module-level mutable state is explicitly private and guarded where necessary, as in `agent/retry_utils.py`.
- TypeScript locals use `camelCase`; constants use descriptive uppercase names when they are module-wide configuration, such as `WORD_BOUNDARY` in `ui-tui/src/lib/fuzzy.ts`.

**Types:**
- Python uses modern annotations (`list[str]`, unions, `Path`) and dataclass/value-object style types where appropriate.
- TypeScript uses explicit interfaces and generic types for public helpers; avoid weakening types to `any`. Use `import type` for type-only imports.

## Code Style

**Formatting:**
- Python follows the repository's standard formatting and explicit UTF-8 file I/O expectations; new text `open`, `read_text`, and `write_text` calls should specify `encoding` unless a test intentionally exercises locale behavior.
- TypeScript uses Prettier configured by `.prettierrc`: two spaces, single quotes, no semicolons, 120-column width, no trailing commas, and omitted arrow-function parentheses where possible.
- Some existing web tests use double quotes and semicolons (`web/src/pages/ChatPage.test.tsx`); when editing a file, preserve its local formatter style and run that workspace's formatter/check command.

**Linting:**
- Python's blocking Ruff rule is `PLW1514` in `pyproject.toml`; `ty` is also run in CI for type diagnostics.
- TypeScript workspaces share `eslint.config.shared.mjs`. Enforced conventions include sorted imports/exports/JSX props, type-only imports, braces on all control-flow bodies, unused-import rejection, and React Hooks rules.
- Use the workspace scripts (`npm run check`, `npm run lint`, or `npm run fmt`) rather than inventing a repo-wide command for a focused change.

## Import Organization

**Order:**
1. Python standard library imports.
2. Third-party imports.
3. First-party/local imports.
4. TypeScript side effects, built-ins, external packages, internal aliases, parent modules, sibling modules, then index imports, matching `eslint.config.shared.mjs`.

**Path Aliases:**
- The web app uses `@/` for `web/src`, configured in `web/vitest.config.ts` and the TypeScript configuration.
- Prefer relative sibling imports in `ui-tui`, as shown in `ui-tui/src/lib/fuzzy.test.ts`, unless the workspace already provides an alias.

## Error Handling

**Patterns:**
- Raise domain-specific exceptions with docstrings in Python, using `agent/errors.py` as the canonical pattern.
- Catch only errors that can be handled or translated. Log expected operational failures with module loggers; use `logger.exception` for unexpected crashes, as in `acp_adapter/entry.py`.
- In TypeScript, return explicit nullable results for expected non-matches (`fuzzyScore` returns `null`) and use typed async error paths. Tests should assert both successful and failure behavior.
- Preserve error context when translating provider, subprocess, filesystem, or network failures; do not silently swallow exceptions except at deliberate resilience boundaries.

## Logging

**Framework:** Python `logging`; user-facing CLI output uses `print` where appropriate. TypeScript logging is localized to runtime/UI needs.

**Patterns:**
- Create module loggers with `logging.getLogger(__name__)`, as in `acp_adapter/permissions.py`.
- Use structured interpolation arguments (`logger.warning("... %s", value)`) rather than eagerly formatted strings.
- Keep secrets and credential values out of logs. Tests in `tests/conftest.py` actively remove credential-shaped environment variables.

## Comments

**When to Comment:**
- Comment non-obvious invariants, platform behavior, compatibility constraints, security boundaries, and synchronization requirements. `agent/retry_utils.py` and `tests/conftest.py` contain representative rationale comments.
- Do not narrate obvious statements. Put durable behavior contracts near the implementation and test the contract where practical.

**JSDoc/TSDoc:**
- Public TypeScript functions and interfaces commonly include concise JSDoc describing inputs, outputs, scoring, nullability, or invariants, as in `ui-tui/src/lib/fuzzy.ts`.
- Python public helpers use docstrings, especially CLI, retry, fixture, and integration boundaries.

## Function Design

**Size:** Keep functions focused on one transformation or boundary. Split provider routing, parsing, side effects, and presentation into separate helpers; `agent/account_usage.py` and `ui-tui/src/lib/fuzzy.ts` demonstrate this decomposition.

**Parameters:** Prefer typed, descriptive parameters. Use keyword-only parameters in Python when ambiguity is possible. In TypeScript, use a parameter object when a function has several related optional settings.

**Return Values:** Make absence and failure explicit (`None`/`null`, typed result objects, or exceptions). Preserve stable ordering and deterministic outputs when ranking or aggregating, as `fuzzyRank` does.

## Module Design

**Exports:** Export the smallest public surface needed by callers. Keep implementation helpers private with `_` in Python or non-exported declarations in TypeScript.

**Barrel Files:** No universal barrel-file convention is enforced. Follow the local workspace pattern and avoid adding broad re-export surfaces unless an existing package already uses one.

---

*Convention analysis: 2026-08-14*
