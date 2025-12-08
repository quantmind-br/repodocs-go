# Project Overview: repodocs-go

## Purpose
`repodocs-go` is a Go CLI tool that extracts documentation from diverse sources (websites, Git repos, sitemaps, pkg.go.dev) and converts it into standardized Markdown. It is designed to handle JavaScript-heavy SPAs via headless Chrome, implements stealth HTTP features for anti-bot measures, and uses persistent caching with BadgerDB.

## Module
- Module path: `github.com/quantmind-br/repodocs-go`
- Go version: 1.25.5

## Key Dependencies
| Dependency | Purpose |
|------------|---------|
| `spf13/cobra` + `spf13/viper` | CLI framework and configuration |
| `go-rod/rod` | Headless Chrome for SPA rendering |
| `bogdanfinn/tls-client` | Stealth HTTP client |
| `dgraph-io/badger/v4` | Persistent cache |
| `JohannesKaufmann/html-to-markdown/v2` | HTML to Markdown conversion |
| `go-shiori/go-readability` | Content extraction (Readability algorithm) |
| `go-git/go-git/v5` | Git repository cloning |
| `gocolly/colly/v2` | Web crawling |
| `rs/zerolog` | Structured logging |

## Entry Points
- **Main CLI**: `cmd/repodocs/main.go`
- **Commands**: `root` (main crawl), `doctor` (system check), `version`

## Strategy Types
The tool uses a Strategy pattern to handle different source types:
- `StrategyLLMS` - llms.txt files
- `StrategySitemap` - XML sitemaps
- `StrategyGit` - Git repositories
- `StrategyPkgGo` - pkg.go.dev packages
- `StrategyCrawler` - General web crawling
