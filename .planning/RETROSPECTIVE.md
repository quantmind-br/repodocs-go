# Project Retrospective

*A living document updated after each milestone. Lessons feed forward into future planning.*

## Milestone: v1.0 — LM Studio Provider

**Shipped:** 2026-04-07
**Phases:** 2 | **Plans:** 4 | **Tasks:** 6

### What Was Built
- LMStudioProvider with OpenAI-compatible chat completions, conditional auth, and 300s cold-start timeout
- Factory registration with zero-config defaults (localhost:1234/v1, no API key required)
- TUI config editor integration (dropdown + validation)
- 14 unit tests + 4 factory tests + 1 integration test

### What Worked
- Reusing OpenAI wire format types eliminated code duplication entirely
- Research phase identified all pitfalls upfront (API key exemption, factory switch atomicity, TUI validation gap)
- Keeping ollama TUI validation fix out of scope prevented scope creep
- All plans executed with zero deviations — research quality directly correlated with execution speed

### What Was Inefficient
- REQUIREMENTS.md checkboxes were never checked during phase execution — only caught during audit
- STATE.md progress tracking showed 0% despite 100% completion (stale updates)

### Patterns Established
- OpenAI-compatible provider pattern: reuse wire format types, add provider-specific error handling
- Keyless provider pattern: exempt from API key guard alongside ollama
- httptest-based provider testing with request/response validation

### Key Lessons
1. Research before planning pays off — zero deviations across all 4 plans because pitfalls were known upfront
2. Pre-existing bugs adjacent to your work are tempting to fix but must stay out of scope (ollama validation gap)
3. Milestone audit should run before completion to catch stale artifacts like unchecked requirement checkboxes

### Cost Observations
- Model mix: primarily opus for planning/execution, sonnet for research agents
- Sessions: ~3 sessions (research/plan, execute phase 1, execute phase 2 + audit)
- Notable: entire milestone completed in a single day

---

## Cross-Milestone Trends

### Process Evolution

| Milestone | Sessions | Phases | Key Change |
|-----------|----------|--------|------------|
| v1.0 | ~3 | 2 | First milestone — established GSD workflow baseline |

### Cumulative Quality

| Milestone | Tests Added | Source Files | Zero-Deviation Plans |
|-----------|-------------|--------------|---------------------|
| v1.0 | 19 | 9 | 4/4 |

### Top Lessons (Verified Across Milestones)

1. Thorough research eliminates plan deviations — verified in v1.0 (4/4 plans zero-deviation)
