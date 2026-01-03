Com base na análise do código-fonte e da estrutura do projeto `repodocs-go`, identifiquei diversas oportunidades de otimização focadas em eficiência (redução de parsing redundante), organização do código e robustez.

Abaixo estão as principais recomendações divididas por áreas de impacto:

### 1. Otimização do Pipeline de Conversão (Eficiência)
Atualmente, o arquivo `internal/converter/pipeline.go` realiza o parsing do HTML (usando `goquery`) múltiplas vezes durante o processamento de uma única página.

* **Problema:** O método `Convert` faz parse do HTML no `extractor.Extract`, depois no `sanitizer.Sanitize`, novamente para extrair `description` e mais uma vez para `headers/links`.
* **Otimização:** Refatorar para utilizar um **único objeto `goquery.Document` compartilhado**.
    * Implementar métodos que aceitem o documento já parseado: `SanitizeDocument(doc)`, `ExtractHeadersFromDoc(doc)`, etc..
    * Isso reduzirá drasticamente o uso de CPU e memória em extrações com centenas de páginas.

### 2. Gerenciamento de Recursos no Renderer (Escalabilidade)
O `internal/renderer/pool.go` gerencia as abas do navegador para renderização de JavaScript.

* **Otimização de Reaproveitamento:** No método `Release` do `TabPool`, a aba é limpa navegando para `about:blank`. Para otimizar, verifique se o navegador está consumindo memória excessiva e implemente um limite de "tempo de vida" para cada aba (ex: fechar e recriar a aba após 50 usos) para evitar memory leaks comuns em instâncias longas do Chrome.
* **Detecção de SPA Proativa:** Aumentar a precisão do `internal/renderer/detector.go` adicionando padrões de frameworks modernos como Astro (client-side) e Qwik, reduzindo chamadas desnecessárias ao renderer pesado quando o conteúdo estático já é suficiente.

### 3. Melhoria na Estratégia de Cache (Performance)
O sistema utiliza o BadgerDB como cache persistente.

* **Compressão de Cache:** Como o cache armazena corpos de resposta HTML/Markdown (texto altamente repetitivo), a implementação de compressão (ex: Gzip ou Zstd) antes de salvar no BadgerDB reduziria significativamente o uso de disco.
* **Cache de Metadados:** Além de salvar o corpo da página, cachear os metadados já extraídos (título, descrição) evitaria re-processar o pipeline de conversão se o hash do conteúdo não mudou.

### 4. Organização e Design do Código
* **Separação de Preocupações (LLM):** No arquivo `internal/llm/metadata.go`, a lógica de extração de JSON das respostas do LLM (usando regex e brace matching) está acoplada ao componente de metadados.
    * **Otimização:** Mover as funções `extractJSON` e `findJSONObjectByBraceMatching` para um pacote utilitário de processamento de texto ou para o pacote `domain`, permitindo que outros componentes (como futuros geradores de tags) reutilizem a lógica robusta de limpeza de JSON.
* **Tratamento de Erros Sentinel:** Centralizar mais erros comuns no `internal/domain/errors.go` para evitar a criação de erros `fmt.Errorf` ad-hoc que dificultam o tratamento programático em camadas superiores.

### 5. Otimização de Estratégias Específicas
* **Docs.rs:** O plano já menciona a substituição do crawling HTML por parsing do JSON oficial da Rustdoc. Esta é a otimização de maior impacto no projeto, pois elimina centenas de requisições HTTP por uma única.
* **GitHub Pages:** Implementar o "Tier 1: LLM-optimized" (procurar `llms.txt` primeiro) como padrão global antes de tentar crawling BFS, economizando tempo e banda.

### Resumo Técnico de Ações Recomendadas
| Componente | Ação | Benefício |
| :--- | :--- | :--- |
| `Pipeline` | Implementar `doc-aware` parsing | Redução de overhead de CPU/RAM. |
| `BadgerCache` | Adicionar compressão Zstd | Economia de espaço em disco. |
| `LLM Enhancer` | Processamento em lote (Batching) | Se o provider suportar, reduz latência total. |
| `WorkerPool` | Ajuste dinâmico de concorrência | Evita bloqueios por IP (Rate Limit) mais eficientemente. |

Essas mudanças alinham o projeto com as melhores práticas de Go para ferramentas de CLI de alta performance e processamento de dados.