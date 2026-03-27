<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# internal/strategies/git

Git repository extraction strategy.

## Purpose

Extracts documentation from Git repositories (GitHub, GitLab, Bitbucket). Supports both HTTP archive download (faster) and git clone (fallback). Handles URL parsing, platform detection, file discovery, and conversion.

## Architecture

- **Strategy**: Coordinator implementing strategies.Strategy interface
- **Parser**: URL parsing and platform detection
- **ArchiveFetcher**: HTTP-based tar.gz download and extraction
- **CloneFetcher**: go-git based repository cloning
- **Processor**: File discovery and document conversion

## Key Files

| File | Description |
|------|-------------|
| `doc.go` | Package documentation |
| `strategy.go` | Strategy struct implementing strategies.Strategy interface. Coordinates fetch+process. |
| `types.go` | Platform enum (GitHub/GitLab/Bitbucket/Generic), RepoInfo, GitURLInfo, FetchResult, DocumentExtensions, ConfigExtensions, IgnoreDirs |
| `parser.go` | URL parsing, platform detection, branch/subpath extraction |
| `archive.go` | HTTP-based tar.gz download and extraction |
| `clone.go` | go-git based repository cloning |
| `fetcher.go` | Fetch coordinator (archive first, clone fallback) |
| `processor.go` | File discovery, filtering, document conversion |
| `strategy_test.go` | Tests |

## Types

- **Platform**: github, gitlab, bitbucket, generic
- **RepoInfo**: Platform, Owner, Repo, URL
- **GitURLInfo**: RepoURL, Platform, Owner, Repo, Branch, SubPath
- **FetchResult**: LocalPath, Branch, Method

## File Filters

- **DocumentExtensions**: .md, .mdx
- **ConfigExtensions**: .json, .yaml, .yml, .toml, .env
- **IgnoreDirs**: .git, node_modules, vendor, __pycache__, .venv, venv, dist, build, .next, .nuxt

## Dependencies

- **External**: github.com/go-git/go-git/v5
- **Internal**: github.com/quantmind-br/repodocs/internal/domain, github.com/quantmind-br/repodocs/internal/output, github.com/quantmind-br/repodocs/internal/state, github.com/quantmind-br/repodocs/internal/utils

## For AI Agents

- CanHandle() detects git URLs: git@, .git suffix, github.com/gitlab.com/bitbucket.org (excludes /blob/, /-/blob/)
- Excludes: docs.github.com, pages.github.io, wiki URLs
- TryArchiveDownload() uses main branch, falls back to master
- CloneRepository() fallback when archive fails
- FilterPath supports subdirectory extraction (e.g., /docs)
- SSH URLs not supported for archive download

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->