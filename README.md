![MiroFish Hero](static/image/mirofish_hero.png)

# MiroFish

> **[Leia em Português](#mirofish-português)**

A swarm intelligence prediction engine. Upload documents describing any scenario, and MiroFish simulates thousands of AI agents reacting on social media to predict how events will unfold.

**Live:** [synth.scty.org](https://synth.scty.org)

> Fork of [666ghj/MiroFish](https://github.com/666ghj/MiroFish) — fully translated to English, local KuzuDB graph storage, Claude/Codex CLI support added.

## What it does

1. **Upload reality seeds** — PDFs, markdown, or text files (news articles, policy drafts, financial reports, anything)
2. **Describe what to predict** — natural language prompt (e.g., "Predict public reaction to this policy over 60 days")
3. **MiroFish builds a world** — extracts entities and relationships into a knowledge graph, generates AI agent personas with distinct personalities and opinions
4. **Agents simulate social media** — dual-platform simulation (Twitter + Reddit) where agents post, reply, like, argue, and follow each other
5. **Get a prediction report** — AI analyzes all simulation data and produces findings. Chat with the report agent or interview individual simulated agents.

## Changes from upstream

| Area | Upstream | This fork |
|------|----------|-----------|
| **Language** | Chinese UI + prompts | Full English (60+ files translated) |
| **LLM providers** | Alibaba Qwen only | OpenAI, Anthropic, Claude CLI, Codex CLI |
| **Graph database** | Hosted graph service | Local KuzuDB (embedded, free) |
| **Entity extraction** | Managed extraction pipeline | LLM-based extraction (uses your own model) |
| **Auth** | Requires API keys | Can use Claude Code or Codex CLI subscriptions (no separate API cost) |

## Quick start

### Prerequisites

- Node.js 18+
- Python 3.11-3.12
- [uv](https://docs.astral.sh/uv/) (Python package manager)

### Setup

```bash
cp .env.example .env
# Edit .env — pick your LLM provider (see below)
npm run setup:all
npm run dev
```

- Frontend: http://localhost:3000
- Backend API: http://localhost:5001

### Docker

```bash
cp .env.example .env
docker compose up -d --build
```

Docker builds the Vue frontend, serves it from the Flask app, and exposes the combined app on port `5001` inside the container.

## LLM providers

Set `LLM_PROVIDER` in `.env`:

| Provider | Config | Cost |
|----------|--------|------|
| `claude-cli` | Just set `LLM_PROVIDER=claude-cli` | Uses your Claude Code subscription |
| `codex-cli` | Just set `LLM_PROVIDER=codex-cli` | Uses your Codex CLI subscription |
| `openai` | Set `LLM_API_KEY` + `LLM_MODEL_NAME` | Pay-per-token |
| `anthropic` | Set `LLM_API_KEY` + `LLM_MODEL_NAME` | Pay-per-token |

```env
# Example: use Codex CLI (no API key needed)
LLM_PROVIDER=codex-cli

# Example: use OpenAI API
LLM_PROVIDER=openai
LLM_API_KEY=sk-...
LLM_MODEL_NAME=gpt-4o-mini
```

## Architecture

```
frontend/          Vue 3 + Vite + D3.js (graph visualization)
backend/
  app/
    api/           Thin Flask REST endpoints (graph, simulation, report)
    core/          Workbench session, session registry, resource loader, tasks
    resources/     Adapters for projects, documents, Kuzu, simulations, reports
    tools/         Composable workbench operations (ingest, build, prepare, run, report)
    services/
      graph_db.py          KuzuDB-backed knowledge graph
      entity_extractor.py  LLM-based entity/relationship extraction
      graph_builder.py     Ontology → graph pipeline
      simulation_runner.py OASIS multi-agent simulation (subprocess)
      report_agent.py      ReACT agent with tool-calling for reports
      kuzu_tools.py        Search, interview, and analysis tools
    utils/
      llm_client.py        Multi-provider LLM client (OpenAI/Anthropic/CLI)
  scripts/         OASIS simulation runner scripts (Twitter + Reddit)
```

Workbench session metadata is persisted under `backend/uploads/workbench_sessions/`, and long-running task state is persisted under `backend/uploads/tasks/`.

The backend is being refactored toward a pi-style shape: one workbench session core, pluggable resource adapters, composable tools, and thin API shells.

## How the pipeline works

```
Document upload → LLM ontology extraction → Knowledge graph (KuzuDB)
    → Entity filtering → Agent persona generation (LLM)
    → OASIS dual-platform simulation (Twitter + Reddit subprocess)
    → Graph memory updates → Report generation (ReACT agent)
    → Interactive chat with report agent or individual agents
```

## Acknowledgments

- [MiroFish](https://github.com/666ghj/MiroFish) by 666ghj — original project
- [OASIS](https://github.com/camel-ai/oasis) by CAMEL-AI — multi-agent social simulation framework
- [KuzuDB](https://github.com/kuzudb/kuzu) — embedded graph database

## License

AGPL-3.0

---

![MiroFish Dashboard](static/image/mirofish_dashboard.png)

---

# MiroFish (Português)

Um motor de previsão baseado em inteligência de enxame. Envie documentos descrevendo qualquer cenário e o MiroFish simula milhares de agentes de IA reagindo em redes sociais para prever como os eventos vão se desenrolar.

**Demo:** [synth.scty.org](https://synth.scty.org)

> Fork de [666ghj/MiroFish](https://github.com/666ghj/MiroFish) — totalmente traduzido para inglês, armazenamento local em grafo com KuzuDB, suporte a Claude/Codex CLI adicionado.

## O que faz

1. **Envie sementes de realidade** — PDFs, markdown ou arquivos de texto (notícias, propostas de políticas, relatórios financeiros, qualquer coisa)
2. **Descreva o que prever** — prompt em linguagem natural (ex.: "Preveja a reação pública a esta política em 60 dias")
3. **MiroFish constrói um mundo** — extrai entidades e relacionamentos em um grafo de conhecimento, gera personas de agentes de IA com personalidades e opiniões distintas
4. **Agentes simulam redes sociais** — simulação em duas plataformas (Twitter + Reddit) onde agentes postam, respondem, curtem, debatem e seguem uns aos outros
5. **Obtenha um relatório de previsão** — a IA analisa todos os dados da simulação e produz conclusões. Converse com o agente de relatórios ou entreviste agentes simulados individualmente.

## Mudanças em relação ao upstream

| Área | Upstream | Este fork |
|------|----------|-----------|
| **Idioma** | Interface e prompts em chinês | Totalmente em inglês (60+ arquivos traduzidos) |
| **Provedores LLM** | Apenas Alibaba Qwen | OpenAI, Anthropic, Claude CLI, Codex CLI |
| **Banco de grafos** | Serviço de grafo hospedado | KuzuDB local (embutido, gratuito) |
| **Extração de entidades** | Pipeline de extração gerenciada | Extração via LLM (usa seu próprio modelo) |
| **Autenticação** | Requer chaves de API | Pode usar assinaturas do Claude Code ou Codex CLI (sem custo de API separado) |

## Início rápido

### Pré-requisitos

- Node.js 18+
- Python 3.11-3.12
- [uv](https://docs.astral.sh/uv/) (gerenciador de pacotes Python)

### Configuração

```bash
cp .env.example .env
# Edite .env — escolha seu provedor LLM (veja abaixo)
npm run setup:all
npm run dev
```

- Frontend: http://localhost:3000
- API Backend: http://localhost:5001

### Docker

```bash
cp .env.example .env
docker compose up -d --build
```

O Docker compila o frontend Vue, serve-o a partir do app Flask e expõe o app combinado na porta `5001` dentro do container.

## Provedores LLM

Defina `LLM_PROVIDER` no `.env`:

| Provedor | Configuração | Custo |
|----------|--------------|-------|
| `claude-cli` | Apenas defina `LLM_PROVIDER=claude-cli` | Usa sua assinatura do Claude Code |
| `codex-cli` | Apenas defina `LLM_PROVIDER=codex-cli` | Usa sua assinatura do Codex CLI |
| `openai` | Defina `LLM_API_KEY` + `LLM_MODEL_NAME` | Pagamento por token |
| `anthropic` | Defina `LLM_API_KEY` + `LLM_MODEL_NAME` | Pagamento por token |

```env
# Exemplo: usar Codex CLI (sem chave de API)
LLM_PROVIDER=codex-cli

# Exemplo: usar API da OpenAI
LLM_PROVIDER=openai
LLM_API_KEY=sk-...
LLM_MODEL_NAME=gpt-4o-mini
```

## Arquitetura

```
frontend/          Vue 3 + Vite + D3.js (visualização de grafos)
backend/
  app/
    api/           Endpoints REST Flask enxutos (grafo, simulação, relatório)
    core/          Sessão workbench, registro de sessões, carregador de recursos, tarefas
    resources/     Adaptadores para projetos, documentos, Kuzu, simulações, relatórios
    tools/         Operações combináveis do workbench (ingestão, construção, preparação, execução, relatório)
    services/
      graph_db.py          Grafo de conhecimento com KuzuDB
      entity_extractor.py  Extração de entidades/relacionamentos via LLM
      graph_builder.py     Pipeline ontologia → grafo
      simulation_runner.py Simulação multi-agente OASIS (subprocesso)
      report_agent.py      Agente ReACT com chamada de ferramentas para relatórios
      kuzu_tools.py        Ferramentas de busca, entrevista e análise
    utils/
      llm_client.py        Cliente LLM multi-provedor (OpenAI/Anthropic/CLI)
  scripts/         Scripts do simulador OASIS (Twitter + Reddit)
```

Os metadados da sessão workbench são persistidos em `backend/uploads/workbench_sessions/`, e o estado de tarefas de longa duração é persistido em `backend/uploads/tasks/`.

O backend está sendo refatorado em formato pi: um núcleo de sessão workbench, adaptadores de recursos plugáveis, ferramentas combináveis e camadas de API enxutas.

## Como o pipeline funciona

```
Upload de documento → Extração de ontologia via LLM → Grafo de conhecimento (KuzuDB)
    → Filtragem de entidades → Geração de personas de agentes (LLM)
    → Simulação OASIS em duas plataformas (Twitter + Reddit, subprocesso)
    → Atualizações de memória do grafo → Geração de relatório (agente ReACT)
    → Chat interativo com o agente de relatórios ou agentes individuais
```

## Agradecimentos

- [MiroFish](https://github.com/666ghj/MiroFish) por 666ghj — projeto original
- [OASIS](https://github.com/camel-ai/oasis) por CAMEL-AI — framework de simulação social multi-agente
- [KuzuDB](https://github.com/kuzudb/kuzu) — banco de dados de grafos embutido

## Licença

AGPL-3.0

---

![MiroFish Architecture](static/image/mirofish_architecture.png)
