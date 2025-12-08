# Request Flow Analysis
The `repodocs-go` project is a Command Line Interface (CLI) application. Therefore, the "request flow" is interpreted as the execution flow of the main command, tracing how command-line arguments and configuration are processed to initiate the core documentation extraction logic.

## Entry Points Overview
The system's primary entry point is the `main` function in `cmd/repodocs/main.go`.

1.  **`main()`**: Calls `rootCmd.Execute()`, which initiates the `cobra` command-line parsing and execution framework.
2.  **`rootCmd`**: The main command, defined with `Use: "repodocs [url]"`.
3.  **`RunE: run`**: The function executed when the main command is run with a URL argument. This function serves as the primary controller for the entire application lifecycle.
4.  **Subcommands**: `doctorCmd` (for checking dependencies) and `versionCmd` (for displaying version information) are secondary entry points.

## Request Routing Map
The routing mechanism is handled by the `cobra` library, mapping command-line input to specific Go functions.

| Command | Handler Function | Description |
| :--- | :--- | :--- |
| `repodocs [url]` | `run(cmd *cobra.Command, args []string)` | The core extraction logic. |
| `repodocs doctor` | Anonymous function in `doctorCmd` | Checks system dependencies (e.g., Chrome/Chromium for rendering). |
| `repodocs version` | `versionCmd` (implicitly handled by Cobra) | Displays the application version. |

The `run` function's internal routing is based on the input URL:
1.  `url := args[0]` is extracted.
2.  `strategyType := DetectStrategy(url)` is called (`internal/app/detector.go`).
3.  The detected `StrategyType` (e.g., `StrategyGit`, `StrategySitemap`, `StrategyCrawler`) determines which concrete `strategies.Strategy` implementation is created and executed via `CreateStrategy` and `strategy.Execute()`.

## Middleware Pipeline
In this CLI context, the "middleware pipeline" consists of initialization and configuration loading steps that preprocess the command execution before the core logic runs.

1.  **`cobra.OnInitialize(initConfig)`**:
    *   **Purpose**: Configuration loading.
    *   **Action**: If the `--config` flag is set, `viper` is configured to use that file.
2.  **`run` Function Preprocessing**:
    *   **Logger Initialization**: `utils.NewLogger` is created based on the `--verbose` flag.
    *   **Configuration Loading**: `config.Load()` reads configuration from files and binds all persistent flags (e.g., `output`, `concurrency`, `cache-ttl`) to the configuration structure via `viper.BindPFlag`.
    *   **Argument Validation**: Checks for the presence of the required `[url]` argument.
    *   **Output Directory Resolution**: Determines the final output directory, prioritizing the explicit `-o` flag over the URL-based default.
    *   **Context Creation**: A cancellable `context.Context` is created for graceful shutdown.
    *   **Signal Handling**: A goroutine is launched to listen for `SIGINT` or `SIGTERM` signals, which trigger context cancellation.

## Controller/Handler Analysis
The `run` function and the `app.Orchestrator` are the primary controllers.

### 1. `run` Function (Main Controller)
*   **Input**: `*cobra.Command` and `[]string` (arguments).
*   **Action**: Gathers all configuration and flags into `app.OrchestratorOptions`.
*   **Delegation**: Creates and delegates execution to the `Orchestrator`.

### 2. `app.Orchestrator` (Core Processor)
*   **Initialization (`NewOrchestrator`)**:
    *   Creates the application's core dependencies (`strategies.Dependencies`), which includes:
        *   HTTP Client/Fetcher.
        *   BadgerDB Cache.
        *   Rod-based Renderer Pool (for JavaScript rendering).
        *   Worker Pool (`utils.WorkerPool`).
*   **Validation (`ValidateURL`)**:
    *   Calls `DetectStrategy(url)` to ensure the URL matches a known extraction strategy.
*   **Execution (`Run`)**:
    *   **Strategy Detection**: Identifies the appropriate strategy (`Git`, `Sitemap`, `Crawler`, etc.).
    *   **Strategy Creation**: Instantiates the concrete strategy (e.g., `strategies.NewCrawlerStrategy`).
    *   **Execution**: Calls `strategy.Execute(ctx, url, strategyOpts)`. This is where the main work (fetching, converting, and writing) occurs, often utilizing the internal worker pool and dependencies.

## Authentication & Authorization Flow
The application is a documentation extraction CLI tool and does not implement a traditional user authentication or authorization flow.

*   **External Credentials**: The system may implicitly handle credentials for external services (e.g., private Git repositories) if the underlying `git` command or fetcher is configured to use them, but there is no explicit, internal authentication mechanism defined in the core flow.
*   **Access Control**: Access control is limited to the user's permissions on the local machine to execute the binary and write to the output directory.

## Error Handling Pathways
Error handling is structured to provide informative messages to the user and ensure graceful resource cleanup.

1.  **CLI Execution Errors**:
    *   If `rootCmd.Execute()` returns an error (e.g., from `run`), the error is printed to `os.Stderr`, and the application exits with `os.Exit(1)`.
2.  **Configuration/Initialization Errors**:
    *   Errors from `config.Load()` or `app.NewOrchestrator()` are wrapped with context (e.g., "failed to load config") and returned by `run`, leading to a clean exit.
3.  **Validation Errors**:
    *   If the URL is missing or `orchestrator.ValidateURL` fails (unsupported format), an error is returned, and the process stops before resource-intensive operations begin.
4.  **Strategy Execution Errors**:
    *   Errors from `strategy.Execute` are wrapped ("strategy execution failed") and returned.
    *   **Context Cancellation**: The `Run` function explicitly checks `if ctx.Err() != nil` after `strategy.Execute` returns. This allows the application to distinguish between a genuine execution failure and a user-initiated graceful shutdown (via `SIGINT`/`SIGTERM`).
5.  **Resource Cleanup**:
    *   `defer orchestrator.Close()` in `run` ensures that resources (like the BadgerDB cache and the Rod browser pool) are released, even if an error occurs during execution.

## Request Lifecycle Diagram

```mermaid
graph TD
    A[CLI Execution: repodocs [url]] --> B(Cobra: Parse Flags & Args);
    B --> C{run Function Start};
    C --> D(Init Logger & Load Config);
    D --> E(Create Context & Signal Handler);
    E --> F(Validate URL & Determine Output Dir);
    F --> G(app.NewOrchestrator);
    G --> H(Init Dependencies: Cache, Fetcher, Renderer);
    H --> I(Detect Strategy: Git, Sitemap, Crawler, etc.);
    I --> J(Create Concrete Strategy);
    J --> K(strategy.Execute(ctx, url, opts));
    K --> L{Strategy Logic: Fetch, Convert, Write};
    L --> M{Execution Complete?};
    M -- Success --> N(Log Completion & Duration);
    M -- Error --> O(Handle Error & Check Context Cancellation);
    O -- Cancelled --> P(Log Shutdown);
    O -- Failed --> Q(Return Error);
    N --> R(orchestrator.Close());
    Q --> R;
    P --> R;
    R --> S[CLI Exit];
```