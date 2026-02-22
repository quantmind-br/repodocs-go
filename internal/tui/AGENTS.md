# AGENTS.md - internal/tui

Interactive terminal UI for configuration using Bubble Tea + Huh forms.

## Structure

| File | Purpose |
|------|---------|
| `app.go` | Main TUI application, state machine |
| `forms.go` | 409 lines - Form field definitions, validation |
| `config_adapter.go` | Config file read/write |
| `categories.go` | Config category organization |
| `styles.go` | Lipgloss styling |
| `validation.go` | Input validation rules |

## Where to Look

| Task | File |
|------|------|
| Add config field | `forms.go` - field definitions |
| Change validation | `validation.go` |
| Modify category structure | `categories.go` |
| Style changes | `styles.go` |
| Config persistence | `config_adapter.go` |

## Key Types

```go
type App struct { /* Bubble Tea model */ }
type ConfigForm struct { /* Huh form definition */ }
```

## Conventions

- Uses `huh` library for form fields
- Uses `lipgloss` for styling
- State machine in `app.go` for navigation
- Validation rules in `validation.go`

## Notes

- **forms.go (409 lines)**: Complex state machine for config flow
- Accessible mode via `ACCESSIBLE=1` env var
