# Reference completa de comandos

> 20 comandos públicos do `vectorgov` CLI, organizados por categoria. Cada comando segue o mesmo padrão: **propósito → quando usar → exemplos → flags → formatos de saída → veja também**.
>
> 💡 **Procurando algo específico?** Use a [Cheat Sheet](cheat-sheet.md) para visão geral em 1 página, ou `Ctrl+F` aqui para buscar por nome.

---

## 🔍 Busca

### `search`

**Busca semântica simples** com 3 modos de performance.

#### Quando usar
- 80% dos casos de uso de busca em legislação
- Chat / RAG / autocomplete
- Quando você não precisa de análise jurídica completa nem de expansão por grafo

#### Exemplos
```bash
# Busca simples
vectorgov search "O que é ETP?"

# Mais resultados, modo preciso
vectorgov search "pesquisa de preços" --top-k 10 --mode precise

# Filtros estruturados
vectorgov search "dispensa" --tipo LEI --ano 2021
vectorgov search "art. 75" --doc LEI-14133-2021

# Saída em JSON estruturado (terminal com syntax highlight)
vectorgov search "licitação" --output json

# JSON bruto para pipes shell
vectorgov search "licitação" --raw | jq '.hits[0].text'
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--top-k/-k` | int (1-20) | `5` | Quantidade de resultados |
| `--mode/-m` | `fast`/`balanced`/`precise` | `balanced` | Trade-off latência/precisão |
| `--tipo/-t` | `LEI`/`DECRETO`/`IN`/`PORTARIA`/`AC` | — | Filtro por tipo |
| `--ano/-a` | int | — | Filtro por ano |
| `--doc/-d` | str | — | Filtro por documento (ex: `LEI-14133-2021`) |
| `--cache` | flag | `false` | Usar cache semântico |
| `--output/-o` | `table`/`json`/`text`/`llm`/`markdown` | `table` | Formato de saída |
| `--raw` | flag | `false` | JSON bruto para pipes |

#### Modos
| Mode | Latência | Quando usar |
|---|---|---|
| `fast` | ~2s | Autocomplete, busca rápida |
| `balanced` | ~5s | Default — chat, RAG (recomendado) |
| `precise` | ~15s | Análises críticas, queries ambíguas |

#### Veja também
- [`smart-search`](#smart-search) — análise jurídica completa
- [`hybrid`](#hybrid) — semântica + grafo
- [`merged`](#merged) — máxima cobertura

---

### `smart-search`

**Análise jurídica completa** com pipeline de Juiz LLM. Decide tudo: chunks, expansão, validação.

#### Quando usar
- Quando precisa de **resposta pronta com curadoria jurídica**
- Quando aceita pagar mais (💰💰) por qualidade superior
- Quando latência ~10-18s é aceitável

#### Exemplos
```bash
vectorgov smart-search "Quando o ETP pode ser dispensado?"
vectorgov smart-search "critérios de julgamento" --output json
vectorgov smart-search "pesquisa de preços" --cache
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--cache` | flag | `false` | Usar cache semântico |
| `--output/-o` | `table`/`json`/`text`/`llm` | `table` | Formato |
| `--raw` | flag | `false` | JSON bruto |

#### Veja também
- [`search`](#search) — alternativa mais barata
- [`hybrid`](#hybrid) — outra alternativa com grafo

---

### `hybrid`

**Semântica + expansão por grafo de citações**. Retorna chunks diretos **e** dispositivos relacionados.

#### Quando usar
- Quando precisa não só dos artigos mais relevantes, mas também dos **artigos citados/citantes**
- Para mostrar "cadeia regulatória" (lei → decreto → IN)

#### Exemplos
```bash
vectorgov hybrid "Critérios de julgamento em licitações"
vectorgov hybrid "Dispensa de licitação" --hops 2 --top-k 15
vectorgov hybrid "licitação" --output json
# Pergunta multi-dispositivo → mais cobertura (custa ~1,7x tokens):
vectorgov hybrid "Quais os critérios de julgamento?" --payload-coverage strict@20 --show-stats
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--top-k/-k` | int (1-50) | `10` | Hits diretos |
| `--hops` | int (1-2) | `1` | Saltos no grafo |
| `--graph-expansion` | `bidirectional`/`forward` | `bidirectional` | Direção da expansão |
| `--token-budget` | int | `6000` | Limite de tokens do contexto |
| `--payload-coverage` | `strict@10`/`strict@20` | `strict@10` | Janela de entrega: lean (default) ou wide (+cobertura, +tokens) |
| `--output/-o` | `table`/`json`/`text` | `table` | Formato |
| `--raw` | flag | `false` | Inclui `graph_nodes` e `stats` |

> 🎛️ Os 4 flags de payload (`--no-nota-espec`, `--no-jurisprudencia`, `--no-proveniencia`, `--no-links`) também valem aqui — veja [Flags de payload](#flags-de-payload-suprimir-features-no-envio-ao-llm).

#### `--payload-coverage` e o custo em tokens

Perguntas **multi-dispositivo** (respondidas por vários artigos/incisos) ficam
mais completas no modo **`strict@20`** (wide) — no golden, +0,231 de cobertura na
categoria multi (vs +0,127 no geral), **sem regressão**. O preço é **~1,7× mais
tokens**, e esse custo é **medível**: `token_count_estimate` aparece no JSON, no
rodapé `--show-stats` e na metadata do formato `llm`.

O default é **`strict@10`** (lean): a **resposta final de um LLM não melhora** com
a janela maior (janelas grandes sofrem de *lost-in-middle*). Por isso o wide vale
a pena para **recall/revisão** — quando você quer ver mais dispositivos — e não
para resposta-direta-de-LLM.

> **Composição do payload:** o `token_count_estimate` conta o payload **completo**
> entregue ao LLM — **não só o texto da lei**. Cada trecho também transporta a
> **nota do especialista** e a **jurisprudência do TCU** (curadoria), além de
> cabeçalhos estruturais. Total = **lei + curadoria + estrutura**. Considere isso
> ao orçar o contexto.

#### Veja também
- [`merged`](#merged) — versão dual-path com filesystem
- [`search`](#search) — alternativa sem grafo

---

### `lookup`

**Resolve referência legal exata** (`"Art. 75 da Lei 14.133"`) para o dispositivo correspondente. Sub-segundo. Suporta batch.

#### Quando usar
- Quando o usuário ou agente já sabe a referência exata
- Para mostrar o texto consolidado de um artigo (caput + parágrafos + incisos)
- Batch lookups (até 20 referências)

#### Exemplos
```bash
# Single
vectorgov lookup "Art. 75 da Lei 14.133"
vectorgov lookup "§ 1º do Art. 33 da Lei 14.133/2021"
vectorgov lookup "Inciso I do § 2 do Art. 4 da IN 67/2021"

# Controle de parent/siblings
vectorgov lookup --no-parent "Art. 1 da Lei 14.133"
vectorgov lookup --no-siblings "Art. 75 da Lei 14.133"

# Batch via auto-split (separadores: vírgula, ;, " e ")
vectorgov lookup "Art. 75, Art. 18 e Art. 33 da Lei 14.133"

# Batch via stdin
printf "Art. 75 da Lei 14.133\nArt. 33 da Lei 14.133" | vectorgov lookup --pipe

# Para colar em LLM (texto puro)
vectorgov lookup -o llm "Art. 11 da Lei 14.133"
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--parent/--no-parent` | flag | `--parent` | Incluir dispositivo pai |
| `--siblings/--no-siblings` | flag | `--siblings` | Incluir irmãos |
| `--output/-o` | `text`/`json`/`llm` | `text` | Formato |
| `--raw` | flag | `false` | JSON bruto sem formatação |
| `--pipe` | flag | `false` | Lê referências de stdin (uma por linha) |

#### Output inclui (quando disponível)
- Texto do dispositivo (consolidado para artigos)
- `evidence_url` — link para o trecho destacado
- `document_url` — link para download do PDF
- `nota_especialista` — comentário do especialista jurídico
- `jurisprudencia_tcu` — jurisprudência relacionada

#### Veja também
- [`explain`](#explain) — lookup + read em uma chamada
- [`read`](#read) — quando você já tem o ID canônico

---

### `grep`

**Busca textual literal** via ripgrep. Sub-segundo.

#### Quando usar
- Quando você sabe a forma exata do texto (`"dispensa de licitação"`)
- Para encontrar palavras-chave específicas
- Quando busca semântica retorna ruído mas você sabe o termo

#### Exemplos
```bash
vectorgov grep "dispensa de licitação"
vectorgov grep "ETP" --doc LEI-14133-2021
vectorgov grep "licitação" --max 10 --context-lines 5
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--doc/-d` | str | — | Filtrar por documento |
| `--max/-n` | int (1-50) | `20` | Máximo de matches |
| `--context-lines/-C` | int (0-10) | `3` | Linhas de contexto ao redor |
| `--output/-o` | `table`/`json`/`text` | `table` | Formato |
| `--raw` | flag | `false` | JSON bruto |

#### Veja também
- [`fs-search`](#fs-search) — busca no índice curado (não literal)

---

### `fs-search`

**Índice curado** com modo `auto` que detecta se a query é referência legal (grep) ou semântica (index).

#### Quando usar
- Para siglas ou termos curados (`"ETP"`, `"PCA"`, `"PNCP"`)
- Para conceitos com aliases curados pelo especialista
- Quando quer combinar precisão + recall (modo `both`)

#### Exemplos
```bash
vectorgov fs-search "art. 75 da Lei 14.133"
vectorgov fs-search "pregão eletrônico"
vectorgov fs-search "dispensa" --mode index
vectorgov fs-search "art. 75" --mode grep
vectorgov fs-search "ETP" --mode both
vectorgov fs-search "art. 75" --doc LEI-14133-2021 --output json
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--doc/-d` | str | — | Filtrar por documento |
| `--top-k/-k` | int (1-50) | `10` | Máximo de resultados |
| `--mode/-m` | `auto`/`index`/`grep`/`both` | `auto` | Estratégia |
| `--output/-o` | `table`/`json`/`text` | `table` | Formato |
| `--raw` | flag | `false` | JSON bruto |

#### Veja também
- [`search`](#search) — busca vetorial (embeddings)
- [`grep`](#grep) — só busca textual literal

---

### `merged`

**Dual-path**: executa `hybrid` + `fs-search` em paralelo, deduplica e rankeia via RRF.

#### Quando usar
- Para **máxima cobertura** — combina o melhor dos dois mundos
- Quando você não sabe se a query é melhor para semântica ou índice curado

#### Exemplos
```bash
vectorgov merged "Modalidades de licitação"
vectorgov merged "Pesquisa de preços" --top-k 15
vectorgov merged "art. 75" --doc LEI-14133-2021

# Controle granular de backends
vectorgov merged "dispensa" --no-filesystem    # apenas semântica
vectorgov merged "licitação" --no-hybrid       # apenas índice curado
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--top-k/-k` | int (1-50) | `10` | Máximo de resultados |
| `--doc/-d` | str | — | Filtrar por documento |
| `--token-budget` | int | `6000` | Limite de tokens |
| `--no-hybrid` | flag | `false` | Desabilita semântica |
| `--no-filesystem` | flag | `false` | Desabilita índice curado |
| `--output/-o` | `table`/`json`/`text` | `table` | Formato |
| `--raw` | flag | `false` | Inclui `mutual_count`, `hybrid_count`, etc. |

#### Veja também
- [`hybrid`](#hybrid) — só semântica + grafo
- [`fs-search`](#fs-search) — só índice curado

---

### `read`

**Lê o texto canônico completo** de um documento ou dispositivo específico. Determinístico, sub-segundo, **gratuito**.

#### Quando usar
- Para mostrar o texto completo de uma lei ou artigo
- Quando você já tem o `document_id` e (opcionalmente) `span_id`

#### Exemplos
```bash
# Documento inteiro
vectorgov read LEI-14133-2021

# Apenas um dispositivo
vectorgov read LEI-14133-2021 --span ART-075
vectorgov read LEI-14133-2021 --span PAR-033-1
vectorgov read LEI-14133-2021 --span INC-005-I

# JSON estruturado
vectorgov read LEI-14133-2021 --span ART-075 -o json
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--span/-s` | str | — | Dispositivo específico (`ART-075`, `PAR-033-1`, etc.) |
| `--output/-o` | `text`/`json` | `text` | Formato |
| `--raw` | flag | `false` | JSON bruto |

#### Diferença para `lookup`
- `lookup` resolve referências em **linguagem natural** ("Art. 75 da Lei 14.133")
- `read` usa **IDs canônicos** diretamente

Use `read` quando já souber o `document_id` e `span_id` exatos (ex: após um `lookup`).

#### Veja também
- [`lookup`](#lookup) — quando você tem a referência em texto
- [`explain`](#explain) — combina os dois

---

### `explain`

**lookup + read** em uma chamada. Resolve a referência e já entrega o texto consolidado completo.

#### Quando usar
- Quando quer "tudo de um artigo" em uma única chamada
- Para gerar contexto rico para um LLM sem fazer 2 requests

#### Exemplos
```bash
vectorgov explain "Art. 75 da Lei 14.133"
vectorgov explain "§ 2 do Art. 33 da Lei 14.133" --output llm
```

#### Veja também
- [`lookup`](#lookup) — só a resolução
- [`read`](#read) — só a leitura por ID

---

## 🤖 LLM helpers

### `context`

**Bloco completo** (busca + system prompt) pronto para colar em qualquer LLM.

#### Quando usar
- Para vibe coding com LLMs (ChatGPT, Claude, Gemini)
- Quando quer pular o passo de "montar o contexto manualmente"

#### Exemplos
```bash
# Bloco padrão (texto puro)
vectorgov context "Quando posso usar dispensa de licitação?"

# Formato messages (JSON OpenAI-compatible)
vectorgov context "ETP" --format messages

# Com smart-search (mais profundo)
vectorgov context "pregão eletrônico" --smart

# Com prompt específico
vectorgov context "pesquisa de preços" --prompt detailed
```

#### Flags
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--format` | `raw`/`messages`/`clipboard` | `raw` | Formato de saída |
| `--smart` | flag | `false` | Usa `smart-search` em vez de `search` |
| `--prompt` | str | `default` | System prompt: `default`, `concise`, `detailed`, `chatbot` |

---

### `tokens`

**Estima quantos tokens** uma busca consumiria. Útil para planejar uso com LLMs.

#### Exemplos
```bash
vectorgov tokens "O que é ETP?"
vectorgov tokens "pesquisa de preços" --top-k 10
vectorgov tokens "licitação" --output json
```

#### Output exemplo
```
Estimativa de Tokens
+-----------------+------------+---------------------------+
| Componente      |     Tokens | Descrição                 |
+-----------------+------------+---------------------------+
| Contexto        |      1,234 | 5 hits da busca           |
| System Prompt   |        200 | Instruções do sistema     |
| Query           |          5 | Pergunta do usuário       |
|-----------------|------------|---------------------------|
| Total           |      1,439 | 5,432 caracteres          |
+-----------------+------------+---------------------------+

Comparação com limites:
  GPT-4o:            OK 1.1% (1,439/128,000)
  Claude 3.5 Sonnet: OK 0.7% (1,439/200,000)
  Gemini 2.0 Flash:  OK 0.1% (1,439/1,000,000)
```

---

### `prompts`

**System prompts pré-otimizados** para uso com LLMs.

#### Subcomandos
```bash
vectorgov prompts list              # Lista estilos disponíveis
vectorgov prompts show juridico --raw  # Mostra o texto completo
```

#### Estilos disponíveis
- `default` — uso geral
- `concise` — respostas curtas
- `detailed` — análises completas
- `chatbot` — conversa multi-turno

---

## 📊 Info & feedback

### `docs`

**Lista normas indexadas** e mostra metadados de uma norma específica. **Gratuito**.

#### Subcomandos
```bash
# Listar
vectorgov docs list
vectorgov docs list --page 2 --limit 20
vectorgov docs list --output json

# Detalhe de um documento
vectorgov docs info LEI-14133-2021
vectorgov docs info IN-65-2021
```

#### Flags `docs list`
| Flag | Tipo | Default | Descrição |
|---|---|---|---|
| `--page/-p` | int | `1` | Página |
| `--limit/-l` | int (1-100) | `50` | Itens por página |
| `--output/-o` | `table`/`json` | `table` | Formato |

---

### `audit`

**Histórico de requisições e estatísticas** de uso da API. **Gratuito**.

#### Subcomandos
```bash
# Logs (últimos 30 dias)
vectorgov audit logs --days 30 --limit 50

# Estatísticas agregadas
vectorgov audit stats --days 7
```

---

### `quota`

**Uso do plano** — cotas de smart-search e créditos restantes. **Gratuito**.

```bash
vectorgov quota
vectorgov quota --output json
```

---

### `feedback`

**Like/dislike** de resultado de busca para melhoria contínua. **Gratuito**.

#### Subcomandos
```bash
# Após uma busca, use o query_id
vectorgov feedback send abc123def456 --like
vectorgov feedback send abc123def456 --dislike
```

---

## 🛠️ Setup & config

### `auth`

**Autenticação** — login, status, logout.

#### Subcomandos
```bash
vectorgov auth login        # Salva API key
vectorgov auth status       # Mostra autenticação atual
vectorgov auth logout       # Remove API key
```

---

### `config`

**Gerencia configurações** em `~/.vectorgov/config.yaml`.

#### Subcomandos
```bash
# Listar tudo
vectorgov config list

# Obter valor específico
vectorgov config get api_key

# Definir valor
vectorgov config set default_mode precise
vectorgov config set default_top_k 10

# Remover
vectorgov config delete default_mode
```

#### Prioridade de configuração
1. Flags da linha de comando (maior)
2. Variáveis de ambiente (`VECTORGOV_*`)
3. Arquivo `~/.vectorgov/config.yaml`
4. Defaults do CLI (menor)

---

### `init`

**Inicializa projeto** com arquivos para ferramentas AI.

```bash
vectorgov init --all       # CLAUDE.md, .cursorrules, AGENTS.md
vectorgov init --claude    # Apenas CLAUDE.md
vectorgov init --cursor    # Apenas .cursorrules
vectorgov init --codex     # Apenas AGENTS.md
```

---

### `version`

**Mostra versão do CLI**.

```bash
vectorgov --version    # ou -V
vectorgov version      # subcomando alternativo
```

---

## 🎛️ Flags de payload (suprimir features no envio ao LLM)

Disponíveis em `search`, `hybrid`, `smart-search`, `merged`, `grep`, `fs-search`,
`lookup`, `explain`, `context` e `ask`. Todas vêm **ligadas por padrão** (payload
completo) — desligue o que não quiser mandar ao LLM. O **texto do dispositivo
nunca é removido**.

| Flag | Desliga |
|---|---|
| `--no-nota-espec` | Nota do especialista |
| `--no-jurisprudencia` | Jurisprudência relacionada (TCU) |
| `--no-proveniencia` | Proveniência (origem) — só no payload do LLM; `document_id`/`span_id`/`breadcrumb` seguem no JSON/raw |
| `--no-links` | Links de evidência (PDF / trecho destacado) |

```bash
# só o texto da lei, sem curadoria nem links
vectorgov lookup "Art. 75 da Lei 14.133" --no-nota-espec --no-jurisprudencia --no-proveniencia --no-links
```

> Requer `vectorgov` (SDK) **>= 0.21.0** para o efeito das flags.

---

## 📊 Telemetria de tokens nas respostas

Os comandos de busca (`search`, `smart-search`, `hybrid`, `lookup`, `merged`,
`grep`, `fs-search` e `ask`) anexam, quando disponível, a **telemetria de tokens**
do payload que seria entregue a um LLM. Os campos aparecem no `--raw`/`--output
json`, no rodapé do formato padrão/`llm` e no rodapé do `--show-stats` (hybrid).

| Campo | Descrição |
|---|---|
| `token_count_estimate` | Estimativa do total de tokens do payload **completo** (lei + curadoria + estrutura) entregue ao LLM. |
| `token_count_method` | Método usado para produzir a contagem (ex.: `estimate`). Indica se o número é aproximado. |
| `token_count_breakdown` | Decomposição do custo: `law_chunks` (texto da lei), `curadoria` (nota do especialista + jurisprudência) e `structure` (cabeçalhos). A soma é igual a `token_count_estimate`. |
| `payload_coverage` | Janela de entrega aplicada (`strict@10` lean por padrão, `strict@20` wide). Ver [`hybrid`](#hybrid). |

Exemplo de rodapé (formato padrão/`llm`):

```
Total: 10 hits | Latencia: 1.3s | payload: strict@20 | ~2372 tokens | (lei=1856 curadoria=0 estrutura=516) | Request ID: ...
```

> Os campos são **opcionais**: aparecem quando o SDK e a API os fornecem; caso
> contrário, são omitidos (o CLI degrada graciosamente). Use-os para **orçar o
> contexto** antes de mandar os resultados a um LLM — o complemento natural do
> comando [`tokens`](#tokens).

---

## 📤 Formatos de saída

Todos os comandos de busca suportam: `table` (padrão), `json`, `text`, `llm` e `--raw`.

### `--output llm` (otimizado para IAs)
Texto puro, sem ANSI, sem JSON. Economiza ~40% de tokens vs JSON.

```
[1/5] Art. 18 da Lei 14.133/2021 (score: 0.97)
Art. 18. A fase preparatória do processo licitatório é caracterizada pelo planejamento ...
EVIDENCE: https://vectorgov.io/api/v1/evidence/leis%3ALEI-14133-2021%23ART-018
PDF: https://vectorgov.io/api/v1/evidence/download/source/LEI-14133-2021
---
```

### `--output table` (padrão interativo)
Tabelas Rich coloridas com formatação automática.

### `--output json` (terminal com syntax highlight)
JSON estruturado bonito para leitura no terminal.

### `--raw` (JSON bruto para pipes)
JSON sem formatação, ideal para `jq` ou redirect:
```bash
vectorgov search --raw "ETP" > resultados.json
vectorgov search --raw "ETP" | jq '.hits[0]'
```

---

## 🤖 Para LLMs e agentes

Defina uma vez e todos os comandos retornam texto otimizado:

```bash
export VECTORGOV_OUTPUT=llm
```

Ou use a flag em cada chamada:
```bash
vectorgov search --output llm "ETP"
vectorgov context --output llm "dispensa de licitação"
vectorgov lookup --output llm "Art. 75 da Lei 14.133"
```

> O CLI **detecta automaticamente** quando o stdout não é um terminal (pipe, redirect, CI) e usa o formato `llm` por padrão. Não precisa configurar nada para integrar com scripts/agentes.

---

## Veja também

- 🧭 [Cheat Sheet](cheat-sheet.md) — todos os comandos em 1 página
- 🍳 [Receitas comuns](recipes.md) — fluxos completos por caso de uso
- 📦 [SDK Python](https://pypi.org/project/vectorgov/) — alternativa programática
- 🌐 [Playground online](https://vectorgov.io/playground)
