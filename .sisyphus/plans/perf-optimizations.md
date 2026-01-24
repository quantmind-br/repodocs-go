# Performance Optimizations Implementation Plan

## Context

### Original Request
Implement 4 performance optimizations from the analyzed proposal file, in priority order with TDD approach.

### Interview Summary
**Key Discussions**:
- User confirmed ALL 4 optimizations should be implemented
- TDD approach selected (tests first, then implementation)
- Project has excellent existing test coverage (79.5% git, 87.1% converter, 89.1% renderer)

**Research Findings**:
- html-to-markdown v2 supports `ConvertNode(*html.Node)` - can use `sel.Get(0)` to extract node from goquery
- Current pipeline parses HTML 2-3 times per document (confirmed in pipeline.go)
- TabPool eagerly creates 5 tabs at startup (pool.go:31-39)
- Discovery probes are sequential for loop (github_pages.go:139-178)

### Metis Review
**Identified Gaps** (addressed):
- Output equivalence for perf-004: Add regression tests comparing before/after outputs
- File size threshold: Use existing 10MB constant, keep as internal constant (no new config flags)
- Parallel probe priority: Return highest-priority success, not first-completed
- Edge cases: Handle os.Stat failures, context cancellation, tab pool lifecycle

---

## Work Objectives

### Core Objective
Implement 4 performance optimizations to improve startup time, reduce CPU usage in conversion, and prevent OOM from large files.

### Concrete Deliverables
- Modified `internal/strategies/git/processor.go` with pre-read size check
- Modified `internal/converter/pipeline.go` and `markdown.go` eliminating redundant parsing
- Modified `internal/renderer/pool.go` with lazy tab creation
- Modified `internal/strategies/github_pages.go` with parallel probe discovery

### Definition of Done
- [ ] All existing tests pass: `make test`
- [ ] New tests added for each optimization
- [ ] No new public API or CLI changes
- [ ] Performance improvements verifiable (startup time, memory usage)

### Must Have
- TDD: Write failing test first, then implement
- Preserve existing behavior (output equivalence)
- Handle edge cases (stat failures, context cancellation)

### Must NOT Have (Guardrails)
- NO new public configuration flags or CLI options
- NO changes to public interfaces or exported types
- NO semantic changes to output (Markdown must be equivalent)
- NO modifications outside the 4 specified files + their test files
- NO suppressing errors silently

---

## Verification Strategy (MANDATORY)

### Test Decision
- **Infrastructure exists**: YES (`make test`, `make test-integration`)
- **User wants tests**: TDD
- **Framework**: Go testing + testify (assert/require)

### TDD Pattern for Each Task

**Task Structure:**
1. **RED**: Write failing test first
   - Test file: `tests/unit/<package>/<file>_test.go`
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
perf-006 (trivial, independent)
         ↓
perf-004 (high impact, independent)
         ↓
perf-002 (medium, independent)
         ↓
perf-001 (low priority, independent)
```

## Parallelization

| Group | Tasks | Reason |
|-------|-------|--------|
| N/A | All sequential | TDD requires focused implementation |

| Task | Depends On | Reason |
|------|------------|--------|
| All | None | Each optimization is in separate package |

---

## TODOs

### Phase 1: Critical Fixes

- [x] 1. perf-006: Pre-check File Size Before Reading

  **What to do**:
  1. Write test for file size limit behavior:
     - Test that files > 10MB are skipped without reading
     - Test that os.Stat errors are handled gracefully
     - Test normal files still process correctly
  2. Implement size check using `os.Stat()` before `os.ReadFile()`
  3. Verify existing tests still pass

  **Must NOT do**:
  - Do NOT change the 10MB threshold value
  - Do NOT add configuration options for the limit
  - Do NOT change error return behavior (still return nil for skipped files)

  **Parallelizable**: NO (first task, establishes pattern)

  **References**:

  **Pattern References**:
  - `internal/strategies/git/processor.go:112-120` - Current implementation to modify
  - `internal/strategies/git/processor.go:42-86` - Similar error handling patterns

  **Test References**:
  - `tests/unit/git/processor_test.go` - Existing processor tests (follow patterns)

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created: `tests/unit/git/processor_size_check_test.go`
  - [ ] Test covers: files > 10MB skipped, os.Stat errors handled, normal files work
  - [ ] `go test -v -run TestProcessFile ./internal/strategies/git/...` → PASS

  **Manual Execution Verification:**
  - [ ] Create temp 11MB file, run processor, verify no OOM
  - [ ] `make test` → all tests pass

  **Commit**: YES
  - Message: `perf(git): check file size before reading to prevent OOM`
  - Files: `internal/strategies/git/processor.go`, `tests/unit/git/processor_size_check_test.go`
  - Pre-commit: `make test`

---

### Phase 2: High-Impact Optimizations

- [x] 2. perf-004: Eliminate Redundant HTML Parsing in Converter

  **What to do**:
  1. Write regression test capturing current output:
     - Convert sample HTML with selector path
     - Convert sample HTML with readability path
     - Store expected Markdown output as golden files
  2. Refactor to use `ConvertNode(*html.Node)` instead of `ConvertString(string)`:
     - Extract `*html.Node` from goquery Selection using `sel.Get(0)`
     - Pass node directly to markdown converter
     - Avoid serializing DOM to string then reparsing
  3. Verify output equivalence with golden files

  **Must NOT do**:
  - Do NOT change the public API of Pipeline.Convert()
  - Do NOT change the output format or structure
  - Do NOT remove goquery dependency (still needed for sanitization)

  **Parallelizable**: NO (depends on understanding from task 1)

  **References**:

  **Pattern References**:
  - `internal/converter/pipeline.go:56-179` - Current Convert() implementation
  - `internal/converter/pipeline.go:115-127` - Selector path serialization
  - `internal/converter/pipeline.go:128-146` - Readability path re-parsing
  - `internal/converter/markdown.go` - Current ConvertString usage

  **API/Type References**:
  - html-to-markdown v2: `ConvertNode(*html.Node)` returns `([]byte, error)`
  - goquery: `sel.Get(0)` returns `*html.Node`

  **Test References**:
  - `tests/unit/converter/pipeline_test.go` - Existing pipeline tests
  - `tests/unit/converter/markdown_test.go` - Existing markdown tests

  **External References**:
  - html-to-markdown v2 docs: `ConvertNode` accepts `*html.Node` from `golang.org/x/net/html`

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created/updated: `tests/unit/converter/pipeline_equivalence_test.go`
  - [ ] Golden files created for selector and readability paths
  - [ ] `go test -v ./internal/converter/...` → PASS (output identical)

  **Manual Execution Verification:**
  - [ ] Run `make test` → all tests pass
  - [ ] Benchmark before/after with: `go test -bench=BenchmarkConvert ./internal/converter/...`

  **Commit**: YES
  - Message: `perf(converter): eliminate redundant HTML parsing using ConvertNode`
  - Files: `internal/converter/pipeline.go`, `internal/converter/markdown.go`, test files
  - Pre-commit: `make test`

---

- [x] 3. perf-002: Lazy Browser Tab Initialization

  **What to do**:
  1. Write test for lazy initialization behavior:
     - Test that NewTabPool returns immediately with zero tabs created
     - Test that Acquire() creates tab on-demand
     - Test that created count doesn't exceed maxTabs
     - Test that context cancellation during Acquire() works
  2. Refactor TabPool:
     - Add `created int` and `createMu sync.Mutex` fields
     - Remove eager creation loop from NewTabPool
     - Add lazy creation logic in Acquire()
  3. Verify existing renderer tests still pass

  **Must NOT do**:
  - Do NOT change the public interface (NewTabPool, Acquire, Release, Close)
  - Do NOT change maxTabs default (5)
  - Do NOT remove the cleanup in Release()

  **Parallelizable**: NO (sequential TDD)

  **References**:

  **Pattern References**:
  - `internal/renderer/pool.go:20-42` - Current NewTabPool with eager creation
  - `internal/renderer/pool.go:45-59` - Current Acquire() to modify
  - `internal/renderer/pool.go:62-81` - Release() pattern to preserve

  **Test References**:
  - `tests/unit/renderer/pool_test.go` - Existing pool tests

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created/updated: `tests/unit/renderer/pool_lazy_test.go`
  - [ ] Test covers: zero initial tabs, on-demand creation, max limit, context cancel
  - [ ] `go test -v ./internal/renderer/...` → PASS

  **Manual Execution Verification:**
  - [ ] Run `make test` → all tests pass
  - [ ] Run `make test-integration` → renderer integration tests pass

  **Commit**: YES
  - Message: `perf(renderer): lazy browser tab initialization for faster startup`
  - Files: `internal/renderer/pool.go`, test files
  - Pre-commit: `make test`

---

### Phase 3: Nice-to-Have Optimizations

- [ ] 4. perf-001: Parallelize GitHub Pages Discovery Probes

  **What to do**:
  1. Write test for parallel probe behavior:
     - Test that all probes are fired concurrently (mock fetcher)
     - Test that highest-priority successful probe is returned (not fastest)
     - Test that context cancellation stops all probes
     - Test that probe failures don't block other probes
  2. Refactor discoverViaHTTPProbes():
     - Fire all probes concurrently using goroutines
     - Collect results in channel with priority index
     - Return highest-priority (lowest index) successful result
  3. Verify existing GitHub Pages tests pass

  **Must NOT do**:
  - Do NOT change probe priority order (defined in GetDiscoveryProbes)
  - Do NOT add timeout per-probe (use context)
  - Do NOT change the fallback to browser crawl behavior

  **Parallelizable**: NO (final task)

  **References**:

  **Pattern References**:
  - `internal/strategies/github_pages.go:139-178` - Current sequential implementation
  - `internal/strategies/github_pages_discovery.go` - Probe definitions and priority
  - `internal/utils/parallel.go` - ParallelForEach pattern (reference only)

  **Test References**:
  - `tests/unit/strategies/github_pages_test.go` - Existing strategy tests

  **Acceptance Criteria**:

  **TDD:**
  - [ ] Test file created/updated: `tests/unit/strategies/github_pages_parallel_test.go`
  - [ ] Test covers: concurrent execution, priority ordering, context cancel, error isolation
  - [ ] `go test -v ./internal/strategies/...` → PASS

  **Manual Execution Verification:**
  - [ ] Run `make test` → all tests pass
  - [ ] Time a real run before/after (should see 1-2s improvement on cold start)

  **Commit**: YES
  - Message: `perf(github-pages): parallelize discovery probes for faster startup`
  - Files: `internal/strategies/github_pages.go`, test files
  - Pre-commit: `make test`

---

## Commit Strategy

| After Task | Message | Files | Verification |
|------------|---------|-------|--------------|
| 1 | `perf(git): check file size before reading to prevent OOM` | processor.go, tests | `make test` |
| 2 | `perf(converter): eliminate redundant HTML parsing using ConvertNode` | pipeline.go, markdown.go, tests | `make test` |
| 3 | `perf(renderer): lazy browser tab initialization for faster startup` | pool.go, tests | `make test` |
| 4 | `perf(github-pages): parallelize discovery probes for faster startup` | github_pages.go, tests | `make test` |

---

## Success Criteria

### Verification Commands
```bash
make test           # All unit tests pass
make test-integration  # Integration tests pass
make lint           # No linting errors
```

### Final Checklist
- [ ] All 4 optimizations implemented
- [ ] All existing tests pass
- [ ] New tests added for each optimization
- [ ] No new public API or config flags
- [ ] Markdown output equivalence verified (perf-004)
- [ ] Memory safety improved (perf-006)
- [ ] Startup time improved (perf-002, perf-001)
