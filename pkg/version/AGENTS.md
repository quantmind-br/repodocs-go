<!-- Parent: ../../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# pkg/version/ - Version Information

Public version info package with build-time variables.

## Purpose

Provides version information for the CLI binary. Build-time variables set via ldflags: Version, BuildTime, Commit. Exposes Info struct with methods for different output formats.

## Key Files

| File | Description |
|------|-------------|
| version.go | Build-time ldflags (Version, BuildTime, Commit), Info struct with Get(), String(), Short(), Full() |
| version_test.go | Version info tests |

## API

```go
var Version   = "dev"    // Set via -ldflags "-X version.Version=x.y.z"
var BuildTime = "unknown" // Set via -ldflags "-X version.BuildTime=..."
var Commit    = "unknown" // Set via -ldflags "-X version.Commit=..."

type Info struct {
    Version   string
    BuildTime string
    Commit    string
    GoVersion string
    OS        string
    Arch      string
}

func Get() Info     // Returns full Info struct
func Short() string // Returns just Version
func Full() string  // Returns formatted string: "repodocs x.y.z (commit: ..., built: ..., go os/arch)"
```

## For AI Agents

- Import: `github.com/quantmind-br/repodocs/pkg/version`
- Build command includes ldflags for version injection
- Used by CLI root command for `--version` flag

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->