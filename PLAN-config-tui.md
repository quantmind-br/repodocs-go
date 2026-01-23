# Plano de Implementacao: TUI Interativa para Configuracao (repodocs config)

## Resumo Executivo

Implementar uma interface TUI (Terminal User Interface) interativa para configuracao da aplicacao RepoDocs, acionada pelo comando `repodocs config`. A TUI permitira que usuarios visualizem, editem e salvem configuracoes de forma intuitiva, substituindo a necessidade de editar manualmente arquivos YAML ou passar multiplas flags CLI.

A implementacao usara a biblioteca **charmbracelet/huh** (construida sobre bubbletea), que oferece componentes de formulario prontos para uso, validacao integrada e excelente acessibilidade.

---

## Analise de Requisitos

### Requisitos Funcionais

- [ ] Comando `repodocs config` como ponto de entrada para a TUI
- [ ] Subcomando `repodocs config show` para exibir configuracao atual (nao-interativo)
- [ ] Subcomando `repodocs config init` para criar arquivo de configuracao inicial
- [ ] Subcomando `repodocs config edit` (ou default) para editar interativamente
- [ ] Navegacao por categorias de configuracao (Output, Concurrency, Cache, Rendering, Stealth, LLM, Logging)
- [ ] Inputs apropriados para cada tipo de dado:
  - Text inputs para strings (directory, user_agent, api_key)
  - Number inputs para inteiros (workers, max_depth, max_tokens)
  - Select/dropdown para opcoes limitadas (log level, log format, llm provider)
  - Toggle/confirm para booleanos (cache.enabled, rendering.force_js)
  - Duration inputs para time.Duration (timeout, cache_ttl)
- [ ] Validacao em tempo real dos valores inseridos
- [ ] Preview das mudancas antes de salvar
- [ ] Salvar configuracao em `~/.repodocs/config.yaml`
- [ ] Cancelamento seguro (Ctrl+C / Esc) sem salvar

### Requisitos Nao-Funcionais

- [ ] **Performance**: TUI deve iniciar em < 100ms
- [ ] **Acessibilidade**: Suporte a modo acessivel para screen readers (huh.WithAccessible)
- [ ] **UX**: Navegacao intuitiva com teclas padrao (Tab, Enter, setas, j/k)
- [ ] **Compatibilidade**: Funcionar em terminais comuns (iTerm, Terminal.app, Windows Terminal, xterm)
- [ ] **Responsividade**: Adaptar-se ao tamanho do terminal
- [ ] **Consistencia**: Seguir padroes visuais do ecossistema Charm (cores, bordas)

---

## Analise Tecnica

### Arquitetura Proposta

```
cmd/repodocs/main.go
       |
       +-- configCmd (novo)
              |
              +-- config show  -> Exibe YAML atual
              +-- config init  -> Cria config inicial
              +-- config edit  -> Abre TUI (default)
                     |
                     v
            internal/tui/
                  |
                  +-- app.go        (Model principal, Init/Update/View)
                  +-- forms.go      (Definicoes dos formularios huh)
                  +-- styles.go     (Temas e estilos lipgloss)
                  +-- categories.go (Menus de navegacao por categoria)
                  +-- validation.go (Funcoes de validacao)
                  +-- config_adapter.go (Conversao Config <-> Form values)
                     |
                     v
            internal/config/
                  |
                  +-- loader.go (usa viper.WriteConfigAs para salvar)
```

### Componentes Afetados

| Arquivo/Modulo | Tipo de Mudanca | Descricao |
|----------------|-----------------|-----------|
| `cmd/repodocs/main.go` | Modificar | Adicionar `configCmd` com subcomandos |
| `internal/tui/` | Criar | Novo pacote para logica da TUI |
| `internal/tui/app.go` | Criar | Model principal do bubbletea |
| `internal/tui/forms.go` | Criar | Definicoes de formularios huh por categoria |
| `internal/tui/styles.go` | Criar | Temas e estilos visuais |
| `internal/tui/categories.go` | Criar | Menu de navegacao entre categorias |
| `internal/tui/validation.go` | Criar | Validadores customizados |
| `internal/tui/config_adapter.go` | Criar | Adapter entre Config struct e form values |
| `internal/config/loader.go` | Modificar | Adicionar funcao `Save(*Config) error` |
| `go.mod` | Modificar | Adicionar dependencias huh, lipgloss |

### Dependencias a Adicionar

```go
// go.mod
require (
    github.com/charmbracelet/huh v0.8.0
    github.com/charmbracelet/lipgloss v1.1.0
    github.com/charmbracelet/bubbletea v1.3.4  // dependencia de huh
)
```

### Estrutura do Pacote TUI

```
internal/tui/
├── app.go              # tea.Model principal, orquestra navegacao
├── forms.go            # Definicoes de formularios huh
├── styles.go           # Theme e estilos lipgloss
├── categories.go       # Lista de categorias, menu principal
├── validation.go       # Validadores (ValidatePort, ValidateDuration, etc)
├── config_adapter.go   # Mapeia Config <-> form values
└── tui_test.go         # Testes unitarios
```

---

## Plano de Implementacao

### Fase 1: Setup e Infraestrutura

**Objetivo**: Configurar dependencias e estrutura basica do pacote TUI

#### Tarefas:

1. **Adicionar dependencias ao go.mod**
   - Arquivos: `go.mod`
   - Comando: `go get github.com/charmbracelet/huh@latest`

   ```bash
   go get github.com/charmbracelet/huh@v0.8.0
   go get github.com/charmbracelet/lipgloss@v1.1.0
   ```

2. **Criar estrutura do pacote internal/tui/**
   - Criar diretorio `internal/tui/`
   - Criar arquivos vazios iniciais

3. **Criar styles.go com tema base**
   - Arquivo: `internal/tui/styles.go`

   ```go
   package tui

   import "github.com/charmbracelet/lipgloss"

   var (
       // Cores do tema
       primaryColor   = lipgloss.AdaptiveColor{Light: "#5A56E0", Dark: "#7571F9"}
       successColor   = lipgloss.AdaptiveColor{Light: "#02BA84", Dark: "#02BF87"}
       errorColor     = lipgloss.AdaptiveColor{Light: "#FE5F86", Dark: "#FE5F86"}
       mutedColor     = lipgloss.AdaptiveColor{Light: "#9B9B9B", Dark: "#5C5C5C"}

       // Estilos base
       TitleStyle = lipgloss.NewStyle().
           Bold(true).
           Foreground(primaryColor).
           MarginBottom(1)

       DescriptionStyle = lipgloss.NewStyle().
           Foreground(mutedColor)

       SuccessStyle = lipgloss.NewStyle().
           Foreground(successColor)

       ErrorStyle = lipgloss.NewStyle().
           Foreground(errorColor)

       BoxStyle = lipgloss.NewStyle().
           Border(lipgloss.RoundedBorder()).
           BorderForeground(primaryColor).
           Padding(1, 2)
   )

   // GetTheme returns the huh theme for forms
   func GetTheme() *huh.Theme {
       return huh.ThemeCharm()
   }
   ```

---

### Fase 2: Config Adapter

**Objetivo**: Criar camada de adaptacao entre Config struct e valores do formulario

#### Tarefas:

1. **Criar config_adapter.go**
   - Arquivo: `internal/tui/config_adapter.go`

   ```go
   package tui

   import (
       "time"
       "github.com/quantmind-br/repodocs-go/internal/config"
   )

   // ConfigValues holds form values that map to Config struct
   type ConfigValues struct {
       // Output
       OutputDirectory string
       OutputFlat      bool
       OutputOverwrite bool
       JSONMetadata    bool

       // Concurrency
       Workers  int
       Timeout  string // Duration as string for form
       MaxDepth int

       // Cache
       CacheEnabled   bool
       CacheTTL       string
       CacheDirectory string

       // Rendering
       ForceJS     bool
       JSTimeout   string
       ScrollToEnd bool

       // Stealth
       UserAgent      string
       RandomDelayMin string
       RandomDelayMax string

       // Logging
       LogLevel  string
       LogFormat string

       // LLM
       LLMProvider    string
       LLMAPIKey      string
       LLMBaseURL     string
       LLMModel       string
       LLMMaxTokens   int
       LLMTemperature float64
   }

   // FromConfig converts a Config to ConfigValues for form editing
   func FromConfig(cfg *config.Config) *ConfigValues {
       return &ConfigValues{
           OutputDirectory: cfg.Output.Directory,
           OutputFlat:      cfg.Output.Flat,
           OutputOverwrite: cfg.Output.Overwrite,
           JSONMetadata:    cfg.Output.JSONMetadata,

           Workers:  cfg.Concurrency.Workers,
           Timeout:  cfg.Concurrency.Timeout.String(),
           MaxDepth: cfg.Concurrency.MaxDepth,

           CacheEnabled:   cfg.Cache.Enabled,
           CacheTTL:       cfg.Cache.TTL.String(),
           CacheDirectory: cfg.Cache.Directory,

           ForceJS:     cfg.Rendering.ForceJS,
           JSTimeout:   cfg.Rendering.JSTimeout.String(),
           ScrollToEnd: cfg.Rendering.ScrollToEnd,

           UserAgent:      cfg.Stealth.UserAgent,
           RandomDelayMin: cfg.Stealth.RandomDelayMin.String(),
           RandomDelayMax: cfg.Stealth.RandomDelayMax.String(),

           LogLevel:  cfg.Logging.Level,
           LogFormat: cfg.Logging.Format,

           LLMProvider:    cfg.LLM.Provider,
           LLMAPIKey:      cfg.LLM.APIKey,
           LLMBaseURL:     cfg.LLM.BaseURL,
           LLMModel:       cfg.LLM.Model,
           LLMMaxTokens:   cfg.LLM.MaxTokens,
           LLMTemperature: cfg.LLM.Temperature,
       }
   }

   // ToConfig converts ConfigValues back to a Config struct
   func (v *ConfigValues) ToConfig() (*config.Config, error) {
       timeout, err := time.ParseDuration(v.Timeout)
       if err != nil {
           return nil, fmt.Errorf("invalid timeout: %w", err)
       }
       cacheTTL, err := time.ParseDuration(v.CacheTTL)
       if err != nil {
           return nil, fmt.Errorf("invalid cache_ttl: %w", err)
       }
       jsTimeout, err := time.ParseDuration(v.JSTimeout)
       if err != nil {
           return nil, fmt.Errorf("invalid js_timeout: %w", err)
       }
       delayMin, err := time.ParseDuration(v.RandomDelayMin)
       if err != nil {
           return nil, fmt.Errorf("invalid random_delay_min: %w", err)
       }
       delayMax, err := time.ParseDuration(v.RandomDelayMax)
       if err != nil {
           return nil, fmt.Errorf("invalid random_delay_max: %w", err)
       }

       return &config.Config{
           Output: config.OutputConfig{
               Directory:    v.OutputDirectory,
               Flat:         v.OutputFlat,
               Overwrite:    v.OutputOverwrite,
               JSONMetadata: v.JSONMetadata,
           },
           Concurrency: config.ConcurrencyConfig{
               Workers:  v.Workers,
               Timeout:  timeout,
               MaxDepth: v.MaxDepth,
           },
           Cache: config.CacheConfig{
               Enabled:   v.CacheEnabled,
               TTL:       cacheTTL,
               Directory: v.CacheDirectory,
           },
           Rendering: config.RenderingConfig{
               ForceJS:     v.ForceJS,
               JSTimeout:   jsTimeout,
               ScrollToEnd: v.ScrollToEnd,
           },
           Stealth: config.StealthConfig{
               UserAgent:      v.UserAgent,
               RandomDelayMin: delayMin,
               RandomDelayMax: delayMax,
           },
           Logging: config.LoggingConfig{
               Level:  v.LogLevel,
               Format: v.LogFormat,
           },
           LLM: config.LLMConfig{
               Provider:    v.LLMProvider,
               APIKey:      v.LLMAPIKey,
               BaseURL:     v.LLMBaseURL,
               Model:       v.LLMModel,
               MaxTokens:   v.LLMMaxTokens,
               Temperature: v.LLMTemperature,
           },
       }, nil
   }
   ```

---

### Fase 3: Validadores Customizados

**Objetivo**: Criar funcoes de validacao para os campos do formulario

#### Tarefas:

1. **Criar validation.go**
   - Arquivo: `internal/tui/validation.go`

   ```go
   package tui

   import (
       "errors"
       "fmt"
       "os"
       "path/filepath"
       "strconv"
       "strings"
       "time"
   )

   // ValidateDuration validates a duration string
   func ValidateDuration(s string) error {
       if s == "" {
           return errors.New("duration cannot be empty")
       }
       _, err := time.ParseDuration(s)
       if err != nil {
           return fmt.Errorf("invalid duration format (use: 30s, 5m, 1h): %v", err)
       }
       return nil
   }

   // ValidatePositiveInt validates a positive integer string
   func ValidatePositiveInt(s string) error {
       n, err := strconv.Atoi(s)
       if err != nil {
           return errors.New("must be a valid number")
       }
       if n < 1 {
           return errors.New("must be at least 1")
       }
       return nil
   }

   // ValidateDirectory validates a directory path
   func ValidateDirectory(s string) error {
       if s == "" {
           return errors.New("directory cannot be empty")
       }
       // Expand ~ to home directory
       if strings.HasPrefix(s, "~") {
           home, err := os.UserHomeDir()
           if err != nil {
               return nil // Allow it, we'll handle at runtime
           }
           s = filepath.Join(home, s[1:])
       }
       // Check if parent exists (for new directories)
       parent := filepath.Dir(s)
       if _, err := os.Stat(parent); os.IsNotExist(err) {
           return fmt.Errorf("parent directory does not exist: %s", parent)
       }
       return nil
   }

   // ValidateTemperature validates LLM temperature (0.0 - 2.0)
   func ValidateTemperature(s string) error {
       f, err := strconv.ParseFloat(s, 64)
       if err != nil {
           return errors.New("must be a valid decimal number")
       }
       if f < 0 || f > 2 {
           return errors.New("temperature must be between 0.0 and 2.0")
       }
       return nil
   }

   // ValidateLogLevel validates log level option
   func ValidateLogLevel(s string) error {
       valid := map[string]bool{"debug": true, "info": true, "warn": true, "error": true}
       if !valid[strings.ToLower(s)] {
           return errors.New("must be one of: debug, info, warn, error")
       }
       return nil
   }

   // ValidateLogFormat validates log format option
   func ValidateLogFormat(s string) error {
       valid := map[string]bool{"pretty": true, "json": true}
       if !valid[strings.ToLower(s)] {
           return errors.New("must be one of: pretty, json")
       }
       return nil
   }
   ```

---

### Fase 4: Formularios por Categoria

**Objetivo**: Criar formularios huh para cada categoria de configuracao

#### Tarefas:

1. **Criar forms.go com formularios por categoria**
   - Arquivo: `internal/tui/forms.go`

   ```go
   package tui

   import (
       "github.com/charmbracelet/huh"
   )

   // Category represents a configuration category
   type Category string

   const (
       CategoryOutput      Category = "Output"
       CategoryConcurrency Category = "Concurrency"
       CategoryCache       Category = "Cache"
       CategoryRendering   Category = "Rendering"
       CategoryStealth     Category = "Stealth"
       CategoryLogging     Category = "Logging"
       CategoryLLM         Category = "LLM"
   )

   // AllCategories returns all available categories
   func AllCategories() []Category {
       return []Category{
           CategoryOutput,
           CategoryConcurrency,
           CategoryCache,
           CategoryRendering,
           CategoryStealth,
           CategoryLogging,
           CategoryLLM,
       }
   }

   // BuildOutputForm creates the Output configuration form
   func BuildOutputForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewInput().
                   Title("Output Directory").
                   Description("Base directory for saved documentation files").
                   Placeholder("./docs").
                   Value(&v.OutputDirectory).
                   Validate(ValidateDirectory),

               huh.NewConfirm().
                   Title("Flat Structure").
                   Description("Disable hierarchical folder structure").
                   Value(&v.OutputFlat),

               huh.NewConfirm().
                   Title("Overwrite Existing").
                   Description("Overwrite existing files without prompting").
                   Value(&v.OutputOverwrite),

               huh.NewConfirm().
                   Title("JSON Metadata").
                   Description("Generate individual .json metadata files").
                   Value(&v.JSONMetadata),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildConcurrencyForm creates the Concurrency configuration form
   func BuildConcurrencyForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewInput().
                   Title("Workers").
                   Description("Number of concurrent workers").
                   Placeholder("5").
                   Value(intToStringPtr(&v.Workers)).
                   Validate(ValidatePositiveInt),

               huh.NewInput().
                   Title("Timeout").
                   Description("Request timeout (e.g., 30s, 1m, 2m30s)").
                   Placeholder("30s").
                   Value(&v.Timeout).
                   Validate(ValidateDuration),

               huh.NewInput().
                   Title("Max Depth").
                   Description("Maximum crawl depth for web strategies").
                   Placeholder("3").
                   Value(intToStringPtr(&v.MaxDepth)).
                   Validate(ValidatePositiveInt),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildCacheForm creates the Cache configuration form
   func BuildCacheForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewConfirm().
                   Title("Cache Enabled").
                   Description("Enable BadgerDB caching layer").
                   Value(&v.CacheEnabled),

               huh.NewInput().
                   Title("Cache TTL").
                   Description("Time-to-live for cached responses (e.g., 24h, 7d)").
                   Placeholder("24h").
                   Value(&v.CacheTTL).
                   Validate(ValidateDuration),

               huh.NewInput().
                   Title("Cache Directory").
                   Description("Path to store cache database").
                   Placeholder("~/.repodocs/cache").
                   Value(&v.CacheDirectory).
                   Validate(ValidateDirectory),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildRenderingForm creates the Rendering configuration form
   func BuildRenderingForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewConfirm().
                   Title("Force JS Rendering").
                   Description("Always use headless browser for JavaScript rendering").
                   Value(&v.ForceJS),

               huh.NewInput().
                   Title("JS Timeout").
                   Description("Timeout for JavaScript execution").
                   Placeholder("60s").
                   Value(&v.JSTimeout).
                   Validate(ValidateDuration),

               huh.NewConfirm().
                   Title("Scroll to End").
                   Description("Scroll page to bottom to trigger lazy loads").
                   Value(&v.ScrollToEnd),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildStealthForm creates the Stealth configuration form
   func BuildStealthForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewInput().
                   Title("User Agent").
                   Description("Custom User-Agent header (leave empty for random)").
                   Placeholder("Mozilla/5.0 ...").
                   Value(&v.UserAgent),

               huh.NewInput().
                   Title("Random Delay Min").
                   Description("Minimum random delay between requests").
                   Placeholder("1s").
                   Value(&v.RandomDelayMin).
                   Validate(ValidateDuration),

               huh.NewInput().
                   Title("Random Delay Max").
                   Description("Maximum random delay between requests").
                   Placeholder("3s").
                   Value(&v.RandomDelayMax).
                   Validate(ValidateDuration),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildLoggingForm creates the Logging configuration form
   func BuildLoggingForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewSelect[string]().
                   Title("Log Level").
                   Description("Minimum log level to display").
                   Options(
                       huh.NewOption("Debug", "debug"),
                       huh.NewOption("Info", "info").Selected(true),
                       huh.NewOption("Warn", "warn"),
                       huh.NewOption("Error", "error"),
                   ).
                   Value(&v.LogLevel),

               huh.NewSelect[string]().
                   Title("Log Format").
                   Description("Output format for log messages").
                   Options(
                       huh.NewOption("Pretty (colored)", "pretty").Selected(true),
                       huh.NewOption("JSON", "json"),
                   ).
                   Value(&v.LogFormat),
           ),
       ).WithTheme(GetTheme())
   }

   // BuildLLMForm creates the LLM configuration form
   func BuildLLMForm(v *ConfigValues) *huh.Form {
       return huh.NewForm(
           huh.NewGroup(
               huh.NewSelect[string]().
                   Title("LLM Provider").
                   Description("AI provider for metadata enhancement").
                   Options(
                       huh.NewOption("None (disabled)", ""),
                       huh.NewOption("OpenAI", "openai"),
                       huh.NewOption("Anthropic", "anthropic"),
                       huh.NewOption("Google", "google"),
                       huh.NewOption("Custom/OpenAI-compatible", "custom"),
                   ).
                   Value(&v.LLMProvider),

               huh.NewInput().
                   Title("API Key").
                   Description("API key for the selected provider").
                   EchoMode(huh.EchoModePassword).
                   Value(&v.LLMAPIKey),

               huh.NewInput().
                   Title("Base URL").
                   Description("Custom base URL (for OpenAI-compatible providers)").
                   Placeholder("https://api.openai.com/v1").
                   Value(&v.LLMBaseURL),

               huh.NewInput().
                   Title("Model").
                   Description("Model name to use").
                   Placeholder("gpt-4").
                   Value(&v.LLMModel),

               huh.NewInput().
                   Title("Max Tokens").
                   Description("Maximum tokens in response").
                   Placeholder("4096").
                   Value(intToStringPtr(&v.LLMMaxTokens)).
                   Validate(ValidatePositiveInt),

               huh.NewInput().
                   Title("Temperature").
                   Description("Sampling temperature (0.0 - 2.0)").
                   Placeholder("0.7").
                   Value(floatToStringPtr(&v.LLMTemperature)).
                   Validate(ValidateTemperature),
           ),
       ).WithTheme(GetTheme())
   }

   // Helper functions for type conversion
   func intToStringPtr(n *int) *string {
       s := strconv.Itoa(*n)
       return &s
   }

   func floatToStringPtr(f *float64) *string {
       s := strconv.FormatFloat(*f, 'f', 2, 64)
       return &s
   }
   ```

---

### Fase 5: Menu Principal e Navegacao

**Objetivo**: Criar menu de navegacao entre categorias

#### Tarefas:

1. **Criar categories.go**
   - Arquivo: `internal/tui/categories.go`

   ```go
   package tui

   import (
       "github.com/charmbracelet/huh"
   )

   // CategoryMenuItem represents a menu item for category selection
   type CategoryMenuItem struct {
       Name        string
       Description string
       Category    Category
   }

   // GetCategoryMenuItems returns menu items for all categories
   func GetCategoryMenuItems() []CategoryMenuItem {
       return []CategoryMenuItem{
           {Name: "Output", Description: "Directory, flat structure, overwrite settings", Category: CategoryOutput},
           {Name: "Concurrency", Description: "Workers, timeout, crawl depth", Category: CategoryConcurrency},
           {Name: "Cache", Description: "Enable/disable, TTL, directory", Category: CategoryCache},
           {Name: "Rendering", Description: "JavaScript rendering, timeout, scroll", Category: CategoryRendering},
           {Name: "Stealth", Description: "User-Agent, request delays", Category: CategoryStealth},
           {Name: "Logging", Description: "Log level and format", Category: CategoryLogging},
           {Name: "LLM", Description: "AI provider, API key, model settings", Category: CategoryLLM},
       }
   }

   // BuildCategoryMenu creates the main category selection menu
   func BuildCategoryMenu(selected *Category) *huh.Form {
       items := GetCategoryMenuItems()
       options := make([]huh.Option[Category], len(items)+1)

       for i, item := range items {
           options[i] = huh.NewOption(item.Name+" - "+item.Description, item.Category)
       }
       // Add save & exit option
       options[len(items)] = huh.NewOption("Save and Exit", Category("save"))

       return huh.NewForm(
           huh.NewGroup(
               huh.NewSelect[Category]().
                   Title("RepoDocs Configuration").
                   Description("Select a category to configure, or save and exit").
                   Options(options...).
                   Value(selected),
           ),
       ).WithTheme(GetTheme())
   }
   ```

---

### Fase 6: Aplicacao Principal (tea.Model)

**Objetivo**: Criar o Model principal que orquestra a TUI

#### Tarefas:

1. **Criar app.go**
   - Arquivo: `internal/tui/app.go`

   ```go
   package tui

   import (
       "fmt"

       tea "github.com/charmbracelet/bubbletea"
       "github.com/charmbracelet/huh"
       "github.com/charmbracelet/lipgloss"
       "github.com/quantmind-br/repodocs-go/internal/config"
   )

   // State represents the current state of the TUI
   type State int

   const (
       StateMenu State = iota
       StateEditing
       StateConfirmSave
       StateSaved
       StateCancelled
   )

   // Model is the main Bubble Tea model for the config TUI
   type Model struct {
       state           State
       currentCategory Category
       values          *ConfigValues
       originalConfig  *config.Config
       categoryMenu    *huh.Form
       currentForm     *huh.Form
       confirmForm     *huh.Form
       width           int
       height          int
       err             error
       saved           bool
   }

   // NewModel creates a new TUI model with the given config
   func NewModel(cfg *config.Config) *Model {
       values := FromConfig(cfg)
       var selectedCategory Category

       m := &Model{
           state:          StateMenu,
           values:         values,
           originalConfig: cfg,
           categoryMenu:   BuildCategoryMenu(&selectedCategory),
       }
       m.currentCategory = selectedCategory

       return m
   }

   // Init implements tea.Model
   func (m *Model) Init() tea.Cmd {
       return m.categoryMenu.Init()
   }

   // Update implements tea.Model
   func (m *Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
       switch msg := msg.(type) {
       case tea.WindowSizeMsg:
           m.width = msg.Width
           m.height = msg.Height
           return m, nil

       case tea.KeyMsg:
           switch msg.String() {
           case "ctrl+c":
               m.state = StateCancelled
               return m, tea.Quit
           case "esc":
               if m.state == StateEditing {
                   // Return to menu
                   m.state = StateMenu
                   m.categoryMenu = BuildCategoryMenu(&m.currentCategory)
                   return m, m.categoryMenu.Init()
               }
               if m.state == StateMenu {
                   m.state = StateCancelled
                   return m, tea.Quit
               }
           }
       }

       var cmd tea.Cmd

       switch m.state {
       case StateMenu:
           form, cmd := m.categoryMenu.Update(msg)
           if f, ok := form.(*huh.Form); ok {
               m.categoryMenu = f
           }

           if m.categoryMenu.State == huh.StateCompleted {
               if m.currentCategory == Category("save") {
                   m.state = StateConfirmSave
                   var confirm bool
                   m.confirmForm = huh.NewForm(
                       huh.NewGroup(
                           huh.NewConfirm().
                               Title("Save configuration?").
                               Description("This will write to ~/.repodocs/config.yaml").
                               Affirmative("Save").
                               Negative("Cancel").
                               Value(&confirm),
                       ),
                   ).WithTheme(GetTheme())
                   return m, m.confirmForm.Init()
               }

               // Open category form
               m.state = StateEditing
               m.currentForm = m.buildFormForCategory(m.currentCategory)
               return m, m.currentForm.Init()
           }

           return m, cmd

       case StateEditing:
           form, cmd := m.currentForm.Update(msg)
           if f, ok := form.(*huh.Form); ok {
               m.currentForm = f
           }

           if m.currentForm.State == huh.StateCompleted {
               // Return to menu
               m.state = StateMenu
               m.categoryMenu = BuildCategoryMenu(&m.currentCategory)
               return m, m.categoryMenu.Init()
           }

           return m, cmd

       case StateConfirmSave:
           form, cmd := m.confirmForm.Update(msg)
           if f, ok := form.(*huh.Form); ok {
               m.confirmForm = f
           }

           if m.confirmForm.State == huh.StateCompleted {
               // Check if user confirmed
               if m.confirmForm.Get("confirm") == true {
                   // Save config
                   if err := m.saveConfig(); err != nil {
                       m.err = err
                   } else {
                       m.saved = true
                   }
                   m.state = StateSaved
               } else {
                   // Return to menu
                   m.state = StateMenu
                   m.categoryMenu = BuildCategoryMenu(&m.currentCategory)
                   return m, m.categoryMenu.Init()
               }
               return m, tea.Quit
           }

           return m, cmd
       }

       return m, cmd
   }

   // View implements tea.Model
   func (m *Model) View() string {
       switch m.state {
       case StateMenu:
           return m.renderWithHeader("Main Menu", m.categoryMenu.View())

       case StateEditing:
           title := fmt.Sprintf("Editing: %s", m.currentCategory)
           footer := "\n" + DescriptionStyle.Render("Press Esc to return to menu")
           return m.renderWithHeader(title, m.currentForm.View()+footer)

       case StateConfirmSave:
           return m.renderWithHeader("Confirm", m.confirmForm.View())

       case StateSaved:
           if m.err != nil {
               return ErrorStyle.Render(fmt.Sprintf("Error saving config: %v", m.err))
           }
           return SuccessStyle.Render("Configuration saved to ~/.repodocs/config.yaml")

       case StateCancelled:
           return DescriptionStyle.Render("Configuration cancelled. No changes saved.")

       default:
           return ""
       }
   }

   func (m *Model) renderWithHeader(title, content string) string {
       header := TitleStyle.Render("RepoDocs Configuration - " + title)
       return BoxStyle.Width(min(m.width-4, 80)).Render(header + "\n\n" + content)
   }

   func (m *Model) buildFormForCategory(cat Category) *huh.Form {
       switch cat {
       case CategoryOutput:
           return BuildOutputForm(m.values)
       case CategoryConcurrency:
           return BuildConcurrencyForm(m.values)
       case CategoryCache:
           return BuildCacheForm(m.values)
       case CategoryRendering:
           return BuildRenderingForm(m.values)
       case CategoryStealth:
           return BuildStealthForm(m.values)
       case CategoryLogging:
           return BuildLoggingForm(m.values)
       case CategoryLLM:
           return BuildLLMForm(m.values)
       default:
           return nil
       }
   }

   func (m *Model) saveConfig() error {
       cfg, err := m.values.ToConfig()
       if err != nil {
           return err
       }
       return config.Save(cfg)
   }

   func min(a, b int) int {
       if a < b {
           return a
       }
       return b
   }

   // Run starts the TUI application
   func Run(cfg *config.Config) error {
       model := NewModel(cfg)
       p := tea.NewProgram(model, tea.WithAltScreen())
       _, err := p.Run()
       return err
   }
   ```

---

### Fase 7: Funcao Save no Config Loader

**Objetivo**: Adicionar funcao para salvar configuracao em YAML

#### Tarefas:

1. **Modificar internal/config/loader.go**
   - Adicionar funcao `Save(*Config) error`

   ```go
   // Save writes the configuration to the default config file
   func Save(cfg *Config) error {
       // Ensure config directory exists
       if err := EnsureConfigDir(); err != nil {
           return fmt.Errorf("failed to create config directory: %w", err)
       }

       // Marshal config to YAML
       data, err := yaml.Marshal(cfg)
       if err != nil {
           return fmt.Errorf("failed to marshal config: %w", err)
       }

       // Write to file
       configPath := ConfigFilePath()
       if err := os.WriteFile(configPath, data, 0644); err != nil {
           return fmt.Errorf("failed to write config file: %w", err)
       }

       return nil
   }
   ```

---

### Fase 8: Integracao com Cobra

**Objetivo**: Adicionar comando `config` ao CLI

#### Tarefas:

1. **Modificar cmd/repodocs/main.go**
   - Adicionar `configCmd` com subcomandos

   ```go
   var configCmd = &cobra.Command{
       Use:   "config",
       Short: "Manage RepoDocs configuration",
       Long: `Manage RepoDocs configuration interactively or view current settings.

   Without subcommands, opens an interactive TUI for editing configuration.`,
       RunE: func(cmd *cobra.Command, args []string) error {
           // Default: open interactive TUI
           cfg, err := config.Load()
           if err != nil {
               // If no config exists, use defaults
               cfg = config.Default()
           }
           return tui.Run(cfg)
       },
   }

   var configShowCmd = &cobra.Command{
       Use:   "show",
       Short: "Show current configuration",
       RunE: func(cmd *cobra.Command, args []string) error {
           cfg, err := config.Load()
           if err != nil {
               return fmt.Errorf("failed to load config: %w", err)
           }

           // Marshal to YAML and print
           data, err := yaml.Marshal(cfg)
           if err != nil {
               return fmt.Errorf("failed to marshal config: %w", err)
           }

           fmt.Println(string(data))
           return nil
       },
   }

   var configInitCmd = &cobra.Command{
       Use:   "init",
       Short: "Create default configuration file",
       RunE: func(cmd *cobra.Command, args []string) error {
           configPath := config.ConfigFilePath()

           // Check if config already exists
           if _, err := os.Stat(configPath); err == nil {
               return fmt.Errorf("config file already exists at %s", configPath)
           }

           // Create default config
           cfg := config.Default()
           if err := config.Save(cfg); err != nil {
               return fmt.Errorf("failed to create config: %w", err)
           }

           fmt.Printf("Created default configuration at %s\n", configPath)
           return nil
       },
   }

   var configEditCmd = &cobra.Command{
       Use:   "edit",
       Short: "Edit configuration interactively",
       RunE: func(cmd *cobra.Command, args []string) error {
           cfg, err := config.Load()
           if err != nil {
               cfg = config.Default()
           }
           return tui.Run(cfg)
       },
   }

   var configPathCmd = &cobra.Command{
       Use:   "path",
       Short: "Show configuration file path",
       Run: func(cmd *cobra.Command, args []string) {
           fmt.Println(config.ConfigFilePath())
       },
   }

   func init() {
       // ... existing init code ...

       // Add config command and subcommands
       configCmd.AddCommand(configShowCmd)
       configCmd.AddCommand(configInitCmd)
       configCmd.AddCommand(configEditCmd)
       configCmd.AddCommand(configPathCmd)
       rootCmd.AddCommand(configCmd)
   }
   ```

---

### Fase 9: Testes

**Objetivo**: Criar testes unitarios para os componentes da TUI

#### Tarefas:

1. **Criar internal/tui/tui_test.go**

   ```go
   package tui

   import (
       "testing"
       "time"

       "github.com/quantmind-br/repodocs-go/internal/config"
       "github.com/stretchr/testify/assert"
       "github.com/stretchr/testify/require"
   )

   func TestFromConfig(t *testing.T) {
       cfg := config.Default()
       values := FromConfig(cfg)

       assert.Equal(t, cfg.Output.Directory, values.OutputDirectory)
       assert.Equal(t, cfg.Cache.Enabled, values.CacheEnabled)
       assert.Equal(t, cfg.Concurrency.Workers, values.Workers)
   }

   func TestToConfig(t *testing.T) {
       values := &ConfigValues{
           OutputDirectory: "./test-docs",
           Workers:         10,
           Timeout:         "60s",
           MaxDepth:        5,
           CacheEnabled:    true,
           CacheTTL:        "48h",
           CacheDirectory:  "/tmp/cache",
           ForceJS:         true,
           JSTimeout:       "120s",
           ScrollToEnd:     true,
           RandomDelayMin:  "2s",
           RandomDelayMax:  "5s",
           LogLevel:        "debug",
           LogFormat:       "json",
       }

       cfg, err := values.ToConfig()
       require.NoError(t, err)

       assert.Equal(t, "./test-docs", cfg.Output.Directory)
       assert.Equal(t, 10, cfg.Concurrency.Workers)
       assert.Equal(t, 60*time.Second, cfg.Concurrency.Timeout)
       assert.Equal(t, true, cfg.Rendering.ForceJS)
   }

   func TestValidateDuration(t *testing.T) {
       tests := []struct {
           input   string
           wantErr bool
       }{
           {"30s", false},
           {"5m", false},
           {"2h30m", false},
           {"24h", false},
           {"invalid", true},
           {"", true},
           {"30", true},
       }

       for _, tt := range tests {
           t.Run(tt.input, func(t *testing.T) {
               err := ValidateDuration(tt.input)
               if tt.wantErr {
                   assert.Error(t, err)
               } else {
                   assert.NoError(t, err)
               }
           })
       }
   }

   func TestValidatePositiveInt(t *testing.T) {
       tests := []struct {
           input   string
           wantErr bool
       }{
           {"1", false},
           {"100", false},
           {"0", true},
           {"-1", true},
           {"abc", true},
       }

       for _, tt := range tests {
           t.Run(tt.input, func(t *testing.T) {
               err := ValidatePositiveInt(tt.input)
               if tt.wantErr {
                   assert.Error(t, err)
               } else {
                   assert.NoError(t, err)
               }
           })
       }
   }

   func TestValidateTemperature(t *testing.T) {
       tests := []struct {
           input   string
           wantErr bool
       }{
           {"0.0", false},
           {"0.7", false},
           {"1.0", false},
           {"2.0", false},
           {"2.1", true},
           {"-0.1", true},
           {"abc", true},
       }

       for _, tt := range tests {
           t.Run(tt.input, func(t *testing.T) {
               err := ValidateTemperature(tt.input)
               if tt.wantErr {
                   assert.Error(t, err)
               } else {
                   assert.NoError(t, err)
               }
           })
       }
   }
   ```

2. **Criar tests/unit/tui/ para testes adicionais**

---

### Fase 10: Acessibilidade e Polish

**Objetivo**: Adicionar suporte a acessibilidade e refinamentos finais

#### Tarefas:

1. **Adicionar flag --accessible ao config command**

   ```go
   var accessibleMode bool

   func init() {
       configCmd.PersistentFlags().BoolVar(&accessibleMode, "accessible", false,
           "Enable accessible mode for screen readers")
   }

   // In Run function:
   func Run(cfg *config.Config, accessible bool) error {
       model := NewModel(cfg)
       if accessible {
           model.SetAccessible(true)
       }
       // ...
   }
   ```

2. **Detectar variavel de ambiente ACCESSIBLE**

   ```go
   accessible := os.Getenv("ACCESSIBLE") != "" || accessibleMode
   ```

---

## Estrategia de Testes

### Testes Unitarios

- [ ] `TestFromConfig` - Conversao Config -> ConfigValues
- [ ] `TestToConfig` - Conversao ConfigValues -> Config
- [ ] `TestValidateDuration` - Validacao de duracoes
- [ ] `TestValidatePositiveInt` - Validacao de inteiros positivos
- [ ] `TestValidateTemperature` - Validacao de temperatura LLM
- [ ] `TestValidateDirectory` - Validacao de diretorios
- [ ] `TestBuildOutputForm` - Criacao de formulario Output
- [ ] `TestBuildCacheForm` - Criacao de formulario Cache
- [ ] `TestCategoryMenuItems` - Itens do menu de categorias

### Testes de Integracao

- [ ] `TestConfigSaveAndLoad` - Salvar e recarregar configuracao
- [ ] `TestConfigInitCommand` - Comando `config init`
- [ ] `TestConfigShowCommand` - Comando `config show`
- [ ] `TestConfigPathCommand` - Comando `config path`

### Casos de Teste Especificos

| ID | Cenario | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | Criar config em sistema limpo | `repodocs config init` | Arquivo criado em ~/.repodocs/config.yaml |
| TC02 | Editar workers | Navegar Output > Workers > "10" | workers: 10 no YAML |
| TC03 | Valor invalido de timeout | Timeout: "invalid" | Erro de validacao exibido |
| TC04 | Cancelar sem salvar | Esc no menu principal | Nenhuma alteracao salva |
| TC05 | Salvar configuracao | Menu > Save and Exit > Confirm | Arquivo YAML atualizado |
| TC06 | Modo acessivel | `ACCESSIBLE=1 repodocs config` | Prompts texto simples |

---

## Riscos e Mitigacoes

| Risco | Probabilidade | Impacto | Mitigacao |
|-------|---------------|---------|-----------|
| Conflito de dependencias com bubbletea/huh | Baixo | Alto | Verificar compatibilidade antes de adicionar |
| Performance lenta em terminais limitados | Baixo | Medio | Testar em multiplos terminais, otimizar rendering |
| Perda de dados ao salvar | Medio | Alto | Backup automatico do config anterior |
| UX confusa para novos usuarios | Medio | Medio | Adicionar help text claro, documentacao |
| Validacao incompleta de valores | Medio | Medio | Testes abrangentes de validacao |

---

## Checklist de Conclusao

- [ ] Dependencias adicionadas (huh, lipgloss)
- [ ] Pacote internal/tui/ criado
- [ ] Config adapter implementado
- [ ] Validadores implementados
- [ ] Formularios por categoria implementados
- [ ] Menu de navegacao implementado
- [ ] Model principal (app.go) implementado
- [ ] Funcao Save() no loader implementada
- [ ] Comandos Cobra adicionados (config, show, init, edit, path)
- [ ] Testes unitarios escritos e passando
- [ ] Testes de integracao escritos e passando
- [ ] Modo acessivel implementado
- [ ] Documentacao atualizada (README, --help)
- [ ] Code review realizado
- [ ] Feature testada manualmente

---

## Notas Adicionais

### Ordem de Implementacao Recomendada

1. Adicionar dependencias e criar estrutura de pastas
2. Implementar config_adapter.go e validation.go (testavel isoladamente)
3. Implementar forms.go com formularios simples primeiro (Output, Logging)
4. Implementar app.go com navegacao basica
5. Adicionar comandos Cobra e testar fluxo completo
6. Implementar formularios restantes
7. Adicionar testes e modo acessivel
8. Polish e documentacao

### Exemplo de Uso Final

```bash
# Abrir TUI interativa
repodocs config

# Ou equivalente
repodocs config edit

# Ver configuracao atual
repodocs config show

# Criar config inicial
repodocs config init

# Ver caminho do arquivo
repodocs config path

# Modo acessivel
ACCESSIBLE=1 repodocs config
```

### Referencias

- [charmbracelet/huh](https://github.com/charmbracelet/huh) - Biblioteca de formularios
- [charmbracelet/bubbletea](https://github.com/charmbracelet/bubbletea) - Framework TUI
- [charmbracelet/lipgloss](https://github.com/charmbracelet/lipgloss) - Styling
- [Exemplo bubbletea+huh](https://github.com/charmbracelet/huh/blob/main/examples/bubbletea/main.go)
