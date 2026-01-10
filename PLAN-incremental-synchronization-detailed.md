# Plano de Implementacao: Incremental Synchronization

> **Phase**: 1.2 (Foundation)  
> **Priority**: High  
> **Status**: Ready for Implementation  
> **Complexity**: Medium | **Value**: Very High  
> **Estimated Effort**: 3-4 days

## Resumo Executivo

Implementar sincronizacao incremental para o repodocs-go, permitindo que execucoes subsequentes processem apenas conteudo alterado. O sistema rastreara hashes de conteudo das paginas processadas em um arquivo de estado JSON, comparando-os em execucoes futuras para determinar se uma pagina precisa ser reprocessada.

**Valor entregue:**
- 10-100x mais rapido em execucoes de manutencao
- Habilita workflows de sincronizacao agendada (cron)
- Reduz custos de API LLM para enhancement de metadados
- Melhora experiencia do usuario em sites grandes

## Analise de Requisitos

### Requisitos Funcionais

- [ ] RF01: Criar arquivo de estado `.repodocs-state.json` no diretorio de output na primeira execucao
- [ ] RF02: Armazenar hash de conteudo, timestamp e caminho do arquivo para cada pagina processada
- [ ] RF03: Flag `--sync` para habilitar modo incremental (opt-in explicito)
- [ ] RF04: Flag `--full-sync` para forcar reprocessamento completo ignorando estado
- [ ] RF05: Comparar content hash antes de processar cada pagina
- [ ] RF06: Detectar paginas deletadas (presentes no estado mas nao descobertas no crawl)
- [ ] RF07: Flag `--prune` para remover arquivos de paginas deletadas
- [ ] RF08: Atualizar estado apos cada escrita bem-sucedida

### Requisitos Nao-Funcionais

- [ ] RNF01: Estado deve ser human-readable (JSON formatado)
- [ ] RNF02: Operacoes de estado devem ser thread-safe (multiplos workers)
- [ ] RNF03: Falha na leitura/escrita de estado nao deve impedir execucao
- [ ] RNF04: Estado deve incluir versao do schema para migracao futura
- [ ] RNF05: Performance: overhead de verificacao de estado < 1ms por pagina

## Analise Tecnica

### Arquitetura Proposta

```
                                    +------------------+
                                    |   CLI (main.go)  |
                                    |  --sync flags    |
                                    +--------+---------+
                                             |
                                             v
                                    +------------------+
                                    |   Orchestrator   |
                                    | (SyncOptions)    |
                                    +--------+---------+
                                             |
                      +----------------------+----------------------+
                      |                                             |
                      v                                             v
           +------------------+                          +------------------+
           | StateManager     |                          | Dependencies     |
           | (internal/state) |<------------------------>| (strategies)     |
           +------------------+                          +------------------+
                      |                                             |
                      |  +------------------------------------------+
                      |  |
                      v  v
           +------------------+
           |   Writer         |
           | (state update)   |
           +------------------+
                      |
                      v
           +------------------+
           | .repodocs-state  |
           |     .json        |
           +------------------+
```

### Componentes Afetados

| Arquivo/Modulo | Tipo de Mudanca | Descricao |
|----------------|-----------------|-----------|
| `internal/state/models.go` | Criar | Definir structs SyncState, PageState |
| `internal/state/manager.go` | Criar | Implementar StateManager com Load/Save/Update |
| `internal/state/errors.go` | Criar | Erros sentinela para estado |
| `internal/domain/options.go` | Modificar | Adicionar SyncOptions ao CommonOptions |
| `cmd/repodocs/main.go` | Modificar | Adicionar flags --sync, --full-sync, --prune |
| `internal/app/orchestrator.go` | Modificar | Injetar StateManager nas Dependencies |
| `internal/strategies/strategy.go` | Modificar | Adicionar StateManager ao Dependencies |
| `internal/strategies/crawler.go` | Modificar | Verificar estado antes de processar |
| `internal/strategies/git/processor.go` | Modificar | Verificar estado antes de processar |
| `internal/output/writer.go` | Modificar | Atualizar estado apos escrita |
| `tests/unit/state/` | Criar | Testes unitarios do StateManager |
| `tests/integration/sync_test.go` | Criar | Teste de integracao do fluxo completo |

### Dependencias

**Internas:**
- `internal/domain` - Para tipos e erros
- `internal/utils` - Para logger e funcoes de path
- Existente `ContentHash` em `domain.Document` (calculado em `converter/pipeline.go`)

**Externas:**
- Nenhuma nova dependencia necessaria (JSON encoding da stdlib)

**Sequenciamento:**
- Phase 1.1 (Manifest) deve estar completa para multi-source state management
- Porem, pode ser implementada em paralelo se limitada a single-source inicialmente

## Plano de Implementacao

### Fase 1: Definir Modelos de Estado
**Objetivo**: Criar estruturas de dados para persistencia de estado

#### Tarefas:

1. **Criar `internal/state/models.go`**
   - Arquivos: `internal/state/models.go`
   
   ```go
   package state

   import "time"

   // Version do schema para migracao futura
   const StateVersion = 1

   // SyncState representa o estado completo de sincronizacao
   type SyncState struct {
       Version   int                  `json:"version"`
       SourceURL string               `json:"source_url"`
       Strategy  string               `json:"strategy,omitempty"`
       LastSync  time.Time            `json:"last_sync"`
       Pages     map[string]PageState `json:"pages"`
   }

   // PageState representa o estado de uma pagina individual
   type PageState struct {
       ContentHash string    `json:"content_hash"`
       FetchedAt   time.Time `json:"fetched_at"`
       FilePath    string    `json:"file_path"`
   }

   // NewSyncState cria um novo estado vazio
   func NewSyncState(sourceURL, strategy string) *SyncState {
       return &SyncState{
           Version:   StateVersion,
           SourceURL: sourceURL,
           Strategy:  strategy,
           LastSync:  time.Now(),
           Pages:     make(map[string]PageState),
       }
   }
   ```

2. **Criar `internal/state/errors.go`**
   - Arquivos: `internal/state/errors.go`
   
   ```go
   package state

   import "errors"

   var (
       // ErrStateNotFound indica que o arquivo de estado nao existe
       ErrStateNotFound = errors.New("state file not found")
       
       // ErrStateCorrupted indica que o arquivo de estado esta corrompido
       ErrStateCorrupted = errors.New("state file is corrupted")
       
       // ErrVersionMismatch indica versao incompativel do schema
       ErrVersionMismatch = errors.New("state version mismatch")
   )
   ```

### Fase 2: Implementar StateManager
**Objetivo**: Criar gerenciador de estado thread-safe

#### Tarefas:

1. **Criar `internal/state/manager.go`**
   - Arquivos: `internal/state/manager.go`
   
   ```go
   package state

   import (
       "context"
       "encoding/json"
       "os"
       "path/filepath"
       "sync"
       "time"

       "github.com/quantmind-br/repodocs-go/internal/utils"
   )

   const StateFileName = ".repodocs-state.json"

   // Manager gerencia o estado de sincronizacao
   type Manager struct {
       baseDir  string
       state    *SyncState
       mu       sync.RWMutex
       dirty    bool
       logger   *utils.Logger
       disabled bool
   }

   // ManagerOptions configura o StateManager
   type ManagerOptions struct {
       BaseDir   string
       SourceURL string
       Strategy  string
       Logger    *utils.Logger
       Disabled  bool // Para --full-sync ou quando sync nao esta habilitado
   }

   // NewManager cria um novo StateManager
   func NewManager(opts ManagerOptions) *Manager {
       return &Manager{
           baseDir:  opts.BaseDir,
           logger:   opts.Logger,
           disabled: opts.Disabled,
           state:    NewSyncState(opts.SourceURL, opts.Strategy),
       }
   }

   // Load carrega o estado do arquivo
   func (m *Manager) Load(ctx context.Context) error {
       if m.disabled {
           return nil
       }

       m.mu.Lock()
       defer m.mu.Unlock()

       path := m.statePath()
       data, err := os.ReadFile(path)
       if os.IsNotExist(err) {
           return ErrStateNotFound
       }
       if err != nil {
           return err
       }

       var state SyncState
       if err := json.Unmarshal(data, &state); err != nil {
           return ErrStateCorrupted
       }

       if state.Version != StateVersion {
           if m.logger != nil {
               m.logger.Warn().
                   Int("file_version", state.Version).
                   Int("expected_version", StateVersion).
                   Msg("State version mismatch, will rebuild state")
           }
           return ErrVersionMismatch
       }

       m.state = &state
       return nil
   }

   // Save persiste o estado no arquivo
   func (m *Manager) Save(ctx context.Context) error {
       if m.disabled {
           return nil
       }

       m.mu.Lock()
       defer m.mu.Unlock()

       if !m.dirty {
           return nil
       }

       m.state.LastSync = time.Now()

       data, err := json.MarshalIndent(m.state, "", "  ")
       if err != nil {
           return err
       }

       path := m.statePath()
       if err := os.MkdirAll(filepath.Dir(path), 0755); err != nil {
           return err
       }

       if err := os.WriteFile(path, data, 0644); err != nil {
           return err
       }

       m.dirty = false
       return nil
   }

   // ShouldProcess verifica se uma pagina precisa ser processada
   func (m *Manager) ShouldProcess(url, contentHash string) bool {
       if m.disabled {
           return true
       }

       m.mu.RLock()
       defer m.mu.RUnlock()

       page, exists := m.state.Pages[url]
       if !exists {
           return true // Pagina nova
       }

       return page.ContentHash != contentHash
   }

   // Update registra uma pagina processada
   func (m *Manager) Update(url string, page PageState) {
       if m.disabled {
           return
       }

       m.mu.Lock()
       defer m.mu.Unlock()

       m.state.Pages[url] = page
       m.dirty = true
   }

   // MarkSeen registra URLs descobertas no crawl atual
   func (m *Manager) MarkSeen(url string) {
       // Usado para deteccao de delecao
   }

   // GetDeletedPages retorna paginas no estado que nao foram vistas
   func (m *Manager) GetDeletedPages(seenURLs map[string]bool) []PageState {
       if m.disabled {
           return nil
       }

       m.mu.RLock()
       defer m.mu.RUnlock()

       var deleted []PageState
       for url, page := range m.state.Pages {
           if !seenURLs[url] {
               deleted = append(deleted, page)
           }
       }
       return deleted
   }

   // Stats retorna estatisticas do estado
   func (m *Manager) Stats() (total, unchanged, changed, new int) {
       m.mu.RLock()
       defer m.mu.RUnlock()
       return len(m.state.Pages), 0, 0, 0 // Implementar contadores
   }

   func (m *Manager) statePath() string {
       return filepath.Join(m.baseDir, StateFileName)
   }
   ```

### Fase 3: Adicionar Flags CLI
**Objetivo**: Expor funcionalidade via linha de comando

#### Tarefas:

1. **Modificar `internal/domain/options.go`**
   - Adicionar campos de sync ao CommonOptions
   
   ```go
   // Adicionar ao CommonOptions
   type CommonOptions struct {
       // ... campos existentes ...
       Sync     bool // Habilitar modo incremental
       FullSync bool // Forcar reprocessamento completo
       Prune    bool // Remover arquivos de paginas deletadas
   }
   ```

2. **Modificar `cmd/repodocs/main.go`**
   - Adicionar flags na funcao `init()`
   
   ```go
   // Em init(), adicionar:
   rootCmd.PersistentFlags().Bool("sync", false, "Enable incremental sync mode")
   rootCmd.PersistentFlags().Bool("full-sync", false, "Force full re-processing (ignore state)")
   rootCmd.PersistentFlags().Bool("prune", false, "Remove files for deleted pages")

   // Bindings para viper (opcional):
   _ = viper.BindPFlag("sync.enabled", rootCmd.PersistentFlags().Lookup("sync"))
   _ = viper.BindPFlag("sync.full", rootCmd.PersistentFlags().Lookup("full-sync"))
   _ = viper.BindPFlag("sync.prune", rootCmd.PersistentFlags().Lookup("prune"))
   ```
   
   - Modificar funcao `run()` para passar flags ao Orchestrator
   
   ```go
   // Em run(), adicionar:
   syncEnabled, _ := cmd.Flags().GetBool("sync")
   fullSync, _ := cmd.Flags().GetBool("full-sync")
   prune, _ := cmd.Flags().GetBool("prune")

   orchOpts := app.OrchestratorOptions{
       CommonOptions: domain.CommonOptions{
           // ... existentes ...
           Sync:     syncEnabled,
           FullSync: fullSync,
           Prune:    prune,
       },
       // ...
   }
   ```

### Fase 4: Integrar StateManager nas Estrategias
**Objetivo**: Usar estado para skip de paginas inalteradas

#### Tarefas:

1. **Modificar `internal/strategies/strategy.go`**
   - Adicionar StateManager ao Dependencies
   
   ```go
   // Adicionar ao Dependencies struct:
   type Dependencies struct {
       // ... campos existentes ...
       StateManager *state.Manager
   }

   // Modificar NewDependencies para aceitar state options
   type DependencyOptions struct {
       // ... campos existentes ...
       SyncEnabled bool
       FullSync    bool
       Prune       bool
   }

   // Em NewDependencies, criar StateManager:
   var stateManager *state.Manager
   if opts.SyncEnabled && !opts.FullSync {
       stateManager = state.NewManager(state.ManagerOptions{
           BaseDir:   opts.OutputDir,
           SourceURL: opts.SourceURL,
           Logger:    logger,
           Disabled:  opts.FullSync,
       })
       // Tentar carregar estado existente
       if err := stateManager.Load(context.Background()); err != nil {
           if !errors.Is(err, state.ErrStateNotFound) {
               logger.Warn().Err(err).Msg("Failed to load state, starting fresh")
           }
       }
   }
   ```

2. **Modificar `internal/strategies/crawler.go`**
   - Verificar estado em `processResponse`
   
   ```go
   func (s *CrawlerStrategy) processResponse(ctx context.Context, r *colly.Response, cctx *crawlContext) {
       // ... codigo existente ate criar doc ...

       // NOVO: Verificar estado antes de processar
       if s.deps.StateManager != nil && doc.ContentHash != "" {
           if !s.deps.StateManager.ShouldProcess(currentURL, doc.ContentHash) {
               s.logger.Debug().Str("url", currentURL).Msg("Skipping unchanged page")
               return
           }
       }

       // ... resto do processamento ...
   }
   ```

3. **Modificar `internal/strategies/git/processor.go`**
   - Verificar estado em `ProcessFile`
   
   ```go
   func (p *Processor) ProcessFile(ctx context.Context, file FileInfo, opts ProcessOptions) (*domain.Document, error) {
       // ... codigo existente ...

       // NOVO: Verificar estado (se StateManager disponivel)
       if opts.StateManager != nil && doc.ContentHash != "" {
           if !opts.StateManager.ShouldProcess(fileURL, doc.ContentHash) {
               p.logger.Debug().Str("file", file.RelPath).Msg("Skipping unchanged file")
               return nil, nil
           }
       }

       // ... resto do processamento ...
   }
   ```

### Fase 5: Atualizar Estado apos Escrita
**Objetivo**: Persistir estado de cada pagina processada

#### Tarefas:

1. **Modificar `internal/strategies/strategy.go` - WriteDocument**
   
   ```go
   // Modificar WriteDocument para atualizar estado:
   func (d *Dependencies) WriteDocument(ctx context.Context, doc *domain.Document) error {
       // ... codigo existente de enhancement ...

       if d.Writer == nil {
           return fmt.Errorf("writer is not configured")
       }

       if err := d.Writer.Write(ctx, doc); err != nil {
           return err
       }

       // NOVO: Atualizar estado apos escrita bem-sucedida
       if d.StateManager != nil {
           filePath := d.Writer.GetPath(doc.URL)
           d.StateManager.Update(doc.URL, state.PageState{
               ContentHash: doc.ContentHash,
               FetchedAt:   doc.FetchedAt,
               FilePath:    filePath,
           })
       }

       return nil
   }
   ```

2. **Salvar estado ao final da execucao**
   - Modificar `internal/app/orchestrator.go`
   
   ```go
   // Em Run(), ao final:
   func (o *Orchestrator) Run(ctx context.Context, url string, opts OrchestratorOptions) error {
       // ... execucao existente ...

       // NOVO: Salvar estado
       if o.deps.StateManager != nil {
           if err := o.deps.StateManager.Save(ctx); err != nil {
               o.logger.Warn().Err(err).Msg("Failed to save state")
           }
       }

       return nil
   }
   ```

### Fase 6: Implementar Deteccao de Delecao
**Objetivo**: Identificar e opcionalmente remover paginas deletadas

#### Tarefas:

1. **Rastrear URLs descobertas durante crawl**
   - Modificar `crawlContext` para rastrear URLs vistas
   
   ```go
   type crawlContext struct {
       // ... campos existentes ...
       seenURLs *sync.Map // NOVO: URLs descobertas neste crawl
   }
   ```

2. **Implementar pruning em `internal/app/orchestrator.go`**
   
   ```go
   // Ao final de Run(), se --prune:
   if opts.Prune && o.deps.StateManager != nil {
       deleted := o.deps.StateManager.GetDeletedPages(seenURLs)
       for _, page := range deleted {
           if err := os.Remove(page.FilePath); err != nil {
               o.logger.Warn().Err(err).Str("file", page.FilePath).Msg("Failed to remove deleted page")
           } else {
               o.logger.Info().Str("file", page.FilePath).Msg("Removed deleted page")
           }
       }
   }
   ```

### Fase 7: Testes
**Objetivo**: Garantir corretude e prevenir regressoes

#### Tarefas:

1. **Criar `tests/unit/state/manager_test.go`**
   
   ```go
   package state_test

   import (
       "context"
       "testing"
       "time"

       "github.com/stretchr/testify/assert"
       "github.com/stretchr/testify/require"

       "github.com/quantmind-br/repodocs-go/internal/state"
   )

   func TestManager_NewAndSave(t *testing.T) {
       dir := t.TempDir()
       mgr := state.NewManager(state.ManagerOptions{
           BaseDir:   dir,
           SourceURL: "https://example.com",
           Strategy:  "crawler",
       })

       // Atualizar estado
       mgr.Update("https://example.com/page1", state.PageState{
           ContentHash: "abc123",
           FetchedAt:   time.Now(),
           FilePath:    "page1.md",
       })

       // Salvar
       err := mgr.Save(context.Background())
       require.NoError(t, err)

       // Carregar em novo manager
       mgr2 := state.NewManager(state.ManagerOptions{
           BaseDir: dir,
       })
       err = mgr2.Load(context.Background())
       require.NoError(t, err)

       // Verificar
       assert.False(t, mgr2.ShouldProcess("https://example.com/page1", "abc123"))
       assert.True(t, mgr2.ShouldProcess("https://example.com/page1", "xyz789"))
       assert.True(t, mgr2.ShouldProcess("https://example.com/page2", "any"))
   }

   func TestManager_ShouldProcess_Unchanged(t *testing.T) {
       // ... test cases ...
   }

   func TestManager_ShouldProcess_Changed(t *testing.T) {
       // ... test cases ...
   }

   func TestManager_Disabled(t *testing.T) {
       mgr := state.NewManager(state.ManagerOptions{
           Disabled: true,
       })
       // Sempre deve retornar true quando disabled
       assert.True(t, mgr.ShouldProcess("any-url", "any-hash"))
   }

   func TestManager_GetDeletedPages(t *testing.T) {
       // ... test cases ...
   }
   ```

2. **Criar `tests/integration/sync_test.go`**
   
   ```go
   //go:build integration

   package integration

   import (
       "context"
       "os"
       "path/filepath"
       "testing"

       "github.com/stretchr/testify/require"
   )

   func TestIncrementalSync_SkipsUnchanged(t *testing.T) {
       // 1. Primeira execucao cria estado
       // 2. Segunda execucao com --sync deve pular paginas inalteradas
       // 3. Verificar logs ou contadores
   }

   func TestIncrementalSync_ProcessesChanged(t *testing.T) {
       // 1. Primeira execucao
       // 2. Modificar conteudo mock
       // 3. Segunda execucao deve reprocessar
   }

   func TestFullSync_IgnoresState(t *testing.T) {
       // --full-sync deve processar tudo
   }

   func TestPrune_RemovesDeletedFiles(t *testing.T) {
       // 1. Criar estado com paginas
       // 2. Executar crawl que nao descobre algumas
       // 3. --prune deve remover arquivos
   }
   ```

## Estrategia de Testes

### Testes Unitarios

- [ ] `state.NewSyncState` - Cria estado vazio corretamente
- [ ] `Manager.Load` - Carrega estado existente
- [ ] `Manager.Load` - Retorna ErrStateNotFound se arquivo nao existe
- [ ] `Manager.Load` - Retorna ErrStateCorrupted se JSON invalido
- [ ] `Manager.Load` - Retorna ErrVersionMismatch se versao incompativel
- [ ] `Manager.Save` - Persiste estado no arquivo correto
- [ ] `Manager.Save` - Nao escreve se nao houver mudancas (dirty flag)
- [ ] `Manager.ShouldProcess` - Retorna true para URL nova
- [ ] `Manager.ShouldProcess` - Retorna true para hash diferente
- [ ] `Manager.ShouldProcess` - Retorna false para hash igual
- [ ] `Manager.ShouldProcess` - Retorna true quando disabled
- [ ] `Manager.Update` - Adiciona nova entrada
- [ ] `Manager.Update` - Atualiza entrada existente
- [ ] `Manager.GetDeletedPages` - Retorna paginas nao vistas
- [ ] Thread-safety - Operacoes concorrentes nao causam race conditions

### Testes de Integracao

- [ ] Fluxo completo: primeira execucao cria estado
- [ ] Fluxo completo: segunda execucao pula inalteradas
- [ ] `--full-sync` ignora estado
- [ ] `--prune` remove arquivos corretamente
- [ ] Multi-source (com manifest) mantem estados separados

### Casos de Teste Especificos

| ID | Cenario | Input | Output Esperado |
|----|---------|-------|-----------------|
| TC01 | Primeira execucao | URL sem estado previo | Cria .repodocs-state.json |
| TC02 | Sync com pagina inalterada | Hash igual ao estado | Skip, log "Skipping unchanged" |
| TC03 | Sync com pagina alterada | Hash diferente | Reprocessa, atualiza estado |
| TC04 | Sync com pagina nova | URL nao no estado | Processa, adiciona ao estado |
| TC05 | Full-sync | --full-sync flag | Processa tudo, ignora estado |
| TC06 | Prune pagina deletada | URL no estado mas nao descoberta | Remove arquivo |
| TC07 | Estado corrompido | JSON invalido | Log warning, inicia fresh |
| TC08 | Versao incompativel | state.version != 1 | Log warning, rebuild |

## Riscos e Mitigacoes

| Risco | Probabilidade | Impacto | Mitigacao |
|-------|---------------|---------|-----------|
| Race condition no estado | Media | Alto | Usar sync.RWMutex em todas operacoes |
| Estado corrompido por crash | Baixa | Medio | Atomic write (temp file + rename) |
| Estado muito grande (>100k paginas) | Baixa | Medio | Considerar BadgerDB se necessario |
| ContentHash nao calculado | Media | Alto | Verificar hash antes de usar estado |
| Inconsistencia apos erro | Media | Medio | Save periodico, nao apenas no final |

## Checklist de Conclusao

- [ ] Codigo implementado
  - [ ] `internal/state/models.go`
  - [ ] `internal/state/manager.go`
  - [ ] `internal/state/errors.go`
  - [ ] Modificacoes em `domain/options.go`
  - [ ] Modificacoes em `cmd/repodocs/main.go`
  - [ ] Modificacoes em `strategies/strategy.go`
  - [ ] Modificacoes em `strategies/crawler.go`
  - [ ] Modificacoes em `strategies/git/processor.go`
  - [ ] Modificacoes em `app/orchestrator.go`
- [ ] Testes escritos e passando
  - [ ] Testes unitarios do StateManager
  - [ ] Testes de integracao do fluxo de sync
- [ ] Documentacao atualizada
  - [ ] README com exemplos de uso
  - [ ] Help text das novas flags
- [ ] Code review realizado
- [ ] Feature testada manualmente em site real

## Notas Adicionais

### Decisoes de Design

1. **JSON vs BadgerDB para estado**: Escolhido JSON por simplicidade, human-readability, e porque o estado e relativamente pequeno (metadata apenas, sem conteudo). BadgerDB pode ser considerado no futuro para sites muito grandes.

2. **Opt-in explicito (`--sync`)**: Modo incremental requer flag explicita para evitar comportamento inesperado. Usuarios devem escolher conscientemente usar sincronizacao.

3. **ContentHash como chave**: Usa o hash ja calculado pelo pipeline de conversao, evitando re-fetch para comparacao.

4. **Estado por diretorio de output**: Cada diretorio de output tem seu proprio arquivo de estado, permitindo multiplos extracts do mesmo source com configuracoes diferentes.

### Exemplos de Uso

```bash
# Primeira execucao (cria estado)
repodocs https://docs.example.com --sync

# Segunda execucao (pula inalteradas)
repodocs https://docs.example.com --sync
# Output: "Skipping 45 unchanged pages, processing 3 changed"

# Forcar reprocessamento completo
repodocs https://docs.example.com --full-sync

# Remover arquivos de paginas deletadas
repodocs https://docs.example.com --sync --prune
```

### Formato do Arquivo de Estado

```json
{
  "version": 1,
  "source_url": "https://docs.example.com",
  "strategy": "crawler",
  "last_sync": "2026-01-10T12:00:00Z",
  "pages": {
    "https://docs.example.com/intro": {
      "content_hash": "a1b2c3d4e5f6...",
      "fetched_at": "2026-01-10T11:55:00Z",
      "file_path": "intro.md"
    },
    "https://docs.example.com/api/auth": {
      "content_hash": "f6e5d4c3b2a1...",
      "fetched_at": "2026-01-10T11:55:30Z",
      "file_path": "api/auth.md"
    }
  }
}
```
