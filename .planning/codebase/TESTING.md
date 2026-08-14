# Testing Patterns

**Analysis Date:** 2026-08-14

## Test Framework

**Runner:**
- Python uses pytest 9.1.1 and pytest-asyncio 1.3.0 from the `dev` extra in `pyproject.toml`.
- Python discovery is restricted to `tests` by `[tool.pytest.ini_options]`; normal runs exclude `integration` through `addopts`.
- TypeScript uses Vitest 4.1.10. Workspace configs include `ui-tui/vitest.config.ts`, `web/vitest.config.ts`, `tests-js/vitest.config.ts`, and `apps/desktop/vitest.config.ts`.

**Assertion Library:**
- Python uses native `assert` statements with pytest fixtures and helpers.
- TypeScript uses Vitest's `expect`, `describe`, and `it` APIs.

**Run Commands:**
```bash
pytest tests/ -v                 # Direct Python suite
scripts/run_tests.sh             # Canonical isolated Python runner
npm run --ws check               # Workspace checks from the root
npm run --prefix ui-tui test    # TUI Vitest suite
npm run --prefix web test       # Web Vitest suite
npm run --prefix tests-js test  # Root TypeScript tests
```

## Test File Organization

**Location:**
- Core Python tests live under `tests/`, grouped by subsystem (`tests/tools`, `tests/gateway`, `tests/cli`, `tests/e2e`, and others).
- Skill tests are colocated under each skill, for example `skills/productivity/docx/tests/test_docx_skill.py`.
- TypeScript tests are generally colocated with source (`ui-tui/src/lib/*.test.ts`, `apps/shared/src/*.test.ts`) or in `src/__tests__` for broader UI/state tests. Web tests are under `web/src`.

**Naming:**
- Use descriptive behavior-oriented names: `test_fetch_account_usage_codex`, `test_strict_fails_on_unfilled`, and `it('returns all items in original order for an empty query')`.

**Structure:**
```
tests/<subsystem>/test_<behavior>.py
skills/<category>/<skill>/tests/test_<skill>_skill.py
ui-tui/src/<module>.test.ts[x]
ui-tui/src/__tests__/<feature>.test.ts[x]
web/src/<component-or-page>.test.tsx
tests-js/<feature>.test.ts
```

## Test Structure

**Suite Organization:**
```typescript
import { describe, expect, it } from 'vitest'

describe('fuzzyScore', () => {
  it('returns null when characters are out of order or absent', () => {
    expect(fuzzyScore('gpt-4o', 'o4g')).toBeNull()
  })
})
```

```python
def test_fetch_account_usage_codex(monkeypatch):
    monkeypatch.setattr('agent.account_usage.httpx.Client', lambda timeout=15.0: _Client(...))
    snapshot = fetch_account_usage('openai-codex')
    assert snapshot is not None
```

**Patterns:**
- Group related behaviors with `describe` in Vitest and `class Test...` or module-level `test_...` functions in pytest.
- Test observable contracts: exact values, ordering, emitted text, nullability, persistence, and error status.
- Prefer deterministic fixtures generated in the test, as in `skills/productivity/docx/tests/test_docx_skill.py`; avoid live network calls in normal tests.
- Use parametrization or focused repeated cases when platform/provider variants share a contract; use registered markers for tests requiring special environments.

## Mocking

**Framework:** pytest `monkeypatch` and hand-written fakes; Vitest `vi.fn`, `vi.mock`, `vi.stubGlobal`, fake timers, and `vi.waitFor`.

**Patterns:**
```typescript
beforeEach(() => {
  vi.clearAllMocks()
  vi.useFakeTimers()
})

afterEach(() => {
  vi.useRealTimers()
  vi.unstubAllGlobals()
})
```

```python
monkeypatch.setattr('agent.account_usage.resolve_runtime_provider', lambda *args, **kwargs: {...})
monkeypatch.setattr('agent.account_usage.httpx.Client', lambda timeout=10.0: _RoutingClient(payloads))
```

**What to Mock:**
- Mock network clients, provider SDKs, WebSocket/browser globals, timers, filesystem boundaries, credentials, and external subprocess inputs when testing local behavior. See `web/src/pages/ChatPage.test.tsx` and `tests/test_account_usage.py`.
- Build small fakes when protocol behavior matters, such as `_Response`, `_Client`, and `_RoutingClient` in `tests/test_account_usage.py`.

**What NOT to Mock:**
- Do not mock the pure function or value transformation under test. Do not make normal tests depend on live APIs or operator credentials.
- Keep integration tests explicitly marked (`integration`, `ssh`, or subsystem-specific markers) and excluded from default runs when they require external services.

## Fixtures and Factories

**Test Data:**
```python
@pytest.fixture(scope='module')
def workdir(tmp_path_factory) -> Path:
    return tmp_path_factory.mktemp('docxskill')
```

- Shared isolation and security fixtures are centralized in `tests/conftest.py`; subsystem fixtures live in directories such as `tests/tools/conftest.py` and `tests/gateway/conftest.py`.
- Use `tmp_path`/`tmp_path_factory` for filesystem state and generate files in the test. `test_docx_skill.py` creates PNG, JSON, and DOCX fixtures without checked-in binary fixtures.
- Use explicit small factory/fake classes for provider responses and protocol objects rather than broad mocks when response shape is part of the contract.

**Location:**
- Shared fixtures: `tests/conftest.py`.
- Subsystem fixtures: `tests/<subsystem>/conftest.py`.
- Skill-local fixtures/helpers: `skills/<category>/<skill>/tests/`.

## Coverage

**Requirements:** No numeric coverage threshold or coverage configuration is detected. CI focuses on test execution, lint/type checks, and targeted workflow lanes.

**View Coverage:** No repository-standard coverage command is configured. If coverage is needed for a focused investigation, use the relevant runner's standard coverage integration without committing generated reports.

## Test Types

**Unit Tests:**
- Predominant pattern: pure helper, parser, state, formatting, provider-routing, and UI behavior tests, with external boundaries mocked.

**Integration Tests:**
- Mark with pytest's `integration` marker or subsystem markers. CI can install provider extras and run selected integration/e2e lanes, but default pytest excludes integration tests.

**E2E Tests:**
- Python E2E tests live under `tests/e2e/`; Docker-specific tests live under `tests/docker/`. They use dedicated fixtures and workflow lanes rather than the default unit suite.
- Browser/UI behavior is primarily covered with Vitest fakes for WebSocket, terminal, storage, timers, and browser globals, as in `web/src/pages/ChatPage.test.tsx`.

## Common Patterns

**Async Testing:**
```typescript
await vi.waitFor(() => expect(FakeWebSocket.instances).toHaveLength(1))
await vi.advanceTimersByTimeAsync(ms)
```

- Python async tests use pytest-asyncio where required; preserve event-loop cleanup and use the repository fixtures rather than creating global loops.

**Error Testing:**
```typescript
expect(fuzzyScore('gpt-4o', 'xyz')).toBeNull()
expect(() => cacheManifests([exampleManifest])).not.toThrow()
```

- Assert exact failure values, exception status, CLI return codes, stderr/stdout payloads, and resilience behavior. `skills/productivity/docx/tests/test_docx_skill.py` verifies subprocess return code and structured JSON for strict failures.
- Run Python files through `scripts/run_tests.sh` when possible: CI uses fresh pytest subprocesses per file to prevent module-level state leakage, while `tests/conftest.py` removes credential-shaped environment variables and redirects `HERMES_HOME`.

---

*Testing analysis: 2026-08-14*
