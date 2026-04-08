# Verify & Clean Up Known Bugs (Rate Limit & Circuit Breaker)

## TL;DR

> **Quick Summary**: All 6 documented bugs in `bugs.md` are already fixed in the current codebase, but regression test coverage is inadequate. This plan adds targeted regression tests for each fix, applies a minor TDD fix to the exported `CalculateBackoff` function (hardcoded jitter), and updates documentation to reflect resolved status.
> 
> **Deliverables**:
> - 7 new regression test functions across 4 test files
> - 1 production code fix (2-line change to `CalculateBackoff` in `internal/llm/retry.go`)
> - Updated `bugs.md` with resolution status per bug
> - Updated `AGENTS.md` Known Bugs section
> 
> **Estimated Effort**: Short
> **Parallel Execution**: YES - 2 waves
> **Critical Path**: Wave 1 (all regression tests + fix) → Wave 2 (documentation) → Final Verification

---

## Context

### Original Request
User requested fixing all 6 known bugs documented in `bugs.md`. Investigation revealed all bugs are already corrected in the current source code, but `bugs.md` (dated 2026-03-29) was never updated.

### Interview Summary
**Key Discussions**:
- All 6 bugs verified as fixed by reading current source code and cross-referencing with bug descriptions
- User chose TDD approach for any new tests
- User chose to fix all 6 at once (all in same rate limit/circuit breaker subsystem)
- After discovering bugs are fixed, user chose "Verify + Clean" approach

**Research Findings**:
- `provider_wrapper.go:103-110` — CB check happens FIRST, token at :122 inside retry closure (Bugs 1, 5 fixed)
- `config.go:46` has `JitterFactor` field, `strategy.go:175` maps it, `retry.go:118` uses it (Bug 2 fixed)
- `circuit_breaker.go:100-104` — `halfOpenAllowed` counter limits requests (Bug 3 fixed)
- `fetcher/retry.go:104-111` — `max(backoff, RetryAfter)` implemented (Bug 4 fixed)
- `config.go:130-167` — full rate limit + circuit breaker validation (Bug 6 fixed)
- Exported `CalculateBackoff()` at `retry.go:188-199` still uses hardcoded `0.1` jitter — only residual issue

### Metis Review
**Identified Gaps** (addressed):
- Regression test coverage is INADEQUATE for 5 of 6 bugs — tests exist but don't verify the specific fix
- Bug 3 test must use serial calls (not goroutines) to avoid flaky tests
- Bug 4 test must verify calculated delay value, not wall-clock time
- Bug 2 test must compare JitterFactor=0 vs >0 baseline, not assert exact random values
- Must check existing `TestCalculateBackoff` assertions before applying fix
- Test file locations must be specified per test (internal vs external package)

---

## Work Objectives

### Core Objective
Verify all 6 bug fixes have adequate regression test coverage, fix the one residual code issue, and update documentation to reflect resolved status.

### Concrete Deliverables
- 7 new test functions: 1 regression test per bug + 1 test for CalculateBackoff fix
- 1 production code fix: `CalculateBackoff` in `internal/llm/retry.go:188-199`
- Updated `bugs.md` with "RESOLVED" status per bug entry
- Updated `AGENTS.md` Known Bugs section

### Definition of Done
- [ ] `make test` passes with zero failures
- [ ] `make lint` produces no new warnings
- [ ] `grep -c "RESOLVED" bugs.md` returns `6`
- [ ] All 7 new test functions pass individually via `go test -run TestName -v`

### Must Have
- One regression test per bug that would FAIL if the fix were reverted
- `CalculateBackoff` uses `cfg.JitterFactor` instead of hardcoded `0.1`
- All 6 bugs marked as RESOLVED in `bugs.md`

### Must NOT Have (Guardrails)
- NO modifications to existing test functions — only add new test functions
- NO refactoring of production code beyond the `CalculateBackoff` fix
- NO changes to CI coverage thresholds
- NO new test frameworks or mock libraries (use existing testify + manual mocks)
- NO integration tests — scope is unit regression tests only
- NO rewriting of `bugs.md` content or changing its language (Portuguese) — only append resolution status

---

## Verification Strategy

> **ZERO HUMAN INTERVENTION** — ALL verification is agent-executed. No exceptions.

### Test Decision
- **Infrastructure exists**: YES
- **Automated tests**: TDD for CalculateBackoff fix; regression tests for already-fixed bugs
- **Framework**: `go test` with `testify/assert` and `testify/require`
- **Pattern**: Write test → confirm it passes (for regression tests on already-fixed code); Write failing test → fix → green (for CalculateBackoff)

### QA Policy
Every task includes agent-executed QA scenarios.
Evidence saved to `.sisyphus/evidence/task-{N}-{scenario-slug}.{ext}`.

- **Tests**: Use Bash — `go test -run TestName -v ./path/...`
- **Full suite**: Use Bash — `make test`
- **Lint**: Use Bash — `make lint`

---

## Execution Strategy

### Parallel Execution Waves

```
Wave 1 (Start Immediately - all independent, MAX PARALLEL):
├── Task 1: Regression tests for Bugs 1+5 (provider_wrapper) [quick]
├── Task 2: Regression test for Bug 2 (JitterFactor/retry) [quick]
├── Task 3: Regression test for Bug 3 (half-open/circuit_breaker) [quick]
├── Task 4: Regression test for Bug 4 (Retry-After/fetcher) [quick]
├── Task 5: Regression test for Bug 6 (config validation) [quick]
└── Task 6: TDD fix for CalculateBackoff (test + fix) [quick]

Wave 2 (After Wave 1 - documentation):
└── Task 7: Update bugs.md + AGENTS.md [quick]

Wave FINAL (After ALL tasks — 4 parallel reviews, then user okay):
├── Task F1: Plan compliance audit (oracle)
├── Task F2: Code quality review (unspecified-high)
├── Task F3: Real manual QA (unspecified-high)
└── Task F4: Scope fidelity check (deep)
-> Present results -> Get explicit user okay
```

### Dependency Matrix

| Task | Depends On | Blocks | Wave |
|------|-----------|--------|------|
| 1 | — | 7, F1-F4 | 1 |
| 2 | — | 7, F1-F4 | 1 |
| 3 | — | 7, F1-F4 | 1 |
| 4 | — | 7, F1-F4 | 1 |
| 5 | — | 7, F1-F4 | 1 |
| 6 | — | 7, F1-F4 | 1 |
| 7 | 1-6 | F1-F4 | 2 |
| F1-F4 | 7 | — | FINAL |

### Agent Dispatch Summary

- **Wave 1**: **6** — T1-T6 → `quick`
- **Wave 2**: **1** — T7 → `quick`
- **FINAL**: **4** — F1 → `oracle`, F2 → `unspecified-high`, F3 → `unspecified-high`, F4 → `deep`

---

## TODOs

> Implementation + Test = ONE Task. Never separate.
> EVERY task MUST have: Recommended Agent Profile + Parallelization info + QA Scenarios.

- [ ] 1. Regression tests for Bugs 1 + 5 (provider_wrapper — token ordering + retry token consumption)

  **What to do**:
  - Add `TestRateLimitedProvider_CircuitOpenPreservesTokens` to `internal/llm/provider_wrapper_test.go`:
    - Create a `RateLimitedProvider` with a mock provider, rate limiter, and circuit breaker
    - Force circuit breaker to `StateOpen` by recording enough failures
    - Call `Complete()` — it should return `ErrLLMCircuitOpen`
    - Verify the mock provider was NOT called (proving token wasn't consumed and request wasn't attempted)
    - Key assertion: if the CB check were moved AFTER `rateLimiter.Wait()`, the rate limiter would be drained on open-circuit calls
  - Add `TestRateLimitedProvider_RetriesConsumeTokens` to the same file:
    - Create a `RateLimitedProvider` with a mock that fails N times then succeeds
    - Configure `MaxRetries >= N`
    - After `Complete()` succeeds, verify the mock provider was called exactly N+1 times (1 initial + N retries)
    - Verify the rate limiter was accessed for EVERY attempt, not just the first
    - Key assertion: if `Wait()` were outside the retry closure, only 1 token would be consumed regardless of retry count

  **Must NOT do**:
  - Do NOT modify existing test functions like `TestRateLimitedProvider_Complete_CircuitBreaker`
  - Do NOT introduce new mock types — use the existing `mockLLMProvider` pattern in the file

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single test file, well-defined test patterns to follow, no complex logic
  - **Skills**: `[]`
  - **Skills Evaluated but Omitted**:
    - None relevant — standard Go testing

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 2, 3, 4, 5, 6)
  - **Blocks**: Task 7 (documentation)
  - **Blocked By**: None (can start immediately)

  **References**:

  **Pattern References** (existing code to follow):
  - `internal/llm/provider_wrapper_test.go:TestRateLimitedProvider_Complete_CircuitBreaker` — Follow this exact mock setup pattern for the new CB test (mock provider struct, config, provider construction)
  - `internal/llm/provider_wrapper_test.go:TestRateLimitedProvider_Complete_Retry` — Follow this pattern for retry token test (mock with error-then-success behavior using `fn` field)

  **API/Type References** (contracts to test against):
  - `internal/llm/provider_wrapper.go:100-144` — The `Complete()` method being tested. Bug 1 fix at lines 103-110 (CB first), Bug 5 fix at line 122 (Wait inside closure)
  - `internal/domain/errors.go:ErrLLMCircuitOpen` — Expected error when circuit is open

  **Test References** (testing patterns to follow):
  - `internal/llm/provider_wrapper_test.go:mockLLMProvider` struct — Reuse this mock type with `fn` field for custom behavior
  - `internal/llm/provider_wrapper_test.go:TestRateLimitedProvider_Complete_Success` — Shows the basic happy-path setup pattern

  **WHY Each Reference Matters**:
  - `provider_wrapper_test.go` existing tests show the exact mock construction and assertion patterns to follow
  - `Complete()` method body shows the exact code path that fixes Bugs 1 and 5

  **Acceptance Criteria**:

  - [ ] `go test ./internal/llm/ -run TestRateLimitedProvider_CircuitOpenPreservesTokens -v` → PASS
  - [ ] `go test ./internal/llm/ -run TestRateLimitedProvider_RetriesConsumeTokens -v` → PASS
  - [ ] `make test` → still passes (no regressions)

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Bug 1 regression — circuit open preserves rate limit tokens
    Tool: Bash
    Preconditions: Clean working tree, `make test` passes
    Steps:
      1. Run `go test ./internal/llm/ -run TestRateLimitedProvider_CircuitOpenPreservesTokens -v`
      2. Verify output contains "PASS"
      3. Verify output contains the test function name
    Expected Result: Test passes, proving CB check happens before token consumption
    Failure Indicators: "FAIL" in output, or test function not found
    Evidence: .sisyphus/evidence/task-1-bug1-circuit-open-preserves-tokens.txt

  Scenario: Bug 5 regression — retries consume rate limit tokens
    Tool: Bash
    Preconditions: Same as above
    Steps:
      1. Run `go test ./internal/llm/ -run TestRateLimitedProvider_RetriesConsumeTokens -v`
      2. Verify output contains "PASS"
    Expected Result: Test passes, proving each retry consumes a token
    Failure Indicators: "FAIL" in output
    Evidence: .sisyphus/evidence/task-1-bug5-retries-consume-tokens.txt
  ```

  **Commit**: YES (groups with Tasks 2-5 into Commit 1)
  - Message: `test(llm,fetcher,config): add regression tests for 6 resolved bugs`
  - Files: `internal/llm/provider_wrapper_test.go`
  - Pre-commit: `go test ./internal/llm/ -run "TestRateLimitedProvider_CircuitOpenPreservesTokens|TestRateLimitedProvider_RetriesConsumeTokens" -v`

- [ ] 2. Regression test for Bug 2 (JitterFactor affects backoff calculation)

  **What to do**:
  - Add `TestRetrier_JitterFactorFromConfig` to `internal/llm/retry_test.go`:
    - Create two `Retrier` instances: one with `JitterFactor=0.0`, one with `JitterFactor=0.5`
    - Call `calculateBackoff(attempt=2)` on both with identical other config (InitialInterval, Multiplier, MaxInterval)
    - For `JitterFactor=0.0`: verify the result equals the deterministic value `InitialInterval * Multiplier^attempt` exactly (no variance)
    - For `JitterFactor=0.5`: run multiple iterations (e.g., 100) and verify results are NOT all identical (proving jitter is applied)
    - Do NOT assert exact jitter values (randomness makes that flaky)

  **Must NOT do**:
  - Do NOT modify existing `TestRetrier_calculateBackoff` or `TestCalculateBackoff`
  - Do NOT seed the random generator globally

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single test function in existing file, straightforward logic
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 3, 4, 5, 6)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `internal/llm/retry_test.go:TestRetrier_calculateBackoff` (line 462) — Shows how to construct `Retrier` and call `calculateBackoff`
  - `internal/llm/retry_test.go:TestNewRetrier` — Shows `RetryConfig` construction

  **API/Type References**:
  - `internal/llm/retry.go:115-132` — The `calculateBackoff` method that uses `r.config.JitterFactor`
  - `internal/llm/retry.go:18-26` — `RetryConfig` struct with `JitterFactor` field

  **WHY Each Reference Matters**:
  - `TestRetrier_calculateBackoff` shows the exact pattern for constructing a Retrier and calling the unexported method
  - `calculateBackoff` method body shows the `JitterFactor > 0` guard and jitter calculation to validate

  **Acceptance Criteria**:

  - [ ] `go test ./internal/llm/ -run TestRetrier_JitterFactorFromConfig -v` → PASS
  - [ ] `make test` → still passes

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Bug 2 regression — JitterFactor=0 produces deterministic backoff
    Tool: Bash
    Preconditions: Clean working tree
    Steps:
      1. Run `go test ./internal/llm/ -run TestRetrier_JitterFactorFromConfig -v`
      2. Verify output contains "PASS"
    Expected Result: Test passes, proving JitterFactor from config affects backoff calculation
    Failure Indicators: "FAIL" in output
    Evidence: .sisyphus/evidence/task-2-bug2-jitter-factor-config.txt
  ```

  **Commit**: YES (groups into Commit 1)
  - Files: `internal/llm/retry_test.go`
  - Pre-commit: `go test ./internal/llm/ -run TestRetrier_JitterFactorFromConfig -v`

- [ ] 3. Regression test for Bug 3 (half-open limits concurrent requests)

  **What to do**:
  - Add `TestCircuitBreaker_HalfOpenLimitsRequests` to `internal/llm/circuit_breaker_test.go`:
    - Create a circuit breaker with `SuccessThresholdHalfOpen=2` and `FailureThreshold=3`
    - Transition to `StateOpen` by recording 3 failures
    - Advance time past `ResetTimeout` (or use a short timeout like 1ms and sleep briefly)
    - Call `Allow()` — should return `true` and transition to `StateHalfOpen` (1st probe)
    - Call `Allow()` again — should return `true` (2nd probe, within limit of 2)
    - Call `Allow()` a 3rd time — should return `false` (REJECTED — limit reached)
    - Call `Allow()` a 4th time — should return `false` (still rejected)
    - Key: use SERIAL calls only, NOT goroutines — avoid flaky timing tests
    - Assert that exactly `SuccessThresholdHalfOpen` calls returned true

  **Must NOT do**:
  - Do NOT modify existing `TestCircuitBreaker_Allow` or `TestCircuitBreaker_HalfOpenFailure`
  - Do NOT use goroutines or concurrent access patterns — serial calls only
  - Do NOT test state transitions (already covered) — focus on REQUEST LIMITING

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single test function, clear serial logic
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 4, 5, 6)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `internal/llm/circuit_breaker_test.go:TestCircuitBreaker_Allow` (line 94) — Shows how to construct a circuit breaker with custom config and test `Allow()` returns
  - `internal/llm/circuit_breaker_test.go:TestCircuitBreaker_HalfOpenFailure` (line 211) — Shows pattern for transitioning to half-open state

  **API/Type References**:
  - `internal/llm/circuit_breaker.go:83-108` — The `Allow()` method. Bug 3 fix at lines 97-104 (halfOpenAllowed counter)
  - `internal/llm/circuit_breaker.go:47-62` — `CircuitBreakerConfig` with `SuccessThresholdHalfOpen` field

  **WHY Each Reference Matters**:
  - Existing half-open tests show how to force the breaker into half-open state (record failures → wait timeout)
  - `Allow()` method body shows the counter logic that limits requests

  **Acceptance Criteria**:

  - [ ] `go test ./internal/llm/ -run TestCircuitBreaker_HalfOpenLimitsRequests -v` → PASS
  - [ ] `make test` → still passes

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Bug 3 regression — half-open rejects excess requests
    Tool: Bash
    Preconditions: Clean working tree
    Steps:
      1. Run `go test ./internal/llm/ -run TestCircuitBreaker_HalfOpenLimitsRequests -v`
      2. Verify output contains "PASS"
    Expected Result: Test passes, proving only SuccessThresholdHalfOpen requests are allowed in half-open
    Failure Indicators: "FAIL" in output
    Evidence: .sisyphus/evidence/task-3-bug3-half-open-limits.txt

  Scenario: Bug 3 — verify 3rd and 4th Allow() calls return false
    Tool: Bash
    Preconditions: Same
    Steps:
      1. Run `go test ./internal/llm/ -run TestCircuitBreaker_HalfOpenLimitsRequests -v`
      2. Check test output for assertion details
    Expected Result: Exactly SuccessThresholdHalfOpen Allow() calls return true, rest return false
    Failure Indicators: More than SuccessThresholdHalfOpen calls return true
    Evidence: .sisyphus/evidence/task-3-bug3-half-open-excess-rejected.txt
  ```

  **Commit**: YES (groups into Commit 1)
  - Files: `internal/llm/circuit_breaker_test.go`
  - Pre-commit: `go test ./internal/llm/ -run TestCircuitBreaker_HalfOpenLimitsRequests -v`

- [ ] 4. Regression test for Bug 4 (Retry-After header respected in fetcher)

  **What to do**:
  - Add `TestRetrier_RespectsRetryAfterHeader` to `tests/unit/fetcher/retry_test.go`:
    - Create a `Retrier` with standard backoff config (e.g., InitialBackoff=100ms, Multiplier=2.0, MaxRetries=3)
    - Create a mock operation that returns a `domain.RetryableError` with `RetryAfter=5` (seconds) on first call, then succeeds
    - Track the time between the first failure and the second attempt using `time.Now()` before and after
    - Assert that the actual wait was >= 5 seconds (the RetryAfter value), NOT the backoff-calculated value (~100ms)
    - Alternative approach if time-based is too flaky: create a test that captures the wait duration by wrapping the Retrier's internal behavior, or verify via the number of calls and timing assertions with generous tolerance (e.g., >= 4.5s)
    - NOTE: The fetcher's `Retrier` at `internal/fetcher/retry.go:104-111` checks `retryableErr.RetryAfter > 0` and uses `max(backoff, retryAfter)` — this is the code being validated

  **Must NOT do**:
  - Do NOT modify existing `TestRetrier_Retry_Error` or `TestParseRetryAfter`
  - Do NOT use sub-millisecond timing assertions — allow generous tolerance for CI environments
  - Do NOT test the LLM retry path (`internal/llm/retry.go`) — Bug 4 is about fetcher retries

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Single test function, well-defined assertion
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3, 5, 6)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `tests/unit/fetcher/retry_test.go:TestRetrier_Retry_Error` (line 214) — Shows how to construct a fetcher `Retrier` and test with failing operations
  - `tests/unit/fetcher/retry_test.go:TestParseRetryAfter` (line 551) — Shows RetryAfter parsing, useful for understanding the error type

  **API/Type References**:
  - `internal/fetcher/retry.go:73-119` — The `Retry()` method. Bug 4 fix at lines 104-111 (RetryAfter check and max)
  - `internal/domain/errors.go:77` — `RetryableError` struct with `RetryAfter int` field
  - `internal/domain/errors.go:IsRetryable()` — Used by the Retry loop to check retryability

  **WHY Each Reference Matters**:
  - `TestRetrier_Retry_Error` shows the exact setup pattern for fetcher retrier tests
  - `retry.go:104-111` is the specific code being validated — the `max(backoff, RetryAfter)` logic
  - `RetryableError` struct shows how to construct the error with `RetryAfter` field

  **Acceptance Criteria**:

  - [ ] `go test ./tests/unit/fetcher/ -run TestRetrier_RespectsRetryAfterHeader -v` → PASS
  - [ ] `make test` → still passes

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Bug 4 regression — Retry-After header value is respected
    Tool: Bash
    Preconditions: Clean working tree
    Steps:
      1. Run `go test ./tests/unit/fetcher/ -run TestRetrier_RespectsRetryAfterHeader -v`
      2. Verify output contains "PASS"
    Expected Result: Test passes, proving wait duration respects RetryAfter when it exceeds calculated backoff
    Failure Indicators: "FAIL" in output, or timeout (test takes too long due to actual 5s sleep)
    Evidence: .sisyphus/evidence/task-4-bug4-retry-after-respected.txt

  Scenario: Bug 4 — verify test doesn't take excessive time
    Tool: Bash
    Preconditions: Same
    Steps:
      1. Run `go test ./tests/unit/fetcher/ -run TestRetrier_RespectsRetryAfterHeader -v -timeout 30s`
      2. Verify test completes within 30s
    Expected Result: Test completes without timeout
    Failure Indicators: Timeout exceeded
    Evidence: .sisyphus/evidence/task-4-bug4-retry-after-timing.txt
  ```

  **Commit**: YES (groups into Commit 1)
  - Files: `tests/unit/fetcher/retry_test.go`
  - Pre-commit: `go test ./tests/unit/fetcher/ -run TestRetrier_RespectsRetryAfterHeader -v`

- [ ] 5. Regression test for Bug 6 (config Validate rejects invalid rate limit values)

  **What to do**:
  - Add `TestConfig_Validate_RateLimitFields` to `tests/unit/config/config_test.go`:
    - Use table-driven sub-tests with `t.Run()` (matches existing pattern in the file)
    - Test cases that MUST return an error (when `RateLimit.Enabled=true`):
      - `RequestsPerMinute: -1` → error containing "requests_per_minute"
      - `BurstSize: -1` → error containing "burst_size"
      - `MaxRetries: -1` → error containing "max_retries"
      - `JitterFactor: -0.5` → error containing "jitter_factor"
      - `JitterFactor: 1.5` → error containing "jitter_factor"
      - `CircuitBreaker.Enabled=true` with `FailureThreshold: 0` → error containing "failure_threshold"
      - `CircuitBreaker.Enabled=true` with `SuccessThresholdHalfOpen: 0` → error containing "success_threshold_half_open"
    - Test case that MUST NOT return an error:
      - Valid config with `RateLimit.Enabled=true`, all positive values → `nil` error
    - Match EXACTLY what `Validate()` checks today at `config.go:130-167` — do not test validations that don't exist
    - Start each test case from `config.Default()` to get valid base config, then override the one field being tested

  **Must NOT do**:
  - Do NOT modify existing `TestConfig_Validate` function
  - Do NOT test validation for `Enabled=false` (current code skips validation when disabled — that's by design)
  - Do NOT invent new validations — only test what `Validate()` currently implements

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Table-driven test, clear pattern to follow
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1, 2, 3, 4, 6)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `tests/unit/config/config_test.go:TestConfig_Validate` (line 46) — Shows existing validation test pattern with table-driven sub-tests
  - `internal/config/defaults.go:Default()` — Use this to get a valid base config for test cases

  **API/Type References**:
  - `internal/config/config.go:105-171` — The `Validate()` method. Bug 6 fix at lines 130-167 (rate limit + circuit breaker validation)
  - `internal/config/config.go:38-56` — `RateLimitConfig` and `CircuitBreakerConfig` structs showing all fields

  **WHY Each Reference Matters**:
  - Existing `TestConfig_Validate` shows the exact table-driven pattern and assertion style to follow
  - `Validate()` method body at lines 130-167 defines the exact validations to test — match them 1:1

  **Acceptance Criteria**:

  - [ ] `go test ./tests/unit/config/ -run TestConfig_Validate_RateLimitFields -v` → PASS
  - [ ] `make test` → still passes

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: Bug 6 regression — invalid rate limit values rejected
    Tool: Bash
    Preconditions: Clean working tree
    Steps:
      1. Run `go test ./tests/unit/config/ -run TestConfig_Validate_RateLimitFields -v`
      2. Verify output contains "PASS"
      3. Verify all sub-tests pass (look for "--- PASS:" lines)
    Expected Result: All table-driven sub-tests pass, proving Validate() catches invalid RL/CB values
    Failure Indicators: "FAIL" in output, or any sub-test fails
    Evidence: .sisyphus/evidence/task-5-bug6-validate-rate-limit.txt

  Scenario: Bug 6 — valid config passes validation
    Tool: Bash
    Preconditions: Same
    Steps:
      1. Verify the "valid config" sub-test case returns nil error
    Expected Result: Valid rate limit config passes without error
    Failure Indicators: Valid config returns unexpected error
    Evidence: .sisyphus/evidence/task-5-bug6-valid-config-passes.txt
  ```

  **Commit**: YES (groups into Commit 1)
  - Files: `tests/unit/config/config_test.go`
  - Pre-commit: `go test ./tests/unit/config/ -run TestConfig_Validate_RateLimitFields -v`

- [ ] 6. TDD fix for exported CalculateBackoff (hardcoded jitter → config jitter)

  **What to do**:
  - **FIRST (RED)**: Add `TestCalculateBackoff_UsesConfigJitter` to `internal/llm/retry_test.go`:
    - Test with `JitterFactor=0.0`: verify backoff is exactly `InitialInterval * Multiplier^attempt` (deterministic, no jitter)
    - Test with `JitterFactor=0.5`: run 50+ iterations and verify results are NOT all identical (jitter applied)
    - Test with `JitterFactor=0.0` and `JitterFactor < 0` (negative): verify no jitter (the `> 0` guard)
    - This test should FAIL against the current `CalculateBackoff` function (which hardcodes `0.1` and always applies jitter)
  - **THEN (GREEN)**: Fix `CalculateBackoff` in `internal/llm/retry.go:188-199`:
    - Replace line 191: `jitter := backoff * 0.1 * (rand.Float64()*2 - 1)` with `cfg.JitterFactor` and add `> 0` guard
    - Match the pattern of the unexported `calculateBackoff` method at lines 115-132:
      ```go
      if cfg.JitterFactor > 0 {
          jitter := backoff * cfg.JitterFactor * (rand.Float64()*2 - 1)
          backoff += jitter
      }
      ```
    - Remove the unconditional `backoff += jitter` line
  - **VERIFY**: Existing `TestCalculateBackoff` at line 407 must still pass. Read it first to understand assertions. If it relies on jitter being applied (it likely does since hardcoded 0.1 was always active), it may need `JitterFactor=0.1` set in its config. Check and adjust the NEW test or production fix accordingly, but do NOT modify the existing test function.

  **Must NOT do**:
  - Do NOT deprecate the function — just fix it
  - Do NOT modify existing `TestCalculateBackoff` at line 407
  - Do NOT change the function signature
  - Do NOT add new parameters

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: 2-line production fix + 1 test function, TDD cycle is trivial
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: YES
  - **Parallel Group**: Wave 1 (with Tasks 1-5)
  - **Blocks**: Task 7
  - **Blocked By**: None

  **References**:

  **Pattern References**:
  - `internal/llm/retry.go:115-132` — The unexported `calculateBackoff` method — this is the CORRECT pattern to replicate. Shows the `JitterFactor > 0` guard and jitter calculation
  - `internal/llm/retry_test.go:TestCalculateBackoff` (line 407) — MUST READ before making changes. Understand its assertions to ensure backward compatibility

  **API/Type References**:
  - `internal/llm/retry.go:188-199` — The exported `CalculateBackoff` function to fix. Line 191 has the hardcoded `0.1`
  - `internal/llm/retry.go:18-26` — `RetryConfig` struct with `JitterFactor float64` field

  **Test References**:
  - `tests/unit/llm/retry_test.go:385` — External test that calls exported `CalculateBackoff`. Must still pass after fix

  **WHY Each Reference Matters**:
  - The unexported `calculateBackoff` (lines 115-132) IS the correct implementation to copy — it already handles the `> 0` guard and uses `cfg.JitterFactor`
  - The existing `TestCalculateBackoff` (line 407) may need its config to include `JitterFactor: 0.1` after the fix — READ IT FIRST before changing production code
  - The external test at `tests/unit/llm/retry_test.go:385` uses the exported function — must verify it still passes

  **Acceptance Criteria**:

  - [ ] `go test ./internal/llm/ -run TestCalculateBackoff_UsesConfigJitter -v` → PASS (new test)
  - [ ] `go test ./internal/llm/ -run TestCalculateBackoff -v` → PASS (existing test)
  - [ ] `go test ./tests/unit/llm/ -run TestCalculateBackoff -v` → PASS (external test)
  - [ ] `make test` → still passes

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: CalculateBackoff fix — JitterFactor=0 produces deterministic output
    Tool: Bash
    Preconditions: Fix applied to retry.go:188-199
    Steps:
      1. Run `go test ./internal/llm/ -run TestCalculateBackoff_UsesConfigJitter -v`
      2. Verify output contains "PASS"
    Expected Result: Test passes, proving CalculateBackoff now uses cfg.JitterFactor
    Failure Indicators: "FAIL" in output
    Evidence: .sisyphus/evidence/task-6-calculate-backoff-config-jitter.txt

  Scenario: CalculateBackoff fix — backward compatibility preserved
    Tool: Bash
    Preconditions: Fix applied
    Steps:
      1. Run `go test ./internal/llm/ -run "^TestCalculateBackoff$" -v`
      2. Run `go test ./tests/unit/llm/ -run TestCalculateBackoff -v`
      3. Verify both contain "PASS"
    Expected Result: Existing tests still pass after the fix
    Failure Indicators: "FAIL" in either test
    Evidence: .sisyphus/evidence/task-6-calculate-backoff-backward-compat.txt

  Scenario: Full test suite passes after fix
    Tool: Bash
    Preconditions: All changes applied
    Steps:
      1. Run `make test`
      2. Verify exit code 0 and no failures
    Expected Result: All short tests pass
    Failure Indicators: Non-zero exit code, "FAIL" in output
    Evidence: .sisyphus/evidence/task-6-make-test-pass.txt
  ```

  **Commit**: YES (Commit 2 — separate from regression tests)
  - Message: `fix(llm): use configured JitterFactor in exported CalculateBackoff`
  - Files: `internal/llm/retry.go`, `internal/llm/retry_test.go`
  - Pre-commit: `make test`

- [ ] 7. Update bugs.md and AGENTS.md documentation

  **What to do**:
  - Update `bugs.md`:
    - Add a `## Status` line after each bug's `**Correcao:**` section with: `**Status:** RESOLVED (2026-04) — Fix verified by regression test`
    - Add to each bug entry which test verifies it:
      - Bug 1: `TestRateLimitedProvider_CircuitOpenPreservesTokens`
      - Bug 2: `TestRetrier_JitterFactorFromConfig`
      - Bug 3: `TestCircuitBreaker_HalfOpenLimitsRequests`
      - Bug 4: `TestRetrier_RespectsRetryAfterHeader`
      - Bug 5: `TestRateLimitedProvider_RetriesConsumeTokens`
      - Bug 6: `TestConfig_Validate_RateLimitFields`
    - Update the `## Resumo` table to add a "Status" column showing "RESOLVED" for all 6
    - Preserve ALL existing content and Portuguese language — only ADD resolution annotations
  - Update `AGENTS.md`:
    - Change the "Known Bugs" section from listing 6 active bugs to marking them as resolved
    - Replace the current text with something like: `All 6 documented issues in bugs.md have been resolved and verified with regression tests. See bugs.md for details.`
    - Keep the reference to `bugs.md` for historical context

  **Must NOT do**:
  - Do NOT rewrite existing bug descriptions in `bugs.md`
  - Do NOT change the language from Portuguese to English in `bugs.md`
  - Do NOT delete `bugs.md` — preserve it as historical documentation
  - Do NOT remove the Known Bugs section from AGENTS.md — update it in place

  **Recommended Agent Profile**:
  - **Category**: `quick`
    - Reason: Simple documentation edits, no code logic
  - **Skills**: `[]`

  **Parallelization**:
  - **Can Run In Parallel**: NO
  - **Parallel Group**: Wave 2 (sequential after Wave 1)
  - **Blocks**: F1-F4
  - **Blocked By**: Tasks 1-6

  **References**:

  **Pattern References**:
  - `bugs.md` — Full file, all 6 bug entries to annotate. Preserve existing Portuguese content
  - `AGENTS.md:Known Bugs` section — Current wording to update

  **WHY Each Reference Matters**:
  - `bugs.md` is the primary document to update — must understand its structure to add annotations correctly
  - `AGENTS.md` references bugs.md and needs consistent messaging

  **Acceptance Criteria**:

  - [ ] `grep -c "RESOLVED" bugs.md` returns `6`
  - [ ] Each bug entry in `bugs.md` has a status line and test reference
  - [ ] AGENTS.md "Known Bugs" section reflects resolved status
  - [ ] No existing content deleted from `bugs.md`

  **QA Scenarios (MANDATORY):**

  ```
  Scenario: bugs.md has all 6 bugs marked as RESOLVED
    Tool: Bash
    Preconditions: Documentation edits applied
    Steps:
      1. Run `grep -c "RESOLVED" bugs.md`
      2. Verify output is "6"
      3. Run `grep "RESOLVED" bugs.md` to see all 6 lines
    Expected Result: Exactly 6 lines containing "RESOLVED"
    Failure Indicators: Count != 6
    Evidence: .sisyphus/evidence/task-7-bugs-md-resolved-count.txt

  Scenario: AGENTS.md reflects resolved status
    Tool: Bash
    Preconditions: Documentation edits applied
    Steps:
      1. Run `grep -i "resolved" AGENTS.md`
      2. Verify output references bugs being resolved
    Expected Result: AGENTS.md mentions bugs are resolved
    Failure Indicators: No match found
    Evidence: .sisyphus/evidence/task-7-agents-md-updated.txt

  Scenario: bugs.md preserves existing content
    Tool: Bash
    Preconditions: Documentation edits applied
    Steps:
      1. Run `grep -c "BUG" bugs.md`
      2. Verify all 6 bug headings still present
      3. Run `grep "Severidade" bugs.md` to verify Portuguese content preserved
    Expected Result: All original headings and Portuguese content intact
    Failure Indicators: Missing headings or translated content
    Evidence: .sisyphus/evidence/task-7-bugs-md-preserved.txt
  ```

  **Commit**: YES (Commit 3)
  - Message: `docs: mark all 6 rate-limit/circuit-breaker bugs as resolved`
  - Files: `bugs.md`, `AGENTS.md`
  - Pre-commit: `grep -c "RESOLVED" bugs.md` (expect 6)

---

## Final Verification Wave (MANDATORY — after ALL implementation tasks)

> 4 review agents run in PARALLEL. ALL must APPROVE. Present consolidated results to user and get explicit "okay" before completing.

- [ ] F1. **Plan Compliance Audit** — `oracle`
  Read the plan end-to-end. For each "Must Have": verify implementation exists (run test, grep file). For each "Must NOT Have": search codebase for forbidden patterns — reject with file:line if found. Check evidence files exist in `.sisyphus/evidence/`. Compare deliverables against plan.
  Output: `Must Have [N/N] | Must NOT Have [N/N] | Tasks [N/N] | VERDICT: APPROVE/REJECT`

- [ ] F2. **Code Quality Review** — `unspecified-high`
  Run `make test` + `make lint`. Review all changed files for: test naming convention match, testify usage, no `as any` or empty catches. Check AI slop: excessive comments, over-abstraction, generic names. Verify `CalculateBackoff` fix is minimal (2-line change, matching unexported method pattern).
  Output: `Build [PASS/FAIL] | Lint [PASS/FAIL] | Tests [N pass/N fail] | Files [N clean/N issues] | VERDICT`

- [ ] F3. **Real Manual QA** — `unspecified-high`
  Run EVERY regression test individually via `go test -run TestName -v`. Verify each passes. Run `make test` for full suite. Verify `grep -c "RESOLVED" bugs.md` returns 6. Verify AGENTS.md Known Bugs section updated. Save output to `.sisyphus/evidence/final-qa/`.
  Output: `Scenarios [N/N pass] | Integration [N/N] | Edge Cases [N tested] | VERDICT`

- [ ] F4. **Scope Fidelity Check** — `deep`
  For each task: read "What to do", read actual diff (`git diff`). Verify 1:1 — everything in spec was built (no missing), nothing beyond spec was built (no creep). Check "Must NOT do" compliance: no existing test modifications, no refactoring, no new frameworks. Flag unaccounted changes.
  Output: `Tasks [N/N compliant] | Contamination [CLEAN/N issues] | Unaccounted [CLEAN/N files] | VERDICT`

---

## Commit Strategy

| # | Scope | Message | Files | Pre-commit |
|---|-------|---------|-------|------------|
| 1 | Regression tests (Bugs 1-6) | `test(llm,fetcher,config): add regression tests for 6 resolved bugs` | `internal/llm/provider_wrapper_test.go`, `internal/llm/retry_test.go`, `internal/llm/circuit_breaker_test.go`, `tests/unit/fetcher/retry_test.go`, `tests/unit/config/config_test.go` | `make test` |
| 2 | CalculateBackoff fix | `fix(llm): use configured JitterFactor in exported CalculateBackoff` | `internal/llm/retry.go`, `internal/llm/retry_test.go` | `make test` |
| 3 | Documentation cleanup | `docs: mark all 6 rate-limit/circuit-breaker bugs as resolved` | `bugs.md`, `AGENTS.md` | — |

---

## Success Criteria

### Verification Commands
```bash
make test          # Expected: PASS, 0 failures
make lint          # Expected: no new warnings
grep -c "RESOLVED" bugs.md  # Expected: 6
go test ./internal/llm/ -run "TestRateLimitedProvider_CircuitOpenPreservesTokens|TestRetrier_JitterFactorFromConfig|TestCircuitBreaker_HalfOpenLimitsRequests|TestRateLimitedProvider_RetriesConsumeTokens|TestCalculateBackoff_UsesConfigJitter" -v  # Expected: all PASS
go test ./tests/unit/fetcher/ -run "TestRetrier_RespectsRetryAfterHeader" -v  # Expected: PASS
go test ./tests/unit/config/ -run "TestConfig_Validate_RateLimitFields" -v  # Expected: PASS
```

### Final Checklist
- [ ] All "Must Have" present
- [ ] All "Must NOT Have" absent
- [ ] All tests pass
- [ ] `bugs.md` has 6 RESOLVED entries
- [ ] AGENTS.md Known Bugs section updated
