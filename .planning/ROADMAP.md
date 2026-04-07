# Roadmap: LM Studio Provider

## Overview

Add first-class LM Studio support to repodocs in two tightly scoped phases. Phase 1 delivers the complete provider implementation — the factory registration, default URL, optional auth header, and config validation must ship as one atomic unit. Phase 2 wires up the TUI dropdown, connection error UX, and full test coverage. The result: users can run metadata enhancement against local LM Studio models with zero-config defaults.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Provider Core** - Implement LM Studio provider and register it in the factory with defaults
- [ ] **Phase 2: TUI, UX & Tests** - Add TUI option, connection error message, and full test coverage

## Phase Details

### Phase 1: Provider Core
**Goal**: Users can configure `provider: lmstudio` in their config and have repodocs make correct OpenAI-compatible requests to a local LM Studio server
**Depends on**: Nothing (first phase)
**Requirements**: PROV-01, PROV-02, PROV-03, PROV-04, PROV-05, PROV-06, CONF-01, CONF-03
**Plans:** 2 plans

Plans:
- [ ] 01-01-PLAN.md — LMStudioProvider struct and factory registration (provider.go + lmstudio.go)
- [ ] 01-02-PLAN.md — TUI validation and provider dropdown (validation.go + forms.go)

**Success Criteria** (what must be TRUE):
  1. User can set `provider: lmstudio` in YAML config and repodocs loads it without error
  2. Metadata enhancement runs against `http://localhost:1234/v1` when no base_url is set
  3. Provider works with no API key configured (no Authorization header sent)
  4. Provider sends `Authorization: Bearer <key>` when an API key is configured
  5. Requests use OpenAI chat completions format and respect 300s timeout

### Phase 2: TUI, UX & Tests
**Goal**: LM Studio is selectable from the TUI config editor, connection failures show a helpful message, and the provider is fully covered by unit and integration tests
**Depends on**: Phase 1
**Requirements**: CONF-02, CONF-04, TEST-01, TEST-02, TEST-03, TEST-04
**Success Criteria** (what must be TRUE):
  1. TUI config editor provider dropdown includes `lmstudio` as a selectable option
  2. User sees a clear error message when LM Studio server is not running (connection refused)
  3. Unit tests pass verifying correct request format and conditional auth header behavior
  4. Provider factory test confirms `lmstudio` case returns a valid provider instance
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Provider Core | 0/2 | Planning complete | - |
| 2. TUI, UX & Tests | 0/? | Not started | - |
