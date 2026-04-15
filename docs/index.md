# VectorGov CLI

**CLI para busca semântica em legislação brasileira** — projetado para humanos no terminal e agentes de IA via stdin/stdout.

```bash
pip install vectorgov-cli
vectorgov auth login
vectorgov search "O que é ETP?"
```

```
[1/5] Art. 18 da Lei 14.133/2021 (score: 0.97)
Art. 18. A fase preparatória do processo licitatório é caracterizada pelo planejamento ...
```

---

## 🚀 Por onde começar

<div class="grid cards" markdown>

-   :material-rocket-launch:{ .lg .middle } **Quickstart**

    ---

    Do `pip install` ao primeiro resultado em 1 minuto.

    [:octicons-arrow-right-24: README](https://github.com/euteajudo/vectorgov-cli-docs#-quickstart-1-minuto)

-   :material-card-text-outline:{ .lg .middle } **Cheat Sheet**

    ---

    Os 20 comandos do CLI em uma única página, com decision tree.

    [:octicons-arrow-right-24: cheat-sheet.md](cheat-sheet.md)

-   :material-book-open-variant:{ .lg .middle } **Reference de comandos**

    ---

    Detalhe técnico de cada comando: flags, formatos, exemplos.

    [:octicons-arrow-right-24: commands.md](commands.md)

-   :material-chef-hat:{ .lg .middle } **Receitas**

    ---

    20 fluxos completos por caso de uso.

    [:octicons-arrow-right-24: recipes.md](recipes.md)

</div>

---

## 🔍 Os 9 comandos de busca

| Comando | Latência | Custo | Pra que serve |
|---|---|---|---|
| [`search`](commands.md#search) | 2-7s | 💰 | Busca semântica simples |
| [`smart-search`](commands.md#smart-search) | 5-18s | 💰💰 | Análise jurídica completa |
| [`hybrid`](commands.md#hybrid) | 3-10s | 💰 | Semântica + grafo |
| [`merged`](commands.md#merged) | 2-5s | 💰 | Dual-path (RRF) |
| [`lookup`](commands.md#lookup) | < 1s | 💰 | Resolve "Art. X da Lei Y" |
| [`grep`](commands.md#grep) | < 1s | 💰 | Busca textual literal |
| [`fs-search`](commands.md#fs-search) | < 1s | 💰 | Índice curado |
| [`read`](commands.md#read) | < 1s | **free** | Texto canônico |
| [`explain`](commands.md#explain) | ~1s | 💰 | lookup + read em uma chamada |

> 🌳 **Qual usar?** Veja a [decision tree na cheat sheet](cheat-sheet.md#qual-comando-usar).

---

## 🍳 Receitas comuns

### Bloco pronto para LLM

```bash
vectorgov context "Quando dispensar licitação?"
```

### Resolver referência legal

```bash
vectorgov lookup "Art. 75 da Lei 14.133"
```

### Inicializar projeto AI

```bash
vectorgov init --all  # CLAUDE.md, .cursorrules, AGENTS.md
```

> 🍳 **Mais receitas**: [recipes.md](recipes.md) tem 20 fluxos completos.

---

## 📚 Categorias completas (20 comandos)

<div class="grid cards" markdown>

-   :material-magnify:{ .lg .middle } **🔍 Busca (9)**

    `search` · `smart-search` · `hybrid` · `merged` · `lookup` · `grep` · `fs-search` · `read` · `explain`

    [:octicons-arrow-right-24: Detalhes](commands.md#busca)

-   :material-robot:{ .lg .middle } **🤖 LLM helpers (3)**

    `context` · `tokens` · `prompts`

    [:octicons-arrow-right-24: Detalhes](commands.md#llm-helpers)

-   :material-chart-bar:{ .lg .middle } **📊 Info & feedback (4)**

    `docs list/info` · `audit logs/stats` · `quota` · `feedback`

    [:octicons-arrow-right-24: Detalhes](commands.md#info-feedback)

-   :material-cog:{ .lg .middle } **🛠️ Setup & config (4)**

    `auth login/status/logout` · `config list/get/set/delete` · `init` · `version`

    [:octicons-arrow-right-24: Detalhes](commands.md#setup-config)

</div>

---

## 🤖 Para LLMs e agentes

```bash
export VECTORGOV_OUTPUT=llm
```

Todos os comandos passam a retornar texto puro otimizado (~40% menos tokens que JSON). O CLI também detecta automaticamente quando o stdout não é um terminal e usa `llm` por padrão.

Veja [docs/cheat-sheet.md](cheat-sheet.md) para o guia completo de uso por agentes.

---

## 🤝 Suporte

- 🐛 [GitHub Issues](https://github.com/euteajudo/vectorgov-cli-docs/issues)
- 📧 contato@vectorgov.io
- 📦 [SDK Python](https://pypi.org/project/vectorgov/) — alternativa programática
- 🌐 [Playground](https://vectorgov.io/playground)
