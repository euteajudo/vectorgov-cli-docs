# Changelog

Todas as mudancas notaveis neste projeto serao documentadas neste arquivo.

O formato e baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Versionamento Semantico](https://semver.org/lang/pt-BR/).

## [0.2.10] - 2026-04-12

### Corrigido

- `vectorgov lookup -o json` em modo batch (auto-split) agora emite
  JSON estruturado com syntax highlight via `console.print_json`.
  Antes, o `-o json` caia no branch text e renderizava Panels Rich
  em vez de JSON.

## [0.2.9] - 2026-04-12

### Corrigido

- `vectorgov lookup` em modo batch (auto-split e `--pipe`) agora exibe
  `evidence_url` e `document_url` de cada resultado no output text,
  seguindo o mesmo padrao do modo single. Antes, os links so apareciam
  no modo `llm` e no `--raw`/`json`.

## [0.2.8] - 2026-04-12

### Corrigido

- `vectorgov lookup` agora renderiza resultados quando o backend faz
  auto-split de multiplas referencias numa mesma query. Antes, uma
  query como `"inc. I do Art. 75, inc. II do Art. 75"` retornava
  `status=batch` do backend, mas o handler single do CLI nao tratava
  esse caso e exibia "Nenhum dispositivo encontrado".
- Nova renderizacao batch em todos os formatos: `text` (Panels), `json`,
  `raw` (JSON estruturado) e `llm` (separador `---` entre refs).

## [0.2.7] - 2026-04-12

### Corrigido

- `vectorgov lookup -o json` agora emite JSON estruturado em vez de
  serializar o dict Python do match inteiro dentro do campo `text` do
  primeiro hit. Causa raiz: quando o backend remove `node_id` por
  seguranca, o SDK retorna `result.match = None` e o CLI faz fallback
  construindo um dict via `_build_match_from_raw`. O codigo usava
  `getattr(h, "text", str(h))` que em dicts cai no `str(h)` e serializa
  o dict inteiro como string.
- Helper `_get()` interno que le campos de dict OU de objeto Hit
  uniformemente, aplicado tambem ao output text (Panel).
- Output `-o json` agora promove `evidence_url` e `document_url` para
  top-level, alem de expor `status`, `query`, `total` e campos
  estruturados em cada hit (`device_type`, `document_id`,
  `article_number`, `breadcrumb`).

## [0.2.6] - 2026-04-12

### Alterado

- Comando `ask` agora e restrito a administradores e escondido do
  `--help` geral. Usuarios nao-admin que tentarem usar recebem erro
  orientando a usar `vectorgov search` ou `vectorgov context`.
- Toda a documentacao publica (README, MAPA_DO_CLI, CHANGELOG) foi
  atualizada para remover referencias ao comando.
- A verificacao usa o novo endpoint `GET /api/v1/sdk/me` do backend,
  que retorna os scopes da API key atual.

## [0.2.5] - 2026-04-12

### Adicionado

- Flag `--version` / `-V` no nivel global do CLI, seguindo a convencao
  de ferramentas como `python`, `node`, `git`, `pip`, `docker`, etc:

      vectorgov --version
      vectorgov -V

  O subcomando `vectorgov version` continua funcionando normalmente
  para compatibilidade.

## [0.2.4] - 2026-04-12

### Alterado

Documentacao interna limpa: removidas menções a tecnologias específicas do
stack backend nos docstrings, comentários e help messages. README, MAPA_DO_CLI
e CHANGELOG também atualizados. Não há mudanças de comportamento nem de API.

## [0.2.3] - 2026-04-12

### Adicionado — features otimizadas para consumo por IAs

**NOTA PARA LLMs E AGENTES DE IA**: esta versao introduz ferramentas
projetadas especificamente para que modelos de linguagem e agentes de
IA consumam o CLI do VectorGov de forma mais eficiente. Se voce e um
LLM usando o CLI em sessoes de vibe coding, estas features sao para voce.

#### `--output llm` (todos os 7 comandos de busca)

Formato de texto puro otimizado para consumo direto por LLMs. Sem
escapes ANSI, sem JSON, sem formatacao Rich. Cada hit separado por
`---` com header `[N/total] fonte (score)`, texto completo e links
de evidencia `EVIDENCE:` e `PDF:` em texto puro.

Economiza ~40% de tokens vs `--raw` (JSON) e elimina poluicao de
escapes vs `--output text` (Rich).

Disponivel em: `search`, `smart-search`, `hybrid`, `lookup`, `grep`,
`merged`, `fs-search`.

#### `VECTORGOV_OUTPUT` (env var) e `default_output` (config)

Define o formato de output padrao para toda a sessao:

    export VECTORGOV_OUTPUT=llm
    vectorgov search "ETP"     # usa llm automaticamente

Ou persistente:

    vectorgov config set default_output llm

Hierarquia: flag explicito > env var > config > default (table).
O `--raw` continua como bypass independente (sempre JSON bruto).

#### `vectorgov explain` (comando novo)

Contexto completo de um dispositivo legal em uma unica chamada.
Combina lookup (resolve a referencia) + texto consolidado (caput +
todos os filhos) + parent + links de evidencia.

    vectorgov explain --output llm "Art. 75 da Lei 14.133"

Ideal para LLMs que precisam entender um artigo completo sem fazer
multiplas chamadas (lookup + read).

#### `--pipe` em lookup (stdin batch)

Le referencias de stdin e faz batch em 1 chamada HTTP:

    printf "Art. 75 da Lei 14.133\nArt. 33 da Lei 14.133" | vectorgov lookup --raw --pipe

Reduz N chamadas para 1 quando o LLM precisa consultar multiplos
artigos em sequencia.

### Alterado

- `lookup`: argumento `query` agora e opcional (None por default) para
  suportar `--pipe`. Se nem query nem --pipe forem fornecidos, mostra
  erro explicativo (exit=1 em vez de exit=2 do typer).

## [0.2.2] - 2026-04-11

### Adicionado — links de evidência em todos os outputs

Antes, apesar do backend retornar `evidence_url` e `document_url` em cada hit
de busca, o CLI descartava esses campos na serialização. Agora todos os 7
comandos de busca expõem os links de evidência em todos os formatos de saída.

**Comandos afetados**: `search`, `smart-search`, `hybrid`, `lookup`, `grep`,
`merged`, `fs-search`.

**Formatos atualizados**:

- **Raw/JSON**: `evidence_url` e `document_url` são campos primeira classe
  em cada hit, sempre absolutos (prefixados com `https://vectorgov.io`)
- **Table** (search): nova coluna "Evidência" com link clicável via Rich
  markup — funciona em terminais modernos (iTerm2, Windows Terminal, VS
  Code), vira texto simples nos outros
- **Text**: linhas dedicadas "Ver trecho: {url}" e "Baixar PDF: {url}"
  indentadas abaixo de cada hit
- **Markdown** (search): nova seção "Fontes" por hit com links nativos

### Corrigido

- `lookup`: workaround para bug do SDK v0.16 que não mapeia `evidence_url`
  e `document_url` para o `LookupResult`. O CLI agora lê de
  `result._raw_response` como fallback quando o objeto não tem os atributos.

### Novos helpers em `utils/output.py`

- `absolute_url(url)`: prefixa com `https://vectorgov.io` quando URL é relativa
- `get_evidence_links(hit)`: extrai e normaliza `evidence_url` + `document_url`
  de um hit (aceita objeto ou dict)
- `hit_to_raw_dict(hit)`: serialização padronizada de hit para JSON
- `render_evidence_lines_text(hit)`: linhas formatadas para text output
- `render_evidence_cell_table(hit)`: célula com link clicável para tabela
- `render_evidence_markdown(hit)`: seção "Fontes" em Markdown

### Notas

Alguns comandos podem retornar `evidence_url: null` e `document_url: null`
porque o backend ainda não popula esses campos neles:
- `grep` — busca textual, sem enriquecimento
- `fs-search` — idem
- `merged` — dual-path que combina hybrid + filesystem; os hits do hybrid
  deveriam carregar os links mas o merge step não está propagando
- `hybrid.graph_nodes` — apenas `direct_evidence` tem os campos populados

O CLI propaga os campos como `null` nesses casos para que agentes
consumidores possam detectar a ausência e degradar graciosamente.
O fix do backend para esses 4 pontos está em SPEC separada.

## [0.2.1] - 2026-04-11

### Adicionado — paridade total SDK × CLI

- **Novo comando `read`**: wrapper para `vg.read_canonical(document_id, span_id)`
  - Lê texto canônico de um documento inteiro ou dispositivo específico
  - Útil para ler um artigo completo sem passar por busca semântica
  - Exemplo: `vectorgov read LEI-14133-2021 --span ART-075`

- `search`: novos flags `--tipo/-t`, `--ano/-a`, `--doc/-d`
  - `--tipo LEI` filtra por tipo de documento
  - `--ano 2021` filtra por ano
  - `--doc LEI-14133-2021` filtra por document_id específico
  - Passa `filters={}` e `document_id_filter=...` ao SDK

- `merged`: novos flags `--doc/-d`, `--no-hybrid`, `--no-filesystem`
  - `--doc` filtra por document_id
  - `--no-hybrid` desabilita busca semântica (usa só índice curado)
  - `--no-filesystem` desabilita índice curado (usa só hybrid)
  - Guard: erro se ambos `--no-*` usados juntos

- `docs list`: novos flags `--page/-p` e `--limit/-l`
  - Paginação conforme o SDK
  - Footer mostra "Página X/Y — Total na base: N documentos"

### Corrigido

- `lookup`: removido flag morto `--doc/-d` que não era passado ao SDK
  (o método `vg.lookup()` não aceita `document_id` — filtro deve ir
  embutido na própria referência, ex: "Art. 75 da Lei 14.133")

### Paridade SDK × CLI

Com a v0.2.1, **todos os 10 métodos de busca do SDK** têm representação no CLI
com todos os parâmetros funcionais:

| SDK method | CLI command | Paridade |
|------------|-------------|----------|
| `search()` | `search` | ✅ 100% (menos `trace_id` interno) |
| `smart_search()` | `smart-search` | ✅ 100% |
| `hybrid()` | `hybrid` | ✅ 100% (menos `trace_id` interno) |
| `lookup()` | `lookup` | ✅ 100% |
| `grep()` | `grep` | ✅ 100% |
| `filesystem_search()` | `fs-search` | ✅ 100% (menos `include_text`) |
| `merged()` | `merged` | ✅ 100% |
| `read_canonical()` | `read` | ✅ 100% (NOVO) |
| `list_documents()` | `docs list` | ✅ 100% |
| `get_document()` | `docs info` | ✅ 100% |

## [0.2.0] - 2026-04-11

### Adicionado
- `smart-search`: Busca inteligente MOC v4 com análise de completude e nível de confiança
- `hybrid`: Busca semântica + expansão por grafo normativo
- `lookup`: Consulta de artigo específico por referência legal
  - Novos flags `--parent/--no-parent` e `--siblings/--no-siblings`
- `grep`: Busca exata por texto no corpo das normas
  - Novos flags `--max/-n` (1-50) e `--context/-C` (0-10)
- `merged`: Busca dual-path combinando semântica + índice curado (RRF)
- `fs-search`: Busca no índice curado, modo `auto/index/grep/both`
- `audit logs`: Histórico de requisições à API
- `audit stats`: Estatísticas agregadas de uso
- `quota`: Consulta de uso do plano (smart_search + créditos)
- `prompts list`: Lista system prompts disponíveis para LLMs
- `prompts show`: Exibe system prompt completo
- `context`: Gera bloco completo (busca + system prompt) pronto para LLMs
- `init`: Inicializa projeto com CLAUDE.md, .cursorrules, AGENTS.md para ferramentas AI

### Corrigido
- `docs list`: crash `'DocumentsResponse' object is not iterable` — agora usa
  `resp.documents` em vez de iterar sobre o wrapper
- `docs list/info`: campos `tipo_documento`, `ano`, `titulo`, `chunks_count` agora
  lidos via `getattr` com fallback (compat SDK v0.13-v0.16+)
- `docs info`: fallback para `list_documents()` quando `get_document()` falha (era
  404 antes do backend implementar o endpoint `/sdk/documents/{id}`)
- `hybrid --raw`: saída sem `graph_nodes` nem `stats`, agora inclui ambos
- `hybrid`: quando `direct_evidence` está vazio (seeds fracos no reranker), usa
  `graph_nodes` como fallback para popular `hits` — garante que queries válidas
  como "Dispensa de licitação" retornem algo
- `lookup --raw`: serialização `str(Hit(...))` em vez de dict — corrigido para
  usar `__slots__` quando disponível
- `lookup`: reconstrução de `match` a partir de `_raw_response` quando o SDK v0.16
  retorna `match=None` (bug do SDK: parser só cria Hit se `node_id` presente, mas
  o backend remove `node_id` via filtro de segurança para não-admin)
- `lookup --raw`: agora inclui `status`, `parent`, `siblings`, `candidates`,
  `children`, `query_id`
- `merged --raw`: saída agora usa `.results` (era `.hits`) + inclui `mutual_count`,
  `hybrid_count`, `filesystem_count`
- `grep`: `--doc/-d` e validação de range para `--max/-n`
- Todos os comandos: removido `base_url="https://vectorgov.io"` incorreto que
  causava 404 (SDK já usa `https://vectorgov.io/api/v1` por default)

### Melhorado
- Cobertura de endpoints: de 50% para ~95% do SDK
- 19 comandos disponíveis (era 7 na v0.1.4)
- Testes automatizados: 136 casos (search, ask, tokens, docs, feedback, auth,
  config, version, smart-search, hybrid, lookup, fs-search, grep, merged)
- Validação end-to-end com >92% de taxa de sucesso

## [0.1.4] - 2025-01-20

### Corrigido
- Comando `feedback` agora usa subcomando `send` para evitar conflito com Typer
  - Uso: `vectorgov feedback send <query_id> --like`

## [0.1.3] - 2025-01-20

### Corrigido
- Caractere box-drawing (─) em tabelas causava erro no Windows (cp1252)
- Substituido por hifen (-) para compatibilidade

## [0.1.2] - 2025-01-20

### Corrigido
- Caracteres Unicode incompativeis com Windows (cp1252)
  - ✓ → OK
  - ✗ → ERRO
  - 👍 → +1
  - 👎 → -1
- Removidos acentos de mensagens para evitar encoding issues

## [0.1.1] - 2025-01-20

### Adicionado
- Comando `tokens` para estimativa de tokens antes de usar LLM
  - Mostra contexto, system prompt e query tokens
  - Compara com limites de modelos populares (GPT-4o, Claude, Gemini)
  - Suporta saida JSON

## [0.1.0] - 2025-01-20

### Adicionado
- Primeira versao publica do VectorGov CLI
- Comando `search` para busca semantica em legislacao
- Comando `ask` para obter contexto para LLMs
- Comando `feedback send` para enviar like/dislike
- Comando `docs list` e `docs info` para listar documentos
- Comando `auth login/status/logout` para autenticacao
- Comando `config set/get/list/delete` para configuracoes
- Suporte a variaveis de ambiente (VECTORGOV_API_KEY)
- Arquivo de configuracao em ~/.vectorgov/config.yaml
- Saida em tabela, JSON ou raw
