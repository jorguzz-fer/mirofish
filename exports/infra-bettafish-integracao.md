# BettaFish — Infraestrutura, Fontes e Integracao com MiroFish

---

## 1. O Que e o BettaFish

Sistema multi-agente de analise de opiniao publica criado por BaiFu (666ghj). Crawla 30+ plataformas de redes sociais, analisa sentimentos, identifica tendencias e gera relatorios. 39.6K stars no GitHub.

5 agentes especializados trabalham em paralelo:
- **QueryEngine** — busca ampla em web/noticias
- **MediaEngine** — analise multimodal (video/imagem)
- **InsightEngine** — mineracao de dados privados
- **ReportEngine** — geracao de relatorio final
- **ForumEngine** — debate entre agentes para quebrar bolhas

---

## 2. Requisitos de Infraestrutura

### Hardware

| Recurso | Minimo | Recomendado | Notas |
|---------|--------|-------------|-------|
| RAM | 2 GB | 4 GB+ | Agentes concorrentes + streaming LLM |
| CPU | 2 cores | 4+ cores | Agentes paralelos + crawler |
| Disco | 10 GB | 50 GB+ | Dados do crawler crescem rapido |
| Rede | 10 Mbps | 50+ Mbps | APIs LLM + scraping |
| GPU | Nao precisa | Opcional CUDA | Apenas para inferencia local de sentimento |
| Python | 3.9+ | 3.11 | |

### Stack de Software

| Componente | Tecnologia |
|------------|-----------|
| Web framework | Flask + FastAPI + Streamlit (x3) |
| Banco principal | PostgreSQL 15+ (ou MySQL 8+) |
| Crawler | Playwright + Chromium |
| ML | PyTorch, Transformers, Sentence-Transformers, XGBoost |
| PDF | WeasyPrint |
| Busca | Tavily, Bocha, Anspire |
| LLM | OpenAI-compatible (qualquer provider) |

### Docker Setup (2 containers)

| Container | Imagem | Portas | Funcao |
|-----------|--------|--------|--------|
| bettafish | Python 3.11-slim | 5000, 8501-8503 | App + 3 interfaces Streamlit |
| bettafish-db | PostgreSQL 15 | 5444 | Banco de dados persistente |

Deploy:
```bash
git clone https://github.com/666ghj/BettaFish.git
cp .env.example .env
# editar .env com suas API keys
docker compose up -d
```

---

## 3. APIs e LLMs Necessarios

### LLMs (5 chaves, uma por agente)

| Variavel | Modelo Padrao | Agente | Funcao |
|----------|--------------|--------|--------|
| QUERY_ENGINE_* | deepseek-chat | QueryEngine | Busca web/noticias |
| MEDIA_ENGINE_* | gemini-2.5-pro | MediaEngine | Analise multimodal |
| INSIGHT_ENGINE_* | kimi-k2 | InsightEngine | Mineracao (500K contexto) |
| REPORT_ENGINE_* | gemini-2.5-pro | ReportEngine | Geracao de relatorio |
| FORUM_HOST_* | qwen-plus | ForumEngine | Coordenacao + debate |

Cada um precisa de 3 variaveis: `*_API_KEY`, `*_BASE_URL`, `*_MODEL_NAME`. Pode apontar todos para o mesmo provider.

### APIs de Busca (3 chaves)

| Variavel | Servico | Funcao |
|----------|---------|--------|
| TAVILY_API_KEY | Tavily | Busca web |
| ANSPIRE_API_KEY | Anspire | Busca multimodal |
| BOCHA_WEB_SEARCH_API_KEY | Bocha | Busca multimodal alternativa |

---

## 4. Fontes que o BettaFish Crawla (30+ plataformas)

### Redes Sociais Chinesas (MindSpider crawler)

| Plataforma | Nome Chines | Equivalente Ocidental |
|-----------|-------------|----------------------|
| Weibo | 微博 | Twitter/X chines |
| Xiaohongshu | 小红书 | Instagram chines |
| Douyin | 抖音 | TikTok chines (versao China) |
| Kuaishou | 快手 | Concorrente do Douyin |
| Bilibili | B站 | YouTube chines |
| Tieba | 贴吧 | Forums do Baidu |
| Zhihu | 知乎 | Quora chines |

### Internacionais (via APIs de busca)

| Fonte | Via |
|-------|-----|
| Google News / Bing News | Tavily |
| Twitter/X | Tavily / Bocha |
| Reddit | Tavily |
| YouTube | Tavily |
| Sites de noticias | Tavily / Bocha |

### Nota sobre plataformas ocidentais
O crawler MindSpider foi feito para plataformas **chinesas**. Para TikTok internacional, Instagram ou Twitter, opcoes:
1. Usar **Tavily/Bocha** para busca generica
2. Adaptar o MindSpider para crawlar plataformas ocidentais
3. Exportar dados manualmente e alimentar o sistema

---

## 5. Custo Estimado

### Por analise
| Item | Custo |
|------|-------|
| Tokens LLM (50K-200K) | $0.50 - $2.00 |
| Buscas API (50-200 requests) | $0.10 - $0.50 |
| **Total por analise** | **$0.60 - $2.50** |

### Mensal

| Uso | Analises/dia | API/mes | VPS/mes | Total |
|-----|-------------|---------|---------|-------|
| Leve | 2 | ~$40-150 | ~$20 | ~$60-170 |
| Moderado | 10 | ~$180-750 | ~$24 | ~$200-775 |
| Pesado | 30+ | ~$550-2250 | ~$30+ | ~$580-2280 |

Para reduzir custos: substituir Gemini 2.5 Pro por DeepSeek V3 ou Qwen em todos os agentes (reducao de 5-10x).

---

## 6. Integracao BettaFish + MiroFish

### Fluxo de Dados

```
BettaFish                              MiroFish
┌──────────────────┐                ┌──────────────────────┐
│ Crawla 30+       │                │                      │
│ plataformas      │───exporta───→  │ Upload como          │
│ (comentarios,    │   JSON/PDF     │ "Reality Seeds"      │
│ posts, trends)   │                │                      │
├──────────────────┤                ├──────────────────────┤
│ 5 Agentes        │                │ Gera ontologia       │
│ analisam         │───insights──→  │ Constroi grafo       │
│ sentimento e     │   como prompt  │ Cria agentes         │
│ tendencias       │                │ Simula futuro        │
├──────────────────┤                ├──────────────────────┤
│ DADOS REAIS      │                │ PREVISAO DO FUTURO   │
│ "O que esta      │                │ "O que vai           │
│  acontecendo"    │                │  acontecer"          │
└──────────────────┘                └──────────────────────┘
```

### Opcao 1: Simples (funciona hoje)
1. Rodar BettaFish para coletar dados sobre seu nicho
2. Exportar relatorio como PDF ou TXT
3. Upload no MiroFish como documento seed
4. Prompt direcionado: "Baseado nesta analise real, simule [cenario]"

### Opcao 2: Media (API entre os dois)
1. BettaFish salva dados em PostgreSQL
2. Script extrai dados relevantes e formata como texto
3. Alimenta MiroFish via API `/api/graph/ontology/generate`
4. Automatizar com cron ou n8n

### Opcao 3: Avancada (pipeline unificado)
1. BettaFish crawla em tempo real
2. Dados alimentam o grafo do MiroFish continuamente
3. Simulacoes rodam periodicamente
4. Dashboard unificado: dados reais + previsoes

---

## 7. Todos os Repositorios do Autor (BaiFu - 666ghj)

| Repo | Stars | Funcao |
|------|-------|--------|
| **BettaFish** | 39.6K | Analise de opiniao publica com 5 agentes. Crawla 30+ plataformas. |
| **MiroFish** | 36.8K | Motor de inteligencia de enxame para previsao. Simula cenarios futuros. |
| **DeepSearchAgent-Demo** | 1.1K | Agente de pesquisa profunda. Gera relatorios sem frameworks. |
| **MindSpider** | 334 | Crawler de redes sociais com IA. Arquivado — virou submodulo do BettaFish. |
| **Go-argue-with-my-AI** | 18 | Adiciona marca d'agua de IA em fotos. |
| **StateMachineDrawing** | 12 | Editor web de maquinas de estado. |

### Ecossistema do autor
```
MindSpider (crawla dados)
    ↓
BettaFish (analisa opiniao publica)
    ↓
MiroFish (simula/preve o futuro)

DeepSearchAgent (pesquisa profunda independente)
```

---

## 8. Aplicacao para o Caso INEMA

### Cenario ideal com BettaFish + MiroFish

1. **BettaFish** crawla TikTok/Instagram/YouTube buscando:
   - Conteudo sobre IA e automacao no Brasil
   - Comentarios em posts de concorrentes
   - Tendencias em hashtags relevantes (#inteligenciaartificial, #automacao, etc.)

2. **BettaFish** gera relatorio de analise:
   - Sentimento do publico sobre ferramentas de IA
   - Quais tipos de hooks geram mais engajamento
   - Gaps no mercado de conteudo

3. **MiroFish** recebe o relatorio como seed e simula:
   - "Se INEMA lancar uma serie sobre agentes IA, como o publico reagiria?"
   - "Quais hooks converteriam mais para INEMA.VIP?"
   - "Como o publico se comportaria se oferecessemos mentoria gratuita por 7 dias?"

4. **Resultado:** Estrategia de conteudo baseada em dados reais + simulacao preditiva

### Limitacao atual
O crawler MindSpider foca em plataformas chinesas. Para o mercado brasileiro, precisaria:
- Adaptar crawlers para TikTok/Instagram/YouTube internacionais
- Ou usar Tavily/Bocha como fonte de busca generica
- Ou exportar dados manualmente das plataformas

---

*Documento gerado em 20/03/2026*
*Fontes: GitHub 666ghj/BettaFish, README-EN.md, docker-compose.yml, requirements.txt*
