# Guia de Migração - Correções e Melhorias MiroFish

> Use este documento para aplicar as correções no projeto original.
> Cada seção tem: arquivo, problema, e o código exato para corrigir.
> Pode ser usado como prompt para o Claude em outro projeto.

---

## Prompt para o Claude no projeto original

Cole este prompt no Claude Code dentro do projeto original:

```
Preciso aplicar as seguintes correções e melhorias no MiroFish.
Leia o arquivo MIGRATION-GUIDE.md e aplique cada correção na ordem.
Não altere lógica existente além do descrito.
```

---

## PARTE 1: CORREÇÕES CRÍTICAS (backend não inicia sem elas)

### 1. Funções ausentes no config.py

**Arquivo:** `backend/app/config.py`
**Problema:** `_get_bool_env` e `_get_cors_origins` são referenciadas mas não existem.

**Adicionar ANTES da classe Config:**
```python
import secrets  # adicionar ao topo junto com os outros imports

def _get_bool_env(key, default=False):
    """Get a boolean value from environment variable."""
    val = os.environ.get(key, '')
    if not val:
        return default
    return val.lower() in ('true', '1', 'yes')


def _get_cors_origins():
    """Get CORS origins from environment or return defaults."""
    origins = os.environ.get('CORS_ORIGINS', '')
    if origins:
        return [o.strip() for o in origins.split(',')]
    return ['http://localhost:3000', 'http://localhost:5173']
```

---

### 2. Import circular no build_graph.py

**Arquivo:** `backend/app/tools/build_graph.py`
**Problema:** Import circular impede o backend de iniciar.

**Remover** esta linha do topo do arquivo:
```python
from ..services.graph_builder import GraphBuilderService
```

**Adicionar** o import dentro da função `run_build()`, logo antes de `builder = GraphBuilderService()`:
```python
from ..services.graph_builder import GraphBuilderService
builder = GraphBuilderService()
```

---

### 3. Claude CLI falha com prompts longos

**Arquivo:** `backend/app/utils/llm_client.py`
**Problema:** Prompt passado como argumento de linha de comando estoura limite do OS (~128KB).

**Substituir:**
```python
result = subprocess.run(
    ["claude", "-p", "--output-format", "json", prompt],
    capture_output=True, text=True, timeout=120,
    cwd="/tmp"
)
```

**Por:**
```python
result = subprocess.run(
    ["claude", "-p", "--output-format", "json"],
    input=prompt,
    capture_output=True, text=True, timeout=300,
    cwd="/tmp"
)
```

**Nota:** Timeout também aumentado de 120s para 300s (5 min) para suportar geração de relatórios longos.

Também atualizar a mensagem de timeout:
```python
except subprocess.TimeoutExpired:
    raise RuntimeError("Claude CLI timed out after 300s")
```

---

## PARTE 2: SUPORTE A MÚLTIPLOS PROVIDERS LLM

### 4. oasis_profile_generator — migrar para LLMClient

**Arquivo:** `backend/app/services/oasis_profile_generator.py`

#### 4a. Substituir import
**Remover:**
```python
from openai import OpenAI
```
**Adicionar:**
```python
from ..utils.llm_client import LLMClient
```

#### 4b. Substituir __init__
**Remover:**
```python
self.api_key = api_key or Config.LLM_API_KEY
self.base_url = base_url or Config.LLM_BASE_URL
self.model_name = model_name or Config.LLM_MODEL_NAME

if not self.api_key:
    raise ValueError("LLM_API_KEY is not configured")

self.client = OpenAI(
    api_key=self.api_key,
    base_url=self.base_url
)
```
**Adicionar:**
```python
self.llm = LLMClient(api_key=api_key, base_url=base_url, model=model_name)
```

#### 4c. Substituir chamada LLM em `_generate_profile_with_llm`
**Remover** o bloco `self.client.chat.completions.create(...)` e parsing de `response.choices[0]`.
**Substituir por:**
```python
content = self.llm.chat(
    messages=[
        {"role": "system", "content": self._get_system_prompt(is_individual)},
        {"role": "user", "content": prompt}
    ],
    temperature=0.7 - (attempt * 0.1),
    response_format={"type": "json_object"},
)
```
E remover o bloco de `finish_reason` / `_fix_truncated_json` (o LLMClient já limpa o output).

---

### 5. simulation_config_generator — migrar para LLMClient

**Arquivo:** `backend/app/services/simulation_config_generator.py`

#### 5a. Substituir import
**Remover:**
```python
from openai import OpenAI
```
**Adicionar:**
```python
from ..utils.llm_client import LLMClient
```

#### 5b. Substituir __init__
**Remover:**
```python
self.api_key = api_key or Config.LLM_API_KEY
self.base_url = base_url or Config.LLM_BASE_URL
self.model_name = model_name or Config.LLM_MODEL_NAME

if not self.api_key:
    raise ValueError("LLM_API_KEY is not configured")

self.client = OpenAI(
    api_key=self.api_key,
    base_url=self.base_url
)
```
**Adicionar:**
```python
self.llm = LLMClient(api_key=api_key, base_url=base_url, model=model_name)
```

#### 5c. Substituir chamada LLM em `_call_llm_with_retry`
**Remover** o bloco `self.client.chat.completions.create(...)` e parsing de `response.choices[0]`.
**Substituir por:**
```python
content = self.llm.chat(
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": prompt}
    ],
    temperature=0.7 - (attempt * 0.1),
    response_format={"type": "json_object"},
)
```

#### 5d. Corrigir referências a self.model_name e self.base_url
**Substituir:**
```python
llm_model=self.model_name,
llm_base_url=self.base_url,
```
**Por:**
```python
llm_model=self.llm.model,
llm_base_url=self.llm.base_url,
```

---

## PARTE 3: PROBLEMA DE FLUXO — SIMULAÇÃO REINICIA AO DAR REFRESH

### 6. Step3 reinicia simulação ao montar

**Arquivo:** `frontend/src/components/Step3Simulation.vue`
**Problema:** `onMounted` chama `doStartSimulation()` com `force: true`, reiniciando a simulação toda vez que a página carrega (refresh, navegação). Perde todos os dados.

**Substituir o onMounted:**
```javascript
// ANTES (problemático):
onMounted(() => {
  addLog('Inicializando execução da simulação - Etapa 3')
  if (props.simulationId) {
    doStartSimulation()
  }
})
```

**Por:**
```javascript
// DEPOIS (detecta estado antes de agir):
onMounted(async () => {
  addLog('Inicializando execução da simulação - Etapa 3')
  if (props.simulationId) {
    // First check if simulation already completed or is running
    try {
      const res = await getRunStatus(props.simulationId)
      if (res.success && res.data) {
        const status = res.data.runner_status
        if (status === 'completed' || status === 'stopped') {
          addLog('✓ Simulação já concluída — pronta para gerar relatório')
          phase.value = 2
          runStatus.value = res.data
          emit('update-status', 'completed')
          return
        }
        if (status === 'running') {
          addLog('Simulação em andamento — retomando monitoramento...')
          phase.value = 1
          runStatus.value = res.data
          startStatusPolling()
          startDetailPolling()
          return
        }
      }
    } catch (e) {
      // Status check failed, proceed to start
    }
    doStartSimulation()
  }
})
```

---

## PARTE 4: PROBLEMA DE FLUXO — PROJETO NÃO DETECTA ETAPA ATUAL

### 7. Navegação entre etapas não persiste estado

**Problema:** Ao abrir um projeto que já passou pelo Step 3 (simulação completa), o frontend volta ao Step 2 e mostra "Iniciar Simulação" — que cria uma NOVA simulação, perdendo a anterior.

**Causa raiz:** O projeto (`project.json`) não guarda `current_step`, `simulation_id`, nem `report_id`. Cada view é independente e não sabe o que já foi feito.

**Correção necessária (múltiplos arquivos):**

#### 7a. Backend — adicionar campos ao projeto
**Arquivo:** `backend/app/models/project.py`
Adicionar ao modelo Project:
```python
current_step: int = 1  # 1-5
simulation_id: Optional[str] = None
report_id: Optional[str] = None
```

Atualizar `current_step` conforme o projeto avança:
- Step 1 completo (grafo construído) → `current_step = 2`
- Step 2 completo (simulação criada) → `current_step = 3`
- Step 3 completo (simulação rodou) → `current_step = 4`
- Step 4 completo (relatório gerado) → `current_step = 5`

#### 7b. Frontend — detectar etapa ao abrir projeto
**Arquivo:** `frontend/src/views/MainView.vue` (e similares)
No `onMounted`, verificar `projectData.current_step` e redirecionar:
```javascript
if (projectData.current_step >= 3 && projectData.simulation_id) {
  router.replace(`/simulation/${projectData.simulation_id}/start`)
  return
}
if (projectData.current_step >= 4 && projectData.report_id) {
  router.replace(`/report/${projectData.report_id}`)
  return
}
```

#### 7c. Frontend — cards na home linkam para step correto
**Arquivo:** `frontend/src/views/Home.vue`
No `openProject`, verificar o step atual:
```javascript
const openProject = (proj) => {
  if (proj.current_step >= 4 && proj.report_id) {
    router.push(`/report/${proj.report_id}`)
  } else if (proj.current_step >= 3 && proj.simulation_id) {
    router.push(`/simulation/${proj.simulation_id}/start`)
  } else {
    router.push({ name: 'Process', params: { projectId: proj.project_id } })
  }
}
```

---

## PARTE 5: ENTREVISTAS OFFLINE (ambiente OASIS encerrado)

### 8. Entrevistas falham quando o ambiente OASIS não está rodando

**Problema:** A ferramenta `interview_agents` do Report Agent precisa que o processo OASIS esteja vivo em memória. Quando a simulação termina ou o backend reinicia, as entrevistas falham:
```
WARNING: Interview API call failed (environment not running?)
```

**Impacto:** O relatório é gerado sem as entrevistas, perdendo dados qualitativos dos agentes.

**Quando funciona:** Gerar relatório IMEDIATAMENTE após a simulação completar, sem reiniciar o backend. O script OASIS entra em "command-waiting mode" e aceita entrevistas.

**Correção recomendada — Entrevistas Offline:**

#### 8a. Salvar respostas dos agentes durante a simulação
**Arquivo:** `backend/scripts/run_parallel_simulation.py`
Ao final de cada rodada, salvar o estado de cada agente (posts, opiniões, memória) em arquivo:
```python
# Ao final de cada rodada, salvar estado dos agentes
agent_states_path = os.path.join(sim_dir, "agent_states.json")
with open(agent_states_path, 'w') as f:
    json.dump({
        agent_id: {
            "name": agent.name,
            "persona": agent.persona,
            "posts": agent.posts,
            "memory": agent.memory_summary,
        }
        for agent_id, agent in agents.items()
    }, f, indent=2)
```

#### 8b. Criar endpoint de entrevista offline
**Arquivo:** `backend/app/api/simulation.py`
Novo endpoint que usa LLM para simular entrevista baseada nos dados salvos:
```python
@simulation_bp.route('/<simulation_id>/interview-offline', methods=['POST'])
def interview_offline(simulation_id):
    """Interview agents using saved state (no live environment needed)"""
    # Load agent_states.json
    # Use LLM to generate responses based on persona + posts + memory
    # Return formatted interview results
```

#### 8c. Atualizar ferramenta de entrevista do Report Agent
**Arquivo:** `backend/app/services/report_agent.py` (ou equivalente)
Quando a entrevista ao vivo falhar, tentar entrevista offline automaticamente:
```python
try:
    result = interview_live(agents, questions)
except SimulationNotRunning:
    result = interview_offline(simulation_id, agents, questions)
```

---

## PARTE 6: MELHORIAS DE INTERFACE

### 9. Parar/Retomar builds de grafo

#### 9a. Registro global de builds ativos
**Arquivo:** `backend/app/services/graph_builder.py`
**Adicionar** antes da classe `GraphBuilderService`:
```python
_active_builds = {}  # task_id -> {"stop": False}

def request_stop(task_id: str):
    if task_id in _active_builds:
        _active_builds[task_id]["stop"] = True
        return True
    return False
```

#### 9b. Suporte a stop no entity_extractor
**Arquivo:** `backend/app/services/entity_extractor.py`
**Adicionar** parâmetros ao `extract_batch`:
- `stop_check=None` — callable que retorna True para parar
- `start_from=0` — chunk index para retomar
- `prior_entities=None` — entidades já extraídas
- `prior_relationships=None` — relacionamentos já extraídos

No loop, antes de processar cada chunk:
```python
if stop_check and stop_check():
    return {"entities": ..., "relationships": ..., "last_chunk_index": i, "stopped": True}
```

#### 9c. Novos endpoints
**Arquivo:** `backend/app/api/graph.py`
- `PUT /api/graph/project/<id>` — atualizar nome
- `POST /api/graph/project/<id>/stop` — parar build, salvar progresso
- `POST /api/graph/project/<id>/resume` — retomar de onde parou

---

### 10. Frontend — Projetos na home + Nome do projeto

**Arquivo:** `frontend/src/views/Home.vue`
- Lista de projetos existentes com cards clicáveis
- Campo "Nome do Projeto" na criação
- Botões: ✎ editar, ◼ parar, ▶ continuar, ↻ refazer, ✕ excluir
- Status com bolinha colorida animada
- Painel de configuração de LLMs com 7 providers e presets

**Arquivo:** `frontend/src/views/MainView.vue` (e outras views)
- Nome do projeto em destaque no topo com project_id abaixo

**Arquivo:** `frontend/src/store/pendingUpload.js`
- Adicionado campo `projectName`

---

## PARTE 7: CONFIGURAÇÃO DE LLMs

### 11. Arquitetura de LLMs por etapa

O MiroFish usa LLMs em 6 etapas. Cada uma pode usar um provider diferente:

| # | Etapa | Arquivo | Usa LLMClient? | Recomendado |
|---|-------|---------|----------------|-------------|
| 1 | Ontologia/Grafo | `entity_extractor.py` | ✓ Sim | Claude ou 70B+ |
| 2 | Personas | `oasis_profile_generator.py` | ✓ Sim (após correção 4) | Claude ou 70B+ |
| 3 | Config simulação | `simulation_config_generator.py` | ✓ Sim (após correção 5) | Claude ou 70B+ |
| 4 | Simulação OASIS | `scripts/run_parallel_simulation.py` | ✗ Não (usa camel-ai) | API OpenAI-compatible |
| 5 | Relatório | report agent via `llm_client.py` | ✓ Sim | Claude ou analítico |
| 6 | Chat/Interação | via `llm_client.py` | ✓ Sim | Conversacional |

**Nota sobre Etapa 4:** O script usa `camel-ai` que exige `OPENAI_API_KEY`. Não funciona com `claude-cli`. Precisa de API key de provider OpenAI-compatible (OpenAI, Ollama, Groq, OpenRouter).

### Exemplo de `.env` híbrido
```bash
# Provider principal (etapas 1-3, 5-6)
LLM_PROVIDER=claude-cli

# API para simulação OASIS (etapa 4) — precisa ser OpenAI-compatible
LLM_API_KEY=ollama
LLM_BASE_URL=http://localhost:11434/v1
LLM_MODEL_NAME=qwen2.5:32b
```

### Providers suportados
| Provider | Base URL | Modelos sugeridos | Precisa API key? |
|----------|----------|-------------------|------------------|
| Claude CLI | — | (usa assinatura) | Não |
| OpenAI | `https://api.openai.com/v1` | gpt-4o, gpt-4o-mini | Sim |
| Anthropic | `https://api.anthropic.com` | claude-sonnet-4, claude-opus-4 | Sim |
| Groq | `https://api.groq.com/openai/v1` | llama-3.3-70b-versatile | Sim (grátis) |
| OpenRouter | `https://openrouter.ai/api/v1` | qwen/qwen-2.5-72b-instruct | Sim |
| Ollama | `http://localhost:11434/v1` | qwen2.5:32b, llama3.1:70b | Não |

---

## Ordem de aplicação recomendada

### Prioridade 1 — Backend não funciona sem estas:
1. Correção 1 (config.py — funções ausentes)
2. Correção 2 (build_graph.py — import circular)
3. Correção 3 (llm_client.py — stdin + timeout)

### Prioridade 2 — Suporte a claude-cli:
4. Correção 4 (oasis_profile_generator → LLMClient)
5. Correção 5 (simulation_config_generator → LLMClient)

### Prioridade 3 — Fluxo não perde dados:
6. Correção 6 (Step3 detecta simulação completada)
7. Correção 7 (projeto persiste etapa atual)
8. Correção 8 (entrevistas offline)

### Prioridade 4 — Interface:
9. Melhoria 9 (parar/retomar builds)
10. Melhoria 10 (projetos na home, nome, LLM config)
11. Configuração 11 (LLMs por etapa)
