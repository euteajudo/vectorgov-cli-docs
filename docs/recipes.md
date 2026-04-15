# Receitas — VectorGov CLI

Fluxos completos por caso de uso. Cada receita é copy-pasteable e funciona com instalação padrão (`pip install vectorgov-cli` + `vectorgov auth login`).

---

## 🤖 Vibe coding com LLMs

### Receita 1: Bloco pronto para colar em ChatGPT/Claude

```bash
vectorgov context "Quando dispensar licitação?"
```

Output cola direto na conversa do LLM. Inclui contexto + system prompt jurídico otimizado.

### Receita 2: Format messages (OpenAI-compatible) via Python

```bash
# Gera as messages no terminal
vectorgov context "ETP" --format messages
```

Ou direto em Python (preferível para apps):

```python
from vectorgov import VectorGov
from openai import OpenAI

vg = VectorGov()  # lê VECTORGOV_API_KEY
result = vg.search("ETP")

response = OpenAI().chat.completions.create(
    model="gpt-4o-mini",
    messages=result.to_messages(query="ETP"),
)
print(response.choices[0].message.content)
```

### Receita 3: Inicializar projeto AI

```bash
vectorgov init --all
# cria CLAUDE.md, .cursorrules, AGENTS.md
```

Os 3 arquivos contêm instruções para agentes (Claude Code, Cursor, Codex) usarem o VectorGov corretamente.

### Receita 4: Estimar tokens antes de mandar para LLM

```bash
vectorgov tokens "Quando dispensar licitação?" --top-k 10
```

Mostra quantos tokens cada resultado consumiria e compara com janelas dos modelos populares (GPT-4o, Claude, Gemini).

---

## 🔍 Busca avançada

### Receita 5: Buscar em uma norma específica

```bash
vectorgov search "credenciamento" --doc LEI-14133-2021 --top-k 10
```

Aplica filtro `document_id_filter` no backend — só retorna resultados dessa lei.

### Receita 6: Filtrar por tipo + ano

```bash
vectorgov search "dispensa" --tipo LEI --ano 2021
vectorgov search "credenciamento" --tipo IN --ano 2022
```

### Receita 7: Análise jurídica completa com Juiz LLM

```bash
vectorgov smart-search "É possível dispensar licitação para obras emergenciais?"
```

Retorna chunks aprovados pelo Juiz LLM + nível de confiança (alta/média/baixa) + raciocínio + dispositivos relacionados via grafo. Custa mais (💰💰) mas entrega resposta pronta.

### Receita 8: Buscar referências relacionadas via grafo

```bash
vectorgov hybrid "critérios de julgamento" --hops 2 --top-k 15
```

Retorna os artigos diretamente relevantes **+** os artigos citados/citantes via grafo de citações (hops=2 expande mais profundamente).

### Receita 9: Máxima cobertura (dual-path)

```bash
vectorgov merged "prazo para impugnação do edital"
```

Combina semântica + índice curado via Reciprocal Rank Fusion (RRF). Hits que aparecem nos dois recebem boost automático.

---

## 📌 Lookup de referências legais

### Receita 10: Resolver uma referência

```bash
vectorgov lookup "Art. 75 da Lei 14.133"
```

Retorna o texto consolidado do artigo (caput + parágrafos + incisos + alíneas).

### Receita 11: Batch de referências (auto-split)

```bash
vectorgov lookup "Art. 75, Art. 18 e Art. 33 da Lei 14.133"
```

O CLI detecta separadores (vírgula, ponto-e-vírgula, " e ") e processa como batch.

### Receita 12: Batch via stdin (uma por linha)

```bash
cat referencias.txt | vectorgov lookup --pipe
# ou
printf "Art. 75 da Lei 14.133\nArt. 33 da Lei 14.133" | vectorgov lookup --pipe
```

Útil para processar listas grandes (até 20 referências por chamada).

### Receita 13: Ler texto completo de um artigo conhecido

```bash
# Se você já sabe o ID
vectorgov read LEI-14133-2021 --span ART-075

# Se você tem a referência em texto
vectorgov explain "Art. 75 da Lei 14.133"
```

`explain` faz `lookup + read` em uma chamada — mais conveniente para vibe coding.

---

## 🐍 Integração programática

### Receita 14: CLI vs SDK (quando usar cada um)

| Cenário | Use |
|---|---|
| Exploração interativa no terminal | **CLI** |
| Vibe coding com LLMs (copy-paste) | **CLI** + `--output llm` |
| Script shell pontual | **CLI** + `--raw` + `jq` |
| Aplicação Python | **SDK** (`pip install vectorgov`) |
| Agente Python (LangChain, etc.) | **SDK** + integrations |
| MCP server (Claude Desktop, Cursor) | **SDK** com extra `[mcp]` |

### Receita 15: Pipeline shell com jq

```bash
# Extrair texto do primeiro hit
vectorgov search --raw "ETP" | jq '.hits[0].text'

# Capturar query_id em variável e mandar feedback
QUERY_ID=$(vectorgov search --raw "ETP" | jq -r '.query_id')
vectorgov feedback send $QUERY_ID --like

# Hybrid: extrair só artigos via grafo
vectorgov hybrid "critérios de julgamento" --hops 2 --raw | jq '.graph_nodes'

# Batch lookup: listar URLs de evidência
vectorgov lookup --raw "Art. 75, Art. 18 da Lei 14.133" | jq -r '.results[].evidence_url'
```

> **Dica**: `jq` é opcional. Para uso interativo, prefira `--output json` (já vem com syntax highlight). Use `jq` só para shell scripts de automação.

### Receita 16: Equivalente Python (sem jq, sem CLI)

```python
from vectorgov import VectorGov

vg = VectorGov()  # lê VECTORGOV_API_KEY

# Busca
result = vg.search("ETP", top_k=5)
for hit in result.hits:
    label = hit.citation or hit.source  # citation = formato jurídico
    print(f"[{hit.score:.0%}] {label}")
    print(hit.text[:200])

# Lookup
r = vg.lookup("Art. 75 da Lei 14.133")
print(r.match.citation)  # 'Art. 75 da Lei 14.133/2021'
print(r.stitched_text)

# Feedback
vg.feedback(query_id=result.query_id, like=True)
```

---

## 📊 Auditoria e quotas

### Receita 17: Ver uso do plano

```bash
vectorgov quota
vectorgov audit stats --days 7
```

### Receita 18: Logs de eventos críticos

```bash
vectorgov audit logs --severity critical --days 30
```

### Receita 19: Listar normas indexadas

```bash
vectorgov docs list                              # Página 1
vectorgov docs list --page 2 --limit 50          # Mais
vectorgov docs info LEI-14133-2021               # Detalhe de uma
```

---

## 🛠️ Troubleshooting

### Receita 20: Sair de problemas comuns

```bash
# Esqueci minha API key
vectorgov auth status

# Refazer login
vectorgov auth logout
vectorgov auth login

# Mudar formato padrão para 'llm' (sessão)
export VECTORGOV_OUTPUT=llm

# Mudar formato padrão (permanente)
vectorgov config set default_output llm

# Ver toda a configuração atual
vectorgov config list

# Ajuda detalhada de qualquer comando
vectorgov search --help
vectorgov lookup --help
```

---

## Veja também

- 🧭 [Cheat Sheet](cheat-sheet.md) — visão geral em 1 página
- 📖 [Reference completa](commands.md) — todas as flags e formatos
- 🐍 [SDK Python](https://pypi.org/project/vectorgov/) — quando preferir programar
