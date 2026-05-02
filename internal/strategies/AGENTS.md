<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-05-01 | Updated: 2026-05-01 -->

# internal/strategies

Extraction strategies implementing `domain.Strategy`. Detection order: `LLMS → PkgGo → DocsRS → Sitemap → Wiki → GitHubPages → Git → Crawler`.

## Structure

```
├── strategy.go              # Options, Dependencies (DI container)
├── git/                     # Subpackage: archive, clone, parser, processor
│   ├── strategy.go          # GitStrategy coordinator
│   ├── archive.go           # HTTP tar.gz fetcher
│   ├── clone.go             # go-git wrapper
│   ├── parser.go            # URL → platform/owner/repo
│   ├── processor.go         # File discovery + doc conversion
│   └── types.go             # Platform enum, file filter maps
├── crawler.go               # Recursive crawler (colly)
├── sitemap.go               # sitemap.xml parser
├── github_pages.go          # SPA-aware GitHub Pages
├── pkggo.go                 # pkg.go.dev extractor
├── docsrs.go                # docs.rs Rustdoc extractor
├── docsrs_types.go          # Rustdoc JSON schema
├── docsrs_renderer.go       # Rustdoc → Markdown (complex)
├── wiki.go                  # GitHub wiki
├── llms.go                  # llms.txt extractor
└── *_discovery.go           # Sitemap/MkDocs/Docusaurus probes
```

## Where to Look

| Task | File | Notes |
|------|------|-------|
| Add strategy | New file + `detector.go` | Embed `*Dependencies`, implement 3 methods |
| Change DI wiring | `strategy.go` `NewDependencies()` | Wires all shared services |
| Git handling | `git/` subpackage | Archive vs clone; platform URLs |
| SPA detection | `github_pages.go` | `looksLikeSPAShell()`, `isEmptyOrErrorContent()` |
| Rustdoc render | `docsrs_renderer.go` | Signature formatting, type linking |
| Crawler bugs | `crawler.go`, `crawler_context.go` | Colly callbacks, visited tracking |

## Conventions

- Constructor: `NewXStrategy(deps *Dependencies)`
- `Dependencies` lazily initializes renderer via `sync.Once`
- Options embed `domain.CommonOptions` for shared fields
- File filters as global maps: `DocumentExtensions`, `IgnoreDirs`

## Anti-Patterns

- Strategy logic belongs here, NOT in `internal/app/orchestrator.go`
- Don't bypass `Dependencies` to create ad hoc service instances

## Gotchas

- `git/strategy_test.go` (~1900 lines) and `git_strategy_test.go` (1530 lines) overlap
- `docsrs_types.go` mirrors Rustdoc JSON; upstream changes break parsing
- Git subpackage uses real `exec.Command("git", ...)` in clone path


<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->