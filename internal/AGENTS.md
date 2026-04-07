<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-01 -->

# internal/

Core implementation packages. Public surface for most work starts in `app/`, shared contracts live in `domain/`, and shared runtime wiring lives in `strategies/strategy.go`.

## Subdirectories

| Directory | Purpose |
|-----------|---------|
| [app/](app/AGENTS.md) | URL detection + top-level orchestration |
| [cache/](cache/AGENTS.md) | Badger persistence for fetched content |
| [config/](config/AGENTS.md) | Config structs, defaults, loader, validation |
| [converter/](converter/AGENTS.md) | HTML/Markdown/plaintext conversion pipeline |
| [domain/](domain/AGENTS.md) | Shared interfaces, models, sentinel errors |
| [fetcher/](fetcher/AGENTS.md) | Stealth HTTP client, retry, transport adapter |
| [git/](git/AGENTS.md) | Thin go-git wrapper for DI/tests |
| [llm/](llm/AGENTS.md) | Provider factory, wrappers, metadata enrichment |
| [manifest/](manifest/AGENTS.md) | YAML/JSON multi-source manifests |
| [output/](output/AGENTS.md) | Writer + metadata collector |
| [renderer/](renderer/AGENTS.md) | Rod renderer + page pool + SPA heuristics |
| [state/](state/AGENTS.md) | Incremental sync state |
| [strategies/](strategies/AGENTS.md) | Extraction strategies + shared `Dependencies` |
| [tui/](tui/AGENTS.md) | Interactive config editor |
| [utils/](utils/AGENTS.md) | URL/fs/logger/worker utilities |

## Boundaries

- `domain/`: contracts only; no implementation logic.
- `app/`: selection + orchestration; no source-specific extraction details.
- `strategies/`: source-specific logic plus shared runtime wiring (`NewDependencies`, lazy renderer, state save/prune helpers).
- `utils/`: shared leaf helpers; safe to depend on broadly.

## Shared Patterns

- Most packages consume `domain` interfaces or `strategies.Dependencies`.
- `app.NewOrchestrator(...)` converts CLI/config state into `strategies.DependencyOptions`.
- Rendering is lazy unless explicitly enabled.
- Metadata collection and sync state are optional features, not always-on core paths.

## Anti-Patterns

- Do not duplicate interfaces outside `domain/`.
- Do not instantiate browser/fetch/cache logic ad hoc inside strategies when `Dependencies` already provides it.
- Do not move source detection logic out of `app/detector.go`.

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
