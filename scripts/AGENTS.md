<!-- Parent: ../AGENTS.md -->
<!-- Generated: 2026-03-15 | Updated: 2026-03-15 -->

# scripts/ - Build and Release

Build and release automation scripts.

## Purpose

Shell scripts for release workflow management. Handles version bumping, changelog generation, git tagging, and triggers GitHub Actions for build/publish.

## Key Files

| File | Description |
|------|-------------|
| release.sh | Interactive release script - prompts for version bump (patch/minor/major/custom), validates semver, checks uncommitted changes, shows changelog since last tag, creates annotated git tag, pushes to origin |

## Usage

```bash
./scripts/release.sh
```

The script:
1. Reads current version from latest git tag
2. Prompts for version bump type (patch/minor/major/custom)
3. Validates semver format (vX.Y.Z)
4. Checks for uncommitted changes
5. Displays commits since last tag
6. Creates annotated git tag
7. Pushes tag to origin
8. GitHub Actions builds and publishes release

## Dependencies

- Git (for tags, log, status)
- GitHub Actions workflow (triggers on tag push)

## For AI Agents

- Run locally to create releases
- Tags trigger CI/CD pipeline
- Monitor: https://github.com/quantmind-br/repodocs/actions

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->