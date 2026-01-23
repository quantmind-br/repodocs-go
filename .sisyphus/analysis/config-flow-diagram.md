# Config Persistence Flow Diagram

## Current Flow (With Data Loss)

```
┌─────────────────────────────────────────────────────────────────┐
│ User runs: repodocs config                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ config.Load()                                                   │
│ - Reads ~/.repodocs/config.yaml                                │
│ - Unmarshals to Config struct (ALL fields present)             │
│                                                                 │
│ Config {                                                        │
│   Output: {...},                                               │
│   Concurrency: {...},                                          │
│   Cache: {...},                                                │
│   Rendering: {...},                                            │
│   Stealth: {...},                                              │
│   Logging: {...},                                              │
│   LLM: {                                                        │
│     Provider: "openai",                                        │
│     APIKey: "sk-...",                                          │
│     RateLimit: {                ← PRESENT                      │
│       Enabled: true,                                           │
│       RequestsPerMinute: 100,                                  │
│       CircuitBreaker: {...}                                    │
│     }                                                           │
│   },                                                            │
│   Exclude: [...]                                               │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ FromConfig(cfg)                                                 │
│ - Converts Config to ConfigValues for form editing             │
│                                                                 │
│ ConfigValues {                                                  │
│   OutputDirectory: "./docs",                                   │
│   Workers: 5,                                                  │
│   CacheEnabled: true,                                          │
│   ForceJS: false,                                              │
│   UserAgent: "",                                               │
│   LogLevel: "info",                                            │
│   LLMProvider: "openai",                                       │
│   LLMAPIKey: "sk-...",                                         │
│   LLMMaxTokens: 4096,                                          │
│   LLMTemperature: 0.7,                                         │
│   LLMEnhanceMetadata: false,                                   │
│   Exclude: [...]                                               │
│   // ❌ RateLimit fields MISSING                               │
│   // ❌ CircuitBreaker fields MISSING                          │
│   // ❌ MaxRetries field MISSING                               │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ TUI Forms (User Edits)                                          │
│ - Shows only fields in ConfigValues                            │
│ - User can edit LLM provider, API key, etc.                    │
│ - RateLimit/CircuitBreaker NOT visible                         │
│                                                                 │
│ User changes: LLMProvider from "openai" to "anthropic"         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ ToConfig()                                                      │
│ - Converts ConfigValues back to Config                         │
│                                                                 │
│ Config {                                                        │
│   Output: {...},                                               │
│   Concurrency: {...},                                          │
│   Cache: {...},                                                │
│   Rendering: {...},                                            │
│   Stealth: {...},                                              │
│   Logging: {...},                                              │
│   LLM: {                                                        │
│     Provider: "anthropic",  ← UPDATED                          │
│     APIKey: "sk-...",                                          │
│     RateLimit: {            ← ZERO VALUES!                     │
│       Enabled: false,       ← WAS true                         │
│       RequestsPerMinute: 0, ← WAS 100                          │
│       CircuitBreaker: {                                        │
│         Enabled: false,     ← WAS true                         │
│         FailureThreshold: 0 ← WAS 5                            │
│       }                                                         │
│     }                                                           │
│   },                                                            │
│   Exclude: [...]                                               │
│ }                                                               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ config.Save(cfg)                                                │
│ - Marshals Config to YAML                                      │
│ - Writes to ~/.repodocs/config.yaml (OVERWRITES)               │
│                                                                 │
│ yaml:                                                           │
│ llm:                                                            │
│   provider: anthropic                                          │
│   api_key: sk-...                                              │
│   rate_limit:                                                  │
│     enabled: false        ← DATA LOSS!                         │
│     requests_per_minute: 0                                     │
│     burst_size: 0                                              │
│     circuit_breaker:                                           │
│       enabled: false      ← DATA LOSS!                         │
│       failure_threshold: 0                                     │
└─────────────────────────────────────────────────────────────────┘
```

## Field Mapping Visualization

```
Config Struct                    ConfigValues Struct
═════════════════════════════════════════════════════════════════

Output                    ✅ →   OutputDirectory
├─ Directory                     OutputFlat
├─ Flat                          OutputOverwrite
├─ Overwrite                     JSONMetadata
└─ JSONMetadata

Concurrency              ✅ →   Workers
├─ Workers                       Timeout (string)
├─ Timeout                       MaxDepth
└─ MaxDepth

Cache                    ✅ →   CacheEnabled
├─ Enabled                       CacheTTL (string)
├─ TTL                           CacheDirectory
└─ Directory

Rendering                ✅ →   ForceJS
├─ ForceJS                       JSTimeout (string)
├─ JSTimeout                     ScrollToEnd
└─ ScrollToEnd

Stealth                  ✅ →   UserAgent
├─ UserAgent                     RandomDelayMin (string)
├─ RandomDelayMin                RandomDelayMax (string)
└─ RandomDelayMax

Logging                  ✅ →   LogLevel
├─ Level                         LogFormat
└─ Format

LLM                      ⚠️  →   LLMProvider
├─ Provider                      LLMAPIKey
├─ APIKey                        LLMBaseURL
├─ BaseURL                       LLMModel
├─ Model                         LLMMaxTokens
├─ MaxTokens                     LLMTemperature
├─ Temperature                   LLMTimeout (string)
├─ Timeout                       LLMEnhanceMetadata
├─ EnhanceMetadata               
├─ MaxRetries             ❌ →   (MISSING)
└─ RateLimit              ❌ →   (MISSING)
   ├─ Enabled                    
   ├─ RequestsPerMinute          
   ├─ BurstSize                  
   ├─ MaxRetries                 
   ├─ InitialDelay               
   ├─ MaxDelay                   
   ├─ Multiplier                 
   └─ CircuitBreaker      ❌ →   (MISSING)
      ├─ Enabled                 
      ├─ FailureThreshold        
      ├─ SuccessThresholdHalfOpen
      └─ ResetTimeout            

Exclude                  ✅ →   Exclude

Legend:
✅ = Properly mapped and persisted
⚠️  = Partially mapped (some fields missing)
❌ = Not mapped, data loss on save
```

## Data Loss Scenarios

### Scenario 1: Edit and Save

```
Original config.yaml:
  llm:
    rate_limit:
      enabled: true
      requests_per_minute: 100

User action: Edit LLM provider, save

Result:
  llm:
    rate_limit:
      enabled: false        ← LOST!
      requests_per_minute: 0
```

### Scenario 2: Load Default, Edit, Save

```
No config file exists

User runs: repodocs config

config.Load() → config.Default() (includes rate_limit defaults)
FromConfig() → ConfigValues (rate_limit dropped)
User edits something
ToConfig() → Config (rate_limit = zero values)
Save() → config.yaml (rate_limit disabled)

Result: Rate limiting disabled even though defaults were loaded
```

### Scenario 3: Multiple Edits

```
Edit 1: Change provider (rate_limit lost)
Edit 2: Change API key (rate_limit still lost)
Edit 3: Change model (rate_limit still lost)

After 3 edits: rate_limit completely gone
```

## Proposed Fix Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ User runs: repodocs config                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ config.Load()                                                   │
│ - Reads ~/.repodocs/config.yaml                                │
│ - Unmarshals to Config struct (ALL fields present)             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ FromConfig(cfg)                                                 │
│ - Converts Config to ConfigValues                              │
│ - ✅ NOW INCLUDES RateLimit fields                             │
│ - ✅ NOW INCLUDES CircuitBreaker fields                        │
│ - ✅ NOW INCLUDES MaxRetries field                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ TUI Forms (User Edits)                                          │
│ - ✅ Shows RateLimit/CircuitBreaker in advanced section        │
│ - User can edit all fields                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ ToConfig()                                                      │
│ - ✅ Reconstructs RateLimit from ConfigValues                  │
│ - ✅ Reconstructs CircuitBreaker from ConfigValues             │
│ - ✅ Preserves all values                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ config.Save(cfg)                                                │
│ - Marshals Config to YAML                                      │
│ - ✅ RateLimit preserved                                       │
│ - ✅ CircuitBreaker preserved                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Code Locations

```
internal/
├── config/
│   ├── config.go          ← Config struct definition
│   ├── defaults.go        ← Default() includes RateLimit
│   └── loader.go          ← Save() does full replacement
└── tui/
    ├── config_adapter.go  ← ConfigValues (INCOMPLETE)
    │                         FromConfig() (INCOMPLETE)
    │                         ToConfig() (INCOMPLETE)
    ├── forms.go           ← TUI form definitions
    └── app.go             ← handleSave() calls ToConfig()
```
