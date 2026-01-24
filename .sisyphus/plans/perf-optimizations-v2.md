# Performance Optimizations V2 - Implementation Plan

## Context

### Original Request
Implement 4 performance optimizations from validated spec file (shotgun-prompt-20260124-161517_response.md):
- perf-001: Parallelize Manifest Source Processing (HIGH priority)
- perf-002: Batch Sitemap Index Processing (MEDIUM priority)
- perf-003: Reduce Scroll Sleep Duration (LOW priority)
- perf-005: Add Configurable Max File Size (LOW priority)

### Interview Summary
**Key Decisions**:
- Full parallelism OK for manifest sources (no ordering requirements)
- Batched processing for sitemap (global lastmod sorting NOT required)
- Max file size configurable via config file only (no CLI flag)
- Project has excellent test coverage and infrastructure

**Research Findings**:
- `utils.ParallelForEach` pattern already used in 5+ strategies
- `RunManifest` (orchestrator.go:246-311) processes sources sequentially
- `processSitemapIndex` (sitemap.go:112-149) collects all URLs, sorts, then processes
- `scrollToEnd` (rod.go:222-259) uses fixed 500ms sleep, max 10 iterations
- Git processor (processor.go:117) has hardcoded 10MB limit

### Metis Review
**Identified Gaps** (addressed):
- Concurrency safety in RunManifest: `ParallelForEach` handles goroutines safely
- Error handling semantics: Use existing `ContinueOnError` flag + aggregate errors
- Stable height rule for scroll: 2 consecutive equal heights = stable
- Config schema for max_file_size: Add to OutputConfig or new GitConfig section

---

## Work Objectives

### Core Objective
Implement 4 performance optimizations to improve manifest execution time, sitemap processing responsiveness, scroll performance, and memory safety with configurable file limits.

### Concrete Deliverables
- Modified `internal/app/orchestrator.go` with parallel manifest source processing
- Modified `internal/strategies/sitemap.go` with batched sitemap index processing
- Modified `internal/renderer/rod.go` with reduced scroll sleep and early exit
- Modified `internal/strategies/git/processor.go` + `internal/config/config.go` with configurable max file size

### Definition of Done
- [x] All existing tests pass: `make test`
- [x] New tests added for each optimization
- [x] Performance improvements verifiable (execution time, responsiveness)
- [x] No breaking changes to public API

### Must Have
- Preserve existing error handling semantics
- Maintain backward compatibility (defaults match current behavior)
- Follow existing code patterns (ParallelForEach, config structure)

### Must NOT Have (Guardrails)
- NO new CLI flags (config file only for perf-005)
- NO changes to public interfaces or exported types
- NO modifications outside specified files + their test files
- NO suppressing errors silently
- NO changing default behaviors (10MB file limit, 500ms becomes 300ms default)
- NO breaking existing sitemap `--limit` flag behavior

---

## Verification Strategy (MANDATORY)

### Test Decision
- **Infrastructure exists**: YES (`make test`, `make test-integration`)
- **User wants tests**: TDD
- **Framework**: Go testing + testify (assert/require)

### TDD Pattern for Each Task

**Task Structure:**
1. **RED**: Write failing test first
   - Test file: `tests/unit/<package>/<file>_test.go` or inline `*_test.go`
   - Test command: `go test -v -run TestName ./internal/<package>/...`
   - Expected: FAIL (test exists, implementation doesn't)
2. **GREEN**: Implement minimum code to pass
   - Command: `go test -v -run TestName ./internal/<package>/...`
   - Expected: PASS
3. **REFACTOR**: Clean up while keeping green
   - Command: `make test`
   - Expected: PASS (all tests)

---

## Task Flow

```
perf-001 (HIGH priority, orchestrator)
         ↓
perf-002 (MEDIUM priority, sitemap)
         ↓
perf-003 (LOW priority, rod) ─┬─ (parallel with perf-005)
                              │
perf-005 (LOW priority, git) ─┘
```

## Parallelization

| Group | Tasks | Reason |
|-------|-------|--------|
| A | perf-003, perf-005 | Independent packages (renderer, git+config) |

| Task | Depends On | Reason |
|------|------------|--------|
| perf-002 | perf-001 | Similar parallel processing pattern to understand |
| perf-003, perf-005 | perf-002 | Can parallelize after core tasks done |

---

## TODOs

### Phase 1: High Priority

- [x] 1. perf-001: Parallelize Manifest Source Processing

  **What to do**:
  1. Write test for parallel manifest execution:
     - Test that multiple sources run concurrently (timing verification)
     - Test that ContinueOnError=true aggregates all errors
     - Test that ContinueOnError=false returns first error
     - Test that context cancellation stops all sources
  2. Refactor `RunManifest`:
     - Create worker function that processes single source
     - Use `utils.ParallelForEach` with configurable concurrency (default: 3)
     - Aggregate results maintaining source index for error reporting
     - Call `FlushMetadata` and `SaveState` AFTER all sources complete
  3. Add source identifier to all log messages for traceability

  **Must NOT do**:
  - Do NOT change the public signature of `RunManifest`
  - Do NOT change the `ManifestResult` struct
  - Do NOT add new config options for manifest concurrency (use existing `Concurrency.Workers`)
  - Do NOT remove ContinueOnError logic

  **Parallelizable**: NO (first task, establishes parallel pattern)

  **References**:

  **Pattern References**:
  - `internal/app/orchestrator.go:246-311` - Current RunManifest sequential loop
  - `internal/strategies/sitemap.go:196-263` - ParallelForEach usage example
  - `internal/strategies/llms.go:107-108` - ParallelForEach with opts.Concurrency
  - `internal/utils/workerpool.go:204-254` - ParallelForEach implementation

  **API/Type References**:
  - `internal/app/orchestrator.go:ManifestResult` - Result struct (Source, Error, Duration)
  - `internal/manifest/manifest.go` - Config.Sources structure

  **Test References**:
  - `tests/unit/app/orchestrator_test.go` - Existing orchestrator tests
  - `internal/utils/workerpool_test.go` - ParallelForEach test patterns

  **Acceptance Criteria**:

  **TDD:**
   - [x] Test file created: `tests/unit/app/orchestrator_parallel_manifest_test.go`
   - [x] Test covers: concurrent execution, error aggregation, context cancellation
   - [x] `go test -v -run TestRunManifest ./internal/app/...` → PASS

   **Manual Execution Verification:**
   - [x] Create manifest with 3 slow sources (rate-limited sites)
   - [x] Time execution before: ~Sum(source_times)
   - [x] Time execution after: ~Max(source_times)
   - [x] `make test` → all tests pass

  **Commit**: YES
  - Message: `perf(orchestrator): parallelize manifest source processing`
  - Files: `internal/app/orchestrator.go`, test files
  - Pre-commit: `make test`

---

### Phase 2: Medium Priority

- [x] 2. perf-002: Batch Sitemap Index Processing

  **What to do**:
  1. Write test for batched processing:
     - Test that URLs from first nested sitemap are processed before fetching second
     - Test that total limit is respected across all batches
     - Test that progress is tracked across batches
     - Test that errors in one nested sitemap don't block others
  2. Refactor `processSitemapIndex`:
     - Remove global `allURLs` collection
     - Process each nested sitemap's URLs immediately after collecting
     - Track total processed count for limit enforcement
     - Maintain progress bar across all batches
  3. Remove global `sortURLsByLastMod` call (not needed with batched approach)

  **Must NOT do**:
  - Do NOT change the `--limit` flag behavior (still limits total pages)
  - Do NOT change the public interface of `SitemapStrategy`
  - Do NOT add streaming complexity (channels, etc.)
  - Do NOT change individual sitemap processing logic

  **Parallelizable**: NO (sequential after perf-001)

  **References**:

  **Pattern References**:
  - `internal/strategies/sitemap.go:112-149` - Current processSitemapIndex
  - `internal/strategies/sitemap.go:151-194` - collectURLsFromSitemap
  - `internal/strategies/sitemap.go:196-263` - processURLs (reuse this)

  **API/Type References**:
  - `internal/domain/sitemap.go:SitemapURL` - URL struct with Loc, LastMod
  - `internal/strategies/common.go:Options` - Limit field

  **Test References**:
  - `tests/unit/strategies/sitemap_strategy_test.go` - Existing sitemap tests

  **Acceptance Criteria**:

  **TDD:**
   - [x] Test file created: `tests/unit/strategies/sitemap_batch_test.go`
   - [x] Test covers: batch processing order, limit across batches, error isolation
   - [x] `go test -v -run TestSitemapBatch ./internal/strategies/...` → PASS

   **Manual Execution Verification:**
   - [x] Test with large sitemap index (10+ nested sitemaps)
   - [x] Verify processing starts immediately (not after collecting all)
   - [x] `make test` → all tests pass

  **Commit**: YES
  - Message: `perf(sitemap): batch sitemap index processing for faster start`
  - Files: `internal/strategies/sitemap.go`, test files
  - Pre-commit: `make test`

---

### Phase 3: Low Priority (Parallelizable)

- [x] 3. perf-003: Reduce Scroll Sleep Duration

  **What to do**:
  1. Write test for scroll optimization:
     - Test that early exit on stable height works (2 consecutive equal heights)
     - Test that reduced sleep doesn't break lazy loading detection
     - Test max iterations still respected
  2. Refactor `scrollToEnd`:
     - Reduce `time.Sleep` from 500ms to 300ms
     - Add early exit condition: if height unchanged for 2 consecutive checks, exit
     - Keep max 10 iterations as safety limit

  **Must NOT do**:
  - Do NOT use `WaitRequestIdle` (fragile with persistent connections)
  - Do NOT add configuration for scroll timing
  - Do NOT change the scroll-back-to-top behavior

  **Parallelizable**: YES (with perf-005)

  **References**:

  **Pattern References**:
  - `internal/renderer/rod.go:222-259` - Current scrollToEnd implementation

  **Test References**:
  - `tests/unit/renderer/` - Existing renderer tests

  **Acceptance Criteria**:

  **TDD:**
   - [x] Test file created: `tests/unit/renderer/scroll_test.go`
   - [x] Test covers: early exit on stable height, max iterations, basic scroll
   - [x] `go test -v -run TestScroll ./internal/renderer/...` → PASS

   **Manual Execution Verification:**
   - [x] Test with lazy-loading site (e.g., infinite scroll)
   - [x] Verify content still loads correctly
   - [x] Measure ~30% reduction in scroll time
   - [x] `make test` → all tests pass

  **Commit**: YES
  - Message: `perf(renderer): reduce scroll sleep for faster page processing`
  - Files: `internal/renderer/rod.go`, test files
  - Pre-commit: `make test`

---

- [x] 4. perf-005: Add Configurable Max File Size

  **What to do**:
  1. Add config field to `internal/config/config.go`:
     - Add `GitConfig` struct with `MaxFileSize` field (type: `string` for human-readable, e.g., "2MB")
     - Add `Git` field to main `Config` struct
     - Add default value: `10MB` (matches current hardcoded value)
     - Add validation in `Config.Validate()`
  2. Update `ProcessOptions` to include parsed max file size
  3. Update `ProcessFile` to use config value instead of hardcoded 10MB
  4. Add size parsing helper (e.g., "2MB" → 2*1024*1024)

  **Must NOT do**:
  - Do NOT add CLI flags for this setting
  - Do NOT change the default behavior (still 10MB)
  - Do NOT add complex size parsing (support MB, GB only)

  **Parallelizable**: YES (with perf-003)

  **References**:

  **Pattern References**:
  - `internal/config/config.go` - Existing config struct patterns
  - `internal/strategies/git/processor.go:112-120` - Current hardcoded check
  - `internal/strategies/git/strategy.go` - How config flows to processor

  **API/Type References**:
  - `internal/config/config.go:Config` - Main config struct
  - `internal/strategies/git/processor.go:ProcessOptions` - Options struct

  **Test References**:
  - `tests/unit/git/processor_test.go` - Existing processor tests
  - `tests/unit/config/` - Config tests

  **Acceptance Criteria**:

  **TDD:**
   - [x] Test file created: `tests/unit/config/git_config_test.go`
   - [x] Test file updated: `tests/unit/git/processor_size_test.go`
   - [x] Test covers: default 10MB, custom 2MB, invalid sizes, validation
   - [x] `go test -v ./internal/config/... ./internal/strategies/git/...` → PASS

   **Manual Execution Verification:**
   - [x] Set `git.max_file_size: "2MB"` in config
   - [x] Run git strategy, verify 5MB files are skipped
   - [x] `make test` → all tests pass

  **Commit**: YES
  - Message: `feat(config): add configurable max file size for git strategy`
  - Files: `internal/config/config.go`, `internal/strategies/git/processor.go`, test files
  - Pre-commit: `make test`

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 1 | `perf(orchestrator): parallelize manifest source processing` | orchestrator.go, tests | `make test` |
| 2 | `perf(sitemap): batch sitemap index processing for faster start` | sitemap.go, tests | `make test` |
| 3 | `perf(renderer): reduce scroll sleep for faster page processing` | rod.go, tests | `make test` |
| 4 | `feat(config): add configurable max file size for git strategy` | config.go, processor.go, tests | `make test` |

---

## Success Criteria

### Verification Commands
```bash
make test              # All unit tests pass
make test-integration  # Integration tests pass
make lint              # No linting errors
```

### Final Checklist
- [x] All 4 optimizations implemented
- [x] All existing tests pass
- [x] New tests added for each optimization
- [x] perf-001: Manifest execution ~2-5x faster for multi-source
- [x] perf-002: Sitemap processing starts immediately
- [x] perf-003: ~30% reduction in scroll time per page
- [x] perf-005: Max file size configurable via config.yaml
