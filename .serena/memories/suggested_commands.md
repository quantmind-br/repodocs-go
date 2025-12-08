# Suggested Commands

## Build Commands
```bash
# Build binary to ./build/repodocs
make build

# Cross-compile for all platforms
make build-all

# Install to ~/.local/bin
make install
```

## Test Commands
```bash
# Run unit tests with race detection
make test

# Run integration tests only
make test-integration

# Run E2E tests
make test-e2e

# Run all tests
make test-all

# Generate HTML coverage report
make coverage

# Run a single test
go test -v -run TestName ./path/to/package/...
```

## Code Quality
```bash
# Run golangci-lint (auto-installs if missing)
make lint

# Format code with gofmt
make fmt

# Run go vet
make vet
```

## Development
```bash
# Run directly without building
make run ARGS="https://example.com -o ./output"

# Run in watch mode (requires air, auto-installs)
make dev

# Download and tidy dependencies
make deps

# Update all dependencies
make deps-update
```

## Clean Up
```bash
# Remove build artifacts and coverage reports
make clean
```

## Running the Tool
```bash
# Run built binary
./build/repodocs https://example.com -o ./output

# Check system requirements
./build/repodocs doctor

# Show version
./build/repodocs version
```

## System Utils (Linux)
Standard Linux commands available: `git`, `ls`, `cd`, `grep`, `find`, `cat`, etc.
