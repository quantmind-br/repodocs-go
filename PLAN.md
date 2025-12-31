# PLAN.md - Integração Multi-Provider LLM para repodocs-go

## Resumo Executivo

Integrar suporte a provedores LLM compatíveis com as APIs **OpenAI**, **Anthropic** e **Google Gemini**. O usuário configura apenas três parâmetros por provider:

- **`api_key`** - Chave de autenticação
- **`base_url`** - URL base do endpoint
- **`model`** - Identificador do modelo

**Não haverá modelos hardcoded.** O usuário especifica qualquer modelo suportado pelo endpoint configurado.

---

## Índice

1. [Objetivo e Escopo](#1-objetivo-e-escopo)
2. [Especificações das APIs](#2-especificações-das-apis)
3. [Arquitetura da Solução](#3-arquitetura-da-solução)
4. [Schema de Configuração](#4-schema-de-configuração)
5. [Camada de Domínio](#5-camada-de-domínio)
6. [Implementação dos Providers](#6-implementação-dos-providers)
7. [Cliente HTTP](#7-cliente-http)
8. [Tratamento de Erros](#8-tratamento-de-erros)
9. [Testes](#9-testes)
10. [Fases de Implementação](#10-fases-de-implementação)
11. [Estrutura de Arquivos](#11-estrutura-de-arquivos)
12. [Exemplos de Uso](#12-exemplos-de-uso)

---

## 1. Objetivo e Escopo

### 1.1 Objetivo Principal

Permitir que `repodocs-go` se comunique com qualquer LLM através de três formatos de API padronizados:

| Formato API | Provedores Compatíveis |
|-------------|------------------------|
| **OpenAI** | OpenAI, Azure OpenAI, Ollama, vLLM, LiteLLM, LocalAI, Together AI, Groq, Mistral, Anyscale, OpenRouter, etc. |
| **Anthropic** | Anthropic Claude, AWS Bedrock (Claude), proxies compatíveis |
| **Google** | Google AI Studio, Vertex AI, proxies compatíveis |

### 1.2 Princípios de Design

1. **Zero modelos hardcoded** - O usuário informa o ID do modelo
2. **Configuração mínima** - Apenas `api_key`, `base_url`, `model`
3. **Compatibilidade máxima** - Funciona com qualquer endpoint que implemente o formato
4. **Cliente HTTP nativo** - Sem dependência de SDKs oficiais para maior controle
5. **Seguir padrões existentes** - Usar patterns já estabelecidos no codebase

### 1.3 Fora do Escopo (v1)

- Streaming (será adicionado em versão futura)
- Function calling / Tools
- Vision / Multimodal
- Embeddings
- Fine-tuning

---

## 2. Especificações das APIs

### 2.1 OpenAI Chat Completions API

**Endpoint:** `POST {base_url}/chat/completions`

**Headers:**
```http
Authorization: Bearer {api_key}
Content-Type: application/json
```

**Request Body:**
```json
{
  "model": "{model}",
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ],
  "max_tokens": 4096,
  "temperature": 0.7
}
```

**Response Body:**
```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion",
  "model": "...",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 20,
    "total_tokens": 30
  }
}
```

**Provedores compatíveis com este formato:**

| Provedor | Base URL Padrão |
|----------|-----------------|
| OpenAI | `https://api.openai.com/v1` |
| Ollama | `http://localhost:11434/v1` |
| vLLM | `http://localhost:8000/v1` |
| LiteLLM | `http://localhost:4000` |
| LocalAI | `http://localhost:8080/v1` |
| Together AI | `https://api.together.xyz/v1` |
| Groq | `https://api.groq.com/openai/v1` |
| Mistral | `https://api.mistral.ai/v1` |
| OpenRouter | `https://openrouter.ai/api/v1` |
| Azure OpenAI | `https://{resource}.openai.azure.com/openai/deployments/{deployment}` |

---

### 2.2 Anthropic Messages API

**Endpoint:** `POST {base_url}/v1/messages`

**Headers:**
```http
x-api-key: {api_key}
anthropic-version: 2023-06-01
Content-Type: application/json
```

**Request Body:**
```json
{
  "model": "{model}",
  "max_tokens": 4096,
  "messages": [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "..."}
  ],
  "system": "optional system prompt"
}
```

**Response Body:**
```json
{
  "id": "msg_xxx",
  "type": "message",
  "role": "assistant",
  "model": "...",
  "content": [
    {
      "type": "text",
      "text": "..."
    }
  ],
  "stop_reason": "end_turn",
  "usage": {
    "input_tokens": 10,
    "output_tokens": 20
  }
}
```

**Provedores compatíveis:**

| Provedor | Base URL Padrão |
|----------|-----------------|
| Anthropic | `https://api.anthropic.com` |
| AWS Bedrock | Via SDK específico |

---

### 2.3 Google Gemini API

**Endpoint:** `POST {base_url}/v1beta/models/{model}:generateContent`

**Headers:**
```http
x-goog-api-key: {api_key}
Content-Type: application/json
```

**Alternativa de autenticação:** `?key={api_key}` como query parameter

**Request Body:**
```json
{
  "contents": [
    {
      "role": "user",
      "parts": [{"text": "..."}]
    },
    {
      "role": "model", 
      "parts": [{"text": "..."}]
    }
  ],
  "systemInstruction": {
    "parts": [{"text": "..."}]
  },
  "generationConfig": {
    "maxOutputTokens": 4096,
    "temperature": 0.7
  }
}
```

**Response Body:**
```json
{
  "candidates": [
    {
      "content": {
        "role": "model",
        "parts": [{"text": "..."}]
      },
      "finishReason": "STOP"
    }
  ],
  "usageMetadata": {
    "promptTokenCount": 10,
    "candidatesTokenCount": 20,
    "totalTokenCount": 30
  }
}
```

**Provedores compatíveis:**

| Provedor | Base URL Padrão |
|----------|-----------------|
| Google AI Studio | `https://generativelanguage.googleapis.com` |
| Vertex AI | `https://{region}-aiplatform.googleapis.com` |

---

## 3. Arquitetura da Solução

### 3.1 Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Configuração                                 │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  config.yaml / ENV vars                                        │  │
│  │                                                                │  │
│  │  llm:                                                          │  │
│  │    provider: openai | anthropic | google                       │  │
│  │    api_key: "sk-..."  (ou REPODOCS_LLM_API_KEY)               │  │
│  │    base_url: "https://..."                                     │  │
│  │    model: "user-specified-model-id"                            │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Domain Layer                                  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  LLMProvider interface                                         │  │
│  │    - Name() string                                             │  │
│  │    - Complete(ctx, req) (*LLMResponse, error)                  │  │
│  │    - Close() error                                             │  │
│  └───────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  LLMRequest / LLMResponse / LLMMessage                         │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Provider Adapters                               │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │  OpenAI         │  │  Anthropic      │  │  Google         │     │
│  │  Adapter        │  │  Adapter        │  │  Adapter        │     │
│  │                 │  │                 │  │                 │     │
│  │  - Constrói     │  │  - Constrói     │  │  - Constrói     │     │
│  │    request no   │  │    request no   │  │    request no   │     │
│  │    formato      │  │    formato      │  │    formato      │     │
│  │    OpenAI       │  │    Anthropic    │  │    Gemini       │     │
│  │                 │  │                 │  │                 │     │
│  │  - Parseia      │  │  - Parseia      │  │  - Parseia      │     │
│  │    response     │  │    response     │  │    response     │     │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘     │
│           │                    │                    │               │
│           └────────────────────┼────────────────────┘               │
│                                │                                    │
│                                ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    HTTP Client (net/http)                      │  │
│  │  - POST request com headers específicos                        │  │
│  │  - Retry com backoff exponencial                               │  │
│  │  - Timeout configurável                                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      External APIs                                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │ OpenAI-compat   │  │ Anthropic API   │  │ Google Gemini   │     │
│  │ endpoints       │  │                 │  │ API             │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Fluxo de Dados

```
1. Usuário configura: provider=openai, base_url=http://localhost:11434/v1, model=llama3.2
                           │
                           ▼
2. Factory cria OpenAIProvider com as configurações
                           │
                           ▼
3. Chamada: provider.Complete(ctx, &LLMRequest{Messages: [...]})
                           │
                           ▼
4. OpenAIProvider converte LLMRequest → OpenAI JSON format
                           │
                           ▼
5. HTTP POST para {base_url}/chat/completions com headers corretos
                           │
                           ▼
6. Parse da resposta JSON → LLMResponse
                           │
                           ▼
7. Retorna LLMResponse para o chamador
```

---

## 4. Schema de Configuração

### 4.1 Struct de Configuração

```go
// internal/config/config.go

// LLMConfig contém configurações do provedor LLM
type LLMConfig struct {
    // Provider: "openai", "anthropic", "google"
    Provider string `mapstructure:"provider"`
    
    // APIKey para autenticação (preferir env var REPODOCS_LLM_API_KEY)
    APIKey string `mapstructure:"api_key"`
    
    // BaseURL do endpoint da API
    // Não há default - usuário deve especificar ou usar env var
    BaseURL string `mapstructure:"base_url"`
    
    // Model ID - não há default, usuário deve especificar
    Model string `mapstructure:"model"`
    
    // Configurações opcionais
    MaxTokens   int           `mapstructure:"max_tokens"`
    Temperature float64       `mapstructure:"temperature"`
    Timeout     time.Duration `mapstructure:"timeout"`
    MaxRetries  int           `mapstructure:"max_retries"`
}

// Adicionar ao Config principal
type Config struct {
    // ... campos existentes ...
    LLM LLMConfig `mapstructure:"llm"`
}
```

### 4.2 Defaults

```go
// internal/config/defaults.go

const (
    // LLM defaults - apenas para parâmetros opcionais
    DefaultLLMMaxTokens   = 4096
    DefaultLLMTemperature = 0.7
    DefaultLLMTimeout     = 60 * time.Second
    DefaultLLMMaxRetries  = 3
)

// NÃO há defaults para: provider, api_key, base_url, model
// Usuário DEVE configurar esses valores
```

### 4.3 Exemplo de Configuração

```yaml
# ~/.repodocs/config.yaml

llm:
  # Provider type (obrigatório): openai, anthropic, google
  provider: openai
  
  # API key (obrigatório, preferir variável de ambiente)
  # api_key: sk-...
  
  # Base URL do endpoint (obrigatório)
  # Exemplos:
  #   OpenAI:    https://api.openai.com/v1
  #   Ollama:    http://localhost:11434/v1
  #   vLLM:      http://localhost:8000/v1
  #   Anthropic: https://api.anthropic.com
  #   Google:    https://generativelanguage.googleapis.com
  base_url: http://localhost:11434/v1
  
  # Model ID (obrigatório) - especificado pelo usuário
  model: llama3.2
  
  # Parâmetros opcionais
  max_tokens: 4096
  temperature: 0.7
  timeout: 60s
  max_retries: 3
```

### 4.4 Variáveis de Ambiente

| Variável | Descrição | Obrigatório |
|----------|-----------|-------------|
| `REPODOCS_LLM_PROVIDER` | Tipo do provider | Sim |
| `REPODOCS_LLM_API_KEY` | Chave de API | Sim |
| `REPODOCS_LLM_BASE_URL` | URL base do endpoint | Sim |
| `REPODOCS_LLM_MODEL` | ID do modelo | Sim |
| `REPODOCS_LLM_MAX_TOKENS` | Máximo de tokens | Não |
| `REPODOCS_LLM_TEMPERATURE` | Temperatura | Não |
| `REPODOCS_LLM_TIMEOUT` | Timeout | Não |

---

## 5. Camada de Domínio

### 5.1 Interface do Provider

```go
// internal/domain/interfaces.go

// LLMProvider define a interface para interação com LLMs
type LLMProvider interface {
    // Name retorna o nome do provider (openai, anthropic, google)
    Name() string
    
    // Complete envia uma requisição e retorna a resposta
    Complete(ctx context.Context, req *LLMRequest) (*LLMResponse, error)
    
    // Close libera recursos
    Close() error
}
```

### 5.2 Tipos de Domínio

```go
// internal/domain/models.go

// MessageRole representa o papel na conversa
type MessageRole string

const (
    RoleSystem    MessageRole = "system"
    RoleUser      MessageRole = "user"
    RoleAssistant MessageRole = "assistant"
)

// LLMMessage representa uma mensagem na conversa
type LLMMessage struct {
    Role    MessageRole
    Content string
}

// LLMRequest representa uma requisição de completion
type LLMRequest struct {
    Messages    []LLMMessage
    MaxTokens   int      // 0 = usar default do provider
    Temperature *float64 // nil = usar default do provider
}

// LLMResponse representa a resposta do LLM
type LLMResponse struct {
    Content      string
    Model        string
    FinishReason string
    Usage        LLMUsage
}

// LLMUsage contém estatísticas de uso de tokens
type LLMUsage struct {
    PromptTokens     int
    CompletionTokens int
    TotalTokens      int
}
```

### 5.3 Erros

```go
// internal/domain/errors.go

var (
    ErrLLMNotConfigured     = errors.New("LLM provider not configured")
    ErrLLMMissingAPIKey     = errors.New("LLM API key is required")
    ErrLLMMissingBaseURL    = errors.New("LLM base URL is required")
    ErrLLMMissingModel      = errors.New("LLM model is required")
    ErrLLMInvalidProvider   = errors.New("invalid LLM provider")
    ErrLLMRequestFailed     = errors.New("LLM request failed")
    ErrLLMRateLimited       = errors.New("LLM rate limit exceeded")
    ErrLLMAuthFailed        = errors.New("LLM authentication failed")
    ErrLLMContextTooLong    = errors.New("LLM context length exceeded")
)

// LLMError representa um erro específico do LLM
type LLMError struct {
    Provider   string
    StatusCode int
    Message    string
    Err        error
}

func (e *LLMError) Error() string {
    if e.StatusCode > 0 {
        return fmt.Sprintf("%s error (HTTP %d): %s", e.Provider, e.StatusCode, e.Message)
    }
    return fmt.Sprintf("%s error: %s", e.Provider, e.Message)
}

func (e *LLMError) Unwrap() error {
    return e.Err
}
```

---

## 6. Implementação dos Providers

### 6.1 Factory

```go
// internal/llm/provider.go

package llm

import (
    "fmt"
    "net/http"
    "time"
    
    "github.com/quantmind-br/repodocs-go/internal/config"
    "github.com/quantmind-br/repodocs-go/internal/domain"
)

// ProviderConfig contém configuração para criar um provider
type ProviderConfig struct {
    Provider    string
    APIKey      string
    BaseURL     string
    Model       string
    MaxTokens   int
    Temperature float64
    Timeout     time.Duration
    MaxRetries  int
    HTTPClient  *http.Client
}

// NewProviderFromConfig cria um provider a partir da configuração
func NewProviderFromConfig(cfg *config.LLMConfig) (domain.LLMProvider, error) {
    if cfg.Provider == "" {
        return nil, domain.ErrLLMNotConfigured
    }
    if cfg.APIKey == "" {
        return nil, domain.ErrLLMMissingAPIKey
    }
    if cfg.BaseURL == "" {
        return nil, domain.ErrLLMMissingBaseURL
    }
    if cfg.Model == "" {
        return nil, domain.ErrLLMMissingModel
    }
    
    pcfg := ProviderConfig{
        Provider:    cfg.Provider,
        APIKey:      cfg.APIKey,
        BaseURL:     cfg.BaseURL,
        Model:       cfg.Model,
        MaxTokens:   cfg.MaxTokens,
        Temperature: cfg.Temperature,
        Timeout:     cfg.Timeout,
        MaxRetries:  cfg.MaxRetries,
    }
    
    return NewProvider(pcfg)
}

// NewProvider cria um novo provider baseado no tipo especificado
func NewProvider(cfg ProviderConfig) (domain.LLMProvider, error) {
    // Criar HTTP client com timeout
    timeout := cfg.Timeout
    if timeout == 0 {
        timeout = 60 * time.Second
    }
    
    httpClient := cfg.HTTPClient
    if httpClient == nil {
        httpClient = &http.Client{Timeout: timeout}
    }
    
    switch cfg.Provider {
    case "openai":
        return NewOpenAIProvider(cfg, httpClient)
    case "anthropic":
        return NewAnthropicProvider(cfg, httpClient)
    case "google":
        return NewGoogleProvider(cfg, httpClient)
    default:
        return nil, fmt.Errorf("%w: %s", domain.ErrLLMInvalidProvider, cfg.Provider)
    }
}
```

### 6.2 OpenAI Provider

```go
// internal/llm/openai.go

package llm

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strings"
    
    "github.com/quantmind-br/repodocs-go/internal/domain"
)

// OpenAI request/response types
type openAIRequest struct {
    Model       string            `json:"model"`
    Messages    []openAIMessage   `json:"messages"`
    MaxTokens   int               `json:"max_tokens,omitempty"`
    Temperature float64           `json:"temperature,omitempty"`
}

type openAIMessage struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}

type openAIResponse struct {
    ID      string `json:"id"`
    Model   string `json:"model"`
    Choices []struct {
        Index   int `json:"index"`
        Message struct {
            Role    string `json:"role"`
            Content string `json:"content"`
        } `json:"message"`
        FinishReason string `json:"finish_reason"`
    } `json:"choices"`
    Usage struct {
        PromptTokens     int `json:"prompt_tokens"`
        CompletionTokens int `json:"completion_tokens"`
        TotalTokens      int `json:"total_tokens"`
    } `json:"usage"`
    Error *struct {
        Message string `json:"message"`
        Type    string `json:"type"`
        Code    string `json:"code"`
    } `json:"error,omitempty"`
}

// OpenAIProvider implementa LLMProvider para APIs compatíveis com OpenAI
type OpenAIProvider struct {
    httpClient  *http.Client
    apiKey      string
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

// NewOpenAIProvider cria um novo OpenAI provider
func NewOpenAIProvider(cfg ProviderConfig, httpClient *http.Client) (*OpenAIProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    
    return &OpenAIProvider{
        httpClient:  httpClient,
        apiKey:      cfg.APIKey,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   cfg.MaxTokens,
        temperature: cfg.Temperature,
    }, nil
}

// Name retorna o nome do provider
func (p *OpenAIProvider) Name() string {
    return "openai"
}

// Complete envia uma requisição de completion
func (p *OpenAIProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // Converter mensagens para formato OpenAI
    messages := make([]openAIMessage, len(req.Messages))
    for i, msg := range req.Messages {
        messages[i] = openAIMessage{
            Role:    string(msg.Role),
            Content: msg.Content,
        }
    }
    
    // Construir request body
    maxTokens := req.MaxTokens
    if maxTokens == 0 {
        maxTokens = p.maxTokens
    }
    
    temp := p.temperature
    if req.Temperature != nil {
        temp = *req.Temperature
    }
    
    openAIReq := openAIRequest{
        Model:       p.model,
        Messages:    messages,
        MaxTokens:   maxTokens,
        Temperature: temp,
    }
    
    body, err := json.Marshal(openAIReq)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal request: %w", err)
    }
    
    // Criar HTTP request
    url := p.baseURL + "/chat/completions"
    httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }
    
    // Headers para OpenAI API
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("Authorization", "Bearer "+p.apiKey)
    
    // Executar request
    resp, err := p.httpClient.Do(httpReq)
    if err != nil {
        return nil, &domain.LLMError{
            Provider: "openai",
            Message:  fmt.Sprintf("request failed: %v", err),
            Err:      err,
        }
    }
    defer resp.Body.Close()
    
    // Ler response body
    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }
    
    // Parse response
    var openAIResp openAIResponse
    if err := json.Unmarshal(respBody, &openAIResp); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    // Verificar erros na response
    if openAIResp.Error != nil {
        return nil, &domain.LLMError{
            Provider:   "openai",
            StatusCode: resp.StatusCode,
            Message:    openAIResp.Error.Message,
        }
    }
    
    // Verificar status code
    if resp.StatusCode != http.StatusOK {
        return nil, p.handleHTTPError(resp.StatusCode, respBody)
    }
    
    // Verificar se há choices
    if len(openAIResp.Choices) == 0 {
        return nil, &domain.LLMError{
            Provider: "openai",
            Message:  "no choices in response",
        }
    }
    
    choice := openAIResp.Choices[0]
    
    return &domain.LLMResponse{
        Content:      choice.Message.Content,
        Model:        openAIResp.Model,
        FinishReason: choice.FinishReason,
        Usage: domain.LLMUsage{
            PromptTokens:     openAIResp.Usage.PromptTokens,
            CompletionTokens: openAIResp.Usage.CompletionTokens,
            TotalTokens:      openAIResp.Usage.TotalTokens,
        },
    }, nil
}

// Close libera recursos
func (p *OpenAIProvider) Close() error {
    return nil
}

// handleHTTPError trata erros HTTP
func (p *OpenAIProvider) handleHTTPError(statusCode int, body []byte) error {
    switch statusCode {
    case http.StatusUnauthorized:
        return &domain.LLMError{
            Provider:   "openai",
            StatusCode: statusCode,
            Message:    "authentication failed",
            Err:        domain.ErrLLMAuthFailed,
        }
    case http.StatusTooManyRequests:
        return &domain.LLMError{
            Provider:   "openai",
            StatusCode: statusCode,
            Message:    "rate limit exceeded",
            Err:        domain.ErrLLMRateLimited,
        }
    default:
        return &domain.LLMError{
            Provider:   "openai",
            StatusCode: statusCode,
            Message:    string(body),
        }
    }
}
```

### 6.3 Anthropic Provider

```go
// internal/llm/anthropic.go

package llm

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strings"
    
    "github.com/quantmind-br/repodocs-go/internal/domain"
)

const anthropicVersion = "2023-06-01"

// Anthropic request/response types
type anthropicRequest struct {
    Model     string              `json:"model"`
    MaxTokens int                 `json:"max_tokens"`
    Messages  []anthropicMessage  `json:"messages"`
    System    string              `json:"system,omitempty"`
}

type anthropicMessage struct {
    Role    string `json:"role"`
    Content string `json:"content"`
}

type anthropicResponse struct {
    ID      string `json:"id"`
    Type    string `json:"type"`
    Role    string `json:"role"`
    Model   string `json:"model"`
    Content []struct {
        Type string `json:"type"`
        Text string `json:"text"`
    } `json:"content"`
    StopReason string `json:"stop_reason"`
    Usage      struct {
        InputTokens  int `json:"input_tokens"`
        OutputTokens int `json:"output_tokens"`
    } `json:"usage"`
    Error *struct {
        Type    string `json:"type"`
        Message string `json:"message"`
    } `json:"error,omitempty"`
}

// AnthropicProvider implementa LLMProvider para API Anthropic
type AnthropicProvider struct {
    httpClient  *http.Client
    apiKey      string
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

// NewAnthropicProvider cria um novo Anthropic provider
func NewAnthropicProvider(cfg ProviderConfig, httpClient *http.Client) (*AnthropicProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    
    maxTokens := cfg.MaxTokens
    if maxTokens == 0 {
        maxTokens = 4096 // Anthropic requer max_tokens
    }
    
    return &AnthropicProvider{
        httpClient:  httpClient,
        apiKey:      cfg.APIKey,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   maxTokens,
        temperature: cfg.Temperature,
    }, nil
}

// Name retorna o nome do provider
func (p *AnthropicProvider) Name() string {
    return "anthropic"
}

// Complete envia uma requisição de completion
func (p *AnthropicProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // Separar system prompt das mensagens
    var systemPrompt string
    messages := make([]anthropicMessage, 0, len(req.Messages))
    
    for _, msg := range req.Messages {
        if msg.Role == domain.RoleSystem {
            systemPrompt = msg.Content
        } else {
            messages = append(messages, anthropicMessage{
                Role:    string(msg.Role),
                Content: msg.Content,
            })
        }
    }
    
    // Construir request
    maxTokens := req.MaxTokens
    if maxTokens == 0 {
        maxTokens = p.maxTokens
    }
    
    anthropicReq := anthropicRequest{
        Model:     p.model,
        MaxTokens: maxTokens,
        Messages:  messages,
        System:    systemPrompt,
    }
    
    body, err := json.Marshal(anthropicReq)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal request: %w", err)
    }
    
    // Criar HTTP request
    url := p.baseURL + "/v1/messages"
    httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }
    
    // Headers específicos da Anthropic
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("x-api-key", p.apiKey)
    httpReq.Header.Set("anthropic-version", anthropicVersion)
    
    // Executar request
    resp, err := p.httpClient.Do(httpReq)
    if err != nil {
        return nil, &domain.LLMError{
            Provider: "anthropic",
            Message:  fmt.Sprintf("request failed: %v", err),
            Err:      err,
        }
    }
    defer resp.Body.Close()
    
    // Ler response body
    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }
    
    // Parse response
    var anthropicResp anthropicResponse
    if err := json.Unmarshal(respBody, &anthropicResp); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    // Verificar erros
    if anthropicResp.Error != nil {
        return nil, &domain.LLMError{
            Provider:   "anthropic",
            StatusCode: resp.StatusCode,
            Message:    anthropicResp.Error.Message,
        }
    }
    
    if resp.StatusCode != http.StatusOK {
        return nil, p.handleHTTPError(resp.StatusCode, respBody)
    }
    
    // Extrair conteúdo de texto
    var content string
    for _, block := range anthropicResp.Content {
        if block.Type == "text" {
            content += block.Text
        }
    }
    
    return &domain.LLMResponse{
        Content:      content,
        Model:        anthropicResp.Model,
        FinishReason: anthropicResp.StopReason,
        Usage: domain.LLMUsage{
            PromptTokens:     anthropicResp.Usage.InputTokens,
            CompletionTokens: anthropicResp.Usage.OutputTokens,
            TotalTokens:      anthropicResp.Usage.InputTokens + anthropicResp.Usage.OutputTokens,
        },
    }, nil
}

// Close libera recursos
func (p *AnthropicProvider) Close() error {
    return nil
}

// handleHTTPError trata erros HTTP
func (p *AnthropicProvider) handleHTTPError(statusCode int, body []byte) error {
    switch statusCode {
    case http.StatusUnauthorized:
        return &domain.LLMError{
            Provider:   "anthropic",
            StatusCode: statusCode,
            Message:    "authentication failed",
            Err:        domain.ErrLLMAuthFailed,
        }
    case http.StatusTooManyRequests:
        return &domain.LLMError{
            Provider:   "anthropic",
            StatusCode: statusCode,
            Message:    "rate limit exceeded",
            Err:        domain.ErrLLMRateLimited,
        }
    default:
        return &domain.LLMError{
            Provider:   "anthropic",
            StatusCode: statusCode,
            Message:    string(body),
        }
    }
}
```

### 6.4 Google Provider

```go
// internal/llm/google.go

package llm

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "strings"
    
    "github.com/quantmind-br/repodocs-go/internal/domain"
)

// Google Gemini request/response types
type googleRequest struct {
    Contents          []googleContent   `json:"contents"`
    SystemInstruction *googleContent    `json:"systemInstruction,omitempty"`
    GenerationConfig  *googleGenConfig  `json:"generationConfig,omitempty"`
}

type googleContent struct {
    Role  string       `json:"role,omitempty"`
    Parts []googlePart `json:"parts"`
}

type googlePart struct {
    Text string `json:"text"`
}

type googleGenConfig struct {
    MaxOutputTokens int     `json:"maxOutputTokens,omitempty"`
    Temperature     float64 `json:"temperature,omitempty"`
}

type googleResponse struct {
    Candidates []struct {
        Content struct {
            Role  string `json:"role"`
            Parts []struct {
                Text string `json:"text"`
            } `json:"parts"`
        } `json:"content"`
        FinishReason string `json:"finishReason"`
    } `json:"candidates"`
    UsageMetadata struct {
        PromptTokenCount     int `json:"promptTokenCount"`
        CandidatesTokenCount int `json:"candidatesTokenCount"`
        TotalTokenCount      int `json:"totalTokenCount"`
    } `json:"usageMetadata"`
    Error *struct {
        Code    int    `json:"code"`
        Message string `json:"message"`
        Status  string `json:"status"`
    } `json:"error,omitempty"`
}

// GoogleProvider implementa LLMProvider para Google Gemini API
type GoogleProvider struct {
    httpClient  *http.Client
    apiKey      string
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

// NewGoogleProvider cria um novo Google provider
func NewGoogleProvider(cfg ProviderConfig, httpClient *http.Client) (*GoogleProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    
    return &GoogleProvider{
        httpClient:  httpClient,
        apiKey:      cfg.APIKey,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   cfg.MaxTokens,
        temperature: cfg.Temperature,
    }, nil
}

// Name retorna o nome do provider
func (p *GoogleProvider) Name() string {
    return "google"
}

// Complete envia uma requisição de completion
func (p *GoogleProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // Converter mensagens para formato Google
    var systemInstruction *googleContent
    contents := make([]googleContent, 0, len(req.Messages))
    
    for _, msg := range req.Messages {
        switch msg.Role {
        case domain.RoleSystem:
            systemInstruction = &googleContent{
                Parts: []googlePart{{Text: msg.Content}},
            }
        case domain.RoleUser:
            contents = append(contents, googleContent{
                Role:  "user",
                Parts: []googlePart{{Text: msg.Content}},
            })
        case domain.RoleAssistant:
            contents = append(contents, googleContent{
                Role:  "model",
                Parts: []googlePart{{Text: msg.Content}},
            })
        }
    }
    
    // Construir request
    googleReq := googleRequest{
        Contents:          contents,
        SystemInstruction: systemInstruction,
    }
    
    // Generation config
    maxTokens := req.MaxTokens
    if maxTokens == 0 {
        maxTokens = p.maxTokens
    }
    
    temp := p.temperature
    if req.Temperature != nil {
        temp = *req.Temperature
    }
    
    if maxTokens > 0 || temp > 0 {
        googleReq.GenerationConfig = &googleGenConfig{
            MaxOutputTokens: maxTokens,
            Temperature:     temp,
        }
    }
    
    body, err := json.Marshal(googleReq)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal request: %w", err)
    }
    
    // Criar HTTP request
    // Google usa o modelo na URL
    url := fmt.Sprintf("%s/v1beta/models/%s:generateContent", p.baseURL, p.model)
    httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }
    
    // Headers para Google API
    httpReq.Header.Set("Content-Type", "application/json")
    httpReq.Header.Set("x-goog-api-key", p.apiKey)
    
    // Executar request
    resp, err := p.httpClient.Do(httpReq)
    if err != nil {
        return nil, &domain.LLMError{
            Provider: "google",
            Message:  fmt.Sprintf("request failed: %v", err),
            Err:      err,
        }
    }
    defer resp.Body.Close()
    
    // Ler response body
    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }
    
    // Parse response
    var googleResp googleResponse
    if err := json.Unmarshal(respBody, &googleResp); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }
    
    // Verificar erros
    if googleResp.Error != nil {
        return nil, &domain.LLMError{
            Provider:   "google",
            StatusCode: googleResp.Error.Code,
            Message:    googleResp.Error.Message,
        }
    }
    
    if resp.StatusCode != http.StatusOK {
        return nil, p.handleHTTPError(resp.StatusCode, respBody)
    }
    
    // Verificar candidates
    if len(googleResp.Candidates) == 0 {
        return nil, &domain.LLMError{
            Provider: "google",
            Message:  "no candidates in response",
        }
    }
    
    candidate := googleResp.Candidates[0]
    
    // Extrair texto
    var content string
    for _, part := range candidate.Content.Parts {
        content += part.Text
    }
    
    return &domain.LLMResponse{
        Content:      content,
        Model:        p.model,
        FinishReason: candidate.FinishReason,
        Usage: domain.LLMUsage{
            PromptTokens:     googleResp.UsageMetadata.PromptTokenCount,
            CompletionTokens: googleResp.UsageMetadata.CandidatesTokenCount,
            TotalTokens:      googleResp.UsageMetadata.TotalTokenCount,
        },
    }, nil
}

// Close libera recursos
func (p *GoogleProvider) Close() error {
    return nil
}

// handleHTTPError trata erros HTTP
func (p *GoogleProvider) handleHTTPError(statusCode int, body []byte) error {
    switch statusCode {
    case http.StatusUnauthorized, http.StatusForbidden:
        return &domain.LLMError{
            Provider:   "google",
            StatusCode: statusCode,
            Message:    "authentication failed",
            Err:        domain.ErrLLMAuthFailed,
        }
    case http.StatusTooManyRequests:
        return &domain.LLMError{
            Provider:   "google",
            StatusCode: statusCode,
            Message:    "rate limit exceeded",
            Err:        domain.ErrLLMRateLimited,
        }
    default:
        return &domain.LLMError{
            Provider:   "google",
            StatusCode: statusCode,
            Message:    string(body),
        }
    }
}
```

---

## 7. Cliente HTTP

### 7.1 Retry com Backoff

```go
// internal/llm/retry.go

package llm

import (
    "context"
    "math"
    "math/rand"
    "net/http"
    "time"
)

// RetryConfig configura o comportamento de retry
type RetryConfig struct {
    MaxRetries      int
    InitialInterval time.Duration
    MaxInterval     time.Duration
    Multiplier      float64
}

// DefaultRetryConfig retorna configuração padrão
func DefaultRetryConfig() RetryConfig {
    return RetryConfig{
        MaxRetries:      3,
        InitialInterval: 1 * time.Second,
        MaxInterval:     30 * time.Second,
        Multiplier:      2.0,
    }
}

// ShouldRetry verifica se deve fazer retry baseado no status code
func ShouldRetry(statusCode int) bool {
    switch statusCode {
    case http.StatusTooManyRequests,
        http.StatusInternalServerError,
        http.StatusBadGateway,
        http.StatusServiceUnavailable,
        http.StatusGatewayTimeout:
        return true
    default:
        return false
    }
}

// CalculateBackoff calcula o tempo de espera para o próximo retry
func CalculateBackoff(attempt int, cfg RetryConfig) time.Duration {
    backoff := float64(cfg.InitialInterval) * math.Pow(cfg.Multiplier, float64(attempt))
    
    // Adicionar jitter (±10%)
    jitter := backoff * 0.1 * (rand.Float64()*2 - 1)
    backoff += jitter
    
    // Limitar ao máximo
    if backoff > float64(cfg.MaxInterval) {
        backoff = float64(cfg.MaxInterval)
    }
    
    return time.Duration(backoff)
}
```

---

## 8. Tratamento de Erros

### 8.1 Classificação de Erros

| HTTP Status | Erro | Retry? | Ação do Usuário |
|-------------|------|--------|-----------------|
| 401, 403 | `ErrLLMAuthFailed` | Não | Verificar API key |
| 429 | `ErrLLMRateLimited` | Sim | Aguardar e retry |
| 400 | `ErrLLMInvalidRequest` | Não | Corrigir request |
| 500-599 | Server error | Sim | Auto-retry |

### 8.2 Logging de Erros

```go
// Em caso de erro, logar informações úteis (sem expor secrets)
logger.Error().
    Str("provider", provider.Name()).
    Str("model", req.Model).
    Int("status_code", err.StatusCode).
    Str("error_type", err.Type).
    Msg("LLM request failed")
```

---

## 9. Testes

### 9.1 Estrutura de Testes

```
tests/
├── unit/
│   └── llm/
│       ├── openai_test.go
│       ├── anthropic_test.go
│       ├── google_test.go
│       └── provider_test.go
├── integration/
│   └── llm/
│       ├── openai_integration_test.go
│       ├── anthropic_integration_test.go
│       └── google_integration_test.go
└── mocks/
    └── llm.go
```

### 9.2 Testes Unitários com Mock Server

```go
// tests/unit/llm/openai_test.go

func TestOpenAIProvider_Complete(t *testing.T) {
    tests := []struct {
        name           string
        responseBody   string
        responseStatus int
        wantErr        bool
        wantContent    string
    }{
        {
            name: "successful completion",
            responseBody: `{
                "id": "chatcmpl-123",
                "model": "gpt-4",
                "choices": [{
                    "index": 0,
                    "message": {"role": "assistant", "content": "Hello!"},
                    "finish_reason": "stop"
                }],
                "usage": {"prompt_tokens": 10, "completion_tokens": 5, "total_tokens": 15}
            }`,
            responseStatus: 200,
            wantErr:        false,
            wantContent:    "Hello!",
        },
        {
            name:           "authentication error",
            responseBody:   `{"error": {"message": "Invalid API key"}}`,
            responseStatus: 401,
            wantErr:        true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                // Verificar headers
                assert.Equal(t, "application/json", r.Header.Get("Content-Type"))
                assert.Contains(t, r.Header.Get("Authorization"), "Bearer ")
                
                w.WriteHeader(tt.responseStatus)
                w.Write([]byte(tt.responseBody))
            }))
            defer server.Close()
            
            provider, err := NewOpenAIProvider(ProviderConfig{
                APIKey:  "test-key",
                BaseURL: server.URL,
                Model:   "test-model",
            }, server.Client())
            require.NoError(t, err)
            
            resp, err := provider.Complete(context.Background(), &domain.LLMRequest{
                Messages: []domain.LLMMessage{
                    {Role: domain.RoleUser, Content: "Hello"},
                },
            })
            
            if tt.wantErr {
                require.Error(t, err)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.wantContent, resp.Content)
            }
        })
    }
}
```

### 9.3 Mock Provider

```go
// tests/mocks/llm.go

type MockLLMProvider struct {
    mock.Mock
}

func (m *MockLLMProvider) Name() string {
    return m.Called().String(0)
}

func (m *MockLLMProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    args := m.Called(ctx, req)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*domain.LLMResponse), args.Error(1)
}

func (m *MockLLMProvider) Close() error {
    return m.Called().Error(0)
}
```

---

## 10. Fases de Implementação

### Fase 1: Fundação (3-4 dias)

**Objetivo:** Infraestrutura básica sem quebrar funcionalidade existente

- [ ] Adicionar tipos de domínio em `internal/domain/`
  - `LLMProvider` interface
  - `LLMRequest`, `LLMResponse`, `LLMMessage` structs
  - Erros específicos de LLM
- [ ] Adicionar `LLMConfig` em `internal/config/config.go`
- [ ] Adicionar defaults em `internal/config/loader.go`
- [ ] Criar estrutura de pacote `internal/llm/`
- [ ] Implementar factory `NewProvider()` e `NewProviderFromConfig()`

**Entregáveis:**
- Interfaces e tipos definidos
- Configuração carregando corretamente
- Factory funcionando (retornando erro para provider não implementado)

### Fase 2: OpenAI Provider (2-3 dias)

**Objetivo:** Suporte completo a endpoints compatíveis com OpenAI

- [ ] Implementar `OpenAIProvider`
  - Construção de request JSON
  - Headers corretos (`Authorization: Bearer`)
  - Parse de response
  - Tratamento de erros HTTP
- [ ] Testes unitários com mock server
- [ ] Teste de integração (opcional, requer API key)

**Entregáveis:**
- Provider funcional para OpenAI e compatíveis (Ollama, vLLM, etc.)
- Cobertura de testes > 80%

### Fase 3: Anthropic Provider (2-3 dias)

**Objetivo:** Suporte completo a API Anthropic

- [ ] Implementar `AnthropicProvider`
  - Headers específicos (`x-api-key`, `anthropic-version`)
  - Tratamento de system prompt separado
  - Parse de content blocks
- [ ] Testes unitários
- [ ] Teste de integração

**Entregáveis:**
- Provider funcional para Anthropic Claude
- Cobertura de testes > 80%

### Fase 4: Google Provider (2-3 dias)

**Objetivo:** Suporte completo a Google Gemini API

- [ ] Implementar `GoogleProvider`
  - URL com modelo no path
  - Header `x-goog-api-key`
  - Format de contents/parts
  - SystemInstruction
- [ ] Testes unitários
- [ ] Teste de integração

**Entregáveis:**
- Provider funcional para Google Gemini
- Cobertura de testes > 80%

### Fase 5: Integração e Documentação (2 dias)

**Objetivo:** Integrar ao sistema e documentar

- [ ] Integrar LLM provider em `Dependencies`
- [ ] Adicionar validação de configuração no startup
- [ ] Atualizar README com documentação de configuração
- [ ] Adicionar exemplos para providers comuns
- [ ] Comando `repodocs doctor` para verificar configuração LLM

**Entregáveis:**
- Sistema integrado
- Documentação completa
- Exemplos de configuração

---

## 11. Estrutura de Arquivos

```
internal/
├── config/
│   ├── config.go        # + LLMConfig struct
│   ├── defaults.go      # + DefaultLLM* constants
│   └── loader.go        # + setDefaults para LLM
├── domain/
│   ├── errors.go        # + ErrLLM* errors
│   ├── interfaces.go    # + LLMProvider interface
│   └── models.go        # + LLMRequest, LLMResponse, etc.
├── llm/
│   ├── provider.go      # Factory functions
│   ├── openai.go        # OpenAI provider implementation
│   ├── anthropic.go     # Anthropic provider implementation
│   ├── google.go        # Google provider implementation
│   └── retry.go         # Retry logic
└── strategies/
    └── strategy.go      # + LLM field in Dependencies

tests/
├── unit/
│   └── llm/
│       ├── openai_test.go
│       ├── anthropic_test.go
│       ├── google_test.go
│       └── provider_test.go
├── integration/
│   └── llm/
│       └── provider_integration_test.go
└── mocks/
    └── llm.go
```

---

## 12. Exemplos de Uso

### 12.1 OpenAI Oficial

```yaml
# config.yaml
llm:
  provider: openai
  base_url: https://api.openai.com/v1
  model: gpt-4o
```

```bash
export REPODOCS_LLM_API_KEY=sk-your-openai-key
```

### 12.2 Ollama (Local)

```yaml
llm:
  provider: openai
  base_url: http://localhost:11434/v1
  model: llama3.2
  api_key: ollama  # Ollama não requer key real
```

### 12.3 vLLM (Self-hosted)

```yaml
llm:
  provider: openai
  base_url: http://your-server:8000/v1
  model: meta-llama/Llama-3-70b-chat-hf
  api_key: token-xyz
```

### 12.4 Anthropic Claude

```yaml
llm:
  provider: anthropic
  base_url: https://api.anthropic.com
  model: claude-3-5-sonnet-20241022
```

```bash
export REPODOCS_LLM_API_KEY=sk-ant-your-anthropic-key
```

### 12.5 Google Gemini

```yaml
llm:
  provider: google
  base_url: https://generativelanguage.googleapis.com
  model: gemini-1.5-pro
```

```bash
export REPODOCS_LLM_API_KEY=your-google-api-key
```

### 12.6 Groq (OpenAI-compatible)

```yaml
llm:
  provider: openai
  base_url: https://api.groq.com/openai/v1
  model: llama3-70b-8192
```

```bash
export REPODOCS_LLM_API_KEY=gsk_your-groq-key
```

### 12.7 Together AI (OpenAI-compatible)

```yaml
llm:
  provider: openai
  base_url: https://api.together.xyz/v1
  model: meta-llama/Llama-3-70b-chat-hf
```

```bash
export REPODOCS_LLM_API_KEY=your-together-key
```

---

## Resumo

Este plano implementa suporte a múltiplos provedores LLM de forma flexível e extensível:

1. **3 formatos de API suportados:** OpenAI, Anthropic, Google
2. **Configuração mínima:** `provider`, `api_key`, `base_url`, `model`
3. **Zero hardcode:** Usuário especifica qualquer modelo
4. **Cliente HTTP nativo:** Máximo controle, sem dependências de SDKs
5. **Compatibilidade ampla:** Funciona com dezenas de provedores que implementam esses padrões

A implementação segue os padrões existentes do codebase (hexagonal architecture, viper config, interface-based design) e pode ser completada em aproximadamente 2 semanas.

---

*Documento gerado: 2025-12-31*  
*Versão: 2.0*
