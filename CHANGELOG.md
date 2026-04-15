# Changelog

Todas as mudancas notaveis neste projeto serao documentadas neste arquivo.

O formato e baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/),
e este projeto adere ao [Versionamento Semantico](https://semver.org/lang/pt-BR/).

## [0.3.2] - 2026-04-15

### Adicionado

- **Coluna `Norma` na tabela do `search`**: cada linha agora mostra a
  qual norma o artigo pertence (ex: `Lei 14.133/2021`, `IN 65/2021`,
  `Decreto 10.947/2022`). Antes, o usuario via apenas `Art. 3º` sem
  saber de qual norma — problema critico quando varias normas tem a
  mesma numeracao. Os outros 5 comandos de busca (`smart-search`,
  `hybrid`, `merged`, `fs-search`, `grep`) ja tinham coluna `Fonte`
  equivalente.

- **Helper `format_document_label(document_id)`**: formata IDs
  internos para exibicao humana. Exemplos: `LEI-14133-2021` ->
  `Lei 14.133/2021`, `IN-65-2021` -> `IN 65/2021`,
  `DECRETO-10947-2022` -> `Decreto 10.947/2022`, `AC-1852-2020-P` ->
  `Acordao 1.852/2020-P`. Usado em colunas de tabela, headers de
  text mode e footers de rodape.

- **TTY detection automatica em `resolve_output_format`**: quando
  stdout nao e um terminal (pipe, redirect, LLM, CI/CD), o formato de
  saida padrao vira `llm` automaticamente em vez de `table`. Agentes
  de IA que executam `vectorgov search "..."` agora recebem texto
  integral otimizado para consumo por modelos, sem precisar passar
  `--output llm` explicitamente. Comportamento GNU-style
  (`ls --color=auto`, `git log`, `jq`).

  **Precedencia completa do resolver de formato**:
  1. `--raw` (bypass tudo)
  2. `--output` explicito diferente do default do comando
  3. `VECTORGOV_OUTPUT` env var
  4. `config.default_output`
  5. **stdout nao e TTY** (novo) -> `llm`
  6. Default do comando

  Preferencias explicitas sempre tem precedencia sobre a detecao
  automatica. Para forcar tabela em pipe, use `VECTORGOV_OUTPUT=table`.

- **Footer do `search` com fontes unicas**: substitui o antigo
  `PDF do primeiro resultado: ...` por uma lista consolidada
  `Fontes (N): Lei 14.133/2021, IN 65/2021, ...` com todas as normas
  envolvidas nos hits, ja formatadas para leitura humana.

### Corrigido

- **Zero truncamento de conteudo em TODOS os formatos de saida**:
  removidas 26 ocorrencias de truncamento (`text[:N] + "..."`)
  espalhadas em 11 arquivos. Antes, o default `table` cortava textos
  em apenas 95 caracteres, tornando impossivel para um LLM ler o
  conteudo completo dos artigos retornados. O truncamento tambem
  existia em `text` (300 chars), `json` (300 chars em varios
  comandos), e ate no proprio `llm` (cap de 2000 chars). Agora todos
  os formatos retornam texto integral.

  **Filosofia**: se o usuario quer menos dados, use `--top-k N`.

  Comandos afetados (todos os 14 single-shot + docs):
  `search`, `smart-search`, `hybrid`, `merged`, `fs-search`, `grep`,
  `lookup`, `explain`, `ask`, `context`, `tokens`, `read`, `quota`,
  `init`, `docs list`.

  Truncamentos de credenciais (`auth login`, `config list/get`) foram
  MANTIDOS pois mascarar API keys com `vg_abc...xyz1` e correto por
  seguranca.

- **Word-wrap automatico em todas as 6 tabelas de busca**: coluna
  `Texto`/`Trecho` agora usa Rich `overflow="fold"`, permitindo
  quebra natural em multiplas linhas da celula preservando o dado
  integral. Em TTY largo fica excelente; em terminal estreito as
  linhas ficam mais altas mas o conteudo e sempre completo.

## [0.3.1] - 2026-04-15

### Corrigido

- **Parsing GNU-style de argumentos**: o CLI agora aceita flags e
  argumentos posicionais em qualquer ordem, conforme o padrao da
  industria (git, curl, kubectl, npm, etc). Antes, os 14 comandos
  single-shot (`search`, `grep`, `hybrid`, `lookup`, `merged`,
  `smart-search`, `fs-search`, `context`, `tokens`, `explain`,
  `read`, `quota`, `init`, `ask`) quebravam com
  `Missing argument 'QUERY'` quando flags vinham depois da query
  (ex: `vectorgov search "ETP" --top-k 3`).

  **Causa raiz**: cada comando estava registrado como multi-command
  group do Click via a combinacao `app.add_typer(cmd.app, ...)` +
  `@app.callback(invoke_without_command=True)`. Em group, o parser
  do Click para de aceitar flags intercaladas com posicionais porque
  tenta resolver subcomandos inexistentes.

  **Fix**: cada um dos 14 comandos single-shot agora e uma funcao
  Python pura registrada diretamente no app raiz via
  `app.command(name=...)(fn)`. Os 6 grupos reais com subcomandos
  (`auth`, `audit`, `config`, `docs`, `feedback`, `prompts`)
  continuam usando `add_typer()`.

  Zero breaking change: a ordem antiga (flags antes da query)
  continua funcionando — o fix e aditivo.

- **`commands/__init__.py` desatualizado**: faltavam os re-exports
  de `fs_search`, `read` e `explain`. Agora todos os 20 comandos
  estao expostos.

## [0.3.0] - 2026-04-12

### Adicionado

- **Request ID em todos os comandos**: `Request ID: <32 chars>` aparece
  no rodape do output text/llm e como campo `request_id` no output JSON
  de todos os comandos de busca (`search`, `lookup`, `hybrid`,
  `smart-search`, `ask`, `merged`, `grep`, `fs-search`, `read`,
  `explain`, `docs list/info`, `context`, `tokens`, `feedback`). Use
  este ID para correlacao com logs no dashboard `/uso-api` e para
  suporte tecnico.
- Helpers novos em `utils/output.py`:
  - `render_request_id_line(request_id, prefix="")`: retorna lista de
    linhas para composicao programatica com `extend()`.
  - `print_request_id_footer(console, request_id)`: imprime rodape
    dim-style via Rich no final de handlers text mode.
  - `extract_request_id(result, raw_resp=None)`: extrai `request_id`
    do result com fallback para `_raw_response` (graceful degradation
    quando o SDK em uso nao expoe o campo).

### Alterado

- Versao bump **minor** (0.2.12 → 0.3.0) reflete contrato novo de saida:
  campo `request_id` aparece no top-level do JSON de todos os comandos
  de busca. Clientes automatizados (scripts, IDEs, CI/CD pipelines)
  devem estar preparados para o novo campo, embora ele seja opcional.
- `merged -o json`, `grep -o json`, `fs-search -o json` agora retornam
  um objeto `{request_id, hits}` em vez de um array simples de hits —
  necessario para incluir `request_id` no nivel superior do output.

### Corrigido

Auditoria sistematica de todos os 21 comandos com todos os formatos
de saida (text/json/llm/raw) no Windows/Git Bash. 9 bugs encontrados
e corrigidos:

1. **UnicodeEncodeError em Windows** (stdout cp1252): `hybrid --raw`,
   `fs-search` text, `context --smart`, `init --all` e outros comandos
   quebravam com caracteres fora de cp1252 (`\u202f` non-breaking space,
   `\u2713` checkmark, etc.). Fix aplicado em `main.py`: reconfigura
   `sys.stdout`/`sys.stderr` para UTF-8 antes de qualquer comando rodar.
   Acentos agora aparecem corretamente no Windows.

2. **`config set default_output`**: reportava chave desconhecida mesmo
   sendo o nome documentado no README. Causa: `config.py` validava
   contra `output_format` (legado) mas `resolve_output_format()` lia
   `default_output` (novo desde 0.2.3). Fix: aceitar ambas as chaves.

3. **`merged --no-filesystem`**: retornava `Erro: 0` em vez de
   `Nenhum resultado encontrado.` quando o backend devolvia lista
   vazia. `typer.Exit(0)` era capturado pelo handler generico de
   Exception. Fix: adicionar `except typer.Exit: raise` antes.

4. **`audit logs -o json`** sem resultados: emitia texto em portugues
   em vez de `[]`. Fix: emitir JSON vazio mesmo quando `entries == 0`.

5. **`prompts show <qualquer_string>`**: aceitava nomes invalidos
   (retornava o prompt default). Fix: validar `style` contra
   `vg.available_prompts` e recusar nomes desconhecidos.

6. **`feedback send <short_id>`**: mostrava erro 422 como JSON raw
   do Pydantic. Fix: validar `query_id < 8` chars localmente com
   mensagem amigavel em PT, e traduzir erros 422/404/401/403 do
   backend antes de exibir.

7. **`audit logs`/`audit stats`** com `typer.Exit(0)`: mesmo bug do
   `merged --no-filesystem`. Fix: adicionar `except typer.Exit: raise`
   nos handlers.

8. **`docs list`** retornava coluna **Titulo** vazia para acordaos e
   alguns outros documentos. Fix: deriva titulo humano do `document_id`
   quando o backend nao retorna (ex: `AC-1.852-2.020-P` ->
   `Acordao 1.852/2020 (Plenario)`).

9. **`quota`** sem plano premium: mostrava `Informacao nao disponivel`
   sem contexto. Fix: mensagens especificas por status HTTP (401/403
   -> sem permissao, 404/405 -> plano gratuito, etc.) + hint com link
   para pricing.

### Compatibilidade

- Graceful degradation: se a versao do SDK usada nao expoe `request_id`
  (SDK < 0.18.0), o rodape e omitido e o campo JSON vem como `null`.
  Bumpar a dep minima do SDK quando 0.18.0 for publicado no PyPI.

## [0.2.12] - 2026-04-12

### Corrigido

- Output `lookup -o llm` agora exibe o texto consolidado corretamente
  quando o match vem em formato dict (fallback do parser). Antes,
  o campo `text` saia vazio em alguns casos.
- Limite do texto no modo llm aumentado de 500 para 2000 chars — mais
  adequado para artigos inteiros consolidados.

### Adicionado

- Modo `-o llm` do lookup agora exibe `nota_especialista` e
  `jurisprudencia_tcu` (quando presentes) em secoes dedicadas com
  separador `---`, seguindo o mesmo padrao do modo text.

## [0.2.11] - 2026-04-12

### Corrigido

- `vectorgov lookup` de um ARTIGO inteiro nao duplica mais os filhos
  na lista de hits. Como o match.text ja contem o stitched_text
  completo (caput + incisos + paragrafos), incluir tambem cada
  children individualmente era redundante (1 hit do match + N hits
  dos filhos com mesmo conteudo). Agora: artigos retornam apenas
  1 hit (o match). Para inciso/paragrafo especifico, children
  continua sendo incluido como antes.

### Adicionado

- Expoe campos de curadoria (SPEC 1C) no output do lookup:
  - `nota_especialista` — comentario do especialista juridico
  - `jurisprudencia_tcu` — texto de jurisprudencia relacionada
  - `acordao_tcu_key` — numero do acordao TCU (ex: "1852/2020")
  - `acordao_tcu_link` — link para o acordao completo
  Disponivel em todos os formatos (text, json, llm, raw). No modo
  text, aparecem em Panels dedicados abaixo dos hits.

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
  serializar o dict Python inteiro dentro do campo `text` do primeiro
  hit. Causa raiz: quando o response público omitia `node_id`, o
  parser caía em fallback que serializava todo o dict como string.
- Helper interno que lê campos de dict ou objeto uniformemente,
  aplicado também ao output text (Panel).
- Output `-o json` agora promove `evidence_url` e `document_url` para
  top-level, além de expor `status`, `query`, `total` e campos
  estruturados em cada hit (`device_type`, `document_id`,
  `article_number`, `breadcrumb`).

## [0.2.6] - 2026-04-12

### Alterado

- Reorganizacao interna de comandos: alguns comandos legados foram
  removidos do `--help` publico. Use `vectorgov search` ou
  `vectorgov context` para os fluxos principais de busca.

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
- `smart-search`: Busca inteligente com análise de completude e nível de confiança
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
- `hybrid`: quando `direct_evidence` está vazio (seeds pouco relevantes), usa
  `graph_nodes` como fallback para popular `hits` — garante que queries válidas
  como "Dispensa de licitação" retornem algo
- `lookup --raw`: serialização correta dos objetos Hit (compat com slots)
- `lookup`: reconstrução de `match` a partir do response bruto quando o SDK
  retorna `match=None` (caso em que o response público não inclui IDs internos)
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
