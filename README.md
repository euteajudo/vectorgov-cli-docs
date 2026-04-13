# VectorGov CLI

Cliente de linha de comando para a API VectorGov - Busca semântica em legislação brasileira.

[![PyPI version](https://img.shields.io/pypi/v/vectorgov-cli.svg)](https://pypi.org/project/vectorgov-cli/)
[![PyPI downloads](https://img.shields.io/pypi/dm/vectorgov-cli.svg)](https://pypi.org/project/vectorgov-cli/)
[![Python versions](https://img.shields.io/pypi/pyversions/vectorgov-cli.svg)](https://pypi.org/project/vectorgov-cli/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## Para LLMs e agentes de IA (v0.2.3)

Se voce e um LLM ou agente de IA usando o CLI do VectorGov em sessoes
de vibe coding, estas features foram projetadas para voce:

```bash
# Formato otimizado para IAs (texto puro, sem ANSI, sem JSON)
vectorgov search --output llm "O que e ETP?"

# Definir como padrao para toda a sessao
export VECTORGOV_OUTPUT=llm

# Contexto completo de um artigo em uma unica chamada
vectorgov explain --output llm "Art. 75 da Lei 14.133"

# Batch: consultar multiplos artigos em 1 chamada HTTP
printf "Art. 75 da Lei 14.133\nArt. 33 da Lei 14.133" | vectorgov lookup --raw --pipe
```

O formato `llm` retorna texto puro com separadores `---` entre hits,
headers `[N/total] fonte (score)` e links `EVIDENCE:` / `PDF:` por hit.
Economiza ~40% de tokens comparado ao JSON (`--raw`) e elimina escapes
ANSI do formato `text` que poluem o contexto do modelo.

---

## Provando a veracidade — evidence_url e document_url

Todo comando de busca (`search`, `smart-search`, `hybrid`, `lookup`, `grep`,
`merged`, `fs-search`) expõe dois campos de evidência em cada hit:

- `evidence_url` — link para o trecho destacado visualmente na norma
- `document_url` — link para baixar o PDF original do documento

Ambos são URLs absolutas (prefixadas com `https://vectorgov.io`) e permanentes
(não expiram). Você pode copiar, compartilhar e clicar direto.

**Exemplo — output text mostra os links indentados abaixo de cada hit**:

```bash
$ vectorgov search --top-k 1 --output text "O que é ETP?"

[1] Art. 3 (score: 0.987)
I - Estudo Técnico Preliminar - ETP: documento constitutivo...
    Ver trecho: https://vectorgov.io/api/v1/evidence/IN-58-2022%23INC-003-I
    Baixar PDF: https://vectorgov.io/api/v1/evidence/download/source/IN-58-2022
```

**Exemplo — `--raw` expõe os campos em cada hit**:

```bash
$ vectorgov search --raw --top-k 1 "ETP" | jq '.hits[0]'
{
  "text": "I - Estudo Técnico Preliminar...",
  "article_number": "3",
  "document_id": "IN-58-2022",
  "score": 0.987,
  "evidence_url": "https://vectorgov.io/api/v1/evidence/IN-58-2022%23INC-003-I",
  "document_url": "https://vectorgov.io/api/v1/evidence/download/source/IN-58-2022"
}
```

**Uso em pipeline com agentes LLM**:

```bash
# Busca + extrai evidências pra montar citações manualmente
vectorgov search --raw "dispensa de licitação" | jq -r '.hits[] | "- \(.document_id), Art. \(.article_number) → \(.evidence_url)"'
```

---

## Resumo de Comandos

| Comando | Descrição |
|---------|-----------|
| `search` | Busca semântica em legislação (filtros: `--tipo`, `--ano`, `--doc`) |
| `smart-search` | Busca inteligente MOC v4 com análise de confiança |
| `hybrid` | Busca semântica + expansão por grafo normativo |
| `lookup` | Consulta de artigo específico por referência legal |
| `grep` | Busca exata por texto no corpo das normas |
| `merged` | Busca dual-path: semântica + índice curado (RRF) |
| `fs-search` | Busca no índice curado — texto exato |
| `read` | Lê texto canônico de documento/dispositivo |
| `explain` | Contexto completo de um dispositivo (lookup + texto consolidado) |
| `context` | Bloco completo (busca + prompt) pronto para LLMs |
| `tokens` | Estimativa de tokens antes de usar LLM |
| `prompts list/show` | System prompts disponíveis para LLMs |
| `docs list/info` | Lista e detalha documentos disponíveis |
| `audit logs/stats` | Histórico e estatísticas de uso da API |
| `quota` | Consulta de uso do plano (smart_search + créditos) |
| `feedback send` | Envia like/dislike para uma busca |
| `auth login/status/logout` | Autenticação |
| `config set/get/list/delete` | Configurações |
| `init` | Inicializa projeto com arquivos para ferramentas AI |

## Instalação

```bash
pip install vectorgov-cli
```

## Configuração

```bash
# Configure sua API key
vectorgov auth login

# Ou via variável de ambiente
export VECTORGOV_API_KEY="vg_sua_chave"
```

## Uso

### Busca

```bash
# Busca simples
vectorgov search "O que é ETP?"

# Com opções
vectorgov search "pesquisa de preços" --top-k 10 --mode precise

# Com filtros (v0.2.1)
vectorgov search "dispensa" --tipo LEI --ano 2021
vectorgov search "art. 75" --doc LEI-14133-2021
vectorgov search "licitação" --tipo IN --ano 2022 --top-k 15

# Saída em JSON
vectorgov search "licitação" --output json

# JSON bruto (para pipes)
vectorgov search "licitação" --raw | jq '.hits[0].text'
```

**Opções**:
- `--top-k/-k` (1-20, padrão: 5) — quantidade de resultados
- `--mode/-m` (fast/balanced/precise) — modo de busca
- `--tipo/-t` (LEI, DECRETO, IN, PORTARIA, AC) — filtro por tipo (auto-uppercase)
- `--ano/-a` (ex: 2021) — filtro por ano
- `--doc/-d` (ex: LEI-14133-2021) — filtro por document_id específico
- `--cache` — usar cache semântico
- `--output/-o` (table/json/text/markdown)
- `--raw` — saída JSON bruto para piping

### Contexto para LLMs

Use o comando `context` para gerar um bloco completo (busca + system prompt) pronto para colar em qualquer LLM (OpenAI, Anthropic, Google, etc).

```bash
# Bloco de contexto em texto puro
vectorgov context "O que é ETP?"

# Formato messages (OpenAI-compatible)
vectorgov context "critérios de julgamento" --format messages
```

**Exemplo de integração com OpenAI:**

```python
from vectorgov import VectorGov
from openai import OpenAI

vg = VectorGov(api_key="vg_xxx")
openai = OpenAI()

results = vg.search("O que é ETP?", top_k=5)

response = openai.chat.completions.create(
    model="gpt-4o",
    messages=results.to_messages("O que é ETP?")
)
print(response.choices[0].message.content)
```

### Feedback

```bash
# Apos uma busca, use o query_id para feedback
vectorgov feedback send abc123def456 --like
vectorgov feedback send abc123def456 --dislike
```

### Estimativa de Tokens

Estima quantos tokens uma busca consumiria, util para planejar uso com LLMs.

```bash
# Estimativa basica
vectorgov tokens "O que e ETP?"

# Com mais resultados
vectorgov tokens "pesquisa de precos" --top-k 10

# Saida em JSON
vectorgov tokens "licitacao" --output json
```

Exemplo de saida:

```
Estimativa de Tokens
+-----------------+------------+---------------------------+
| Componente      |     Tokens | Descricao                 |
+-----------------+------------+---------------------------+
| Contexto        |      1,234 | 5 hits da busca           |
| System Prompt   |        200 | Instrucoes do sistema     |
| Query           |          5 | Pergunta do usuario       |
|-----------------|------------|---------------------------|
| Total           |      1,439 | 5,432 caracteres          |
+-----------------+------------+---------------------------+

Comparacao com limites de modelos:
  GPT-4o: OK 1.1% (1,439/128,000)
  GPT-4o-mini: OK 1.1% (1,439/128,000)
  Claude 3.5 Sonnet: OK 0.7% (1,439/200,000)
  Gemini 2.0 Flash: OK 0.1% (1,439/1,000,000)
```

### Documentos

```bash
# Lista documentos disponíveis
vectorgov docs list

# Paginação (v0.2.1)
vectorgov docs list --page 1 --limit 20
vectorgov docs list --page 2 --limit 20

# Saída em JSON
vectorgov docs list --output json

# Informações de um documento
vectorgov docs info LEI-14133-2021
vectorgov docs info IN-65-2021
```

**Opções `docs list`**:
- `--page/-p` (padrão: 1) — página a exibir
- `--limit/-l` (1-100, padrão: 50) — itens por página
- `--output/-o` (table/json)

### Smart Search

Busca inteligente MOC v4 com análise de completude e nível de confiança (ALTO/MEDIO/BAIXO).

```bash
# Busca inteligente
vectorgov smart-search "Quando o ETP pode ser dispensado?"

# Saída em JSON
vectorgov smart-search "critérios de julgamento" --output json

# Com cache
vectorgov smart-search "pesquisa de preços" --cache
```

### Hybrid

Busca semântica enriquecida com expansão por grafo normativo. Retorna
evidências diretas + artigos citados (expansão via grafo).

```bash
# Busca híbrida
vectorgov hybrid "Critérios de julgamento em licitações"

# Com mais hops e resultados
vectorgov hybrid "Dispensa de licitação" --hops 2 --top-k 15

# JSON bruto com graph_nodes e stats
vectorgov hybrid "licitação" --raw | jq '.graph_nodes, .stats'
```

**Opções**:
- `--top-k/-k` (1-50, padrão: 10) — quantidade de resultados diretos
- `--hops` (1 ou 2, padrão: 1) — saltos no grafo normativo
- `--graph-expansion` (bidirectional/forward) — tipo de expansão
- `--token-budget` — limite de tokens do contexto expandido
- `--output/-o` (table/json/text)
- `--raw` — inclui `graph_nodes` e `stats` no output

**Comportamento v0.2.0**: quando a busca vetorial não retorna seeds relevantes
(ex: termos fracos no reranker), o CLI usa automaticamente `graph_nodes` como
fallback para popular `hits` — garante que queries legítimas como "Dispensa de
licitação" retornem algo.

### Lookup

Consulta de dispositivo legal por referência em linguagem natural. Resolve
referências como "Art. 75 da Lei 14.133", "§ 1º do Art. 33", "Inciso I do § 2
do Art. 4", etc. Quando o dispositivo é um **artigo**, o retorno já vem com o
texto completo consolidado (caput + incisos + parágrafos + alíneas).

```bash
# Referência completa (inclua o nome da lei na própria referência)
vectorgov lookup "Art. 75 da Lei 14.133"
vectorgov lookup "§ 1º do Art. 33 da Lei 14.133/2021"
vectorgov lookup "Inciso I do § 2 do Art. 4 da IN 67/2021"

# Controle de parent e siblings
vectorgov lookup --no-parent "Art. 1 da Lei 14.133"
vectorgov lookup --no-siblings "Art. 75 da Lei 14.133"
```

**Opções**:
- `--parent/--no-parent` (padrão: --parent) — incluir dispositivo pai
- `--siblings/--no-siblings` (padrão: --siblings) — incluir dispositivos irmãos
- `--output/-o` (text/json/llm) — formato de saída
- `--raw` — JSON bruto sem formatação (para pipes)
- `--pipe` — lê referências de stdin (uma por linha, batch)

#### Formatos de saída

O `lookup` suporta 4 formatos de saída para atender usos diferentes:

```bash
# 1. text (padrão) — Panels Rich com bordas e cores
vectorgov lookup "Art. 11 da Lei 14.133"

# 2. json — JSON estruturado com syntax highlight no console
vectorgov lookup -o json "Art. 11 da Lei 14.133"

# 3. llm — Texto puro otimizado para colar em LLMs (sem ANSI/Rich)
vectorgov lookup -o llm "Art. 11 da Lei 14.133"

# 4. raw — JSON bruto para pipes e scripts
vectorgov lookup --raw "Art. 11 da Lei 14.133" | jq '.match.text'
```

Todos os formatos incluem, quando disponível:
- Texto do dispositivo (consolidado para artigos)
- `evidence_url` — link para o trecho destacado na norma
- `document_url` — link para download do PDF original
- `nota_especialista` — comentário do especialista jurídico (curadoria SPEC 1C)
- `jurisprudencia_tcu` — jurisprudência relacionada (quando presente)

No formato **text**, nota e jurisprudência aparecem em Panels dedicados
(amarelo e magenta) abaixo dos hits. No **llm**, aparecem em seções com
separador `---`. No **json** e **raw**, são campos top-level.

#### Batch lookup (múltiplas referências)

O lookup suporta batch de 2 formas:

```bash
# 1. Auto-split: múltiplas refs separadas por vírgula, ";" ou " e "
vectorgov lookup "Art. 75 da Lei 14.133 e Art. 18 da Lei 14.133"
vectorgov lookup "inc.I do § 2 do art. 4 da IN 67/2021, inc.II do § 2 do art. 4 da IN 67/2021"

# 2. Via stdin com --pipe (uma ref por linha, até 20)
printf "Art. 75 da Lei 14.133\nArt. 33 da Lei 14.133" | vectorgov lookup --pipe

# Batch em formato llm (bom para colar em LLM)
vectorgov lookup -o llm "Art. 11, Art. 18 e Art. 75 da Lei 14.133"

# Batch em raw JSON para processar com jq
vectorgov lookup --raw "Art. 11, Art. 18 da Lei 14.133" | jq '.results[].evidence_url'
```

**Nota**: o flag `--doc` foi removido na v0.2.1. O SDK `vg.lookup()` não aceita
`document_id` separado — para filtrar por documento, inclua o nome na própria
referência (ex: `"Art. 75 da Lei 14.133"` em vez de `"Art. 75" --doc ...`).

### Grep

Busca exata por texto no corpo das normas. Diferente do `search` (semântico), o `grep` procura ocorrências literais.

```bash
# Busca textual exata
vectorgov grep "dispensa de licitação"

# Filtrado por documento
vectorgov grep "ETP" --doc LEI-14133-2021

# Controle de quantidade e contexto (v0.2.0)
vectorgov grep "licitação" --max 10 --context-lines 5
vectorgov grep "art. 75" --doc LEI-14133-2021 --max 3
```

**Opções**:
- `--doc/-d` — filtrar por documento específico
- `--max/-n` (1-50, padrão: 20) — máximo de resultados
- `--context-lines/-C` (0-10, padrão: 3) — linhas de contexto ao redor do match
- `--output/-o` (table/json/text)
- `--raw` — JSON bruto

### Merged

Busca dual-path combinando busca semântica + índice curado
com Reciprocal Rank Fusion (RRF).

```bash
# Busca merged (ambos backends ativos por padrão)
vectorgov merged "Modalidades de licitação"
vectorgov merged "Pesquisa de preços" --top-k 15

# Filtro por documento (v0.2.1)
vectorgov merged "art. 75" --doc LEI-14133-2021

# Controle granular de backends (v0.2.1)
vectorgov merged "dispensa" --no-filesystem    # apenas busca semântica
vectorgov merged "licitação" --no-hybrid       # apenas índice curado
```

**Opções**:
- `--top-k/-k` (1-50, padrão: 10) — quantidade de resultados
- `--doc/-d` — filtrar por documento específico
- `--token-budget` — limite de tokens para contexto
- `--no-hybrid` — desabilita busca semântica (usa só índice curado)
- `--no-filesystem` — desabilita índice curado (usa só busca semântica)
- `--output/-o` (table/json/text)
- `--raw` — inclui `mutual_count`, `hybrid_count`, `filesystem_count` no output

### Read

Lê o texto canônico completo de um documento ou dispositivo específico (v0.2.1).

```bash
# Documento inteiro
vectorgov read LEI-14133-2021

# Apenas um artigo específico
vectorgov read LEI-14133-2021 --span ART-075
vectorgov read LEI-14133-2021 --span PAR-033-1

# JSON bruto
vectorgov read LEI-14133-2021 --span ART-075 --raw | jq .text
```

**Opções**:
- `--span/-s` (ex: ART-075, PAR-033-1, INC-005-I) — dispositivo específico
- `--output/-o` (text/json)
- `--raw` — JSON bruto

**Diferença para `lookup`**: `lookup` resolve referências em linguagem natural
("Art. 75 da Lei 14.133"); `read` usa IDs canônicos diretamente. Use `read`
quando já souber o `document_id` e `span_id` exatos (ex: após um `lookup`).

### Fs-search

Busca no índice curado — alternativa ao `search` vetorial.
Ideal para termos exatos e referências legais precisas (v0.2.0).

```bash
# Busca simples
vectorgov fs-search "art. 75 da Lei 14.133"
vectorgov fs-search "pregão eletrônico"

# Modo específico
vectorgov fs-search "dispensa" --mode index    # só busca indexada
vectorgov fs-search "art. 75" --mode grep      # só busca textual
vectorgov fs-search "ETP" --mode both          # índice + textual combinados

# Filtrar por documento
vectorgov fs-search "art. 75" --doc LEI-14133-2021 --output json
```

**Opções**:
- `--doc/-d` — filtrar por documento
- `--top-k/-k` (1-50, padrão: 10) — quantidade de resultados
- `--mode/-m` (auto/index/grep/both, padrão: auto) — estratégia de busca
- `--output/-o` (table/json/text)
- `--raw` — JSON bruto

**Diferença para `search`**: `search` é busca vetorial (embeddings), `fs-search`
é busca textual no índice curado. Use `fs-search` quando souber o termo exato
ou quer citar uma norma por referência literal.

### Context

Gera bloco completo (busca + system prompt) pronto para colar em LLMs. O comando principal para vibe coding.

```bash
# Bloco de contexto em texto puro (padrão)
vectorgov context "Quando posso usar dispensa de licitação?"

# Formato messages (JSON OpenAI-compatible)
vectorgov context "ETP" --format messages

# Com busca inteligente (smart-search)
vectorgov context "pregão eletrônico" --smart

# Com system prompt específico
vectorgov context "pesquisa de preços" --prompt detailed
```

Formatos: `raw` (padrão, texto puro), `messages` (JSON OpenAI), `clipboard`

### Audit

Histórico de requisições e estatísticas de uso da API.

```bash
# Logs de requisições (últimos 30 dias)
vectorgov audit logs --days 30 --limit 50

# Estatísticas agregadas (últimos 7 dias)
vectorgov audit stats --days 7
```

### Quota

Consulta de uso do plano: cotas de smart_search e créditos restantes.

```bash
# Ver quota do plano atual
vectorgov quota

# Saída em JSON
vectorgov quota --output json
```

### Prompts

System prompts disponíveis para uso com LLMs.

```bash
# Lista prompts disponíveis
vectorgov prompts list

# Exibe um prompt completo
vectorgov prompts show juridico --raw
```

### Init

Inicializa projeto com arquivos de configuração para ferramentas AI (Claude Code, Cursor, Codex).

```bash
# Tudo de uma vez
vectorgov init --all

# Apenas CLAUDE.md
vectorgov init --claude

# Apenas .cursorrules
vectorgov init --cursor

# Apenas AGENTS.md
vectorgov init --codex
```

### Configuração

```bash
# Ver configuração atual
vectorgov config list

# Definir configuração
vectorgov config set default_mode precise
vectorgov config set default_top_k 10

# Ver valor específico
vectorgov config get api_key

# Remover configuração
vectorgov config delete default_mode
```

### Autenticação

```bash
# Login (salva API key)
vectorgov auth login

# Status da autenticação
vectorgov auth status

# Logout (remove API key)
vectorgov auth logout
```

## Formatos de Saída

Todos os comandos de busca suportam: `--output table` (padrão), `--output json`,
`--output text`, `--output llm` (v0.2.3) e `--raw` (JSON bruto para pipes).

### LLM (otimizado para IAs, v0.2.3)

```bash
vectorgov search "O que é ETP?" --output llm
```

```
[1/5] IN-58-2022, Art. 3 (score: 0.987)
I - Estudo Técnico Preliminar - ETP: documento constitutivo...
EVIDENCE: https://vectorgov.io/api/v1/evidence/IN-58-2022%23INC-003-I
PDF: https://vectorgov.io/api/v1/evidence/download/source/IN-58-2022
---
[2/5] IN-65-2021, Art. 1 (score: 0.856)
Esta Instrução Normativa dispõe sobre a elaboração...
```

Texto puro, sem escapes ANSI, sem JSON. Economiza ~40% de tokens vs `--raw`.

### Tabela (padrão para search)

```bash
vectorgov search "O que é ETP?" --output table
```

```
Resultados para: O que é ETP?
Total: 5 | Latência: 1234ms | Cache: Não

┏━━━┳━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━┓
┃ # ┃ Artigo    ┃ Texto                                                          ┃ Score   ┃
┡━━━╇━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━┩
│ 1 │ Art. 3    │ ETP - Estudo Técnico Preliminar: documento constitutivo...     │ 0.892   │
│ 2 │ Art. 1    │ Esta Instrução Normativa dispõe sobre a elaboração...          │ 0.856   │
└───┴───────────┴────────────────────────────────────────────────────────────────┴─────────┘
```

### JSON

```bash
vectorgov search "O que é ETP?" --output json
```

```json
{
  "query": "O que é ETP?",
  "total": 5,
  "cached": false,
  "latency_ms": 1234,
  "hits": [
    {
      "text": "ETP - Estudo Técnico Preliminar...",
      "article_number": "3",
      "score": 0.892
    }
  ]
}
```

## Integração com Outros Comandos

```bash
# Buscar e processar com jq
vectorgov search "ETP" --raw | jq '.hits[0].text'

# Buscar e salvar
vectorgov search "licitação" --output json > resultados.json

# Usar em scripts
QUERY_ID=$(vectorgov search "ETP" --raw | jq -r '.query_id')
vectorgov feedback send $QUERY_ID --like

# Contexto completo para colar no ChatGPT/Claude
vectorgov context "dispensa de licitação" --format raw

# Lookup + pipe
vectorgov lookup "Art. 75 da Lei 14.133" --raw | jq '.text'

# Grep exato em documento específico
vectorgov grep "pregão eletrônico" --doc LEI-14.133-2021 --output json

# Hybrid com expansão por grafo
vectorgov hybrid "critérios de julgamento" --hops 2 --raw | jq '.cited_expansion'
```

## Variáveis de Ambiente

| Variável | Descrição |
|----------|-----------|
| `VECTORGOV_API_KEY` | API key para autenticação |
| `VECTORGOV_OUTPUT` | Formato de output padrão: `llm`, `table`, `json`, `text` (v0.2.3) |
| `VECTORGOV_DEFAULT_MODE` | Modo de busca padrão (fast, balanced, precise) |
| `VECTORGOV_DEFAULT_TOP_K` | Número padrão de resultados |

## Arquivo de Configuração

Localização: `~/.vectorgov/config.yaml`

```yaml
api_key: vg_sua_chave
default_mode: balanced
default_top_k: 5
default_output: table       # ou llm, json, text (v0.2.3)
```

## Ajuda

```bash
# Ajuda geral
vectorgov --help

# Ajuda de comando específico
vectorgov search --help
vectorgov context --help

# Versão do CLI
vectorgov --version    # ou -V
vectorgov version      # subcomando (alternativa)
```

## Links

- [Documentação](https://vectorgov.io/documentacao)
- [Playground](https://vectorgov.io/playground)
- [SDK Python](https://pypi.org/project/vectorgov/)
