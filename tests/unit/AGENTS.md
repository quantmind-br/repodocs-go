<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-29 -->

# tests/unit

Largest test subtree. External black-box coverage for most packages, plus a top-level `app_test` lane for exported behavior that does not fit a single package subdir.

## Structure

```text
tests/unit/
├── app/            # orchestrator + detector black-box tests
├── cache/
├── config/
├── converter/
├── domain/
├── fetcher/
├── git/
├── llm/
├── manifest/
├── output/
├── renderer/
├── state/
├── strategies/
│   └── git/        # strategy/git subpackage tests
└── utils/
```

## What lives here

- Package-style black-box tests: `cache_test`, `converter_test`, `renderer_test`, `strategies_test`, etc.
- Top-level files in `tests/unit/` often use `package app_test` and exercise mixed exported behavior across the repo.
- Deepest concentration is under `strategies/`, `renderer/`, `llm/`, and `converter/`.

## Commands

```bash
go test ./tests/unit/...
go test -run TestName ./tests/unit/...
go test ./tests/unit/strategies/...
go test ./tests/unit/renderer/...
```

## Dependencies

- Shared helpers: `tests/testutil/`
- Generated mocks: `tests/mocks/`
- Static inputs: `tests/testdata/`, `tests/fixtures/`

## Conventions

- Prefer exported-API testing (`package *_test`) over white-box internals.
- `t.Run(...)`, table-driven cases, and `t.TempDir()` are common.
- Package subdirectories usually mirror `internal/<pkg>/` names.
- For git strategy specifics, see `tests/unit/strategies/git/`.

## Where to Look

| Need | Location |
|------|----------|
| Orchestrator/detector behavior | `app/` and top-level `app_test` files |
| Strategy coverage | `strategies/` |
| Rendering/browser behavior | `renderer/` |
| LLM wrappers/providers | `llm/` |
| Converter pipeline behavior | `converter/` |

## Anti-Patterns

- Do not add new helpers here when they belong in `tests/testutil/`.
- Do not turn black-box suites into implementation-coupled tests without a reason.
- Avoid duplicating the same scenario across top-level `app_test` files and package-specific dirs.

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
