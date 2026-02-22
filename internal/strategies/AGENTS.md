# AGENTS.md - internal/strategies

**Generated:** 2026-02-20 | **Package:** internal/strategies

Documentation extraction strategies implementing the Strategy interface.

## Strategies

| Strategy | Files | Handles |
|----------|-------|---------|
| LLMS | `llms.go` | Sites with `/llms.txt` or `/llms-full.txt` |
| PkgGo | `pkggo.go` | `pkg.go.dev/*` Go package docs |
| DocsRS | `docsrs.go`, `docsrs_*.go` | `docs.rs/*` Rust crate docs (JSON API) |
| Sitemap | `sitemap.go` | Sites with `sitemap.xml` (gzip supported) |
| Git | `git/*.go` | Git repos with `/docs` or markdown files |
| GitHub Pages | `github_pages*.go` | `*.github.io` sites (SPA-aware) |
| Wiki | `wiki.go`, `wiki_parser.go` | GitHub/GitLab wiki repos |
| Crawler | `crawler.go` | Fallback: any HTTP/HTTPS URL |

Detection order follows this table (top = highest priority).

## Where to Look

| Task | File |
|------|------|
| Add URL detection logic | Strategy's `CanHandle()` method |
| Modify extraction flow | Strategy's `Execute()` method |
| Change shared deps | `strategy.go` → `Dependencies` struct |
| Rustdoc JSON parsing | `docsrs_types.go`, `docsrs_json.go` |
| Rustdoc → Markdown | `docsrs_renderer.go` (628 lines, complex) |
| GitHub Pages SPA detection | `github_pages_discovery.go` |
| Git clone/archive logic | `git/clone.go`, `git/archive.go` |
| Wiki markdown parsing | `wiki_parser.go` |

## Adding a New Strategy

1. Create `newstrategy.go` with struct embedding `*Dependencies`
2. Implement interface:
   ```go
   func (s *NewStrategy) Name() string
   func (s *NewStrategy) CanHandle(url string) bool
   func (s *NewStrategy) Execute(ctx context.Context, url string, opts Options) error
   ```
3. Add constructor: `NewNewStrategy(deps *Dependencies) *NewStrategy`
4. Register in `internal/app/detector.go` detection order
5. Add tests in `newstrategy_test.go`

## Shared Patterns

**All strategies use Dependencies for:**
- `deps.Fetcher` - HTTP requests with caching
- `deps.Renderer` - Headless browser (JS sites)
- `deps.Converter` - HTML → Markdown pipeline
- `deps.Writer` - Output with frontmatter
- `deps.WriteDocument()` - Standard doc output flow

**Common method signatures:**
```go
func (s *XStrategy) SetFetcher(f domain.Fetcher)  // Testing injection
```

## Complexity Notes

- **DocsRS**: Own type system (`docsrs_types.go`), JSON parser (`docsrs_json.go`), and Markdown renderer (`docsrs_renderer.go`). Handles Rust generics, lifetimes, trait bounds.
- **GitHub Pages**: Multi-phase discovery (HTTP probes → browser render → link extraction). SPA shell detection in `looksLikeSPAShell()`.
- **Git**: Subpackage with clone/archive/processor separation. Supports both full clone and archive fetch.
