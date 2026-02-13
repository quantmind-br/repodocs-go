# AGENTS.md - renderer

**Generated:** 2026-02-13 | **Package:** `internal/renderer`

## Overview
Headless browser pool (Rod/Chromium) for rendering JavaScript-heavy documentation sites and SPAs.

## Structure
```
internal/renderer/
├── pool.go            # TabPool management (concurrency & recycling)
├── rod.go             # Main Renderer implementation & browser lifecycle
├── stealth.go         # Bot detection evasion (stealth.Page, viewport masking)
└── detector.go        # Heuristics for SPA detection (React, Vue, Next.js, etc.)
```

## Where to Look
| Task | File |
|------|------|
| Add/update SPA patterns | `detector.go` |
| Change browser launch flags | `rod.go` |
| Optimize tab recycling logic | `pool.go` |
| Improve bot avoidance | `stealth.go` |
| Modify JS wait/scroll logic | `rod.go` |

## Key Types
- `Renderer`: Main orchestrator implementing JS rendering via `rod.Browser`.
- `TabPool`: Manages a buffered channel of `rod.Page` instances for thread-safe reuse.
- `RendererOptions`: Configuration for timeouts, concurrency, and stealth settings.

## Conventions
- **Tab Lifecycle**: Always `Acquire` from pool and `Release` via `defer` in `Render()`.
- **Browser State**: Clean tabs before recycling (navigate to `about:blank`).
- **Context**: Every browser operation MUST respect the passed `context.Context` for timeouts.
- **Lazy Init**: Tabs are created on-demand up to `maxTabs` in the pool.
- **CI Safety**: `NoSandbox` is enabled automatically if `os.Getenv("CI")` is set.

## Anti-Patterns
- **Direct Rod Usage**: Avoid using `rod` package directly in strategies; use `Renderer`.
- **Global Browser Instance**: Do not use global browser variables; manage via `Renderer` struct.
- **Hardcoded Selectors**: Avoid hardcoded "wait for" selectors; pass via `RenderOptions`.
- **Resource Leaks**: Never return from `Acquire` without a corresponding `Release`.
- **UI Interaction**: This renderer is for extraction, not form filling or complex interaction.
