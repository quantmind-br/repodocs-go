<!-- Parent: ../../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# cmd/repodocs/ - CLI Entry Point

Single CLI entry point using Cobra framework.

## Purpose

Main CLI application. Contains all commands and flags in main.go. Handles configuration loading via Viper, runs the extraction pipeline through the internal app orchestrator, and manages interactive TUI for config initialization.

## Key Files

| File | Description |
|------|-------------|
| main.go | 539 lines - Root command, init subcommands (config, manifest, run), all CLI flags, Viper config binding |
| main_test.go | CLI command tests |

## Key Commands

- **root**: Extract documentation from a URL (`repodocs [url]`)
- **init config**: Interactive TUI config wizard
- **init manifest**: Generate manifest template
- **run**: Execute extraction from manifest file

## Flags

Global flags: `--config`, `--output`, `--concurrency`, `--limit`, `--max-depth`, `--exclude`, `--filter`, `--nofolders`, `--force`, `--verbose`, `--no-cache`, `--cache-ttl`, `--refresh-cache`, `--render-js`, `--timeout`

## Dependencies

- **Internal**: `internal/app`, `internal/config`, `internal/domain`, `internal/manifest`, `internal/tui`, `internal/utils`, `pkg/version`
- **External**: spf13/cobra, spf13/viper, gopkg.in/yaml.v3

## For AI Agents

- Entry point for running extraction: `go run cmd/repodocs/main.go [url]`
- All flags must be registered in `init()` function
- Config file defaults to `~/.repodocs/config.yaml`

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->