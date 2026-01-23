# Plano de Implementacao: Ollama Provider

## Resumo Executivo

Implementar compatibilidade com o provider Ollama para permitir que usuarios utilizem modelos LLM locais (como Llama, Mistral, Gemma, etc.) para enriquecimento de metadados via IA. O Ollama oferece uma API compativel com OpenAI em `/v1/chat/completions`, o que simplifica significativamente a integracao.

## Analise de Requisitos

### Requisitos Funcionais
- [ ] Suporte ao provider "ollama" na configuracao LLM
- [ ] Conexao com servidor Ollama local (default: `http://localhost:11434`)
- [ ] Suporte a API OpenAI-compatible (`/v1/chat/completions`)
- [ ] Suporte a todos os modelos disponiveis no Ollama (llama3, mistral, gemma, codellama, etc.)
- [ ] Configuracao de temperatura, max_tokens e outros parametros
- [ ] Tratamento de erros especificos do Ollama
- [ ] Documentacao de uso na TUI de configuracao

### Requisitos Nao-Funcionais
- [ ] Performance: timeout configuravel para modelos locais (podem ser mais lentos)
- [ ] Compatibilidade: manter consistencia com outros providers (openai, anthropic, google)
- [ ] Testabilidade: testes unitarios com mock server
- [ ] Usabilidade: API key opcional (Ollama local nao requer autenticacao)

## Analise Tecnica

### Arquitetura Proposta

```
                                    +------------------+
                                    |   NewProvider()  |
                                    +--------+---------+
                                             |
              +------------+------------+----+----+------------+
              |            |            |         |            |
              v            v            v         v            v
        +---------+  +-----------+  +--------+  +--------+  +--------+
        | OpenAI  |  | Anthropic |  | Google |  | Ollama |  | Future |
        +---------+  +-----------+  +--------+  +--------+  +--------+
              |            |            |         |            |
              +------------+------------+---------+------------+
                                    |
                           +--------v--------+
                           | domain.LLMProvider |
                           +-----------------+
```

O Ollama sera implementado como um novo provider que implementa a interface `domain.LLMProvider`. Como o Ollama oferece API compativel com OpenAI, podemos reutilizar grande parte da logica do `OpenAIProvider`, adaptando apenas:
1. Headers de autenticacao (opcional para Ollama)
2. BaseURL default (`http://localhost:11434/v1`)
3. Nome do provider
4. Tratamento de erros especificos

### Componentes Afetados

| Arquivo/Modulo | Tipo de Mudanca | Descricao |
|----------------|-----------------|-----------|
| `internal/llm/ollama.go` | Criar | Novo provider Ollama |
| `internal/llm/ollama_test.go` | Criar | Testes unitarios do provider |
| `internal/llm/provider.go` | Modificar | Adicionar case "ollama" no switch |
| `internal/llm/provider_test.go` | Modificar | Adicionar testes para novo provider |
| `internal/tui/validation.go` | Modificar | Adicionar "ollama" aos providers validos |
| `internal/config/defaults.go` | Modificar | Adicionar constante para BaseURL default do Ollama |
| `tests/unit/llm/ollama_test.go` | Criar | Testes unitarios adicionais |
| `README.md` | Modificar | Documentar uso do Ollama |

### Dependencias

- **Dependencias de pacotes/bibliotecas**: Nenhuma nova (usa `net/http` padrao)
- **Dependencias de outras features**: Nenhuma
- **Dependencias de APIs externas**: Ollama server (local ou remoto)

## Plano de Implementacao

### Fase 1: Provider Core

**Objetivo**: Implementar o provider Ollama com API compativel com OpenAI

#### Tarefas:

1. **Criar arquivo `internal/llm/ollama.go`**
   - Descricao: Implementar struct `OllamaProvider` que implementa `domain.LLMProvider`
   - Arquivos envolvidos: `internal/llm/ollama.go`
   - Codigo de exemplo:

```go
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

// OllamaProvider implements the LLM provider interface for Ollama
// Ollama exposes an OpenAI-compatible API at /v1/chat/completions
type OllamaProvider struct {
    httpClient  *http.Client
    baseURL     string
    model       string
    maxTokens   int
    temperature float64
}

// NewOllamaProvider creates a new Ollama provider instance
func NewOllamaProvider(cfg ProviderConfig, httpClient *http.Client) (*OllamaProvider, error) {
    baseURL := strings.TrimSuffix(cfg.BaseURL, "/")
    
    return &OllamaProvider{
        httpClient:  httpClient,
        baseURL:     baseURL,
        model:       cfg.Model,
        maxTokens:   cfg.MaxTokens,
        temperature: cfg.Temperature,
    }, nil
}

func (p *OllamaProvider) Name() string {
    return "ollama"
}

func (p *OllamaProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // Reutiliza estruturas OpenAI (API compativel)
    messages := make([]openAIMessage, len(req.Messages))
    for i, msg := range req.Messages {
        messages[i] = openAIMessage{
            Role:    string(msg.Role),
            Content: msg.Content,
        }
    }

    maxTokens := req.MaxTokens
    if maxTokens == 0 {
        maxTokens = p.maxTokens
    }

    temp := p.temperature
    if req.Temperature != nil {
        temp = *req.Temperature
    }

    ollamaReq := openAIRequest{
        Model:       p.model,
        Messages:    messages,
        MaxTokens:   maxTokens,
        Temperature: temp,
    }

    body, err := json.Marshal(ollamaReq)
    if err != nil {
        return nil, fmt.Errorf("failed to marshal request: %w", err)
    }

    // Ollama usa endpoint compativel com OpenAI
    url := p.baseURL + "/chat/completions"
    httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    httpReq.Header.Set("Content-Type", "application/json")
    // Ollama nao requer Authorization header para servidor local

    resp, err := p.httpClient.Do(httpReq)
    if err != nil {
        return nil, &domain.LLMError{
            Provider: "ollama",
            Message:  fmt.Sprintf("request failed: %v", err),
            Err:      err,
        }
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response: %w", err)
    }

    var ollamaResp openAIResponse
    if err := json.Unmarshal(respBody, &ollamaResp); err != nil {
        return nil, fmt.Errorf("failed to parse response: %w", err)
    }

    if ollamaResp.Error != nil {
        return nil, p.handleAPIError(resp.StatusCode, ollamaResp.Error.Message)
    }

    if resp.StatusCode != http.StatusOK {
        return nil, p.handleHTTPError(resp.StatusCode, respBody)
    }

    if len(ollamaResp.Choices) == 0 {
        return nil, &domain.LLMError{
            Provider: "ollama",
            Message:  "no choices in response",
        }
    }

    choice := ollamaResp.Choices[0]

    return &domain.LLMResponse{
        Content:      choice.Message.Content,
        Model:        ollamaResp.Model,
        FinishReason: choice.FinishReason,
        Usage: domain.LLMUsage{
            PromptTokens:     ollamaResp.Usage.PromptTokens,
            CompletionTokens: ollamaResp.Usage.CompletionTokens,
            TotalTokens:      ollamaResp.Usage.TotalTokens,
        },
    }, nil
}

func (p *OllamaProvider) Close() error {
    return nil
}

func (p *OllamaProvider) handleAPIError(statusCode int, message string) error {
    // Ollama erros comuns
    switch statusCode {
    case http.StatusNotFound:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    message,
            Err:        fmt.Errorf("model not found - ensure model is pulled: ollama pull %s", p.model),
        }
    case http.StatusServiceUnavailable:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    "ollama server unavailable",
            Err:        domain.ErrLLMRequestFailed,
        }
    default:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    message,
            Err:        domain.ErrLLMRequestFailed,
        }
    }
}

func (p *OllamaProvider) handleHTTPError(statusCode int, body []byte) error {
    switch statusCode {
    case http.StatusTooManyRequests:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    "rate limit exceeded",
            Err:        domain.ErrLLMRateLimited,
        }
    case http.StatusNotFound:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    "model not found",
            Err:        domain.ErrLLMRequestFailed,
        }
    default:
        return &domain.LLMError{
            Provider:   "ollama",
            StatusCode: statusCode,
            Message:    string(body),
        }
    }
}
```

2. **Modificar `internal/llm/provider.go`**
   - Descricao: Adicionar case "ollama" no switch de NewProvider
   - Arquivos envolvidos: `internal/llm/provider.go`
   - Modificacao:

```go
// Em NewProvider(), adicionar case no switch:
case "ollama":
    return NewOllamaProvider(cfg, httpClient)
```

3. **Modificar `internal/llm/provider.go` - Validacao de API Key**
   - Descricao: Tornar API Key opcional para provider Ollama
   - Arquivos envolvidos: `internal/llm/provider.go`
   - Modificacao em `NewProviderFromConfig`:

```go
func NewProviderFromConfig(cfg *config.LLMConfig) (domain.LLMProvider, error) {
    if cfg.Provider == "" {
        return nil, domain.ErrLLMNotConfigured
    }
    // API key opcional para Ollama
    if cfg.APIKey == "" && cfg.Provider != "ollama" {
        return nil, domain.ErrLLMMissingAPIKey
    }
    if cfg.BaseURL == "" {
        return nil, domain.ErrLLMMissingBaseURL
    }
    if cfg.Model == "" {
        return nil, domain.ErrLLMMissingModel
    }
    // ...resto da funcao
}
```

### Fase 2: Configuracao e Defaults

**Objetivo**: Adicionar constantes e validacao para o provider Ollama

#### Tarefas:

1. **Modificar `internal/config/defaults.go`**
   - Descricao: Adicionar constante para BaseURL default do Ollama
   - Arquivos envolvidos: `internal/config/defaults.go`
   - Codigo:

```go
const (
    // ... constantes existentes ...
    
    // Ollama defaults
    DefaultOllamaBaseURL = "http://localhost:11434/v1"
    DefaultOllamaTimeout = 120 * time.Second // Modelos locais podem ser mais lentos
)
```

2. **Modificar `internal/tui/validation.go`**
   - Descricao: Adicionar "ollama" a lista de providers validos
   - Arquivos envolvidos: `internal/tui/validation.go`
   - Verificar se existe validacao de provider e adicionar "ollama"

### Fase 3: Testes Unitarios

**Objetivo**: Garantir cobertura de testes para o novo provider

#### Tarefas:

1. **Criar `internal/llm/ollama_test.go`**
   - Descricao: Testes unitarios completos para OllamaProvider
   - Arquivos envolvidos: `internal/llm/ollama_test.go`
   - Codigo de exemplo:

```go
package llm

import (
    "context"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
    "time"

    "github.com/quantmind-br/repodocs-go/internal/domain"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestNewOllamaProvider(t *testing.T) {
    cfg := ProviderConfig{
        BaseURL:     "http://localhost:11434/v1/",
        Model:       "llama3",
        MaxTokens:   1000,
        Temperature: 0.7,
    }

    provider, err := NewOllamaProvider(cfg, &http.Client{Timeout: 30 * time.Second})
    require.NoError(t, err)
    assert.NotNil(t, provider)
    assert.Equal(t, "ollama", provider.Name())
}

func TestOllamaProvider_Complete_Success(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        assert.Equal(t, "POST", r.Method)
        assert.Equal(t, "/chat/completions", r.URL.Path)
        assert.Equal(t, "application/json", r.Header.Get("Content-Type"))
        // Ollama nao requer Authorization
        assert.Empty(t, r.Header.Get("Authorization"))

        response := openAIResponse{
            ID:    "test-id",
            Model: "llama3",
            Choices: []struct {
                Index   int `json:"index"`
                Message struct {
                    Role    string `json:"role"`
                    Content string `json:"content"`
                } `json:"message"`
                FinishReason string `json:"finish_reason"`
            }{
                {
                    Message: struct {
                        Role    string `json:"role"`
                        Content string `json:"content"`
                    }{
                        Role:    "assistant",
                        Content: "Test response from Ollama",
                    },
                    FinishReason: "stop",
                },
            },
            Usage: struct {
                PromptTokens     int `json:"prompt_tokens"`
                CompletionTokens int `json:"completion_tokens"`
                TotalTokens      int `json:"total_tokens"`
            }{
                PromptTokens:     10,
                CompletionTokens: 5,
                TotalTokens:      15,
            },
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(response)
    }))
    defer server.Close()

    cfg := ProviderConfig{
        BaseURL: server.URL,
        Model:   "llama3",
    }
    provider, err := NewOllamaProvider(cfg, server.Client())
    require.NoError(t, err)

    ctx := context.Background()
    req := &domain.LLMRequest{
        Messages: []domain.LLMMessage{
            {Role: domain.RoleUser, Content: "Hello"},
        },
    }

    resp, err := provider.Complete(ctx, req)
    require.NoError(t, err)
    assert.Equal(t, "Test response from Ollama", resp.Content)
    assert.Equal(t, "llama3", resp.Model)
    assert.Equal(t, "stop", resp.FinishReason)
}

func TestOllamaProvider_Complete_ModelNotFound(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusNotFound)
        w.Write([]byte(`{"error": "model not found"}`))
    }))
    defer server.Close()

    cfg := ProviderConfig{
        BaseURL: server.URL,
        Model:   "nonexistent-model",
    }
    provider, err := NewOllamaProvider(cfg, server.Client())
    require.NoError(t, err)

    ctx := context.Background()
    req := &domain.LLMRequest{
        Messages: []domain.LLMMessage{
            {Role: domain.RoleUser, Content: "Hello"},
        },
    }

    resp, err := provider.Complete(ctx, req)
    assert.Error(t, err)
    assert.Nil(t, resp)
    
    var llmErr *domain.LLMError
    assert.ErrorAs(t, err, &llmErr)
    assert.Equal(t, "ollama", llmErr.Provider)
    assert.Equal(t, http.StatusNotFound, llmErr.StatusCode)
}

func TestOllamaProvider_Complete_ServerUnavailable(t *testing.T) {
    // Tenta conectar a um servidor que nao existe
    cfg := ProviderConfig{
        BaseURL: "http://localhost:99999/v1",
        Model:   "llama3",
    }
    provider, err := NewOllamaProvider(cfg, &http.Client{Timeout: 1 * time.Second})
    require.NoError(t, err)

    ctx := context.Background()
    req := &domain.LLMRequest{
        Messages: []domain.LLMMessage{
            {Role: domain.RoleUser, Content: "Hello"},
        },
    }

    resp, err := provider.Complete(ctx, req)
    assert.Error(t, err)
    assert.Nil(t, resp)
    
    var llmErr *domain.LLMError
    assert.ErrorAs(t, err, &llmErr)
    assert.Equal(t, "ollama", llmErr.Provider)
}

func TestOllamaProvider_Close(t *testing.T) {
    cfg := ProviderConfig{
        BaseURL: "http://localhost:11434/v1",
        Model:   "llama3",
    }
    provider, err := NewOllamaProvider(cfg, &http.Client{})
    require.NoError(t, err)

    err = provider.Close()
    assert.NoError(t, err)
}
```

2. **Modificar `internal/llm/provider_test.go`**
   - Descricao: Adicionar testes para criacao de provider Ollama
   - Arquivos envolvidos: `internal/llm/provider_test.go`
   - Adicionar casos de teste:

```go
// Em TestNewProviderFromConfig, adicionar:
{
    name: "valid ollama config",
    cfg: &config.LLMConfig{
        Provider: "ollama",
        BaseURL:  "http://localhost:11434/v1",
        Model:    "llama3",
        // APIKey opcional para Ollama
    },
    wantErr: nil,
},
{
    name: "valid ollama config with api key",
    cfg: &config.LLMConfig{
        Provider: "ollama",
        APIKey:   "optional-key",
        BaseURL:  "http://localhost:11434/v1",
        Model:    "llama3",
    },
    wantErr: nil,
},

// Em TestNewProvider, adicionar:
{
    name: "valid ollama",
    cfg: ProviderConfig{
        Provider: "ollama",
        BaseURL:  "http://localhost:11434/v1",
        Model:    "llama3",
    },
    wantErr: false,
},
```

### Fase 4: Documentacao e TUI

**Objetivo**: Documentar o uso do Ollama e atualizar interface de configuracao

#### Tarefas:

1. **Atualizar README.md**
   - Descricao: Adicionar secao sobre uso do Ollama
   - Arquivos envolvidos: `README.md`
   - Conteudo a adicionar:

```markdown
### Using Ollama for Local LLM

RepoDocs supports [Ollama](https://ollama.ai/) for running LLM inference locally:

1. Install Ollama: https://ollama.ai/download
2. Pull a model: `ollama pull llama3`
3. Configure RepoDocs:

```yaml
# ~/.repodocs/config.yaml
llm:
  provider: ollama
  base_url: http://localhost:11434/v1
  model: llama3
  enhance_metadata: true
```

Or use the interactive config:
```bash
repodocs config
```

**Supported models**: Any model available in Ollama (llama3, mistral, gemma, codellama, etc.)

**Note**: Ollama runs locally, so no API key is required.
```

2. **Modificar `internal/tui/validation.go`**
   - Descricao: Adicionar "ollama" a lista de providers validos
   - Arquivos envolvidos: `internal/tui/validation.go`
   - Modificacao na funcao `ValidateLLMProvider`:

```go
// ValidateLLMProvider validates LLM provider values
func ValidateLLMProvider(s string) error {
    if s == "" {
        return nil // Empty is valid (LLM disabled)
    }
    validProviders := map[string]bool{
        "openai":    true,
        "anthropic": true,
        "google":    true,
        "ollama":    true,  // <-- ADICIONAR
    }
    if !validProviders[strings.ToLower(s)] {
        return fmt.Errorf("invalid LLM provider: must be openai, anthropic, google, or ollama")
    }
    return nil
}
```

## Estrategia de Testes

### Testes Unitarios
- [ ] TestNewOllamaProvider - criacao do provider
- [ ] TestOllamaProvider_Complete_Success - resposta bem sucedida
- [ ] TestOllamaProvider_Complete_ModelNotFound - modelo nao encontrado (404)
- [ ] TestOllamaProvider_Complete_ServerUnavailable - servidor offline
- [ ] TestOllamaProvider_Complete_EmptyChoices - resposta sem choices
- [ ] TestOllamaProvider_Complete_WithContextCancellation - cancelamento de contexto
- [ ] TestOllamaProvider_Close - fechamento do provider
- [ ] TestNewProviderFromConfig_Ollama - criacao via config
- [ ] TestNewProviderFromConfig_Ollama_NoAPIKey - API key opcional

### Testes de Integracao
- [ ] Teste com servidor Ollama real (manual ou CI com Ollama)
- [ ] Teste de timeout com modelo lento

### Casos de Teste Especificos

| ID | Cenario | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | Provider criado com sucesso | Config valida | Provider nao-nil, err nil |
| TC02 | Resposta bem sucedida | Request valido | Response com content |
| TC03 | Modelo nao encontrado | Modelo inexistente | Erro 404 com mensagem clara |
| TC04 | Servidor offline | URL invalida | Erro de conexao |
| TC05 | API Key opcional | Config sem APIKey | Provider criado com sucesso |
| TC06 | Timeout | Request longo | Erro de timeout |

## Riscos e Mitigacoes

| Risco | Probabilidade | Impacto | Mitigacao |
|-------|---------------|---------|-----------|
| Ollama API muda | Baixo | Alto | Usar API OpenAI-compatible que e mais estavel |
| Performance lenta em modelos grandes | Medio | Medio | Timeout configuravel, documentar requisitos |
| Usuario nao tem Ollama instalado | Medio | Baixo | Mensagens de erro claras, documentacao |
| Inconsistencia com outros providers | Baixo | Medio | Reutilizar tipos OpenAI, testes de paridade |

## Checklist de Conclusao

- [ ] Codigo implementado
  - [ ] `internal/llm/ollama.go` criado
  - [ ] `internal/llm/provider.go` modificado
  - [ ] `internal/config/defaults.go` modificado
  - [ ] `internal/tui/validation.go` verificado/modificado
- [ ] Testes escritos e passando
  - [ ] `internal/llm/ollama_test.go` criado
  - [ ] `internal/llm/provider_test.go` atualizado
  - [ ] `go test ./internal/llm/...` passa
- [ ] Documentacao atualizada
  - [ ] README.md atualizado com secao Ollama
- [ ] Code review realizado
- [ ] Feature testada manualmente com Ollama local
- [ ] Lint e format passando (`make lint && make fmt`)

## Notas Adicionais

### API Ollama OpenAI-Compatible

O Ollama expoe uma API compativel com OpenAI em `/v1/chat/completions`. Isso permite reutilizar as estruturas de request/response do OpenAI (`openAIRequest`, `openAIResponse`), simplificando a implementacao.

**Diferencas principais**:
1. **Autenticacao**: Ollama local nao requer API key
2. **BaseURL**: Default e `http://localhost:11434/v1` (nao `https://api.openai.com/v1`)
3. **Modelos**: Nomes diferentes (llama3, mistral vs gpt-4, gpt-3.5-turbo)
4. **Performance**: Modelos locais podem ser mais lentos dependendo do hardware

### Consideracoes de Timeout

Modelos locais no Ollama podem ser significativamente mais lentos que APIs cloud, especialmente:
- Na primeira requisicao (carregamento do modelo na memoria)
- Com modelos grandes (70B+)
- Em hardware limitado (CPU-only)

Recomendacao: timeout de 120s para Ollama vs 60s para outros providers.

### Exemplo de Configuracao

```yaml
# config.yaml para uso com Ollama
llm:
  provider: ollama
  base_url: http://localhost:11434/v1
  model: llama3
  max_tokens: 4096
  temperature: 0.7
  timeout: 120s
  enhance_metadata: true
```

### Proximos Passos Apos Implementacao

1. Considerar adicionar health check para verificar se Ollama esta rodando
2. Considerar suporte a streaming (futuro)
3. Documentar modelos recomendados para diferentes casos de uso
