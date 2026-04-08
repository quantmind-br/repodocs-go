## 2026-04-07
- Added regression tests covering LLM circuit-open token preservation, retry token consumption, Retry-After handling, and rate-limit config validation.
- Existing circuit breaker half-open test already used `SuccessThresholdHalfOpen: 1`; no code change needed there.
