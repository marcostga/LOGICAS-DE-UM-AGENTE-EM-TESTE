# Pré-Cortex — Estágio Sensorial do Ghost

> **Propósito:** Primeiro estágio de processamento do Ghost. Recebe a entrada crua do usuário, pré-processa, filtra ruído, detecta padrões/emoções, pontua atenção e decide a prioridade do que merece ser processado adiante. É o "sistema sensorial" antes do Cortex.

---

## Arquitetura Geral

```mermaid
graph TB
    subgraph Input["📥 Entrada Crua"]
        RAW[texto do usuário]
    end

    subgraph PreCortex["🧪 Pré-Cortex"]
        direction TB
        PIP[PipelinePreCortex<br/>Orquestrador]
        CTX[ContextoAprendizado<br/>memória estatística]

        PIP --> R[Receptor<br/>receber + validar]
        PIP --> P[Preprocessador<br/>limpar + tokenizar]
        PIP --> F[FiltroRuido<br/>repetição + score ruído]
        PIP --> A[Atencao<br/>score + merece atenção]
        PIP --> D[DeteccaoPadroes<br/>tópicos + emoção + padrões]
        PIP --> PR[Priorizacao<br/>nível 1-5 + decisões]
    end

    subgraph Output["📤 Saída"]
        OUT[
          receptor: original, timestamp, ...,
          preprocessado: tokens, entidades, ...,
          filtro: valido, score_ruido, ...,
          atencao: score, merece_atencao,
          deteccao: topicos, emocao, ...,
          prioridade: nivel, decisoes, ...
        ]
    end

    subgraph Next["➡️ Próximo Estágio"]
        BR[Barramento<br/>processar_entrada]
    end

    RAW --> PIP
    CTX -.-> R
    CTX -.-> P
    CTX -.-> F
    CTX -.-> A
    CTX -.-> D
    CTX -.-> PR
    PIP --> OUT
    OUT --> BR
```

---

## Organograma do Pré-Cortex

```mermaid
graph LR
    subgraph PreCortex["🧪 Pré-Cortex"]
        direction TB
        CTX[ContextoAprendizado<br/>frequências + importâncias<br/>+ entidades + limiares]

        CTX --> R[├── Receptor<br/>validar + timestamp]
        CTX --> P[├── Preprocessador<br/>limpar + tokenizar]
        CTX --> F[├── FiltroRuido<br/>repetição + utilidade]
        CTX --> A[├── Atencao<br/>pontuar + limiar]
        CTX --> D[├── DeteccaoPadroes<br/>tópicos + emoção]
        CTX --> PR[└── Priorizacao<br/>nivel 1-5 + ações]

        PIP[PipelinePreCortex] --> R
        PIP --> P
        PIP --> F
        PIP --> A
        PIP --> D
        PIP --> PR
    end
```

---

## Arquivos e Responsabilidades

### `__init__.py` — Classe `PipelinePreCortex` (42 linhas)

**Responsabilidade:** Orquestrador. Instancia todas as 6 classes do Pré-Cortex com um `ContextoAprendizado` compartilhado e executa o pipeline sequencial.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `__init__()` | — | — | Cria `ContextoAprendizado()` + Receptor + Preprocessador + FiltroRuido + Atencao + DeteccaoPadroes + Priorizacao |
| `processar(entrada, modelo)` | `str`, `callable?` | `dict?` | Pipeline completo: receptor → preprocessador → filtro → atenção → detecção → priorização |
| `feedback_reflexao(texto)` | `str` | — | Passa texto de auto-reflexão para o `ContextoAprendizado` |

**Exporta:** `ContextoAprendizado`, `Receptor`, `Preprocessador`, `FiltroRuido`, `Atencao`, `DeteccaoPadroes`, `Priorizacao`, `PipelinePreCortex`

**Fluxo de `processar()`:**

```mermaid
sequenceDiagram
    participant BR as Barramento
    participant PIP as PipelinePreCortex
    participant R as Receptor
    participant P as Preprocessador
    participant F as FiltroRuido
    participant A as Atencao
    participant D as DeteccaoPadroes
    participant PR as Priorizacao
    participant CTX as ContextoAprendizado

    BR->>PIP: processar(entrada, modelo)

    PIP->>R: receber(entrada, modelo)
    R-->>PIP: {original, timestamp, modelo, ...}

    alt entrada vazia
        R-->>PIP: None
        PIP-->>BR: None (aborta)
    end

    PIP->>P: processar(texto)
    P->>CTX: registrar(tokens)
    P->>CTX: registrar_entidade(e)
    P-->>PIP: {limpo, tokens, entidades, ...}

    PIP->>F: filtrar(dados_pre)
    F->>CTX: is_stopword, score_importancia
    F-->>PIP: {valido, score_ruido, tokens_uteis, ...}

    PIP->>A: pontuar(dados_receptor, dados_pre)
    A->>CTX: score_importancia, is_interrogative
    A->>CTX: append historico_scores
    A-->>PIP: {score, merece_atencao}

    PIP->>D: analisar(dados_receptor, dados_pre)
    D->>CTX: score_importancia, entidades_conhecidas
    D-->>PIP: {topicos, emocao, topicos_recorrentes, ...}

    PIP->>PR: decidir(R, P, F, A, D)
    PR->>CTX: limiar_nivel, limiar_ruido
    PR-->>PIP: {nivel, score_final, decisoes, acoes}

    PIP-->>BR: {receptor, preprocessado, filtro,<br/>atencao, deteccao, prioridade}
```

---

### `contexto.py` — Classe `ContextoAprendizado` (105 linhas)

**Responsabilidade:** Núcleo estatístico do Pré-Cortex. Mantém frequências, importâncias, entidades conhecidas, polaridade, históricos de scores e limiares adaptativos. **É o único estado persistente entre chamadas** — todos os outros módulos dependem dele.

| Atributo | Tipo | Descrição |
|---|---|---|
| `freq_palavras` | `Counter` | Frequência bruta de cada palavra |
| `freq_docs` | `Counter` | Frequência em documentos (quantas mensagens cada termo aparece) |
| `importancia` | `defaultdict(float)` | Score de importância acumulado por palavra |
| `entidades_conhecidas` | `set` | Entidades com letra maiúscula já registradas |
| `total_mensagens` | `int` | Contador de mensagens processadas |
| `historico_scores` | `list` | Histórico de scores de atenção |
| `historico_tamanhos` | `list` | Histórico de tamanhos de entrada |
| `freq_em_perguntas` | `Counter` | Frequência de cada termo em perguntas |
| `polaridade` | `defaultdict(float)` | Polaridade acumulada por palavra |
| `historico_score_ruido` | `list` | Histórico de scores de ruído |
| `_pesos_aprendidos` | `dict` | Pesos: idf=0.5, entidade=2.0, reflexao=2.0, registro=1.0 |

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `registrar(tokens)` | `list[str]` | — | Incrementa `freq_palavras` e `freq_docs` |
| `registrar_pergunta(tokens)` | `list[str]` | — | Marca tokens que aparecem em perguntas |
| `registrar_entidade(palavra)` | `str` | — | Adiciona ao `entidades_conhecidas`, +1.0 de importância |
| `feedback_reflexao(texto)` | `str` | — | Auto-reflexão: +2.0 de importância para palavras-chave |
| `is_stopword(palavra)` | `str` | `bool` | Stopword se `freq_docs(p) / total > 0.7` |
| `score_importancia(palavra)` | `str` | `float` | `importancia + 2.0(se entidade) + idf * 0.5` |
| `is_interrogative(palavra)` | `str` | `bool` | Voto se >50% das ocorrências foram em perguntas |
| `aprender_polaridade(palavra, delta)` | `str`, `float` | — | Acumula polaridade |
| `score_polaridade(palavra)` | `str` | `float` | Polaridade acumulada |
| `limiar_atencao()` | — | `float` | Percentil 60 do histórico de scores, mínimo 0.5 |
| `limiar_min_tokens()` | — | `int` | Percentil 20 dos tamanhos históricos, mínimo 1 |
| `limiar_nivel(percentil)` | — | `float` | Percentil `p * 20` dos scores históricos |
| `limiar_ruido_baixo()` | — | `float` | Percentil 30 do histórico de ruído, ou 0.3 |
| `limiar_ruido_medio()` | — | `float` | Percentil 60 do histórico de ruído, ou 0.6 |

---

### `receptor.py` — Classe `Receptor` (19 linhas)

**Responsabilidade:** Porta de entrada. Valida se há conteúdo, extrai metadados da entrada.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `receber(entrada, modelo)` | `str`, `callable?` | `dict?` | `None` se vazio; senão: `{original, timestamp, modelo, tamanho, e_comando}` |

**Saída:**
```python
{
    "original": "texto do usuário",
    "timestamp": "2026-05-26T12:00:00",
    "modelo": None,           # modelo de linguagem opcional
    "tamanho": 15,           # len(texto)
    "e_comando": False,       # começa com "/"?
}
```

---

### `preprocessador.py` — Classe `Preprocessador` (42 linhas)

**Responsabilidade:** Limpeza, tokenização, extração de entidades e filtragem de stopwords. **Escreve no `ContextoAprendizado`** (`registrar`, `registrar_entidade`).

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `limpar(texto)` | `str` | `str` | Trim, normaliza espaços, limita a 1024 chars |
| `tokenizar(texto)` | `str` | `list[str]` | Regex `\b[^\W\d_]+\b` → lowercase |
| `filtrar_stopwords(tokens)` | `list[str]` | `list[str]` | Remove stopwords do ContextoAprendizado |
| `entidades(texto)` | `str` | `list[str]` | Palavras com maiúscula: `[A-Z][a-z]{2,}` |
| `processar(texto)` | `str` | `dict` | Pipeline completo de pré-processamento |

**Saída de `processar()`:**
```python
{
    "original": "texto",
    "limpo": "texto limpo",
    "tokens": ["texto", "limpo"],           # todas as palavras
    "tokens_uteis": ["texto"],              # sem stopwords
    "entidades": ["Ghost"],
    "total_tokens": 2,
    "total_uteis": 1,
}
```

---

### `filtro_ruido.py` — Classe `FiltroRuido` (74 linhas)

**Responsabilidade:** Filtragem adaptativa de ruído. Avalia taxa de repetição, utilidade dos termos e pontua quanto "ruído" a entrada tem. **Não descarta mensagens — só pontua o nível de ruído.**

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `filtrar(dados_pre)` | `dict` | `dict` | `{valido, motivo, score_ruido, tokens_uteis}` |

**Lógica de score_ruído:**

```mermaid
graph TD
    TOKENS[tokens] --> MIN[len tokens < limiar_min_tokens?]
    MIN -->|sim| RUIM[valido=False, score_ruido=1.0]
    MIN -->|não| REP[taxa de repetição 1 - set/tokens]

    REP --> LIMIAR[repetição > percentil 90?]
    LIMIAR -->|sim| SCORE_REP[score_ruido = repetição]

    UTEIS[tokens_uteis] --> ENT[tem entidades<br/>conhecidas?]
    ENT -->|sim| SCORE_ENT[score_ruido = max score, 1 - score_médio / max_score]
    ENT -->|não| PEN[penalidade adicional +percentil 30 do histórico]

    SCORE_REP --> FINAL
    SCORE_ENT --> FINAL
    PEN --> FINAL
    FINAL[score_ruido final<br/>append no histórico]
```

**Exemplo adaptativo:**
- Entrada `"aaaa aaaa aaaa aaaa aaaa"` → `score_ruido ~1.0, repetitivo`
- Entrada `"Ghost aprende conceitos novos"` → `score_ruido ~0.0, válido`
- Entrada `"é uma coisa que talvez"` → `score_ruido ~0.3` (sem entidades)

---

### `atencao.py` — Classe `Atencao` (56 linhas)

**Responsabilidade:** Pontua a relevância da mensagem com base em entidades, vocabulário, tamanho e se é pergunta. Decide se merece atenção do sistema.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `pontuar(dados_receptor, dados_pre)` | `dict`, `dict` | `dict` | `{score, merece_atencao}` |

**Cálculo do score:**

| Fator | Peso | Detalhe |
|---|---|---|
| Entidades conhecidas | 1.5× | Soma dos `score_importancia` das entidades |
| Vocabulário | 1.0× | Média dos `score_importancia` dos tokens úteis |
| Tamanho | 0.5-2.0× | Adaptativo: ≤p20=0.5, ≥p80=2.0 |
| Pergunta | +2.0 | Se tem `?` ou palavra interrogativa conhecida |

**Decisão:** `merece_atencao = score >= ContextoAprendizado.limiar_atencao()` (percentil 60 dos scores históricos).

---

### `deteccao.py` — Classe `DeteccaoPadroes` (97 linhas)

**Responsabilidade:** Detecção de tópicos, emoção/intensidade, padrões emocionais e co-ocorrências entre tópicos ao longo do tempo.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `analisar(dados_receptor, dados_pre)` | `dict`, `dict` | `dict` | `{topicos, emocao, topicos_recorrentes, padrao_emocional}` |
| `reset()` | — | — | Limpa todos os históricos |

**Detecção de emoção (5 níveis):**

```mermaid
graph LR
    TEXTO[texto + tokens_úteis] --> PONT[intensidade 0-5]
    PONT --> NIVEL{compara com<br/>percentis históricos}

    subgraph Sinais
        A1[? → +1]
        A2[! → +N]
        A3[UPPERCASE → +1]
        A4[score_importância > p80 → +1]
        A5[score_importância > p80*0.5 → +1]
    end

    NIVEL -->|≥ 80% max| URG[urgente]
    NIVEL -->|≥ 60% max| ALT[alta]
    NIVEL -->|≥ 40% max| MED[média]
    NIVEL -->|≥ 20% max| BAI[baixa]
    NIVEL -->|resto| NEUT[neutro]
```

**Saída:**
```python
{
    "topicos": ["inteligência", "artificial"],
    "emocao": "baixa",
    "topicos_recorrentes": ["ghost", "aprender", "código"],
    "padrao_emocional": [("neutro", 12), ("baixa", 3)],
}
```

---

### `priorizacao.py` — Classe `Priorizacao` (50 linhas)

**Responsabilidade:** Decisão final: combina score de atenção e score de ruído para definir um nível de 1 a 5, e aciona decisões binárias.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `decidir(R, P, F, A, D)` | 5 dicts | `dict` | `{nivel, score_final, score_ruido, decisoes, acoes}` |

**Matriz de Nível:**

| Nível | Condição | Ações |
|---|---|---|
| **5** | score ≥ n5 **E** ruído < baixo (ou emoção urgente) | Neo4j + Grafo + SQLite + Refletir + Feedback |
| **4** | score ≥ n4 **E** ruído < médio | Neo4j + Grafo + SQLite + Refletir + Feedback |
| **3** | score ≥ n3 **E** ruído < médio | Neo4j + Grafo + SQLite + Refletir |
| **2** | score ≥ n2 | Grafo + SQLite |
| **1** | default | SQLite apenas |

**Decisões mapeadas:**
```python
{
    "neo4j": nivel >= 3,        # Armazenar em banco grafo
    "grafo_ativo": nivel >= 2,  # Manipular grafo em memória
    "sqlite": True,             # Sempre persistir
    "refletir": nivel >= 3,     # Auto-reflexão
    "feedback": nivel >= 4,     # Feedback aprendizado
}
```

---

## Fluxo de Dados Completo

```mermaid
sequenceDiagram
    participant U as Usuário
    participant BR as Barramento
    participant PIP as PipelinePreCortex
    participant CTX as ContextoAprendizado

    U->>BR: "Ghost, o que é inteligência artificial?"
    BR->>PIP: processar("Ghost, o que é inteligência artificial?")

    PIP->>PIP: Receptor.receber()
    Note over PIP: {original, timestamp, tamanho=40, e_comando=False}

    PIP->>PIP: Preprocessador.processar()
    Note over PIP: {limpo, tokens:[ghost,o,que,é,inteligência,artificial],<br/>entidades:[Ghost]}

    PIP->>CTX: registrar(tokens)
    PIP->>CTX: registrar_entidade("ghost")

    PIP->>PIP: FiltroRuido.filtrar()
    Note over PIP: {valido:True, score_ruido:0.0,<br/>tokens_uteis:[inteligência,artificial]}

    PIP->>PIP: Atencao.pontuar()
    Note over PIP: {score:6.5, merece_atencao:True}

    PIP->>PIP: DeteccaoPadroes.analisar()
    Note over PIP: {topicos:[inteligência,artificial],<br/>emocao:"media", ...}

    PIP->>PIP: Priorizacao.decidir()
    Note over PIP: {nivel:5, decisoes:{neo4j:True, grafo_ativo:True,<br/>sqlite:True, refletir:True, feedback:True}}

    PIP-->>BR: {receptor, preprocessado, filtro,<br/>atencao, deteccao, prioridade}

    BR->>BR: prossegue para o Cortex...
```

---

## Integrações com Outros Subsistemas

```mermaid
graph TB
    subgraph Ghost["🧠 Ghost - Fluxo Sensorial"]
        PC[PreCortex<br/>PipelinePreCortex]
        CTX[ContextoAprendizado]
        BR[Barramento]
        CX[Cortex]
        LP[Lógicas]
        IF[interface.py]
    end

    IF -->|"entrada do usuário"| BR
    BR -->|"processar(entrada)"| PC
    PC -->|"resultado: 6 estágios"| BR
    BR -->|"dados + pre_cortex"| CX
    BR -->|"dados + pre_cortex"| LP
    
    PC -.->|"regista tokens/entidades"| CTX
    CX -.->|"feedback_reflexao"| PC
```

### Mapa de Conexões

| Componente | Conexão com Pré-Cortex |
|---|---|
| **Barramento** | Chama `PipelinePreCortex.processar()` e recebe o dict completo de 6 estágios. Inclui `"pre_cortex"` como chave no retorno de `processar_entrada()` |
| **Cortex** | Recebe os dados do Pré-Cortex via Barramento. Também chama `PipelinePreCortex.feedback_reflexao()` após auto-reflexão para ajustar importâncias |
| **Lógicas** | Recebem os dados indiretamente — o Barramento passa os resultados do Pré-Cortex junto com os do Cortex para o prompt final |
| **interface.py** | Não chama o Pré-Cortex diretamente — tudo passa pelo Barramento. O prompt final inclui metadados do Pré-Cortex (nível de atenção, emoção, tópicos) |
| **Persistência** | Nenhuma direta. O `ContextoAprendizado` vive em memória dentro do `PipelinePreCortex`, que é criado pelo `Barramento` e vive enquanto o processo durar |

---

## Pipeline de Processamento Visual

```mermaid
graph LR
    ENTRADA[entrada crua] --> R[Receptor<br/>validar + timestamp]
    R --> P[Preprocessador<br/>limpar + tokenizar + entidades]
    P --> F[FiltroRuido<br/>repetição + score 0-1]
    P --> A[Atencao<br/>score numérico + limiar]
    P --> D[DeteccaoPadroes<br/>tópicos + emoção 0-5]
    F --> PR[Priorizacao<br/>nível 1-5 + decisões]
    A --> PR
    D --> PR
    PR --> SAIDA[dict final 6 estágios]
```

---

## Padrões de Design Presentes

| Padrão | Implementação |
|---|---|
| **Pipeline** | `PipelinePreCortex.processar()` executa 6 estágios em sequência, cada um recebendo dados do anterior |
| **Facade** | `PipelinePreCortex` esconde todas as 6 classes atrás de `processar()` |
| **Strategy** | Cada estágio encapsula uma estratégia diferente de análise (recepção, limpeza, filtro, atenção, detecção, priorização) |
| **Blackboard** | `ContextoAprendizado` é o quadro-negro compartilhado entre todos os estágios — todos leem e alguns escrevem |
| **Adapter** | O Pré-Cortex inteiro é um adaptador entre a entrada crua do usuário e a representação estruturada que o Cortex/Lógicas consomem |
| **Template Method** | Todos os módulos recebem `contexto` no `__init__` e expõem métodos públicos que operam sobre ele |
| **Stateful Computation** | `ContextoAprendizado` mantém estado acumulado (frequências, limiares adaptativos, históricos) que evolui a cada mensagem |

---

## Estatísticas do Diretório

| Métrica | Valor |
|---|---|
| Arquivos Python | 8 |
| Linhas de código | ~485 |
| Classes | 7 |
| Estágios do pipeline | 6 (Receptor → Preprocessador → FiltroRuido → Atencao → DeteccaoPadroes → Priorizacao) |
| Dependências externas | `re`, `math`, `collections` (stdlib) |
| Estado compartilhado | `ContextoAprendizado` (injetado em todos) |

---

## Resumo

> **O Pré-Cortex é o sistema sensorial do Ghost.** É a primeira coisa que toca a entrada do usuário — validando, limpando, pontuando, filtrando ruído, detectando emoção e decidindo o nível de prioridade.
>
> Cada mensagem passa por 6 estágios: o **Receptor** valida e timestamps, o **Preprocessador** limpa e tokeniza (e já alimenta o `ContextoAprendizado` com frequências), o **FiltroRuido** calcula um score adaptativo de ruído baseado em repetição e utilidade, a **Atencao** pontua relevância (entidades, vocabulário, pergunta), a **DeteccaoPadroes** extrai tópicos e classifica emoção em 5 níveis, e a **Priorizacao** combina tudo num nível 1-5 que decide quais sistemas serão acionados (Neo4j, grafo ativo, SQLite, reflexão, feedback).
>
> **O `ContextoAprendizado` é o cérebro estatístico do Pré-Cortex** — mantém frequências, importâncias, entidades conhecidas, polaridade e limiares adaptativos. Todos os estágios leem dele, e o Preprocessador escreve (registra tokens e entidades). Os limiares evoluem com o tempo: o que é "atenção merecida" hoje pode não ser amanhã, conforme o Ghost aprende o que é normal.
>
> **É o único subsistema que não usa banco de dados ou LLM** — tudo é estatístico, em memória, e extremamente rápido. Uma mensagem passa pelo pipeline inteiro em microssegundos.
>
> **Mas atenção:** o `ContextoAprendizado` é resetado se o processo reiniciar. Não há persistência entre sessões. É pura memória de curto-prazo estatística.
