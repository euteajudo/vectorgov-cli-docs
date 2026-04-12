# MAPA DO CLI VECTORGOV

> **Versão**: 0.2.3
> **Data**: Abril 2026
> **Objetivo**: Documentação completa da arquitetura e funcionamento do CLI VectorGov

---

## 📋 Índice

1. [Visão Geral](#visão-geral)
2. [Arquitetura de Alto Nível](#arquitetura-de-alto-nível)
3. [Estrutura de Arquivos](#estrutura-de-arquivos)
4. [Comandos](#comandos)
5. [Fluxo de Dados](#fluxo-de-dados)
6. [Configurações](#configurações)
7. [Integração com SDK](#integração-com-sdk)
8. [Exemplos de Uso](#exemplos-de-uso)
9. [Links Úteis](#links-úteis)

---

## 📖 Visão Geral

O VectorGov CLI é uma ferramenta de linha de comando para interagir com a API VectorGov diretamente do terminal. Ideal para scripts, automações, pipes Unix e integração com outras ferramentas.

### Características Principais

| Característica | Descrição |
|----------------|-----------|
| **Baseado no SDK** | Usa o SDK Python internamente para chamadas à API |
| **Múltiplos Formatos** | Saída em tabela, JSON ou raw para pipes |
| **Autocomplete** | Suporte a shell completion (bash, zsh) |
| **Configuração Flexível** | Via variáveis de ambiente, arquivo YAML ou flags |
| **Windows/Linux/Mac** | Compatível com todos os sistemas operacionais |

### Quando Usar o CLI vs SDK

| Cenário | Recomendação |
|---------|--------------|
| Scripts shell/bash | **CLI** |
| Automações com pipes | **CLI** |
| Testes rápidos no terminal | **CLI** |
| Aplicações Python | **SDK** |
| Integração com LLMs | **SDK** |
| Aplicações web/API | **SDK** |

---

## 🏗️ Arquitetura de Alto Nível

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ARQUITETURA DO CLI                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Terminal                    CLI (21 comandos)           SDK               │
│   ┌─────────┐               ┌─────────┐                ┌─────────┐         │
│   │$ vectorgov│─────────────▶│  Typer  │───────────────▶│VectorGov│         │
│   │  search  │              │ (main.py)│               │(client) │         │
│   │  hybrid  │              │         │               │         │         │
│   │  context │              │         │               │         │         │
│   │  ...     │              │         │               │         │         │
│   └─────────┘               └────┬────┘               └────┬────┘         │
│                                  │                         │               │
│                                  │                         │ HTTPS         │
│                                  │                         ▼               │
│   ┌─────────┐               ┌────▼────┐          ┌─────────────────┐       │
│   │ stdout  │◀──────────────│ Output  │          │   API VectorGov │       │
│   │ (table/ │               │Formatter│          │vectorgov.io/api │       │
│   │  json)  │               │         │          │                 │       │
│   └─────────┘               └─────────┘          └─────────────────┘       │
│                                                                             │
│   ┌─────────┐                                                               │
│   │ Arquivos│◀─── init (CLAUDE.md, .cursorrules, AGENTS.md)                 │
│   │ locais  │                                                               │
│   └─────────┘                                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📁 Estrutura de Arquivos

```
vectorgov-cli/
├── src/vectorgov/cli/
│   ├── __init__.py              # Versão e metadata
│   ├── main.py                  # Entry point Typer (registro de comandos)
│   ├── commands/
│   │   ├── ask.py               # Contexto para LLMs
│   │   ├── audit.py             # Logs e estatísticas de uso
│   │   ├── auth.py              # Login, logout, status
│   │   ├── config.py            # Gerenciamento de configurações
│   │   ├── context.py           # Bloco completo para LLMs
│   │   ├── docs.py              # Listar/detalhar documentos
│   │   ├── explain.py           # Contexto completo de dispositivo (v0.2.3)
│   │   ├── feedback.py          # Enviar like/dislike
│   │   ├── fs_search.py         # Busca no índice curado
│   │   ├── grep_cmd.py          # Busca textual exata
│   │   ├── hybrid.py            # Busca + grafo normativo
│   │   ├── init.py              # Inicializar projeto AI
│   │   ├── lookup.py            # Consulta artigo por referência
│   │   ├── merged.py            # Dual-path semântica + curado
│   │   ├── prompts.py           # System prompts para LLMs
│   │   ├── quota.py             # Uso do plano
│   │   ├── read.py              # Texto canônico de documento (v0.2.1)
│   │   ├── search.py            # Busca semântica
│   │   ├── smart_search.py      # Busca inteligente MOC v4
│   │   └── tokens.py            # Estimativa de tokens
│   └── utils/
│       ├── config.py            # ConfigManager (YAML + env vars)
│       └── output.py            # OutputFormatter (tabela, JSON, raw, llm)
├── pyproject.toml               # Configuração do pacote
├── README.md                    # Documentação de uso
├── CHANGELOG.md                 # Histórico de versões
└── LICENSE                      # MIT License
```

### Arquivos Principais

| Arquivo | Responsabilidade |
|---------|------------------|
| `main.py` | Registro de todos os 21 comandos Typer |
| `commands/*.py` | Implementação de cada comando (1 arquivo por comando) |
| `utils/config.py` | Gerenciamento de configuração (arquivo + env) |
| `utils/output.py` | Formatação de saída (tabela Rich, JSON, raw, llm) |

---

## 🔧 Comandos

### Comandos de Busca

| Comando | Descrição | Exemplo | Flags principais |
|---------|-----------|---------|------------------|
| `search` | Busca semântica (filtros client-side) | `vectorgov search "dispensa" --tipo LEI --ano 2021` | `--top-k`, `--mode`, `--tipo`, `--ano`, `--doc`, `--cache` |
| `smart-search` | Busca inteligente MOC v4 com confiança | `vectorgov smart-search "Quando o ETP pode ser dispensado?"` | `--cache`, `--output` |
| `hybrid` | Busca semântica + grafo normativo | `vectorgov hybrid "Critérios de julgamento" --hops 2` | `--top-k`, `--hops`, `--graph-expansion`, `--token-budget` |
| `lookup` | Consulta artigo por referência legal | `vectorgov lookup "Art. 75 da Lei 14.133"` | `--parent/--no-parent`, `--siblings/--no-siblings` |
| `grep` | Busca exata por texto | `vectorgov grep "dispensa" --max 10 --context-lines 5` | `--doc`, `--max/-n`, `--context-lines/-C` |
| `merged` | Busca dual-path (semântica + curado, RRF) | `vectorgov merged "licitação" --no-filesystem` | `--top-k`, `--doc`, `--no-hybrid`, `--no-filesystem`, `--token-budget` |
| `fs-search` | Busca no índice curado | `vectorgov fs-search "art. 75 da Lei 14.133" --mode grep` | `--doc`, `--top-k`, `--mode` (auto/index/grep/both) |
| `read` | Lê texto canônico de documento/dispositivo | `vectorgov read LEI-14133-2021 --span ART-075` | `--span/-s` |
| `explain` | Contexto completo de dispositivo (lookup + texto consolidado) | `vectorgov explain "Art. 75 da Lei 14.133" --output llm` | `--output` (default: llm) |

### Comandos de Contexto para LLMs

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `ask` | Contexto para LLMs | `vectorgov ask "Quando o ETP pode ser dispensado?"` |
| `context` | Bloco completo (busca + prompt) para LLMs | `vectorgov context "dispensa de licitação" --format messages` |
| `tokens` | Estimativa de tokens | `vectorgov tokens "pesquisa de preços" --top-k 10` |
| `prompts list` | Lista system prompts disponíveis | `vectorgov prompts list` |
| `prompts show` | Exibe system prompt completo | `vectorgov prompts show juridico --raw` |

### Comandos de Feedback

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `feedback send` | Envia like/dislike | `vectorgov feedback send <query_id> --like` |

### Comandos de Documentos

| Comando | Descrição | Exemplo | Flags |
|---------|-----------|---------|-------|
| `docs list` | Lista documentos (paginado, v0.2.1) | `vectorgov docs list --page 2 --limit 20` | `--page/-p`, `--limit/-l` (1-100), `--output` |
| `docs info` | Detalhes de documento | `vectorgov docs info LEI-14133-2021` | (sem flags) |

### Comandos de Uso e Auditoria

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `audit logs` | Histórico de requisições à API | `vectorgov audit logs --days 30 --limit 50` |
| `audit stats` | Estatísticas agregadas de uso | `vectorgov audit stats --days 7` |
| `quota` | Uso do plano (smart_search + créditos) | `vectorgov quota --output json` |

### Comandos de Autenticação

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `auth login` | Configura API key | `vectorgov auth login` |
| `auth status` | Verifica autenticação | `vectorgov auth status` |
| `auth logout` | Remove API key | `vectorgov auth logout` |

### Comandos de Configuração

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `config list` | Lista configurações | `vectorgov config list` |
| `config get` | Obtém valor | `vectorgov config get api_key` |
| `config set` | Define valor | `vectorgov config set default_top_k 10` |
| `config delete` | Remove valor | `vectorgov config delete default_mode` |

### Comandos de Inicialização

| Comando | Descrição | Exemplo |
|---------|-----------|---------|
| `init --all` | Gera todos os arquivos AI | `vectorgov init --all` |
| `init --claude` | Gera CLAUDE.md | `vectorgov init --claude` |
| `init --cursor` | Gera .cursorrules | `vectorgov init --cursor` |
| `init --codex` | Gera AGENTS.md | `vectorgov init --codex` |

---

## 🔄 Fluxo de Dados

### Busca Simples

```
1. Usuário executa: vectorgov search "O que é ETP?"
2. main.py:search() parseia argumentos
3. ConfigManager carrega API key (env/arquivo)
4. VectorGov SDK faz chamada HTTP à API
5. API retorna resultados JSON
6. OutputFormatter formata para tabela/JSON
7. Resultado é impresso no stdout
```

### Busca Híbrida (Grafo)

```
1. Usuário executa: vectorgov hybrid "critérios de julgamento" --hops 2
2. main.py:hybrid() parseia argumentos
3. VectorGov SDK faz busca semântica + expansão por grafo normativo
4. API retorna evidence_direct + cited_expansion
5. OutputFormatter exibe evidências diretas e artigos citados via grafo
```

### Context (Bloco LLM)

```
1. Usuário executa: vectorgov context "dispensa de licitação" --format messages
2. main.py:context() parseia argumentos
3. SDK busca resultados (search ou smart-search se --smart)
4. SDK busca system prompt (prompts show)
5. Combina busca + prompt em bloco formatado
6. Retorna em formato raw, messages (OpenAI JSON) ou clipboard
```

### Integração com Pipes Unix

```bash
# Busca → jq → processamento
vectorgov search "ETP" --raw | jq '.hits[0].text'

# Busca → salvar arquivo
vectorgov search "licitação" --output json > resultados.json

# Busca → obter query_id → feedback
QUERY_ID=$(vectorgov search "ETP" --raw | jq -r '.query_id')
vectorgov feedback send $QUERY_ID --like

# Contexto completo para pipe em LLM
vectorgov context "ETP" --format messages --raw | jq '.messages'

# Hybrid → extrair artigos citados via grafo
vectorgov hybrid "critérios" --raw | jq '.cited_expansion[].span_id'
```

---

## ⚙️ Configurações

### Prioridade de Configuração

```
1. Flags de linha de comando (maior prioridade)
2. Variáveis de ambiente
3. Arquivo ~/.vectorgov/config.yaml
4. Valores padrão (menor prioridade)
```

### Variáveis de Ambiente

| Variável | Descrição | Exemplo |
|----------|-----------|---------|
| `VECTORGOV_API_KEY` | API key | `vg_xxx...` |
| `VECTORGOV_OUTPUT` | Formato de output padrão (v0.2.3) | `llm`, `table`, `json`, `text` |
| `VECTORGOV_DEFAULT_MODE` | Modo de busca | `fast`, `balanced`, `precise` |
| `VECTORGOV_DEFAULT_TOP_K` | Resultados padrão | `5` |

### Arquivo de Configuração

**Localização**: `~/.vectorgov/config.yaml`

```yaml
api_key: vg_sua_chave
default_mode: balanced
default_top_k: 5
default_output: table       # ou llm, json, text
```

---

## 🔗 Integração com SDK

O CLI usa o SDK Python internamente:

```python
# Internamente no CLI (main.py)
from vectorgov import VectorGov

vg = VectorGov(api_key=config.api_key)
results = vg.search(query, top_k=top_k, mode=mode)
```

### Dependências

```toml
# pyproject.toml
dependencies = [
    "vectorgov>=0.10.0",  # SDK Python
    "typer>=0.9.0",       # Framework CLI
    "rich>=13.0.0",       # Tabelas e formatação
    "pyyaml>=6.0",        # Arquivo de configuração
]
```

---

## 📚 Exemplos de Uso

### Busca Básica

```bash
# Busca simples
vectorgov search "O que é ETP?"

# Com mais resultados
vectorgov search "pesquisa de preços" --top-k 10

# Modo preciso (mais lento, mais acurado)
vectorgov search "licitação" --mode precise
```

### Busca Inteligente (Smart Search)

```bash
# Retorna confiança ALTO/MEDIO/BAIXO + raciocínio jurídico
vectorgov smart-search "Quando o ETP pode ser dispensado?"

# Com cache (respostas frequentes)
vectorgov smart-search "pesquisa de preços" --cache
```

### Busca Híbrida (Grafo)

```bash
# Semântica + expansão por citações normativas
vectorgov hybrid "Critérios de julgamento em licitações"

# Com 2 hops de profundidade no grafo
vectorgov hybrid "Dispensa de licitação" --hops 2 --top-k 15
```

### Lookup, Explain e Grep

```bash
# Consulta direta de artigo (inclua o nome da lei na referência)
vectorgov lookup "Art. 75 da Lei 14.133"
vectorgov lookup "Art. 24, § 2º da Lei 14.133/2021"

# Contexto completo de um artigo (lookup + texto consolidado)
vectorgov explain "Art. 75 da Lei 14.133" --output llm

# Busca textual exata
vectorgov grep "dispensa de licitação"
vectorgov grep "ETP" --doc LEI-14.133-2021
```

### Contexto para LLMs (vibe coding)

```bash
# Bloco completo pronto para colar no ChatGPT/Claude
vectorgov context "Quando posso usar dispensa de licitação?"

# Formato OpenAI messages (para uso programático)
vectorgov context "ETP" --format messages

# Com busca inteligente + prompt específico
vectorgov context "pesquisa de preços" --smart --prompt detailed
```

### Saída JSON para Scripts

```bash
# JSON formatado
vectorgov search "ETP" --output json

# JSON raw (para pipes)
vectorgov search "ETP" --raw | jq '.hits | length'
```

### Contexto com ask

```bash
# Obtém contexto e mostra código de exemplo
vectorgov ask "Quando o ETP pode ser dispensado?" --code

# Saída em formato messages (pronto para LLM)
vectorgov ask "critérios de julgamento" --output json
```

### Estimativa de Tokens

```bash
# Estima tokens antes de usar LLM
vectorgov tokens "O que é ETP?" --top-k 5

# Comparação com limites de modelos
vectorgov tokens "pesquisa de preços" --output json
```

### Auditoria e Quota

```bash
# Logs de uso dos últimos 7 dias
vectorgov audit logs --days 7

# Estatísticas agregadas
vectorgov audit stats --days 30

# Ver quota do plano
vectorgov quota
```

### Inicialização de Projeto

```bash
# Gera todos os arquivos para ferramentas AI
vectorgov init --all

# Apenas para Claude Code
vectorgov init --claude
```

---

## 🔗 Links Úteis

| Recurso | URL |
|---------|-----|
| **PyPI** | https://pypi.org/project/vectorgov-cli/ |
| **Documentação** | https://vectorgov.io/documentacao/cli-instalacao |
| **SDK Python** | https://pypi.org/project/vectorgov/ |
| **API Reference** | https://vectorgov.io/documentacao |
| **Playground** | https://vectorgov.io/playground |

---

## 📝 Histórico de Versões

| Versão | Data | Mudanças |
|--------|------|----------|
| 0.2.3 | 12/04/2026 | Features otimizadas para IAs: `--output llm` (texto puro sem ANSI/JSON), `VECTORGOV_OUTPUT` env var + config `default_output`, comando `explain` (lookup+texto consolidado em 1 chamada), `--pipe` em lookup (batch stdin). 21 comandos (era 19). |
| 0.2.2 | 11/04/2026 | Links de evidência expostos em todos os outputs (raw, json, table, text, markdown) dos 7 comandos de busca. Novos helpers em `utils/output.py`: `absolute_url`, `get_evidence_links`, `hit_to_raw_dict`, `render_evidence_lines_text`, `render_evidence_cell_table`, `render_evidence_markdown`. Workaround para bug do SDK v0.16 em `lookup` (campos ausentes no `LookupResult` mas presentes em `_raw_response`). |
| 0.2.1 | 11/04/2026 | Paridade 10/10 SDK × CLI: novo `read` (canonical reader), filtros `--tipo/--ano/--doc` em `search`, flags granulares `--no-hybrid/--no-filesystem` em `merged`, paginação `--page/--limit` em `docs list`, remoção do flag morto `--doc` de `lookup` |
| 0.2.0 | 11/04/2026 | 11 novos comandos + fixes críticos: smart-search, hybrid, lookup, grep, merged, **fs-search**, context, audit, quota, prompts, init + correções em docs/hybrid/lookup + flags `--parent/--siblings` em lookup + `--max/--context-lines` em grep |
| 0.1.4 | 20/01/2025 | Comando `feedback send` para evitar conflito Typer |
| 0.1.3 | 20/01/2025 | Correção caracteres box-drawing no Windows |
| 0.1.2 | 20/01/2025 | Correção Unicode no Windows (cp1252) |
| 0.1.1 | 20/01/2025 | Comando `tokens` para estimativa |
| 0.1.0 | 20/01/2025 | Primeira versão pública |

---

*Este documento é atualizado a cada release do CLI.*
