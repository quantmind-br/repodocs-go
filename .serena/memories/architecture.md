# Architecture: Hexagonal Architecture with Strategy Pattern

## High-Level Flow
```
cmd/repodocs/main.go          CLI entry point (Cobra/Viper)
        ↓
internal/app/orchestrator.go  Central coordinator
        ↓
internal/app/detector.go      Selects strategy based on URL pattern
        ↓
internal/strategies/*         Strategy implementations
        ↓
internal/strategies/strategy.go  Dependencies struct (Composition Root)
```

## Package Structure

| Package | Responsibility |
|---------|---------------|
| `cmd/repodocs` | CLI entry point with Cobra commands |
| `internal/app` | Orchestrator and strategy detector |
| `internal/domain` | Interfaces and models (core contracts) |
| `internal/strategies` | Strategy implementations + Dependencies DI |
| `internal/fetcher` | Stealth HTTP client with retry |
| `internal/renderer` | Headless Chrome via rod, tab pooling |
| `internal/converter` | Pipeline: Encoding → Readability → Sanitization → Markdown |
| `internal/cache` | BadgerDB persistent cache |
| `internal/output` | Markdown + JSON metadata file writing |
| `internal/config` | Configuration management |
| `internal/utils` | Utilities including logging |
| `pkg/version` | Version information |

## Key Interfaces (internal/domain/interfaces.go)
- `Strategy` - Documentation extraction strategy
- `Fetcher` - HTTP fetching
- `Renderer` - Browser rendering for SPAs
- `Cache` - Caching layer
- `Converter` - HTML to Markdown conversion
- `Writer` - Output writing

## Key Models (internal/domain/models.go)
- `Document` - Extracted document with metadata
- `Page` - Web page representation
- `CacheEntry` - Cache storage entry
- `Sitemap` / `SitemapURL` - Sitemap structures
- `LLMSLink` - llms.txt link entries

## Composition Root
All service instantiation happens in `strategies.NewDependencies()`. This is the single place where all infrastructure components are created and wired together.

## Design Rules
1. **Depend on interfaces**: Infrastructure packages import `internal/domain` interfaces, not concrete types
2. **Composition Root**: All instantiation in `strategies.NewDependencies()`
3. **Strategy detection**: Update `internal/app/detector.go` when adding new source types
4. **Pipeline sequence**: Converter steps run in order: Encoding → Readability → Sanitization → Markdown
