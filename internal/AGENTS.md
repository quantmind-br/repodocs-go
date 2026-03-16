<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/ - Core Packages

Container directory for all internal packages implementing the documentation extraction engine.

## Purpose

Provides all core functionality: strategy routing, HTTP fetching, HTML-to-Markdown conversion, caching, AI integration, and output generation. Packages are interface-driven with the Strategy pattern for extensibility.

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| [app/](app/AGENTS.md) | Orchestrator + Detector (strategy routing) |
| [strategies/](strategies/AGENTS.md) | 8 extraction strategies (crawler, git, sitemap, llms, wiki, pkggo, docsrs, github_pages) |
| [converter/](converter/AGENTS.md) | HTML to Markdown pipeline (readability, sanitizer, encoding) |
| [fetcher/](fetcher/AGENTS.md) | Stealth HTTP client (tls-client, bot avoidance) |
| [renderer/](renderer/AGENTS.md) | Headless browser pool (Rod/Chromium) |
| [cache/](cache/AGENTS.md) | BadgerDB persistence |
| [llm/](llm/AGENTS.md) | Multi-provider AI (OpenAI, Anthropic, Google) + circuit breakers |
| [output/](output/AGENTS.md) | Markdown writer with YAML frontmatter |
| [domain/](domain/AGENTS.md) | Interfaces, models, sentinel errors |
| [tui/](tui/AGENTS.md) | Interactive config (Bubble Tea/Huh) |
| [config/](config/AGENTS.md) | YAML config handling |
| [git/](git/AGENTS.md) | Git client wrapper |
| [manifest/](manifest/AGENTS.md) | Multi-source manifest loading |
| [state/](state/AGENTS.md) | Sync state management |
| [utils/](utils/AGENTS.md) | Shared utilities |

## Dependencies

- **Internal**: All packages depend on `domain/` for interfaces and models
- **External**: tls-client, rod, badgerdb, openaianthropic google-go-ai, cobra, viper

## For AI Agents

- All public APIs defined in `domain/` interfaces
- Strategy pattern in `strategies/` for adding new extraction sources
- Use `app/` orchestrator to run extraction pipelines
- Cache layer in `cache/` uses BadgerDB key-value store

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->