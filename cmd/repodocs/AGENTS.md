<!-- Parent: ../../AGENTS.md -->
<!-- Generated: 2026-04-01 | Updated: 2026-04-01 -->

# cmd/repodocs/

Single Cobra entrypoint. All CLI behavior is in `main.go`; there is no separate `run` or `manifest` subcommand.

## Files

| File | Purpose |
|------|---------|
| `main.go` | Root command, all persistent flags, `doctor`, `version`, `config *`, `--manifest` execution path |
| `main_test.go` | CLI/flag/command coverage |

## Actual Commands

- `repodocs [url]` — single-source extraction.
- `repodocs doctor` — internet/browser/write/cache checks.
- `repodocs version` — print build/version info.
- `repodocs config` — opens interactive config editor.
- `repodocs config edit|show|init|path` — explicit config subcommands.
- `repodocs --manifest path/to/file.yaml` — batch mode; still uses root command.

## Important Behaviors

- If `--manifest` is set, URL args are rejected and execution switches to `runManifest()`.
- Output directory is auto-generated from the URL unless the user explicitly passed `--output`.
- Graceful shutdown handled via `os.Interrupt`/`SIGTERM` cancellation.
- `config` command runs the edit TUI by default.
- Accessible TUI mode: `--accessible` or `ACCESSIBLE=1`.

## Flag Groups

- General: `--config`, `--output`, `--concurrency`, `--limit`, `--max-depth`, `--exclude`, `--filter`, `--nofolders`, `--force`, `--verbose`
- Cache: `--no-cache`, `--cache-ttl`, `--refresh-cache`
- Rendering: `--render-js`, `--timeout`
- Output/meta: `--json-meta`, `--dry-run`, `--split`, `--include-assets`
- Selectors: `--content-selector`, `--exclude-selector`, `--user-agent`
- Manifest/sync: `--manifest`, `--sync`, `--full-sync`, `--prune`

## Where to Look

| Task | Location |
|------|----------|
| Add/change flags | `init()` near root command |
| Change single-URL flow | `run()` |
| Change manifest flow | `runManifest()` |
| Change dependency checks | `doctorCmd`, `checkInternet`, `checkChrome`, `checkWritePermissions`, `checkCacheDir` |
| Change config UX | `configCmd` + `runConfigEdit/Show/Init` |

## Anti-Patterns

- Do not document nonexistent subcommands; manifest mode is a flag on the root command.
- Do not duplicate business logic here if it belongs in `internal/app` or lower layers.
- Keep flag registration centralized in `main.go` to match existing style.

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
