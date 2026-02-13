# internal/app - Orchestration & Detection

**Generated:** 2026-02-13 | **Context:** Composition Root

## OVERVIEW
Composition root and strategy router coordinating documentation extraction from diverse sources.

## STRUCTURE
```
internal/app/
├── detector.go      # URL patterns → Strategy mapping
├── detector_test.go
├── orchestrator.go  # Main coordination, deps lifecycle, execution
└── orchestrator_test.go
```

## WHERE TO LOOK
| Task | File | Notes |
|------|------|-------|
| Add new URL detection rule | `detector.go` | Update `DetectStrategy` and `StrategyType` enum |
| Modify dependency injection | `orchestrator.go` | `NewOrchestrator` initializes `strategies.Dependencies` |
| Change main execution flow | `orchestrator.go` | `Run` handles single URLs; `RunManifest` handles batching |
| Tweak concurrency/timeouts | `orchestrator.go` | Orchestrator transforms `OrchestratorOptions` to deps |
| Fix manifest processing | `orchestrator.go` | Orchestrates multi-source logic and error tolerance |

## KEY TYPES
- `Orchestrator`: High-level runner coordinating `strategies.Dependencies` and strategy execution.
- `StrategyType`: Enum representing available extraction methods (crawler, git, sitemap, etc.).
- `Detector`: Logic for identifying the appropriate handling strategy via `DetectStrategy`.
- `OrchestratorOptions`: Main configuration struct for the execution pipeline.

## CONVENTIONS
- Follows root `AGENTS.md` regarding imports, naming, and error wrapping.
- **Interface-Driven**: Interacts with extraction logic solely through `domain.Strategy`.
- **Context First**: All public methods in `Orchestrator` accept `context.Context` for cancellation.

## ANTI-PATTERNS
- **Strategy Leaks**: Do not add strategy-specific code (e.g., git cloning details) to the orchestrator.
- **Direct Instantiation**: Avoid instantiating strategies directly; use the `StrategyFactory` injection point.
- **Side Effects in Detection**: `DetectStrategy` must be a pure function of the URL.
- See root `AGENTS.md` for general project constraints.
