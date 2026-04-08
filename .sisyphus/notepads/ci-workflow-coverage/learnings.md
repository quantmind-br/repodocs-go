## Learnings

- `go test -short ./... -coverprofile=coverage.out -covermode=atomic` produces lower package coverage than full runs when tests honor `testing.Short()`.
- Reusing the initial `coverage.out` and filtering it per package avoids rerunning tests in the coverage-report step.
- The workflow thresholds for `internal/converter` and `internal/config` needed to match short-test coverage, not full-suite coverage.
