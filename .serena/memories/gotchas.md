# Known Gotchas and Important Notes

## Runtime Requirements

### Chromium/Chrome Required
The renderer package needs local Chromium or Chrome installation. Run `repodocs doctor` to verify system requirements are met.

### Non-Standard HTTP Clients
The fetcher uses `fhttp` and `tls-client` for stealth features. These behave differently from standard `net/http`:
- Different TLS fingerprinting
- Custom HTTP/2 settings
- May require different error handling patterns

## Caching

### Persistent Cache
BadgerDB cache persists between runs. If encountering stale data issues:
- Clear the cache directory manually, OR
- Use `--no-cache` flag

Cache location is configured through the config system.

## Concurrency

### Tab Pooling
The renderer manages a browser tab pool for parallel rendering. When debugging concurrency issues:
- Check `internal/renderer/pool.go`
- Monitor tab lifecycle and pool state
- Be aware of potential resource leaks

## Testing

### Test Categories
- Unit tests: `make test` (use `-short` flag)
- Integration tests: `make test-integration` (may require network)
- E2E tests: `make test-e2e` (requires full system setup)

### Test Data
- Fixtures in `tests/testdata/fixtures/`
- Golden files in `tests/testdata/golden/`

## Configuration

### Viper Binding
The project uses Viper for configuration. All settings flow through `config.Config` struct. Be aware of:
- Environment variable binding
- Config file precedence
- Flag binding order

## Build

### CGO Disabled
Builds use `CGO_ENABLED=0` for fully static binaries. This affects:
- No C library dependencies
- Portable across Linux distributions
- Some Go packages may have reduced functionality
