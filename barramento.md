# Barramento — Sistema Nervoso Central do Ghost

> **Propósito:** Orquestrador central que conecta todos os subsistemas do Ghost. É o único componente que conhece e coordena todos os pipelines, contextos, agentes, recursos e metas simultaneamente.

---

## Arquitetura Geral

```mermaid
graph TB
    subgraph Interface["🎨 src/interface.py"]
        UI[Chat REPL]
        EC[Entrada do Usuário]
    end

    subgraph Pipeline_Entrada["🧩 Pipelines de Entrada"]
        PC[Pre-Cortex<br/>processar texto]
        CX[Cortex<br/>inferir + abstrair]
        LG[Lógicas<br/>raciocínio]
    end

    subgraph Barramento["⚡ Barramento"]
        direction TB
        RO[Roteamento<br/>pub/sub]
        GC[GerenciadorContexto<br/>sessão atual]
        GM[GerenciadorMemoria<br/>wrapper orquestrador]
        PT[PlanejadorTarefas<br/>fila priorizada]
        AV[Avaliador<br/>score entrada/resposta]
        MM[MonitorMetas<br/>metas dinâmicas]
        GR[GestorRecursos<br/>registro thread-safe]

        CORE[Core: processar_entrada]
    end

    subgraph Saida["📤 Saída"]
        LLM[Modelo LLM]
        COA[CoAgente<br/>swarm de agentes]
        MP[Memória Primeiro<br/>resposta direta]
    end

    subgraph Armazenamento["💾 Armazenamento"]
        MEM[OrquestradorMemoria<br/>memorias/]
        N4[Neo4j]
        CH[ChromaDB]
        SQ[SQLite]
    end

    UI -->|texto, modelo, usuario| EC
    EC -->|processar_entrada| CORE
    CORE -->|1 pipeline_pre| PC
    CORE -->|2 cortex| CX
    CORE -->|3 logicas| LG
    CORE -->|4 avaliar| AV
    CORE -->|5 agendar| PT
    CORE -->|6 despachar evento| RO

    CORE -.->|retorna dict<br/>pre_cortex + cortex<br/>+ logicas + avaliacao| UI
    UI -->|LLM prompt| LLM
    UI -->|processar_memoria_primeiro| MP
    UI -->|processar_coagente| COA

    GM -->|delega| MEM
    MEM --> N4
    MEM --> CH
    MEM --> SQ

    COA --> MEM

    RO -->|entrada_recebida| MM
    RO -->|ferramenta_chamada| Log[logger]
```

---

## Organograma do Barramento

```mermaid
graph LR
    subgraph Barramento["Barramento"]
        direction TB
        B[Barramento<br/>Classe Principal]

        B --> R[├── Roteamento<br/>eventos pub/sub]
        B --> C[├── GerenciadorContexto<br/>sessão + estatísticas]
        B --> M[├── GerenciadorMemoria<br/>interface p/ Orquestrador]
        B --> T[├── PlanejadorTarefas<br/>fila priorizada]
        B --> A[├── Avaliador<br/>score 0-10]
        B --> MM[├── MonitorMetas<br/>metas dinâmicas]
        B --> G[├── GestorRecursos<br/>recursos thread-safe]
        B --> COA[├── CoAgente<br/>swarm inteligente]
        B --> REG[├── RegistroAgentes<br/>log de operações]
        B --> P[├── _pre<br/>PipelinePreCortex]
        B --> CX[├── _cortex<br/>Cortex]
        B --> LG[├── _logicas<br/>LogicasPipeline]
    end
```

---

## Arquivos e Responsabilidades

### `__init__.py` — Classe `Barramento` (215 linhas)

**Responsabilidade:** Orquestrador central. Coordena a ordem de processamento, conecta pipelines, gerencia eventos e expõe API unificada.

| Método | Função |
|---|---|
| `__init__(pipeline_pre, cortex_pipeline, logicas_pipeline, orquestrador, modelo)` | Cria todos os subcomponentes, registra recursos, configura rotas de eventos |
| `processar_entrada(texto, modelo, usuario)` | **Método principal**: executa Pre-Cortex → Cortex → Lógicas → Avaliação → Eventos |
| `avaliar_resposta(pergunta, resposta, modelo)` | Avalia qualidade da resposta e despacha evento |
| `processar_memoria_primeiro(texto)` | Tenta responder sem LLM (memória semântica ou procedural) |
| `processar_coagente(texto, contexto, usuario)` | Detecta intenções e delega para swarm de agentes |
| `feedback_reflexao(texto, console)` | Processa reflexão pós-resposta, retroalimenta pipelines |
| `resumo_completo()` | Coleta estatísticas de todos os subsistemas para diagnóstico |

**Eventos pub/sub registrados:**

| Evento | Callback | Gatilho |
|---|---|---|
| `entrada_recebida` | `_on_entrada()` → atualiza metas + log | Toda entrada do usuário |
| `reflexao_concluida` | `_on_reflexao()` → atualiza metas + sugestões | Após reflexão |
| `resposta_gerada` | `_on_resposta()` → placeholder | Após resposta do LLM |
| `ferramenta_chamada` | `_on_ferramenta()` → log | Quando ferramenta executada |

---

### `roteamento.py` — Classe `Roteamento` (32 linhas)

**Responsabilidade:** Sistema de eventos pub/sup para comunicação interna.

- `inscrever(evento, callback)` — Registra ouvinte para um evento
- `despachar(evento, dados)` — Notifica todos os ouvintes, captura exceções individualmente
- `eventos_disponiveis()` — Lista eventos com ouvintes
- `historico_recente(n)` — Últimos N eventos despachados

**Diagrama:**

```mermaid
graph LR
    EMISSOR[Qualquer componente] -->|despachar evento| RO[Barramento.roteamento]
    RO -->|callback 1| CB1[Ouvinte 1]
    RO -->|callback 2| CB2[Ouvinte 2]
    RO -->|callback N| CBN[Ouvinte N]
    RO -->|registro| HIST[Histórico]
```

---

### `gerenciador_contexto.py` — Classe `GerenciadorContexto` (44 linhas)

**Responsabilidade:** Unifica o acesso aos contextos do Pre-Cortex e Cortex, mantém estado da sessão atual.

- `registrar_troca(texto)` — Incrementa contador da sessão
- `estatisticas_combined()` — Métricas combinadas de Pre-Cortex (mensagens, entidades, stopwords) e Cortex (termos, coocorrências, análises)
- `resumo_sessao()` — Sessão atual (início, total de trocas, último assunto)

---

### `gerenciador_memoria.py` — Classe `GerenciadorMemoria` (31 linhas)

**Responsabilidade:** Fachada (Facade) para o `OrquestradorMemoria`. Abstrai toda a complexidade do sistema de memórias por trás de uma interface simples.

- `salvar_mensagem(papel, conteudo, modelo, usuario)` → `OrquestradorMemoria.processar_mensagem()`
- `buscar_contexto(consulta, n)` → `OrquestradorMemoria.buscar_contexto_memoria()`
- `refletir(msg_usuario, msg_assistente, modelo)` → `OrquestradorMemoria.refletir()`
- `estatisticas()` → `OrquestradorMemoria.estatisticas()`
- `encerrar(merges)` → `OrquestradorMemoria.encerrar()`
- `orquestrador` (property) → Acesso direto ao orquestrador subjacente

---

### `planejador_tarefas.py` — Classe `PlanejadorTarefas` (56 linhas)

**Responsabilidade:** Fila de tarefas internas com priorização. Ordena por prioridade decrescente.

- `agendar(nome, prioridade, dados)` — Adiciona tarefa com ID único e timestamp
- `executar_proxima()` — Remove e retorna tarefa de maior prioridade
- `concluir(id, resultado)` — Marca tarefa como concluída
- `processar_fila(limite)` — Executa N tarefas em lote
- `ultimas_executadas(n)` — Histórico das últimas N execuções

**Diagrama:**

```mermaid
graph LR
    ORIGEM[processar_entrada<br/>feedback_reflexao] -->|agendar| FILA[Fila Priorizada]
    FILA -->|ordenado por<br/>prioridade decrescente| EXEC[processar_fila]
    EXEC -->|concluir| HIST[Histórico<br/>executadas]
```

---

### `avaliador.py` — Classe `Avaliador` (78 linhas)

**Responsabilidade:** Avalia qualidade de entradas e respostas com scores de 0 a 10.

| Método | Critérios |
|---|---|
| `avaliar_entrada(texto, dados_pre)` | Importância média dos tokens (peso 0.3) + score do Pre-Cortex (peso 0.2) |
| `avaliar_resposta(pergunta, resposta, modelo)` | Média histórica (baseline) + bônus por tamanho adequado / penalidade por resposta muito curta |

**Fatores de ajuste na resposta:**
- > 2× o tamanho esperado → +1 (detalhada)
- < 0.3× o tamanho esperado → −2 (muito curta)
- Sem histórico → compara com tamanho da pergunta

---

### `monitor_metas.py` — Classe `MonitorMetas` (80 linhas)

**Responsabilidade:** Define e monitora metas de aprendizado, sugere ações ao usuário.

| Critério | Métrica | Exemplo de meta |
|---|---|---|
| `entidades` | `len(PreCortex.entidades_conhecidas)` | Aprender 8 entidades novas |
| `termos_cortex` | `len(Cortex.freq_termos)` | Acumular 50 termos no vocabulário |
| `mensagens` | `PreCortex.total_mensagens` | 20 mensagens trocadas |
| `coocorrencias` | Total em `Cortex.coocorrencias` | 30 pares de coocorrência |

**Sugestões automáticas** (ativadas após 4+ mensagens):
- Poucas entidades → "fale sobre seus interesses"
- Vocabulário baixo → "use palavras mais variadas"

**Metas dinâmicas:** Se `valor_alvo` não for definido, calcula automaticamente como `base + 5` onde `base = max(3, atual // 3)`.

---

### `gestor_recursos.py` — Classe `GestorRecursos` (36 linhas)

**Responsabilidade:** Registro centralizado e thread-safe de recursos do sistema.

| Nome do Recurso | Objeto | Tipo |
|---|---|---|
| `pre_cortex` | `PipelinePreCortex` | `pipeline` |
| `cortex` | `Cortex` | `pipeline` |
| `logicas` | `LogicasPipeline` | `pipeline` |
| `orquestrador` | `GerenciadorMemoria` | `memoria` |
| `coagente` | `CoAgente` | `orquestracao` |

- `registrar(nome, recurso, tipo)` — Adiciona com timestamp
- `obter(nome)` — Retorna o recurso, atualiza `ultimo_uso`
- `liberar(nome)` — Remove e retorna sucesso
- Thread-safe via `threading.Lock`

---

## Fluxo de Dados Completo

```mermaid
sequenceDiagram
    participant User as Usuário
    participant UI as interface.py
    participant BR as Barramento
    participant PC as Pre-Cortex
    participant CX as Cortex
    participant LG as Lógicas
    participant AV as Avaliador
    participant MM as MonitorMetas
    participant LLM as Modelo LLM

    User->>UI: texto
    UI->>BR: processar_entrada(texto, modelo)

    BR->>BR: contexto.registrar_troca(texto)
    BR->>PC: _pre.processar(texto, modelo)
    PC-->>BR: dados_pre

    alt dados_pre is None
        BR-->>UI: None (aborta)
    end

    BR->>CX: _cortex.processar(dados_pre)
    CX-->>BR: dados_cortex

    BR->>LG: _logicas.processar(texto, entidades)
    LG-->>BR: dados_logicas

    BR->>AV: avaliar_entrada(texto, dados_pre)
    AV-->>BR: {score, criterios}

    BR->>BR: tarefas.agendar("atualizar_metas")
    BR->>BR: tarefas.processar_fila()

    BR->>MM: despachar evento entrada_recebida
    MM->>MM: progresso(entidades, termos, coocorrencias, mensagens)

    BR-->>UI: {pre_cortex, cortex, logicas, avaliacao}

    UI->>LLM: constrói prompt com contexto + memórias
    LLM-->>UI: resposta gerada

    UI->>BR: avaliar_resposta(pergunta, resposta, modelo)
    BR->>AV: avaliar_resposta()
    AV-->>BR: {score, tamanho, modelo}
    BR-->>UI: resultado

    UI->>BR: feedback_reflexao(reflexao_texto)
    BR->>PC: feedback_reflexao(texto)
    BR->>CX: registrar_texto(texto)
    BR->>MM: despachar evento reflexao_concluida
    MM-->>UI: sugestões (se houver)

    UI-->>User: resposta formatada
```

---

## Fluxo Alternativo: Memória Primeiro

```mermaid
sequenceDiagram
    participant User as Usuário
    participant UI as interface.py
    participant BR as Barramento
    participant MP as Memória Primeiro
    participant MEM as OrquestradorMemoria

    User->>UI: texto
    UI->>BR: processar_memoria_primeiro(texto)

    BR->>MP: coagente.memoria_primeiro.consultar(texto)

    alt encontrou na memória semântica
        MP-->>BR: {fonte: "memoria", resposta, confiança}
        BR-->>UI: resposta direta
        UI-->>User: ✅ resposta sem LLM
    else não encontrou
        BR->>MP: coagente.memoria_procedural.consultar(texto)
        alt encontrou padrão procedural
            MP->>MP: verificar permissões → executar ferramenta
            MP-->>BR: {fonte: "procedural", resposta, confiança}
            BR-->>UI: resposta da ferramenta
            UI-->>User: ✅ resposta sem LLM
        else nada encontrado
            BR-->>UI: None → segue fluxo normal com LLM
        end
    end
```

---

## Integrações com Outros Subsistemas

```mermaid
graph TB
    subgraph Ghost["🧠 Ghost — Sistema Completo"]
        direction TB

        B[Barramento]
        I[interface.py]
        PC[Pre-Cortex]
        CX[Cortex]
        LG[Lógicas]
        COA[CoAgente]
        MEM[OrquestradorMemoria]
        TL[Tools]
        RG[Registro/Logger]

        I -->|cria e usa| B
        B -->|orquestra| PC
        B -->|orquestra| CX
        B -->|orquestra| LG
        B -->|cria| COA
        B -->|gera contexto| I
        B -->|memoria| MEM
        B -->|log de eventos| RG
        I -->|swarm exec| COA
        COA -->|ferramentas| TL
        MEM -->|Neo4j + Chroma| DB[(Neo4j<br/>ChromaDB<br/>SQLite)]
    end
```

### Mapa de Conexões

| Componente | Como se Conecta ao Barramento |
|---|---|
| **interface.py** | Cria `Barramento(...)` com os 3 pipelines + orquestrador. Chama `processar_entrada()`, `processar_memoria_primeiro()`, `processar_coagente()`, `avaliar_resposta()`, `feedback_reflexao()` |
| **Pre-Cortex** | Passado como `pipeline_pre` no construtor. Usa `_pre.processar()` e `_pre.feedback_reflexao()`. Seu `_ctx` alimenta `Avaliador` e `MonitorMetas` |
| **Cortex** | Passado como `cortex_pipeline`. Usa `_cortex.processar()`. Seu `_ctx` alimenta `GerenciadorContexto` e `MonitorMetas` |
| **Lógicas** | Passado como `logicas_pipeline`. Usa `_logicas.processar()`. Recebe a `consciência` do CoAgente via `logicas_pipeline.set_consciencia()` |
| **CoAgente** | Criado internamente com `CoAgente(modelo, orquestrador_memoria)`. Exposto como `self.coagente` |
| **OrquestradorMemoria** | Envolto em `GerenciadorMemoria`. Passado para CoAgente. Conecta a Neo4j, ChromaDB, SQLite |
| **Tools** | Acessadas indiretamente via CoAgente e `processar_memoria_primeiro()` |
| **Registro/Logger** | Importado e usado para log de eventos de entrada, ferramentas e reflexão |

---

## Padrões de Design Presentes

| Padrão | Implementação |
|---|---|
| **Facade** | `GerenciadorMemoria` abstrai toda a complexidade do `OrquestradorMemoria` |
| **Observer/Pub-Sub** | `Roteamento` com `inscrever()` e `despachar()` |
| **Mediator** | `Barramento` media a comunicação entre todos os subsistemas |
| **Fila Prioritizada** | `PlanejadorTarefas` ordena tarefas por prioridade |
| **Singleton indireto** | Um único `Barramento` por sessão do Ghost |
| **Adapter** | `GerenciadorContexto` unifica interfaces de `PreCortex._ctx` e `Cortex._ctx` |
| **Registry** | `GestorRecursos` mantém catálogo centralizado de recursos |

---

## Resumo

> **O Barramento é o cérebro operacional do Ghost.** Cada entrada do usuário passa por ele em 6 etapas sequenciais: contexto → Pre-Cortex → Cortex → Lógicas → Avaliação → Eventos. Ele não gera a resposta final (o LLM faz isso), mas prepara todo o terreno: analisa a entrada, extrai entidades, infere conceitos, avalia qualidade, gerencia metas, e alimenta o CoAgente com inteligência distribuída. Sem o Barramento, os pipelines seriam ilhas isoladas — ele é o sistema nervoso que os unifica.
