# Documentation Gaps Analysis Report

## Executive Summary

RepoDocs is a Go-based CLI tool for extracting documentation from websites into Markdown. The project has **moderate-to-good documentation coverage** with a comprehensive README, extensive inline Go doc comments (85% of exported symbols), and auto-generated per-package AGENTS.md files. However, significant gaps exist in API documentation for several key packages, missing standard open-source documentation files, unexplained magic numbers in complex algorithms, and a lack of package-level documentation for most internal packages.

**Key Stats:**
- **102** non-test Go files, **174** test files
- **~340** exported symbols across **17** packages
- **85%** exported symbol documentation coverage (**~290** documented, **~50** undocumented)
- **3** package-level doc comments out of **~15** internal packages
- **96** markdown files (many auto-generated AGENTS.md)
- **0** TODO/FIXME/HACK comments (clean codebase)
- **No** CONTRIBUTING.md, CHANGELOG.md, CODE_OF_CONDUCT.md, or SECURITY.md

---

## Current Documentation Assessment

### README.md
- **Status:** Present
- **Quality:** Good, with notable gaps
- **Key Strengths:**
  - Clear project description and feature list
  - Installation instructions (from source)
  - Quick start examples
  - Manifest schema documentation with tables
  - Architecture overview
  - Configuration commands documented
  - Common CLI flags table
- **Key Gaps:**
  - Prerequisites state "Go 1.21" but `go.mod` specifies `go 1.24.1`
  - No FAQ or troubleshooting section
  - No contributing guidelines (points to generic bullet list only)
  - No changelog or version history
  - No security policy
  - Missing `--accessible` flag in common flags table
  - No library/API usage examples (only CLI usage)

### API Documentation
- **Documented functions:** 244 / 292 (**84%**)
- **Documented types:** 182 / 209 (**87%**)
- **Coverage by module:**
  | Package | Coverage | Assessment |
  |---------|----------|------------|
  | `internal/domain` | Excellent | All errors, interfaces, models documented |
  | `internal/converter` | Good | Pipeline stages well-documented |
  | `internal/fetcher` | Good | Client, retry, transport documented |
  | `internal/cache` | Good | Interface and implementation documented |
  | `internal/config` | Good | Config structs and defaults documented |
  | `internal/llm` | Poor | Provider types (Anthropic, Google, Ollama, OpenAI) undocumented |
  | `internal/output` | Poor | `MetadataCollector` and `Writer` types undocumented |
  | `internal/state` | Poor | `Manager` type and all methods undocumented |
  | `internal/strategies/git/` | Very Poor | All archive/clone/parser/processor/strategy symbols undocumented |
  | `internal/strategies/wiki_parser.go` | Very Poor | All `Wiki*` types and parsing helpers undocumented |
  | `internal/strategies/docsrs*.go` | Poor | Strategy, renderer, JSON types undocumented |
  | `cmd/repodocs` | N/A | No exported symbols (by design) |

### Inline Comments
- **Overall density:** Medium
- **Complex areas with missing comments:**
  - `internal/strategies/docsrs_renderer.go` — Complex Rustdoc JSON to Markdown rendering lacks "why" comments for type assertion patterns
  - `internal/strategies/github_pages.go` — SPA detection heuristics have unexplained magic numbers
  - `internal/renderer/rod.go` — Scroll behavior has unexplained iteration limits and delays
  - `internal/strategies/llms.go` — Dual regex parsing not explained
  - `internal/utils/url.go` — `sanitizeForDirName()` has no comments at all

### Examples & Tutorials
- **Status:** Partial
- **Present:** 4 manifest YAML examples in `examples/manifests/`
- **Missing:**
  - Getting started tutorial
  - API/library usage examples
  - Common use case walkthroughs
  - Docker usage example
  - Configuration file example beyond manifest schema

### Architecture Documentation
- **Status:** Good (internal), Poor (user-facing)
- **Present:** AGENTS.md files per package provide excellent internal architecture docs
- **Present:** README has high-level pipeline description
- **Missing:**
  - No architecture diagram in README
  - No data flow diagram for users
  - No component interaction documentation
  - Strategy detection order not documented for users

### Troubleshooting
- **Status:** Partial
- **Present:**
  - `TESTING.md` has test troubleshooting section
  - `bugs.md` documents resolved issues (in Portuguese)
  - `doctor` command mentioned in README
- **Missing:**
  - No FAQ in README
  - No common errors and solutions
  - No debugging guide for users
  - No migration guide

---

## Documentation Gaps

### High Priority

#### DOC-001: Missing Package-Level Documentation for Internal Packages

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/app/`
- `internal/cache/`
- `internal/config/`
- `internal/converter/`
- `internal/domain/`
- `internal/fetcher/`
- `internal/git/`
- `internal/llm/`
- `internal/output/`
- `internal/renderer/`
- `internal/state/`
- `internal/strategies/`
- `internal/tui/`
- `internal/utils/`
- `cmd/repodocs/`

**Current State:**
Only 3 packages have `// Package` doc comments: `internal/manifest`, `internal/strategies/git`, and `internal/tui/styles.go`. All other packages lack package-level documentation that explains their purpose and responsibilities.

**Proposed Content:**
Add `doc.go` files or package comments to each internal package explaining:
- Package purpose and responsibilities
- Key types and functions
- Dependencies and consumers
- Usage examples where non-obvious

**Rationale:**
Package-level documentation is the first thing developers see when browsing code. It orients new contributors and helps IDE tooltips provide useful context. This is especially important for a project with 14 internal packages.

**Estimated Effort:** small

---

#### DOC-002: LLM Provider Types and Constructors Undocumented

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/llm/anthropic.go` — `AnthropicProvider`, `NewAnthropicProvider`
- `internal/llm/google.go` — `GoogleProvider`, `NewGoogleProvider`
- `internal/llm/ollama.go` — `OllamaProvider`, `NewOllamaProvider`
- `internal/llm/openai.go` — `OpenAIProvider`, `NewOpenAIProvider`
- `internal/llm/provider.go` — `ProviderConfig`

**Current State:**
All four LLM provider implementations and their configuration struct lack Go doc comments. Only `LMStudioProvider` and the wrapper types have documentation.

**Proposed Content:**
Add doc comments to each provider type and constructor:
```go
// AnthropicProvider implements domain.LLMProvider for Anthropic's Claude API.
// It supports message-based completions with configurable model and parameters.
type AnthropicProvider struct { ... }

// NewAnthropicProvider creates a new Anthropic provider with the given configuration.
// The httpClient is used for all API requests; if nil, a default client is used.
func NewAnthropicProvider(cfg ProviderConfig, httpClient *http.Client) (*AnthropicProvider, error)
```

**Rationale:**
LLM integration is a key differentiating feature. Developers extending or debugging LLM functionality need to understand provider differences. The lack of docs forces reading implementation code.

**Estimated Effort:** trivial

---

#### DOC-003: Git Strategy Sub-Package Completely Undocumented

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/strategies/git/archive.go` — `ArchiveFetcher`, `NewArchiveFetcher`
- `internal/strategies/git/clone.go` — `CloneFetcher`, `NewCloneFetcher`, `DetectDefaultBranch`
- `internal/strategies/git/fetcher.go` — `RepoFetcher`
- `internal/strategies/git/parser.go` — `Parser`, `NewParser`, `NormalizeFilterPath`, `ExtractPathFromTreeURL`
- `internal/strategies/git/processor.go` — `Processor`, `NewProcessor`, `ExtractTitleFromPath`
- `internal/strategies/git/strategy.go` — `Strategy`, `StrategyDependencies`, `NewStrategy`, `ExecuteOptions`

**Current State:**
All ~25 exported symbols in the git sub-package lack documentation. This is one of the most complex strategy implementations (archive fetching, cloning, parsing, processing).

**Proposed Content:**
Add doc comments to all exported types and functions. The `git/` sub-package already has a `doc.go` file but it only contains `// Package git`. Expand it to explain the sub-package architecture and add docs to all exported symbols.

**Rationale:**
Git strategy is a core extraction method. The sub-package has complex logic spanning archive fetching, git cloning, URL parsing, and file processing. Without documentation, it's very difficult for new contributors to understand or modify.

**Estimated Effort:** small

---

#### DOC-004: Output Package Types Undocumented

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/output/collector.go` — `MetadataCollector`, `CollectorOptions`, `NewMetadataCollector`, plus all methods (`Add`, `Flush`, `Count`, `GetIndex`, `IsEnabled`, `SetStrategy`, `SetSourceURL`)
- `internal/output/writer.go` — `Writer`, `WriterOptions`, `NewWriter`

**Current State:**
The entire `MetadataCollector` type and all its methods lack documentation. The `Writer` type also lacks documentation. This is notable because `Writer` in the same package IS used by the orchestrator but has no interface contract explanation.

**Proposed Content:**
```go
// MetadataCollector accumulates metadata from processed documents and generates
// the consolidated repodocs.json index file.
type MetadataCollector struct { ... }

// Add collects metadata from a processed document.
func (c *MetadataCollector) Add(doc *domain.Document) error

// Flush writes the accumulated metadata index to the output directory.
func (c *MetadataCollector) Flush() error
```

**Rationale:**
Metadata collection is a key output feature (generates `repodocs.json`). The collector is used by the orchestrator but its behavior and lifecycle are opaque without documentation.

**Estimated Effort:** trivial

---

#### DOC-005: Missing Standard Open-Source Documentation Files

**Category:** readme  
**Target Audience:** contributors, users, maintainers

**Affected Areas:**
- Root directory

**Current State:**
The project lacks standard open-source documentation files: CONTRIBUTING.md, CHANGELOG.md, CODE_OF_CONDUCT.md, SECURITY.md. Only LICENSE exists.

**Proposed Content:**
1. **CONTRIBUTING.md**: Development setup, code style, test requirements, PR process, issue templates
2. **CHANGELOG.md**: Version history following Keep a Changelog format
3. **CODE_OF_CONDUCT.md**: Community standards
4. **SECURITY.md**: Vulnerability reporting process, supported versions

**Rationale:**
These files are expected by open-source contributors and are often required by package registries and enterprise users. The current README has a 4-bullet "Contributing" section that is insufficient for external contributors.

**Estimated Effort:** small

---

#### DOC-006: Complex Rustdoc Renderer Lacks Algorithm Comments

**Category:** inline_comments  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/strategies/docsrs_renderer.go`
  - `RenderTypeMap()` (lines 214-318)
  - `renderFunctionSignature()` (lines 133-197)
  - `resolveCrossRefs()` (lines 346-373)

**Current State:**
The Rustdoc JSON to Markdown renderer is the most complex file in the project (628 lines). It handles 12+ Rust type formats but lacks comments explaining:
- Why Rustdoc JSON uses `[]interface{}` for type representations
- What each type format looks like and how it's mapped
- The cross-reference resolution algorithm

**Proposed Content:**
Add a file-level comment block explaining the Rustdoc JSON schema mapping. Add inline comments for each major type format branch. Add comments explaining the `[]interface{}` pattern.

**Rationale:**
This is the project's top complexity hotspot (per AGENTS.md). The renderer handles Rust's rich type system (generics, traits, lifetimes) mapped to Markdown. Without comments, future maintainers cannot safely modify this code.

**Estimated Effort:** medium

---

#### DOC-007: SPA Detection Heuristics Have Unexplained Magic Numbers

**Category:** inline_comments  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/strategies/github_pages.go`
  - `looksLikeSPAShell()` (lines 506-540)
  - `isEmptyOrErrorContent()` (lines 543-577)
  - `processURLs()` concurrency limits (lines 389-473)
- `internal/renderer/rod.go`
  - `scrollToEnd()` (lines 222-258)

**Current State:**
Multiple magic numbers without explanation:
```go
if len(html) < 500 { return true }        // WHY 500?
if len(bodyText) < 100 { return true }     // WHY 100?
if concurrency > 5 { concurrency = 5 }     // WHY 5?
for i := 0; i < 10; i++                   // WHY 10?
time.Sleep(300 * time.Millisecond)        // WHY 300ms?
if stableCount >= 2 { break }             // WHY 2?
```

**Proposed Content:**
Extract magic numbers into named constants with explanatory comments:
```go
// MinSPAContentLength is the minimum HTML content length below which
// a page is considered a SPA shell (likely waiting for JS hydration).
// This accounts for small HTML wrappers around JS apps.
const MinSPAContentLength = 500

// MaxScrollIterations limits how many times we scroll to prevent
// infinite scroll pages from hanging.
const MaxScrollIterations = 10
```

**Rationale:**
These thresholds directly affect user experience and reliability. Without documentation, future changes risk breaking SPA detection or causing hangs on infinite-scroll pages.

**Estimated Effort:** small

---

#### DOC-008: Dual Regex Pattern in LLMS Strategy Not Explained

**Category:** inline_comments  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/strategies/llms.go` (lines 197-204, 206-264)

**Current State:**
The llms.txt parser uses two different regex patterns (`linkRegex` and `bareLinkRegex`) to parse links. The code comments explain WHAT each regex matches but not WHY both are needed.

**Proposed Content:**
```go
// Two regex patterns are needed because the llms.txt format is not strictly
// standardized. Some producers use Markdown-style links [Title](url): desc,
// while others use bare links (url): desc. We try both formats to maximize
// compatibility with different llms.txt implementations.
```

**Rationale:**
This is a workaround for format inconsistency in the emerging llms.txt standard. Documenting this prevents future developers from "simplifying" the code and breaking compatibility.

**Estimated Effort:** trivial

---

### Medium Priority

#### DOC-009: State Manager Package Undocumented

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/state/manager.go` — `Manager`, `ManagerOptions`, `NewManager`, `StateFileName`
- `internal/state/models.go` — `StateVersion`, `SyncState`, `FileState`

**Current State:**
All exported symbols in the state package lack documentation. The state manager handles incremental sync (tracking which files have been processed).

**Proposed Content:**
Add package-level doc and type/function docs explaining the incremental sync state model.

**Rationale:**
State management is critical for large documentation extractions. Understanding the sync model helps developers debug issues with partial re-runs.

**Estimated Effort:** small

---

#### DOC-010: Wiki Parser Types and Functions Undocumented

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
- `internal/strategies/wiki_parser.go` — `WikiPage`, `WikiStructure`, `WikiSection`, `WikiInfo`, plus all parsing helpers

**Current State:**
All 10+ exported symbols in the wiki parser lack documentation.

**Proposed Content:**
Add doc comments explaining the wiki page structure and parsing logic.

**Estimated Effort:** small

---

#### DOC-011: README Go Version Mismatch

**Category:** readme  
**Target Audience:** users, contributors

**Affected Areas:**
- `README.md` (line 28)
- `go.mod`

**Current State:**
README says "Go 1.21 or later" but `go.mod` specifies `go 1.24.1`. This could confuse users with older Go versions.

**Proposed Content:**
Update README to match `go.mod`: "Go 1.24 or later."

**Rationale:**
Incorrect prerequisite information causes installation failures and user frustration.

**Estimated Effort:** trivial

---

#### DOC-012: No FAQ or Troubleshooting in README

**Category:** troubleshooting  
**Target Audience:** users

**Affected Areas:**
- `README.md`

**Current State:**
The README has no FAQ section. Common issues (browser not found, cache problems, rate limiting) are not addressed in the main documentation.

**Proposed Content:**
Add an FAQ section covering:
- "Chrome/Chromium not found" error
- "No strategy found for URL" error
- Cache clearing instructions
- Rate limiting behavior
- Output directory structure
- Manifest validation errors

**Rationale:**
Reduces support burden and improves user self-service.

**Estimated Effort:** small

---

#### DOC-013: Missing Library/API Usage Examples

**Category:** examples  
**Target Audience:** users, developers

**Affected Areas:**
- Root documentation
- `examples/` directory

**Current State:**
All examples are CLI-focused. There are no examples showing how to use RepoDocs as a Go library (importing packages, using the orchestrator programmatically).

**Proposed Content:**
Add `examples/library/` with Go code examples:
- Basic programmatic extraction
- Custom strategy implementation
- Using the converter pipeline directly
- Metadata enhancement with LLM

**Rationale:**
While primarily a CLI tool, RepoDocs is structured as a library (`internal/` packages). Users wanting to embed it in their applications need guidance.

**Estimated Effort:** medium

---

#### DOC-014: Strategy Detection Order Not Documented for Users

**Category:** architecture  
**Target Audience:** users, contributors

**Affected Areas:**
- `README.md`
- `internal/app/detector.go`

**Current State:**
The detection order (`LLMS → PkgGo → DocsRS → Sitemap → Wiki → GitHubPages → Git → Crawler`) is documented in AGENTS.md but not in user-facing documentation.

**Proposed Content:**
Add a "How URL Detection Works" section to README explaining:
1. Detection order and why it matters
2. How to force a specific strategy with `--strategy`
3. Examples of URLs and which strategy handles them

**Rationale:**
Users need to understand why a URL is handled a certain way and how to override when auto-detection fails.

**Estimated Effort:** small

---

#### DOC-015: Deprecated Types Still Referenced Without Clear Migration Path

**Category:** api_docs  
**Target Audience:** developers, maintainers

**Affected Areas:**
- `internal/domain/models.go` — `Metadata`, `MetadataIndex`, `DocumentMetadata`
- `internal/app/orchestrator.go` — Uses deprecated types internally

**Current State:**
Deprecated types have `// Deprecated:` comments but there's no documentation explaining when they were deprecated, why, or how to migrate. The AGENTS.md mentions replacements but doesn't explain the migration.

**Proposed Content:**
Add migration notes to AGENTS.md or a dedicated MIGRATION.md explaining:
- `Metadata` → `SimpleMetadata`
- `MetadataIndex` → `SimpleMetadataIndex`
- `DocumentMetadata` → `SimpleDocumentMetadata`
- Timeline for removal

**Rationale:**
Prevents confusion for contributors encountering both old and new types in the codebase.

**Estimated Effort:** small

---

### Low Priority

#### DOC-016: Package-Level Doc Files Missing for Most Packages

**Category:** api_docs  
**Target Audience:** developers, contributors

**Affected Areas:**
All internal packages except `manifest`, `strategies/git`, and `tui`.

**Current State:**
Most packages rely on AGENTS.md for architecture documentation but lack Go-style `doc.go` files with `// Package` comments.

**Proposed Content:**
Create `doc.go` files for major packages:
- `internal/app/doc.go`
- `internal/converter/doc.go`
- `internal/fetcher/doc.go`
- `internal/renderer/doc.go`
- `internal/strategies/doc.go`

**Rationale:**
Standard Go convention. Improves `go doc` output and IDE hover information.

**Estimated Effort:** small

---

#### DOC-017: bugs.md Written in Portuguese

**Category:** readme  
**Target Audience:** contributors, maintainers

**Affected Areas:**
- `bugs.md`

**Current State:**
The bugs analysis document is entirely in Portuguese. While the project may have Brazilian origins, English is the standard for open-source documentation.

**Proposed Content:**
Translate `bugs.md` to English or add an English version `bugs.en.md`.

**Rationale:**
Non-Portuguese speaking contributors cannot understand the bug analysis and regression test rationale.

**Estimated Effort:** small

---

#### DOC-018: Test README Files in Portuguese

**Category:** readme  
**Target Audience:** contributors

**Affected Areas:**
- `tests/testutil/README.md`

**Current State:**
The test utilities README is in Portuguese.

**Proposed Content:**
Translate to English.

**Estimated Effort:** trivial

---

#### DOC-019: No Architecture Diagram in README

**Category:** architecture  
**Target Audience:** users, contributors

**Affected Areas:**
- `README.md`

**Current State:**
README describes architecture in text but has no visual diagram.

**Proposed Content:**
Add an architecture diagram (Mermaid or image) showing:
- URL → Detector → Strategy → Fetcher/Renderer → Converter → Writer flow
- Optional LLM Enhancer and Metadata Collector paths

**Rationale:**
Visual learners benefit from diagrams. The pipeline is complex enough to warrant one.

**Estimated Effort:** small

---

#### DOC-020: Worker Pool Implementation Undocumented

**Category:** inline_comments  
**Target Audience:** developers

**Affected Areas:**
- `internal/utils/workerpool.go`

**Current State:**
The worker pool utility (276 lines) is a key concurrency primitive but lacks comments explaining its design.

**Proposed Content:**
Add comments explaining:
- Worker lifecycle management
- Error propagation strategy
- Shutdown behavior

**Rationale:**
Concurrency code is error-prone. Documentation helps prevent race condition bugs during modifications.

**Estimated Effort:** small

---

## Documentation Coverage Summary

| Category | Status | Notes |
|----------|--------|-------|
| README | Incomplete | Good content but missing FAQ, troubleshooting, correct Go version |
| API Docs | 85% coverage | Good for domain/converter/fetcher; poor for strategies/git, llm providers, output |
| Inline Comments | Medium | Complex algorithms need "why" comments; magic numbers need explanation |
| Examples | Incomplete | CLI examples present; no library usage examples |
| Architecture | Incomplete | Good internal AGENTS.md docs; no user-facing architecture diagram |
| Troubleshooting | Incomplete | TESTING.md has some; README lacks FAQ |

---

## Statistics

| Metric | Value |
|--------|-------|
| Total Files Analyzed | 102 non-test Go files |
| Total Markdown Files | 96 |
| Documented Functions | 244 |
| Undocumented Functions | 48 |
| Documented Types | 182 |
| Undocumented Types | 27 |
| Documentation Coverage | 85% |
| Packages with Doc Comments | 3 / 15 |

| Priority | Count |
|----------|-------|
| High | 8 |
| Medium | 7 |
| Low | 5 |

| Category | Count |
|----------|-------|
| README | 3 |
| API Docs | 8 |
| Inline Comments | 4 |
| Examples | 1 |
| Architecture | 2 |
| Troubleshooting | 2 |

---

## Recommendations

### Immediate Actions (This Sprint)
1. **Fix README Go version** (DOC-011) — 5 minutes
2. **Document LLM providers** (DOC-002) — 30 minutes
3. **Extract magic numbers to constants** (DOC-007) — 1 hour
4. **Add dual regex explanation** (DOC-008) — 15 minutes

### Short Term (Next 2 Sprints)
5. **Document git strategy sub-package** (DOC-003) — 2 hours
6. **Document output package** (DOC-004) — 1 hour
7. **Add package-level docs** (DOC-001) — 2 hours
8. **Add Rustdoc renderer comments** (DOC-006) — 2 hours

### Medium Term (Next Month)
9. **Create CONTRIBUTING.md, CHANGELOG.md, SECURITY.md** (DOC-005) — 2 hours
10. **Add FAQ and troubleshooting to README** (DOC-012) — 2 hours
11. **Add architecture diagram** (DOC-019) — 1 hour
12. **Create library usage examples** (DOC-013) — 3 hours

### Quality Gates
- All new exported symbols must have Go doc comments
- Magic numbers in new code must be named constants with comments
- Complex algorithms (>50 lines) require "why" comments explaining design decisions
- Package-level doc comments required for new packages
