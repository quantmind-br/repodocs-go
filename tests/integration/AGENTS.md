<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-01 -->

# tests/integration

Cross-component and environment-sensitive tests. Most files use `package integration`; a few nested dirs use package-specific integration suites (`fetcher_test`, `renderer_test`, `llm_test`).

## Structure

```text
tests/integration/
├── cache/
├── config/
├── fetcher/
├── llm/
├── renderer/
├── strategies/
└── *.go            # orchestrator, manifest, writer, cache-flow, converter pipeline
```

## What lives here

- Realer multi-component paths: orchestrator + manifest + writer + cache flow.
- Strategy integrations under `strategies/`.
- Package-specific external integrations in nested dirs like `fetcher/`, `renderer/`, `llm/`.
- No build tags detected in this subtree; integration scope is directory-based.

## Commands

```bash
go test ./tests/integration/...
go test -run TestName ./tests/integration/...
go test ./tests/integration/strategies/...
```

## Shared Inputs

- `tests/testutil/` for reusable servers/loggers/temp dirs.
- `tests/testdata/` and `tests/fixtures/` for deterministic fixtures.
- Some tests exercise network/browser behavior more directly than unit suites.

## Where to Look

| Need | Location |
|------|----------|
| Full orchestrator flows | `orchestrator_test.go`, `manifest_test.go` |
| Cross-package cache behavior | `cache_flow_test.go`, `cache/` |
| Renderer/fetcher/llm integration | nested `renderer/`, `fetcher/`, `llm/` dirs |
| Source-specific integrations | `strategies/` |

## Conventions

- Keep assertions at behavior boundaries; leave fine-grained edge cases to unit tests.
- Reuse `tests/testutil/` instead of ad hoc setup.
- Prefer this tree when a test needs multiple concrete packages working together.

## Anti-Patterns

- Do not claim build-tag isolation here; current evidence shows plain directory/package separation.
- Do not move fast unit-style edge cases here just because they touch two files.
- Avoid brittle assertions on timing/network details unless the test is specifically about that behavior.

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
