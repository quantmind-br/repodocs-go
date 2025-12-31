# PLAN: Rate Limiting para API de LLM

## Resumo Executivo

Implementar controle de rate limiting para requisições à API de LLM, incluindo:
- **Rate limiting proativo** (token bucket) para evitar atingir limites
- **Retry com exponential backoff** para recuperação de erros 429
- **Circuit breaker** para evitar sobrecarga em falhas consecutivas

---

## 1. Análise do Estado Atual

### 1.1 O que existe

| Arquivo | Conteúdo | Status |
|---------|----------|--------|
| `internal/llm/retry.go` | `ShouldRetry()`, `CalculateBackoff()` | ❌ Não utilizado |
| `internal/llm/google.go` | Detecta 429 → retorna `ErrLLMRateLimited` | ✅ Funciona |
| `internal/llm/openai.go` | Detecta 429 → retorna `ErrLLMRateLimited` | ✅ Funciona |
| `internal/llm/anthropic.go` | Detecta 429 → retorna `ErrLLMRateLimited` | ✅ Funciona |
| `internal/llm/metadata.go` | Chama `provider.Complete()` sem retry | ❌ Sem proteção |

### 1.2 Fluxo Atual (Problemático)

```
MetadataEnhancer.Enhance()
    ↓
provider.Complete()
    ↓
HTTP Request → API
    ↓
429 Too Many Requests
    ↓
return ErrLLMRateLimited  ← FALHA IMEDIATA, SEM RETRY
```

### 1.3 Problemas

1. **Sem throttling proativo**: Requisições enviadas tão rápido quanto possível
2. **Sem retry**: Erro 429 causa falha imediata do documento
3. **Sem backoff**: Mesmo se implementasse retry, não há espera entre tentativas
4. **Sem circuit breaker**: Continua tentando mesmo após múltiplas falhas

---

## 2. Arquitetura Proposta

### 2.1 Padrão: Decorator (Wrapper)

Criar um `RateLimitedProvider` que encapsula qualquer `LLMProvider`:

```
┌─────────────────────────────────────────────────────────────┐
│                    RateLimitedProvider                       │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐ │
│  │ TokenBucket │  │   Retrier   │  │   CircuitBreaker     │ │
│  │ (proativo)  │  │ (reativo)   │  │   (proteção)         │ │
│  └─────────────┘  └─────────────┘  └──────────────────────┘ │
│                           ↓                                  │
│                  ┌─────────────────┐                        │
│                  │  LLMProvider    │                        │
│                  │ (OpenAI/Google) │                        │
│                  └─────────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Componentes

| Componente | Responsabilidade | Algoritmo |
|------------|------------------|-----------|
| **TokenBucket** | Limitar taxa de requisições | Token Bucket |
| **Retrier** | Retry em erros transientes | Exponential Backoff + Jitter |
| **CircuitBreaker** | Parar após falhas consecutivas | Three-state FSM |

### 2.3 Fluxo Proposto

```
MetadataEnhancer.Enhance()
    ↓
RateLimitedProvider.Complete()
    ↓
    ├── [1] TokenBucket.Wait() ← Aguarda token disponível
    ↓
    ├── [2] CircuitBreaker.Allow() ← Verifica se pode prosseguir
    ↓
    ├── [3] Retrier.Execute() ← Executa com retry
    │       ↓
    │       provider.Complete()
    │       ↓
    │       ├── Success → return response
    │       ├── 429 → wait(backoff) → retry
    │       ├── 5xx → wait(backoff) → retry
    │       └── 4xx → return error (não retry)
    ↓
    └── [4] CircuitBreaker.Record() ← Registra sucesso/falha
```

---

## 3. Especificação Técnica

### 3.1 Token Bucket Rate Limiter

**Arquivo:** `internal/llm/ratelimit.go`

```go
type TokenBucket struct {
    tokens     float64       // Tokens disponíveis
    capacity   float64       // Capacidade máxima (burst size)
    refillRate float64       // Tokens por segundo (convertido de RPM)
    lastRefill time.Time     // Último refill
    mu         sync.Mutex    // Thread safety
}

// NewTokenBucket cria um novo token bucket
// requestsPerMinute é convertido internamente para tokens/segundo
func NewTokenBucket(requestsPerMinute int, burstSize int) *TokenBucket {
    return &TokenBucket{
        tokens:     float64(burstSize),  // Começa cheio
        capacity:   float64(burstSize),
        refillRate: float64(requestsPerMinute) / 60.0,  // RPM → RPS
        lastRefill: time.Now(),
    }
}

// Interface
type RateLimiter interface {
    // Wait bloqueia até um token estar disponível ou context cancelado
    Wait(ctx context.Context) error
    
    // TryAcquire tenta adquirir sem bloquear
    TryAcquire() bool
    
    // Available retorna tokens disponíveis
    Available() float64
}
```

**Algoritmo Token Bucket:**
```
1. Calcular tempo desde último refill
2. Adicionar tokens: tokens += elapsed * refillRate
3. Cap em capacity: tokens = min(tokens, capacity)
4. Se tokens >= 1: consumir e retornar
5. Senão: calcular tempo de espera e dormir
```

**Configuração:**
- `requests_per_minute`: Taxa de refill (tokens/minuto) - internamente convertido para tokens/segundo
- `burst_size`: Capacidade máxima do bucket

### 3.2 Retry com Exponential Backoff

**Arquivo:** `internal/llm/retry.go` (modificar existente)

```go
type RetryConfig struct {
    MaxRetries      int           // Máximo de tentativas (0 = desabilitado)
    InitialDelay    time.Duration // Delay inicial (ex: 1s)
    MaxDelay        time.Duration // Delay máximo (ex: 60s)
    Multiplier      float64       // Multiplicador (ex: 2.0)
    JitterFactor    float64       // Fator de jitter (ex: 0.1 = ±10%)
}

type Retrier struct {
    config RetryConfig
    logger *utils.Logger
}

// Execute executa fn com retry automático
func (r *Retrier) Execute(ctx context.Context, fn func() error) error

// IsRetryable verifica se o erro permite retry
func IsRetryable(err error) bool
```

**Algoritmo Exponential Backoff com Jitter:**
```
delay = initialDelay * (multiplier ^ attempt)
delay = min(delay, maxDelay)
jitter = delay * jitterFactor * random(-1, 1)
delay = delay + jitter
sleep(delay)
```

**Erros Retryable:**
- `429 Too Many Requests`
- `500 Internal Server Error`
- `502 Bad Gateway`
- `503 Service Unavailable`
- `504 Gateway Timeout`
- Network timeout errors

**Erros NÃO Retryable:**
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- Context cancelled

### 3.3 Circuit Breaker

**Arquivo:** `internal/llm/circuit_breaker.go`

```go
type CircuitState int

const (
    StateClosed   CircuitState = iota  // Normal, permitindo requisições
    StateOpen                          // Bloqueado, rejeitando requisições
    StateHalfOpen                      // Testando se pode reabrir
)

type CircuitBreaker struct {
    state           CircuitState
    failures        int           // Falhas consecutivas
    failureThreshold int          // Limite para abrir circuito
    successThreshold int          // Sucessos para fechar (half-open)
    resetTimeout    time.Duration // Tempo para tentar half-open
    lastFailure     time.Time
    mu              sync.RWMutex
}

// Interface
type CircuitBreaker interface {
    // Allow verifica se requisição é permitida
    Allow() bool
    
    // RecordSuccess registra sucesso
    RecordSuccess()
    
    // RecordFailure registra falha
    RecordFailure()
    
    // State retorna estado atual
    State() CircuitState
}
```

**Estados:**
```
     ┌──────────────────────────────────────┐
     │                                      │
     ▼                                      │
┌─────────┐  failures >= threshold  ┌──────────┐
│ CLOSED  │ ───────────────────────►│   OPEN   │
│(normal) │                         │(bloqueado)│
└─────────┘                         └──────────┘
     ▲                                      │
     │                                      │ timeout expired
     │                                      ▼
     │   success                    ┌───────────┐
     └──────────────────────────────│ HALF-OPEN │
                                    │ (testando)│
               failure              └───────────┘
               ───────────────────────────►│
                                           │
                                           ▼
                                       (volta OPEN)
```

### 3.4 Rate Limited Provider Wrapper

**Arquivo:** `internal/llm/provider_wrapper.go`

```go
type RateLimitedProvider struct {
    provider       domain.LLMProvider
    rateLimiter    RateLimiter
    retrier        *Retrier
    circuitBreaker *CircuitBreaker
    logger         *utils.Logger
}

type RateLimitedProviderConfig struct {
    // Rate limiting
    RequestsPerMinute int           // Convertido internamente para tokens/segundo
    BurstSize         int
    
    // Retry
    MaxRetries     int
    InitialDelay   time.Duration
    MaxDelay       time.Duration
    Multiplier     float64
    
    // Circuit breaker
    FailureThreshold int
    ResetTimeout     time.Duration
}

func NewRateLimitedProvider(
    provider domain.LLMProvider,
    cfg RateLimitedProviderConfig,
    logger *utils.Logger,
) *RateLimitedProvider

func (p *RateLimitedProvider) Name() string
func (p *RateLimitedProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error)
func (p *RateLimitedProvider) Close() error
```

**Implementação de Complete():**
```go
func (p *RateLimitedProvider) Complete(ctx context.Context, req *domain.LLMRequest) (*domain.LLMResponse, error) {
    // 1. Aguardar rate limit
    if err := p.rateLimiter.Wait(ctx); err != nil {
        return nil, fmt.Errorf("rate limit wait cancelled: %w", err)
    }
    
    // 2. Verificar circuit breaker
    if !p.circuitBreaker.Allow() {
        return nil, domain.ErrLLMCircuitOpen
    }
    
    // 3. Executar com retry
    var response *domain.LLMResponse
    err := p.retrier.Execute(ctx, func() error {
        var err error
        response, err = p.provider.Complete(ctx, req)
        return err
    })
    
    // 4. Atualizar circuit breaker
    if err != nil {
        p.circuitBreaker.RecordFailure()
        return nil, err
    }
    
    p.circuitBreaker.RecordSuccess()
    return response, nil
}
```

---

## 4. Configuração

### 4.1 Estrutura de Config

**Arquivo:** `internal/config/config.go`

```go
type LLMConfig struct {
    Provider        string            `mapstructure:"provider"`
    APIKey          string            `mapstructure:"api_key"`
    BaseURL         string            `mapstructure:"base_url"`
    Model           string            `mapstructure:"model"`
    MaxTokens       int               `mapstructure:"max_tokens"`
    Temperature     float64           `mapstructure:"temperature"`
    Timeout         time.Duration     `mapstructure:"timeout"`
    MaxRetries      int               `mapstructure:"max_retries"`      // DEPRECATED: usar RateLimit.MaxRetries
    EnhanceMetadata bool              `mapstructure:"enhance_metadata"`
    RateLimit       RateLimitConfig   `mapstructure:"rate_limit"`       // NOVO
}

type RateLimitConfig struct {
    Enabled           bool          `mapstructure:"enabled"`
    RequestsPerMinute int           `mapstructure:"requests_per_minute"`
    BurstSize         int           `mapstructure:"burst_size"`
    MaxRetries        int           `mapstructure:"max_retries"`
    InitialDelay      time.Duration `mapstructure:"initial_delay"`
    MaxDelay          time.Duration `mapstructure:"max_delay"`
    Multiplier        float64       `mapstructure:"multiplier"`
    CircuitBreaker    CircuitBreakerConfig `mapstructure:"circuit_breaker"`
}

type CircuitBreakerConfig struct {
    Enabled          bool          `mapstructure:"enabled"`
    FailureThreshold int           `mapstructure:"failure_threshold"`
    ResetTimeout     time.Duration `mapstructure:"reset_timeout"`
}
```

### 4.2 Valores Default

**Arquivo:** `internal/config/defaults.go`

```go
var DefaultRateLimitConfig = RateLimitConfig{
    Enabled:           true,
    RequestsPerMinute: 60,            // 60 req/min (= 1 req/s)
    BurstSize:         10,            // Permite burst de 10
    MaxRetries:        3,             // 3 tentativas
    InitialDelay:      1 * time.Second,
    MaxDelay:          60 * time.Second,
    Multiplier:        2.0,
    CircuitBreaker: CircuitBreakerConfig{
        Enabled:          true,
        FailureThreshold: 5,          // Abre após 5 falhas consecutivas
        ResetTimeout:     30 * time.Second,
    },
}
```

### 4.3 Exemplo de config.yaml

```yaml
llm:
  provider: google
  api_key: ${REPODOCS_LLM_API_KEY}
  base_url: https://generativelanguage.googleapis.com
  model: gemini-2.0-flash
  enhance_metadata: true
  
  rate_limit:
    enabled: true
    requests_per_minute: 60    # 60 requisições por minuto (= 1 req/s)
    burst_size: 10             # Permite burst de até 10 requisições
    max_retries: 3             # Máximo 3 tentativas por requisição
    initial_delay: 1s          # Delay inicial entre retries
    max_delay: 60s             # Delay máximo entre retries
    multiplier: 2.0            # Multiplicador exponencial
    
    circuit_breaker:
      enabled: true
      failure_threshold: 5     # Abre após 5 falhas consecutivas
      reset_timeout: 30s       # Tenta reabrir após 30s
```

### 4.4 Limites por Provider

| Provider | Rate Limit Típico | Sugestão `requests_per_minute` |
|----------|------------------|-------------------------------|
| OpenAI | 60 RPM (tier 1) → 10K+ RPM (tier 5) | 50-500 |
| Google Gemini | 60 RPM (free) → 1000+ RPM (paid) | 50-100 |
| Anthropic | 60 RPM (tier 1) → 4000 RPM (tier 4) | 50-200 |
| Ollama (local) | Sem limite externo | 1000+ |

---

## 5. Integração

### 5.1 Modificar strategy.go

**Arquivo:** `internal/strategies/strategy.go`

```go
// Antes (atual):
if opts.LLMConfig != nil && opts.LLMConfig.EnhanceMetadata {
    provider, err := llm.NewProviderFromConfig(opts.LLMConfig)
    if err != nil {
        logger.Warn().Err(err).Msg("Failed to create LLM provider")
    } else {
        llmProvider = provider
        metadataEnhancer = llm.NewMetadataEnhancer(provider)
    }
}

// Depois (com rate limiting):
if opts.LLMConfig != nil && opts.LLMConfig.EnhanceMetadata {
    baseProvider, err := llm.NewProviderFromConfig(opts.LLMConfig)
    if err != nil {
        logger.Warn().Err(err).Msg("Failed to create LLM provider")
    } else {
        // Wrap com rate limiting se habilitado
        if opts.LLMConfig.RateLimit.Enabled {
            llmProvider = llm.NewRateLimitedProvider(
                baseProvider,
                llm.RateLimitedProviderConfig{
                    RequestsPerMinute: opts.LLMConfig.RateLimit.RequestsPerMinute,
                    BurstSize:         opts.LLMConfig.RateLimit.BurstSize,
                    MaxRetries:        opts.LLMConfig.RateLimit.MaxRetries,
                    InitialDelay:      opts.LLMConfig.RateLimit.InitialDelay,
                    MaxDelay:          opts.LLMConfig.RateLimit.MaxDelay,
                    Multiplier:        opts.LLMConfig.RateLimit.Multiplier,
                    // Circuit breaker config...
                },
                logger,
            )
        } else {
            llmProvider = baseProvider
        }
        metadataEnhancer = llm.NewMetadataEnhancer(llmProvider)
    }
}
```

### 5.2 Novos Erros

**Arquivo:** `internal/domain/errors.go`

```go
// Erros existentes
var ErrLLMRateLimited = errors.New("rate limit exceeded")

// Novos erros
var ErrLLMCircuitOpen = errors.New("circuit breaker is open")
var ErrLLMMaxRetriesExceeded = errors.New("max retries exceeded")
var ErrLLMRateLimitWaitCancelled = errors.New("rate limit wait cancelled")
```

---

## 6. Logging e Observabilidade

### 6.1 Logs Estruturados

```go
// Rate limit wait
logger.Debug().
    Float64("tokens_available", bucket.Available()).
    Float64("tokens_needed", 1.0).
    Dur("estimated_wait", estimatedWait).
    Msg("Waiting for rate limit token")

// Retry attempt
logger.Warn().
    Int("attempt", attempt).
    Int("max_retries", cfg.MaxRetries).
    Dur("backoff", backoff).
    Err(err).
    Msg("Retrying LLM request after error")

// Circuit breaker state change
logger.Warn().
    Str("from_state", fromState).
    Str("to_state", toState).
    Int("consecutive_failures", failures).
    Msg("Circuit breaker state changed")

// Final success after retries
logger.Info().
    Int("attempts", attempts).
    Dur("total_duration", duration).
    Msg("LLM request succeeded after retries")

// Final failure
logger.Error().
    Int("attempts", attempts).
    Err(finalErr).
    Msg("LLM request failed after all retries")
```

### 6.2 Métricas (Opcional/Futuro)

```go
type RateLimitMetrics struct {
    RequestsTotal      int64         // Total de requisições
    RequestsSucceeded  int64         // Requisições com sucesso
    RequestsFailed     int64         // Requisições que falharam
    RequestsRetried    int64         // Requisições que precisaram retry
    RetryAttemptsTotal int64         // Total de tentativas de retry
    RateLimitWaitsTotal int64        // Vezes que aguardou rate limit
    RateLimitWaitTime  time.Duration // Tempo total aguardando
    CircuitBreakerOpens int64        // Vezes que circuit breaker abriu
}
```

---

## 7. Testes

### 7.1 Testes Unitários

**Arquivo:** `tests/unit/llm/ratelimit_test.go`

```go
func TestTokenBucket_Refill(t *testing.T)
func TestTokenBucket_Wait_Success(t *testing.T)
func TestTokenBucket_Wait_ContextCancelled(t *testing.T)
func TestTokenBucket_Concurrent(t *testing.T)
func TestTokenBucket_BurstSize(t *testing.T)
func TestTokenBucket_ZeroRate(t *testing.T)
```

**Arquivo:** `tests/unit/llm/retry_test.go`

```go
func TestRetrier_Success_FirstAttempt(t *testing.T)
func TestRetrier_Success_AfterRetry(t *testing.T)
func TestRetrier_Failure_MaxRetries(t *testing.T)
func TestRetrier_NonRetryableError(t *testing.T)
func TestRetrier_ContextCancelled(t *testing.T)
func TestRetrier_BackoffCalculation(t *testing.T)
func TestRetrier_JitterRange(t *testing.T)
func TestIsRetryable_429(t *testing.T)
func TestIsRetryable_5xx(t *testing.T)
func TestIsRetryable_4xx(t *testing.T)
```

**Arquivo:** `tests/unit/llm/circuit_breaker_test.go`

```go
func TestCircuitBreaker_InitialState(t *testing.T)
func TestCircuitBreaker_OpensAfterThreshold(t *testing.T)
func TestCircuitBreaker_RejectsWhenOpen(t *testing.T)
func TestCircuitBreaker_TransitionsToHalfOpen(t *testing.T)
func TestCircuitBreaker_ClosesOnSuccess(t *testing.T)
func TestCircuitBreaker_ReopensOnFailure(t *testing.T)
func TestCircuitBreaker_Concurrent(t *testing.T)
```

**Arquivo:** `tests/unit/llm/provider_wrapper_test.go`

```go
func TestRateLimitedProvider_RespectRateLimit(t *testing.T)
func TestRateLimitedProvider_RetriesOn429(t *testing.T)
func TestRateLimitedProvider_RetriesOn5xx(t *testing.T)
func TestRateLimitedProvider_NoRetryOn4xx(t *testing.T)
func TestRateLimitedProvider_CircuitBreakerIntegration(t *testing.T)
func TestRateLimitedProvider_ContextCancellation(t *testing.T)
```

### 7.2 Testes de Integração

**Arquivo:** `tests/integration/llm_ratelimit_test.go`

```go
func TestRateLimitedProvider_UnderLoad(t *testing.T) {
    // Simula 100 requisições concorrentes
    // Verifica que rate limit é respeitado
}

func TestRateLimitedProvider_RecoveryAfter429(t *testing.T) {
    // Mock server que retorna 429 nas primeiras N requisições
    // Verifica retry e sucesso eventual
}

func TestRateLimitedProvider_CircuitBreakerRecovery(t *testing.T) {
    // Mock server que falha, depois recupera
    // Verifica transições de estado
}
```

### 7.3 Testes de Benchmark

**Arquivo:** `tests/benchmark/llm_ratelimit_benchmark_test.go`

```go
func BenchmarkTokenBucket_Wait(b *testing.B)
func BenchmarkTokenBucket_Concurrent(b *testing.B)
func BenchmarkCircuitBreaker_Allow(b *testing.B)
func BenchmarkRateLimitedProvider_Complete(b *testing.B)
```

---

## 8. Arquivos a Criar/Modificar

### 8.1 Arquivos Novos

| Arquivo | Descrição | LOC Est. |
|---------|-----------|----------|
| `internal/llm/ratelimit.go` | TokenBucket rate limiter | ~100 |
| `internal/llm/circuit_breaker.go` | Circuit breaker | ~120 |
| `internal/llm/provider_wrapper.go` | RateLimitedProvider | ~150 |
| `tests/unit/llm/ratelimit_test.go` | Testes do rate limiter | ~200 |
| `tests/unit/llm/circuit_breaker_test.go` | Testes do circuit breaker | ~180 |
| `tests/unit/llm/provider_wrapper_test.go` | Testes do wrapper | ~250 |
| `tests/integration/llm_ratelimit_test.go` | Testes de integração | ~150 |

### 8.2 Arquivos Modificados

| Arquivo | Modificação | LOC Est. |
|---------|-------------|----------|
| `internal/config/config.go` | Adicionar RateLimitConfig, CircuitBreakerConfig | +30 |
| `internal/config/defaults.go` | Adicionar defaults | +20 |
| `internal/config/loader.go` | Carregar novas configs | +15 |
| `internal/domain/errors.go` | Novos erros | +5 |
| `internal/llm/retry.go` | Refatorar para uso real | +50 |
| `internal/strategies/strategy.go` | Integrar rate limiting | +20 |
| `configs/config.yaml.template` | Documentar novas opções | +20 |

### 8.3 Total Estimado

- **Código novo:** ~500 LOC
- **Testes novos:** ~780 LOC
- **Modificações:** ~160 LOC
- **Total:** ~1440 LOC

---

## 9. Ordem de Implementação

### Fase 1: Infraestrutura (2-3h)

1. [ ] Criar `RateLimitConfig` e `CircuitBreakerConfig` em `config.go`
2. [ ] Adicionar defaults em `defaults.go`
3. [ ] Adicionar loading em `loader.go`
4. [ ] Adicionar novos erros em `errors.go`
5. [ ] Atualizar `config.yaml.template`

### Fase 2: Rate Limiter (2-3h)

6. [ ] Implementar `TokenBucket` em `ratelimit.go`
7. [ ] Escrever testes unitários em `ratelimit_test.go`
8. [ ] Verificar thread safety com testes concorrentes

### Fase 3: Retry Logic (2h)

9. [ ] Refatorar `retry.go` com `Retrier` struct
10. [ ] Implementar `IsRetryable()`
11. [ ] Implementar backoff com jitter
12. [ ] Escrever testes unitários

### Fase 4: Circuit Breaker (2-3h)

13. [ ] Implementar `CircuitBreaker` em `circuit_breaker.go`
14. [ ] Implementar state machine
15. [ ] Escrever testes unitários

### Fase 5: Provider Wrapper (2-3h)

16. [ ] Implementar `RateLimitedProvider` em `provider_wrapper.go`
17. [ ] Integrar TokenBucket, Retrier, CircuitBreaker
18. [ ] Adicionar logging estruturado
19. [ ] Escrever testes unitários

### Fase 6: Integração (1-2h)

20. [ ] Modificar `strategy.go` para usar wrapper
21. [ ] Testar integração end-to-end
22. [ ] Escrever testes de integração

### Fase 7: Documentação e Cleanup (1h)

23. [ ] Atualizar README.md
24. [ ] Atualizar AGENTS.md se necessário
25. [ ] Executar `make lint` e `make test`
26. [ ] Code review e cleanup

### Tempo Total Estimado: 12-17 horas

---

## 10. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Performance do mutex no TokenBucket | Baixa | Médio | Benchmark e otimização se necessário |
| Jitter insuficiente causando thundering herd | Baixa | Alto | Usar jitter de ±20% |
| Circuit breaker muito agressivo | Média | Médio | Valores conservadores de default |
| Configuração complexa para usuário | Média | Baixo | Bons defaults, documentação clara |
| Incompatibilidade com retry existente | Baixa | Baixo | Deprecar campo antigo, manter compatibilidade |

---

## 11. Critérios de Sucesso

### 11.1 Funcionais

- [ ] Rate limit de 60 req/min é respeitado com tolerância de ±10%
- [ ] Retry automático em 429 com backoff exponencial
- [ ] Circuit breaker abre após N falhas consecutivas
- [ ] Circuit breaker fecha após sucesso em half-open
- [ ] Context cancellation interrompe operações corretamente

### 11.2 Não-Funcionais

- [ ] Overhead < 1ms por requisição (sem rate limit wait)
- [ ] Memória adicional < 1KB por provider
- [ ] 100% cobertura de testes nos componentes novos
- [ ] Zero race conditions (verificado com `-race`)

### 11.3 Usabilidade

- [ ] Funciona out-of-the-box com defaults sensatos
- [ ] Configuração clara e bem documentada
- [ ] Logs informativos sem ser verboso
- [ ] Erro claro quando circuit breaker está aberto

---

## 12. Rollback Plan

Se problemas críticos forem encontrados:

1. **Rápido:** Setar `rate_limit.enabled: false` no config
2. **Médio:** Reverter PR e deploy versão anterior
3. **Completo:** Feature flag para desabilitar completamente

---

## 13. Referências

- [Token Bucket Algorithm](https://en.wikipedia.org/wiki/Token_bucket)
- [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Google API Rate Limits](https://ai.google.dev/gemini-api/docs/rate-limits)
- [OpenAI Rate Limits](https://platform.openai.com/docs/guides/rate-limits)
- [Anthropic Rate Limits](https://docs.anthropic.com/en/api/rate-limits)
