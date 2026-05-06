<!-- Generated: 2026-05-01 | Updated: 2026-05-01 -->

# AGENTS.md - repodocs

**Generated:** 2026-05-01 | **Updated:** 2026-05-01 | **Commit:** 69f8eaa | **Branch:** main

## Overview

Go CLI/library that extracts documentation from websites, Git repos, sitemaps, pkg.go.dev, docs.rs, wikis, and `llms.txt` into Markdown. Core architecture: detector → strategy → fetch/render → convert → write, with optional LLM metadata enrichment and sync-state tracking.

## Structure

```text
repodocs/
├── cmd/repodocs/        # Cobra CLI entrypoint; all commands and flags in main.go
├── configs/             # Config template copied on install
├── examples/manifests/  # Sample multi-source manifests
├── internal/
│   ├── app/             # URL detection + orchestrator
│   ├── cache/           # Badger-backed cache
│   ├── config/          # YAML/env/CLI config model + validation
│   ├── converter/       # HTML/Markdown/plaintext conversion pipeline
│   ├── domain/          # Interfaces, models, sentinel errors
│   ├── fetcher/         # Stealth HTTP client + retry + transport
│   ├── git/             # Thin go-git wrapper for DI/testability
│   ├── llm/             # Provider factory + resilience wrappers + metadata
│   ├── manifest/        # YAML/JSON manifest loading
│   ├── output/          # Markdown writer + metadata collector
│   ├── renderer/        # Rod/Chromium renderer + tab pool
│   ├── state/           # Incremental sync state manager
│   ├── strategies/      # Extraction strategies + shared Dependencies
│   ├── tui/             # Bubble Tea/Huh config editor
│   └── utils/           # URL/fs/logger/worker pool helpers
├── pkg/version/         # Build-time version info
├── scripts/             # Release automation
└── tests/               # External unit/integration/e2e/benchmark suites
```

## Flow

```text
URL or manifest
  → internal/app.DetectStrategy / RunManifest
  → internal/strategies/*
  → fetcher and/or renderer
  → converter pipeline
  → output writer
  → optional metadata collector / LLM enhancer / sync state
```

Detection order: `LLMS → PkgGo → DocsRS → Sitemap → Wiki → GitHubPages → Git → Crawler`

## Commands

```bash
make build        # CGO_ENABLED=0 build → ./build/repodocs
make test         # go test -v -race -short ./...
make test-all     # full go test -v -race ./...
make coverage     # coverage.out + ./coverage/coverage.html
make lint         # gofmt -s -w . + golangci-lint run ./...
make deps         # go mod download && go mod tidy
make install      # install binary + default config template
make release-dry  # goreleaser snapshot build
./scripts/release.sh

go test ./tests/unit/...
go test ./tests/integration/...
go test ./tests/e2e/...
go test ./tests/benchmark/... -bench=.
```

## Where to Look

| Task | Location | Notes |
|------|----------|-------|
| Add new source type | `internal/strategies/` + `internal/app/detector.go` | Implement strategy, register constructor, preserve detection order |
| Change composition/DI | `internal/app/orchestrator.go` + `internal/strategies/strategy.go` | Orchestrator builds options; `NewDependencies` wires shared services |
| Modify CLI behavior | `cmd/repodocs/main.go` | Root command, `doctor`, `version`, `config *`, `--manifest` path |
| Change config model/defaults | `internal/config/` | `config.go`, `defaults.go`, `loader.go` |
| HTML → Markdown issues | `internal/converter/` | Encoding → readability → sanitizer → markdown |
| JS rendering issues | `internal/renderer/` | Rod launcher, tab pool, stealth, SPA detection |
| Caching / sync | `internal/cache/`, `internal/state/` | Badger cache and incremental state are separate concerns |
| LLM provider or metadata changes | `internal/llm/` | Provider factory, wrappers, metadata prompt |
| Test helpers / mocks | `tests/testutil/`, `tests/mocks/` | Shared factories vs generated mocks |

## Conventions

- Imports: stdlib / external / internal, blank-line separated.
- Interfaces live in `internal/domain`; implementations live elsewhere.
- Public APIs: `context.Context` first, `error` last.
- Errors: wrap with context; sentinel errors in `internal/domain/errors.go`.
- Constructors: `NewX(...)`; option structs common (`ClientOptions`, `ProviderConfig`, `OrchestratorOptions`).
- Logging: `zerolog` via `internal/utils/logger.go`.
- CLI output dir: auto-derived from URL unless `-o/--output` was explicitly set.

## Project-Specific Rules

- `cmd/` is the only entrypoint; do not import from `cmd/` inside `internal/`.
- Strategy-specific behavior belongs in `internal/strategies/*`, not in `internal/app/orchestrator.go`.
- Avoid direct `rod` usage outside `internal/renderer`.
- New `nolint` directives are discouraged; existing documented exception is `internal/converter/encoding.go`.
- Deprecated config/model fields still exist; avoid using `LLMConfig.MaxRetries`, `Metadata`, `MetadataIndex`, `DocumentMetadata` in new code.
- `_ = err` appears in some tests as “no panic” assertions; avoid it in production paths.

## Tooling Notes

- `.golangci.yml`: only `govet` and `misspell`; tests skipped; explicit exclude for `internal/converter/encoding.go` ineffectual assignment.
- CI (`.github/workflows/ci.yml`): Linux + Windows, overall coverage plus per-package thresholds.
- Release (`.github/workflows/release.yml`): tag push `v*` → GoReleaser.
- Browser dependency: Chrome/Chromium optional unless JS rendering is used.

## CI Coverage Thresholds

| Package | Threshold | Package | Threshold |
|---------|-----------|---------|-----------|
| `internal/domain` | 85% | `internal/strategies` | 34% |
| `internal/converter` | 85% | `internal/app` | 48% |
| `internal/output` | 80% | `cmd/repodocs` | 48% |
| `internal/git` | 80% | `internal/config` | 62% |
| `internal/state` | 95%* | `internal/cache` | 75% |
| `internal/llm` | 70%* | `internal/fetcher` | 70% |
| `internal/renderer` | 28% | `internal/utils` | 90%* |

*Updated since CI thresholds were set; actual coverage may differ. Edit thresholds in `.github/workflows/ci.yml`.

## Known Bugs

All 6 documented issues in `bugs.md` have been **resolved** as of 2026-04-07. Regression tests added:

| # | Bug | Regression Test |
|---|-----|-----------------|
| 1 | Rate limit token consumed before CB check | `TestRateLimitedProvider_CircuitOpenPreservesTokens` |
| 2 | JitterFactor always 0.0 | `TestRetrier_JitterFactorFromConfig`, `TestCalculateBackoff_UsesConfigJitter` |
| 3 | Half-open allows unlimited requests | `TestCircuitBreaker_HalfOpenLimitsRequests` |
| 4 | Retry-After header parsed but never respected | `TestRetrier_RespectsRetryAfterHeader` |
| 5 | Retries don't consume rate limit tokens | `TestRateLimitedProvider_RetriesConsumeTokens` |
| 6 | Validate() doesn't validate rate limit config | `TestConfig_Validate_RateLimitFields` |

Additional fix: exported `CalculateBackoff` in `internal/llm/retry.go` now uses `cfg.JitterFactor` instead of hardcoded `0.1`.

## Complexity Hotspots

| File | Why it matters |
|------|----------------|
| `internal/strategies/docsrs_renderer.go` | Rustdoc JSON → Markdown renderer; large signature/type formatting logic |
| `internal/strategies/github_pages.go` | Discovery + SPA fallback + deduplication |
| `internal/strategies/docsrs_types.go` | Large Rustdoc schema model |
| `internal/utils/url.go` | Cache-key-critical URL normalization |
| `internal/tui/forms.go` | Dense config UI state/field definitions |
| `internal/app/orchestrator.go` | Main execution path for single URL + manifest runs |
| `internal/strategies/git/strategy_test.go` | ~1900 lines; tests 5 components (Strategy, Archive, Clone, Processor, Parser) |
| `internal/strategies/git_strategy_test.go` | 1530 lines; duplicates git/strategy_test.go coverage |
| `internal/renderer/renderer_test.go` | 1133 lines; browser-dependent tests with heavy setup |

## Project Scale

- 276 Go files (~100k lines), 174 test files
- 14 internal packages, 10 extraction strategies
- Externalized tests: `tests/unit/`, `tests/integration/`, `tests/e2e/`


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->

<!-- gitnexus:start -->
# GitNexus — Code Intelligence

This project is indexed by GitNexus as **repodocs-go** (9636 symbols, 24639 relationships, 292 execution flows). Use the GitNexus MCP tools to understand code, assess impact, and navigate safely.

> If any GitNexus tool warns the index is stale, run `npx gitnexus analyze` in terminal first.

## Always Do

- **MUST run impact analysis before editing any symbol.** Before modifying a function, class, or method, run `gitnexus_impact({target: "symbolName", direction: "upstream"})` and report the blast radius (direct callers, affected processes, risk level) to the user.
- **MUST run `gitnexus_detect_changes()` before committing** to verify your changes only affect expected symbols and execution flows.
- **MUST warn the user** if impact analysis returns HIGH or CRITICAL risk before proceeding with edits.
- When exploring unfamiliar code, use `gitnexus_query({query: "concept"})` to find execution flows instead of grepping. It returns process-grouped results ranked by relevance.
- When you need full context on a specific symbol — callers, callees, which execution flows it participates in — use `gitnexus_context({name: "symbolName"})`.

## Never Do

- NEVER edit a function, class, or method without first running `gitnexus_impact` on it.
- NEVER ignore HIGH or CRITICAL risk warnings from impact analysis.
- NEVER rename symbols with find-and-replace — use `gitnexus_rename` which understands the call graph.
- NEVER commit changes without running `gitnexus_detect_changes()` to check affected scope.

## Resources

| Resource | Use for |
|----------|---------|
| `gitnexus://repo/repodocs-go/context` | Codebase overview, check index freshness |
| `gitnexus://repo/repodocs-go/clusters` | All functional areas |
| `gitnexus://repo/repodocs-go/processes` | All execution flows |
| `gitnexus://repo/repodocs-go/process/{name}` | Step-by-step execution trace |

## CLI

| Task | Read this skill file |
|------|---------------------|
| Understand architecture / "How does X work?" | `.claude/skills/gitnexus/gitnexus-exploring/SKILL.md` |
| Blast radius / "What breaks if I change X?" | `.claude/skills/gitnexus/gitnexus-impact-analysis/SKILL.md` |
| Trace bugs / "Why is X failing?" | `.claude/skills/gitnexus/gitnexus-debugging/SKILL.md` |
| Rename / extract / split / refactor | `.claude/skills/gitnexus/gitnexus-refactoring/SKILL.md` |
| Tools, resources, schema reference | `.claude/skills/gitnexus/gitnexus-guide/SKILL.md` |
| Index, status, clean, wiki CLI commands | `.claude/skills/gitnexus/gitnexus-cli/SKILL.md` |

<!-- gitnexus:end -->
