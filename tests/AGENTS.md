<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-01 -->

# tests/

External test tree. Heavy use of black-box `package *_test` suites plus shared mocks/fixtures/utilities.

## Structure

```text
tests/
├── unit/              # largest subtree; mirrors internal packages and shared app-level black-box tests
├── integration/       # cross-component runs; mostly package `integration`
├── e2e/               # CLI workflow tests (`package e2e`)
├── benchmark/         # performance comparisons (currently git clone vs archive)
├── mocks/             # generated gomock + one legacy testify mock
├── testutil/          # preferred shared helpers for new tests
├── helpers/           # legacy helpers; keep for compatibility
├── fixtures/          # static binary/text fixtures
└── testdata/          # configs, golden files, additional fixtures
```

Child docs: [unit/](unit/AGENTS.md), [integration/](integration/AGENTS.md), [mocks/](mocks/AGENTS.md), [testutil/](testutil/AGENTS.md), [helpers/](helpers/AGENTS.md)

## Commands

```bash
go test ./tests/unit/...
go test ./tests/integration/...
go test ./tests/e2e/...
go test ./tests/benchmark/... -bench=.
go test ./... -short
```

No `go:build integration` tags were found under `tests/`; integration tests are organized by directory/package, not build tags.

## Shared Test Infrastructure

- `tests/testutil/`: preferred factories/assertions/temp dirs/servers.
- `tests/mocks/`: generated mocks from `internal/domain/interfaces.go` and `internal/git/interface.go`.
- `tests/helpers/`: older helper package; prefer `testutil` for new code.
- `tests/fixtures/` and `tests/testdata/`: reusable HTML/XML/archive/config/golden inputs.

## Patterns

- Black-box package naming: `fetcher_test`, `renderer_test`, `llm_test`, etc.
- Some top-level unit files use `package app_test` as a general exported-API harness.
- Table-driven tests and `t.TempDir()` are common throughout the tree.
- Benchmarks are isolated in `tests/benchmark/`, not mixed into normal package dirs.

## Anti-Patterns

- Prefer `tests/testutil/` over `tests/helpers/` for new code.
- Do not edit generated files in `tests/mocks/` by hand.
- Keep package-boundary intent: external tests here, white-box internal tests beside implementation when needed.


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
