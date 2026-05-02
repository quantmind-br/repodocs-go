<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-05-01 | Updated: 2026-05-01 -->

# tests/

Externalized test suites: unit, integration, e2e, benchmark. Black-box `package X_test` pattern. 174 test files total.

## Structure

```text
tests/
├── unit/              # Black-box tests mirroring internal/
│   ├── strategies/    # Strategy tests (also in internal/strategies/)
│   ├── llm/           # Provider, retry, circuit breaker tests
│   ├── app/           # Orchestrator tests
│   └── ...            # One subdir per internal package
├── integration/       # Cross-component; package `integration`
│   ├── strategies/    # Real HTTP/git tests
│   ├── llm/           # Live provider tests (skipped by default)
│   └── renderer/      # Browser tests
├── e2e/               # CLI workflow tests
├── benchmark/         # Performance benchmarks
├── testutil/          # Preferred shared helpers (PREFERRED)
│   ├── documents.go   # Document factories
│   ├── http.go        # TestServer wrapper
│   ├── temp.go        # Temp dir with auto-cleanup
│   ├── cache.go       # In-memory BadgerDB
│   └── strategies.go  # NewTestDependencies, NewMinimalDependencies
├── mocks/             # Generated gomock mocks
├── helpers/           # Legacy helpers (deprecated)
├── fixtures/          # Static binary/text fixtures
│   ├── docsrs/json/   # Rustdoc JSON samples
│   ├── git/           # sample-repo.tar.gz
│   ├── llms/          # llms.txt samples
│   └── pkggo/         # pkg.go.dev HTML
└── testdata/          # Configs, golden files, HTML fixtures
```

## Commands

```bash
go test ./tests/unit/...
go test ./tests/integration/...
go test ./tests/e2e/...
go test ./tests/benchmark/... -bench=.
go test ./... -short
```

## Where to Look

| Task | Location | Notes |
|------|----------|-------|
| Write unit tests | `tests/unit/<package>/` | Use `package X_test` (black-box) |
| Write integration tests | `tests/integration/<area>/` | Use `package integration`; skip with `testing.Short()` |
| Create test documents | `testutil/documents.go` | `NewDocument()`, `NewHTMLDocument()` |
| Mock HTTP calls | `testutil/http.go` | `NewTestServer(t)` with route handlers |
| Mock interfaces | `mocks/` | gomock mocks for all domain interfaces |
| Add fixtures | `fixtures/<strategy>/` | Per-strategy binary/text fixtures |
| Regenerate mocks | `mocks/README.md` | `go generate ./...` |

## Conventions

- Table-driven: `tests := []struct{...}` + `t.Run(name, func(t *testing.T){})`
- Test names: `Test<Subject>_<Scenario>`
- Auto-cleanup: `testutil.TempDir(t)`, `testutil.NewTestServer(t)`, `testutil.NewBadgerCache(t)` use `t.Cleanup()`
- Black-box: `package app_test` not `package app`
- Integration tests skip in `-short`: `if testing.Short() { t.Skip(...) }`
- Browser tests check availability: `if !renderer.IsAvailable() { t.Skip(...) }`

## Anti-Patterns

- Prefer `testutil/` over `helpers/` (legacy)
- Don't edit generated mocks (`mocks/*.go`)
- `_ = err` only in tests as "no panic" assertions
- Don't use real network in unit tests — use `testutil.NewTestServer()`
- Avoid `t.Parallel()` with shared `NewTestDependencies()` (Badger conflicts)


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
