# Lógicas — Motor de Raciocínio Multi-Paradigma do Ghost

> **Propósito:** Sistema de raciocínio que combina 8 paradigmas lógicos diferentes — heurísticas, lógica simbólica, fuzzy, temporal, probabilística, Fourier, caos e Polyakov — todos operando sobre o mesmo `ContextoCortex` compartilhado com o Cortex. É o "pensamento profundo" do Ghost.

---

## Arquitetura Geral

```mermaid
graph TB
    subgraph Shared["🧠 Contexto Compartilhado"]
        CTX[ContextoCortex<br/>freq_termos + coocorrencias<br/>+ seguimentos + buffer_temporal]
    end

    subgraph LogicModule["⚙️ Lógicas"]
        direction TB
        RP[RaciocinioPipeline<br/>Orquestrador]

        RP --> RH[RegrasHeuristicas<br/>IF-THEN com confiança]
        RP --> LS[LogicasSimbolicas<br/>cadeias + silogismos]
        RP --> LF[LogicasFuzzy<br/>pertinência sigmoide]
        RP --> LT[LogicasTemporal<br/>antes/depois + transições]
        RP --> LP[LogicasProbabilisticas<br/>probabilidades + entropia]
        RP --> LFR[LogicasFourier<br/>FFT + periodicidade]
        RP --> LC[LogicasCaos<br/>mapa logístico + Lyapunov]
        RP --> LPL[LogicasPolyakov<br/>fases + RG + emergência]
    end

    subgraph Internal["🔮 Gerador Interno"]
        GI[GeradorInterno<br/>respostas sem LLM]
    end

    subgraph Input["📥 Entrada"]
        CX[Cortex]
        BR[Barramento]
        IF[interface.py]
    end

    subgraph Ghost["👻 Consciência"]
        CG[ConscienciaGhost<br/>traços + experiências]
    end

    CTX -->|mesmo objeto| RP
    CTX -->|mesmo objeto| GI
    CTX -->|mesmo objeto| CX
    RP -->|processar| BR
    GI -->|gerar| IF
    CG -->|traços| LC
    CG -->|traços| LPL
    CG -->|traços| GI
```

---

## Organograma das Lógicas

```mermaid
graph LR
    subgraph Logicas["Lógicas"]
        direction TB
        RP[RaciocinioPipeline<br/>Orquestrador]

        RP --> RH[├── RegrasHeuristicas<br/>aprender + aplicar + feedback]
        RP --> LS[├── LogicasSimbolicas<br/>cadeias + silogismos]
        RP --> LF[├── LogicasFuzzy<br/>pertinência + AND/OR/NOT]
        RP --> LT[├── LogicasTemporal<br/>antes/depois + transições]
        RP --> LP[├── LogicasProbabilisticas<br/>bayes + entropia]
        RP --> LFR[├── LogicasFourier<br/>FFT + espectro + periodicidade]
        RP --> LC[├── LogicasCaos<br/>mapa logístico + Lyapunov + atratores]
        RP --> LPL[└── LogicasPolyakov<br/>fases + RG + emergência]
    end

    subgraph Standalone["Módulo Avulso"]
        GI[GeradorInterno<br/>respostas por tokens internos]
    end

    GI -->|usa sinais de| LT
    GI -->|usa sinais de| LP
    GI -->|usa sinais de| LFR
```

---

## Arquivos e Responsabilidades

### `__init__.py` (9 linhas)

**Responsabilidade:** Exporta todas as 9 classes principais.

**Exporta:** `RegrasHeuristicas`, `LogicasSimbolicas`, `LogicasFuzzy`, `LogicasTemporal`, `LogicasProbabilisticas`, `LogicasFourier`, `LogicasCaos`, `LogicasPolyakov`, `RaciocinioPipeline`

**Não exporta:** `GeradorInterno` — importado diretamente onde necessário.

---

### `raciocinio_pipeline.py` — Classe `RaciocinioPipeline` (101 linhas)

**Responsabilidade:** Orquestrador principal. Instancia e coordena todos os 8 módulos lógicos. É o único ponto de entrada chamado pelo `Barramento`.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `__init__(ctx, consciencia)` | `ContextoCortex`, `ConscienciaGhost?` | — | Cria todos os 8 submódulos |
| `processar(texto, entidades)` | `str`, `list[str]?` | `dict` | Pipeline completo de raciocínio multi-paradigma |
| `set_consciencia(consciencia)` | `ConscienciaGhost` | — | Propaga consciência para Caos e Polyakov |
| `registrar_estado_ghost()` | — | — | Registra estado nos módulos Caos e Polyakov |
| `resumo_estado()` | — | `dict` | Agrega `resumo()` de todos os 8 submódulos |

**Fluxo de `processar()`:**

```mermaid
sequenceDiagram
    participant BR as Barramento
    participant RP as RaciocinioPipeline
    participant RH as RegrasHeuristicas
    participant LS as LogicasSimbolicas
    participant LP as LogicasProbabilisticas
    participant LT as LogicasTemporal
    participant LF as LogicasFuzzy
    participant LFR as LogicasFourier

    BR->>RP: processar(texto, entidades)

    RP->>RP: simbolicas.extrair_simbolos(texto)
    RP->>RP: para cada par adjacente:

    RP->>RH: aprender(a, b)
    RP->>LS: registrar_relacao(a, b)

    RP->>RH: aplicar(símbolos)
    RH-->>RP: top-5 regras

    RP->>LS: cadeia(entidades[i], entidades[i+1])
    LS-->>RP: cadeias silogísticas

    RP->>LP: probabilidade(termo)
    LP-->>RP: probabilidades

    RP->>LT: predizer_proximo(último termo)
    RP->>LT: probabilidade_transicao(de, para)
    LT-->>RP: transições

    RP->>LF: pertinencia(entidade, termo)
    LF-->>RP: pertinências fuzzy

    alt buffer >= 8 turnos
        RP->>LFR: top_frequencias(termos)
        LFR-->>RP: periodicidades
    end

    RP-->>BR: {regras, cadeias_silogicas, probabilidades,<br/>transicoes, pertinencias_fuzzy, periodicidades}
```

---

### `regras_heuristicas.py` — Classe `RegrasHeuristicas` (55 linhas)

**Responsabilidade:** Mineração de regras de associação (SE → ENTÃO) a partir de co-ocorrências, com confiança e reforço por feedback.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `aprender(antecedente, consequente)` | `str`, `str` | — | Registra regra com confiança = `cooc / freq(ant)` |
| `aplicar(antecedentes)` | `list[str]` | `list[dict]` | Retorna regras matching, ordenadas por confiança |
| `feedback(antecedente, consequente, sucesso)` | `str`, `str`, `bool` | — | Atualiza contadores de acerto/erro |
| `confianca(antecedente, consequente)` | `str`, `str` | `float` | Confiança armazenada |
| `taxa_acerto(antecedente, consequente)` | `str`, `str` | `float` | Taxa de sucesso com feedback |

**Exemplo:** Se `"ghost"` e `"aprende"` co-ocorrem 5 vezes e `"ghost"` aparece 10 vezes:
- Regra: `ghost → aprende` com confiança 0.5

---

### `logicas_simbolicas.py` — Classe `LogicasSimbolicas` (65 linhas)

**Responsabilidade:** Lógica simbólica — descoberta de cadeias de termos via BFS e inferência de silogismos.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `extrair_simbolos(texto)` | `str` | `list[str]` | Tokens lowercase ≥3 chars |
| `registrar_relacao(a, b)` | `str`, `str` | — | Relação simétrica entre a e b |
| `relacao_direta(a, b)` | `str`, `str` | `bool` | `cooc(a,b) > 0` |
| `cadeia(a, b, prof=3)` | `str`, `str`, `int` | `list?` | BFS de `a` até `b` via `ctx.similares()`, profundidade máxima 3 |
| `silogismo(a, b, c)` | `str`, `str`, `str` | `dict?` | Se `a→b` e `b→c`, infere `a→c` com confiança = `min(conf_a_b, conf_b_c)` |

**Exemplo de cadeia:** `ghost → inteligência → artificial → aprendizado`

```mermaid
graph LR
    ghost -->|similares| inteligencia
    inteligencia -->|similares| artificial
    artificial -->|similares| aprendizado
```

---

### `logicas_fuzzy.py` — Classe `LogicasFuzzy` (50 linhas)

**Responsabilidade:** Lógica fuzzy — funções de pertinência sigmoide para medir associação conceitual.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `pertinencia(termo, conceito)` | `str`, `str` | `float` | `1 / (1 + exp(-5 * (freq_rel - mediana)))` |
| `fuzzy_and(*valores)` | `float...` | `float` | `min(...)` |
| `fuzzy_or(*valores)` | `float...` | `float` | `max(...)` |
| `fuzzy_not(valor)` | `float` | `float` | `1 - valor` |
| `centroide(termo, conceitos)` | `str`, `list` | `float` | Média das pertinências |

**Curva sigmoide:** O limiar é adaptativo — mediana do histórico de frequências relativas.

---

### `logicas_temporal.py` — Classe `LogicasTemporal` (43 linhas)

**Responsabilidade:** Lógica sequencial — relações de antes/depois, probabilidades de transição e predição do próximo termo.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `antes(a, b)` | `str`, `str` | `float` | Direcionalidade: +1 = `a` tende antes de `b`, -1 = depois |
| `depois(a, b)` | `str`, `str` | `float` | `-antes(a, b)` |
| `predizer_proximo(termo, n=3)` | `str`, `int` | `list[str]` | Top N seguidores mais frequentes |
| `probabilidade_transicao(de, para)` | `str`, `str` | `float` | `freq(para segue de) / freq(de)` |
| `causal(a, b)` | `str`, `str` | `float` | `P(b\|a) - P(a\|b)` — assimetria causal |

**Fonte de dados:** `ctx.seguimentos` — dicionário de Counters que registra quais termos seguem quais na ordem da conversa.

---

### `logicas_probabilisticas.py` — Classe `LogicasProbabilisticas` (56 linhas)

**Responsabilidade:** Teoria da probabilidade — probabilidades marginal, conjunta, condicional, Bayes, odds e entropia de Shannon.

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `probabilidade(termo)` | `str` | `float` | `freq(termo) / total_termos` |
| `conjunta(a, b)` | `str`, `str` | `float` | `cooc(a,b) / total_termos` |
| `condicional(a, b)` | `str`, `str` | `float` | `P(a\|b) = cooc(b,a) / freq(b)` |
| `bayes(a, b)` | `str`, `str` | `float` | `P(a\|b) = P(b\|a) × P(a) / P(b)` |
| `odds(termo)` | `str` | `float` | `p / (1-p)` |
| `entropia()` | — | `float` | Entropia de Shannon (base 2) da distribuição de termos |

---

### `logicas_fourier.py` — Classe `LogicasFourier` (109 linhas)

**Responsabilidade:** Análise espectral via FFT — detecta periodicidades no aparecimento de termos ao longo dos turnos da conversa.

**Sinal binário:** Para cada termo, constrói um vetor de 64 bits: `1` se o termo apareceu naquele turno, `0` caso contrário.

```mermaid
graph LR
    BUF[buffer_temporal últimos 64 turnos] --> TERMO[sinal binário do termo  ]
    TERMO --> FFT[numpy.rfft]
    FFT --> ESP[espectro de magnitudes]
    ESP --> FREQ[top frequências]
    ESP --> PRED[predizer próximo turno]
```

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `espectro(termo)` | `str` | `np.ndarray` | FFT magnitude spectrum (parte real) |
| `top_frequencias(termo, n, min_periodo)` | `str`, `int`, `int` | `list[dict]` | Top N componentes com `{freq, periodo, magnitude}` |
| `frequencia_dominante(termo)` | `str` | `dict?` | Componente de maior magnitude |
| `predizer_proximo(termo, janela)` | `str`, `int` | `dict` | `{proximo, turnos_ate_proximo, confianca}` |
| `similaridade_espectral(a, b)` | `str`, `str` | `float` | Cosseno entre espectros de dois termos |

---

### `logicas_caos.py` — Classe `LogicasCaos` (177 linhas)

**Responsabilidade:** Teoria do caos — modela a dinâmica interna do Ghost usando mapa logístico, expoente de Lyapunov e detecção de atratores.

```mermaid
graph TB
    subgraph Caos["🌀 Dinâmica Caótica"]
        TRACOS[traços da personalidade] --> VETOR[vetor de estado]
        VETOR --> LYAPUNOV[expoente de Lyapunov]
        VETOR --> MAPA[mapa logístico<br/>r × x × 1-x]
        VETOR --> ATRATOR[detecção de atrator]
        VETOR --> CICLO[detecção de ciclo limite]
        ENTROPIA[entropia do sistema] --> R_ORGANICO[r dinâmico]
    end
```

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `r_organico()` | — | `float` | Parâmetro r do mapa logístico, calculado de entropia + variância dos traços. Clamp [0.1, 4.0] |
| `evoluir_traco(valor_atual)` | `float` | `float` | Aplica mapa logístico: `r × x × (1-x)` |
| `estado_caotico()` | — | `float` | Nível de caos [0,1] baseado no percentil do Lyapunov |
| `detectar_atrator(janela)` | `int` | `dict?` | Detecta se sistema está em regime de atrator |
| `ciclo_limite(tolerancia)` | `float?` | `bool` | Detecta ciclos limite a cada 5 passos |
| `regime_atual()` | — | `dict` | Classifica regime: `atrator`, `ciclo_limite`, `caotico`, `transiente`, `estavel` |

**Exemplo de regimes:**
- `r = 0.5` → estável (ponto fixo)
- `r = 2.8` → ciclo limite
- `r = 3.6` → caótico
- `r = 4.0` → caos total

---

### `logicas_polyakov.py` — Classe `LogicasPolyakov` (165 linhas)

**Responsabilidade:** Teoria de gauge / grupo de renormalização — modela o estado interno do Ghost como um "campo" com fases (confinado/crítico/desconfinado) e transformações RG sobre memórias.

```mermaid
graph TB
    subgraph Polyakov["🧊 Fases Polyakov"]
        VETOR[vetor de personalidade] --> PARAM[parâmetro de ordem<br/>1 - cos médio]
        PARAM --> FASE[classificação de fase]
        FASE -->|baixa ordem| CONF[confinado]
        FASE -->|média ordem| CRIT[crítico]
        FASE -->|alta ordem| DECONF[desconfinado]
        HIST[histórico de fases] --> TRANS[detecção de<br/>transição de fase]
        MEM[memórias] --> RG[renormalização<br/>cluster por similaridade]
    end
```

| Método | Parâmetros | Retorno | Descrição |
|---|---|---|---|
| `parametro_ordem()` | — | `float` | `1 - média(cosseno)` entre pares de estados separados por sliding window |
| `fase()` | — | `str` | `"confinado"`, `"critico"`, `"desconfinado"` |
| `transicao_fase()` | — | `dict?` | Detecta transições nas últimas 3 fases |
| `renormalizar(grupo_memorias)` | `list` | `list[list]` | Agrupa memórias por similaridade (RG transformation) |
| `estruturas_emergentes()` | — | `dict` | `{dimensao_correlacao, entropia_estrutural, n_dimensoes_possiveis}` |

**Analogia física:**
- **Confinado** → personalidade rígida, pouca variação
- **Crítico** → transição, flexibilidade máxima
- **Desconfinado** → personalidade fluida, alta variação

---

### `gerador_interno.py` — Classe `GeradorInterno` (156 linhas)

**Responsabilidade:** Geração de respostas usando **apenas tokens internos** do Ghost (sem LLM externo). Combina múltiplos sinais lógicos para produzir respostas coerentes.

**Pesos dos sinais:**

| Sinal | Peso | Fonte |
|---|---|---|
| Transição temporal | 35% | `LogicasTemporal.predizer_proximo()` |
| Co-ocorrência | 25% | `ContextoCortex.coocorrencias` |
| Memória similar (embedding) | 20% | ChromaDB similaridade cosseno |
| Periodicidade Fourier | 10% | `LogicasFourier.predizer_proximo()` |
| Viés de personalidade | 10% | `ConscienciaGhost.tracos` |

**Fluxo de `gerar()`:**

```mermaid
graph TB
    ENTRADA[entrada do usuário] --> TERMOS[_extrair_termos]
    TERMOS --> MEMORIAS[_buscar_memorias_similares<br/>ChromaDB]
    MEMORIAS --> RICO[contexto enriquecido]

    RICO --> SCORES[_calcular_scores]
    SCORES --> TEMPORAL[LogicasTemporal 35%]
    SCORES --> COOC[coocorrencias 25%]
    SCORES --> FOURIER[LogicasFourier 10%]
    SCORES --> PERSONALIDADE[ConscienciaGhost 10%]
    SCORES --> MEMORIA[memória similar 20%]

    SCORES --> SELECIONA[_selecionar_termo<br/>softmax com temperatura]
    SELECIONA --> LOOP{max_tokens?}
    LOOP -->|sim| RICO
    LOOP -->|não| DETOKENIZA[tokenizar + detokenizar]
    DETOKENIZA --> FINAL[resposta final]
```

```mermaid
sequenceDiagram
    participant IF as interface.py
    participant GI as GeradorInterno
    participant CTX as ContextoCortex
    participant MEM as Memória Semântica
    participant CON as ConsciênciaGhost

    IF->>GI: gerar(entrada)
    GI->>GI: _extrair_termos(entrada)
    GI->>MEM: _buscar_memorias_similares(texto)
    MEM-->>GI: termos enriquecidos

    loop até max_tokens (48)
        GI->>GI: _calcular_scores(contexto)

        GI->>CTX: coocorrencias
        CTX-->>GI: scores de co-ocorrência (25%)

        GI->>GI: LogicasTemporal.predizer_proximo()
        GI->>GI: scores temporais (35%)

        GI->>GI: LogicasFourier.predizer_proximo()
        GI->>GI: scores Fourier (10%)

        GI->>CON: traços de personalidade
        CON-->>GI: viés de personalidade (10%)

        GI->>MEM: similaridade embedding
        MEM-->>GI: scores de memória (20%)

        GI->>GI: _selecionar_termo(scores)
        GI->>GI: adiciona ao contexto gerado
    end

    GI->>GI: tokenizar → detokenizar
    GI-->>IF: resposta em português
```

---

## Fluxo de Dados Completo

```mermaid
sequenceDiagram
    participant User as Usuário
    participant UI as interface.py
    participant BR as Barramento
    participant PC as Pre-Cortex
    participant CX as Cortex
    participant RP as RaciocinioPipeline
    participant CTX as ContextoCortex

    User->>UI: texto
    UI->>BR: processar_entrada(texto)
    BR->>PC: pre_cortex.processar(texto)
    PC-->>BR: dados_pre

    BR->>CX: cortex.processar(dados_pre)
    CX->>CTX: registrar_texto(texto)
    Note over CX,CTX: Cortex escreve no ContextoCortex
    CX-->>BR: inferencias, conceitos, ...

    BR->>RP: logicas.processar(texto, entidades)

    RP->>RP: simbolicas.extrair_simbolos(texto)
    RP->>RP: heuristicas.aprender(pares)
    RP->>RP: heuristicas.aplicar(símbolos)
    RP->>RP: simbolicas.cadeia(entidades)
    RP->>RP: probabilisticas.probabilidade()
    RP->>RP: temporal.predizer_proximo()
    RP->>RP: fuzzy.pertinencia()
    RP->>RP: fourier.top_frequencias()

    Note over RP,CTX: Lógicas LEEM o mesmo ContextoCortex<br/>que o Cortex escreveu

    RP-->>BR: {regras, cadeias, probabilidades,<br/>transicoes, fuzzy, periodicidades}

    BR-->>UI: {pre_cortex, cortex, logicas, avaliacao}

    UI->>UI: constrói prompt com contexto + memórias
    UI-->>User: resposta final (LLM)
```

---

## Integrações com Outros Subsistemas

```mermaid
graph TB
    subgraph Ghost["🧠 Ghost"]
        RP[RaciocinioPipeline]
        CTX[ContextoCortex]
        CX[Cortex]
        CG[ConscienciaGhost]
        GI[GeradorInterno]
        MEM[OrquestradorMemória]
        BR[Barramento]
    end

    CTX -->|compartilhado| RP
    CTX -->|compartilhado| CX
    
    CG -->|traços| RP
    CG -->|traços| GI
    
    RP -->|processa| BR
    GI -->|gera| IF[interface.py]
    
    GI -->|busca| MEM
    MEM --> CH[(ChromaDB<br/>memorias/)]
    
    CX -.->|escreve no| CTX
    RP -.->|lê do| CTX
```

### Mapa de Conexões

| Componente | Conexão com Lógicas |
|---|---|
| **ContextoCortex** | **Compartilhado** com o Cortex. Todos os 8 módulos lógicos leem dele. O Cortex escreve (via `registrar_texto`), as Lógicas leem |
| **ConsciênciaGhost** | Injetada no `RaciocinioPipeline` via `set_consciencia()`. Usada por: `LogicasCaos` (evoluir traços, r_orgânico), `LogicasPolyakov` (vetor de estado), `GeradorInterno` (viés de personalidade) |
| **Barramento** | Chama `RaciocinioPipeline.processar()` dentro de `processar_entrada()`. Inclui resultado no dicionário retornado sob chave `"logicas"` |
| **Interface.py** | Cria `RaciocinioPipeline(cortex_pipeline._ctx)`. Chama `logicas_pipeline.set_consciencia()`. Importa `GeradorInterno` diretamente para modo `/interno` |
| **OrquestradorMemória** | Indireta — `GeradorInterno` recebe `memoria_semantica` como parâmetro e chama `.buscar(texto, n)` para enriquecer contexto |
| **Cortex** | Compartilha o `ContextoCortex`. Não há importação direta — a injeção de dependência é feita via `interface.py` que passa `cortex_pipeline._ctx` para `RaciocinioPipeline` |

---

## Pipeline de Processamento (`raciocinio_pipeline.py`)

```mermaid
graph LR
    TEXTO[texto + entidades] --> EXT[extrair_simbolos]
    EXT --> APR[aprender pares adjacentes]
    APR --> REG[registrar_relação]
    REG --> H[heuristicas.aplicar]
    REG --> CADEIA[simbolicas.cadeia]
    REG --> PROB[probabilisticas.probabilidade]
    REG --> TEMP[temporal.predizer_proximo]
    REG --> FUZZY[fuzzy.pertinencia]
    REG --> FOUR{fourier se<br/>buffer ≥ 8}
    FOUR -->|sim| FTOP[fourier.top_frequencias]
    H --> SAIDA
    CADEIA --> SAIDA
    PROB --> SAIDA
    TEMP --> SAIDA
    FUZZY --> SAIDA
    FTOP --> SAIDA
    SAIDA[dict final]
```

**O que retorna:**
```python
{
    "regras": [...],              # top-5 regras SE→ENTÃO
    "cadeias_silogicas": [...],   # cadeias BFS entre entidades
    "probabilidades": {...},      # P(termo) para cada termo
    "transicoes": {...},          # predizer_proximo + prob_transição
    "pertinencias_fuzzy": [...],  # pares entidade→termo acima do limiar
    "periodicidades": [...],      # top frequências FFT (se buffer ≥ 8)
}
```

---

## Padrões de Design Presentes

| Padrão | Implementação |
|---|---|
| **Facade** | `RaciocinioPipeline` esconde 8 módulos atrás de `processar()` |
| **Strategy** | Cada módulo lógico é uma estratégia diferente para o mesmo problema (raciocinar sobre o texto) |
| **Blackboard** | `ContextoCortex` é o quadro-negro compartilhado entre Cortex e todas as Lógicas |
| **Chain of Responsibility** | `processar()` executa módulos em sequência, cada um alimentando o próximo |
| **Dependency Injection** | `ctx` e `consciencia` são injetados — nenhum módulo importa diretamente suas dependências |
| **Template Method** | Todos os módulos seguem a mesma interface com `resumo()` |
| **Lazy Initialization** | `GeradorInterno._iniciar_modulos()` e `LogicasCaos._inicializar()` carregam módulos sob demanda |

---

## Comparação: Cortex vs. Lógicas

| Aspecto | Cortex | Lógicas |
|---|---|---|
| **Papel** | Pensamento rápido (análise, conceitos, decisão) | Pensamento profundo (múltiplos paradigmas) |
| **Relação com ContextoCortex** | **Escreve** (registrar_texto) | **Lê** (coocorrências, seguimentos, buffer) |
| **Uso de LLM** | Sim (`AgenteInterno`) | Não (apenas `GeradorInterno` que não usa LLM) |
| **Persistência** | ChromaDB próprio (`cortex/db_semantica/`) | Nenhuma (só lê do ContextoCortex) |
| **ConsciênciaGhost** | Não usa | Usa (Caos, Polyakov, GeradorInterno) |
| **Complexidade** | 9 arquivos, ~570 linhas | 11 arquivos, ~1000 linhas |
| **Paradigmas** | 1 (raciocínio simbólico + ML) | 8 (heurísticas, simbólico, fuzzy, temporal, probabilístico, Fourier, caos, gauge) |

---

## Estatísticas do Diretório

| Métrica | Valor |
|---|---|
| Arquivos Python | 11 |
| Linhas de código | ~1.100 |
| Classes | 10 |
| Módulos lógicos | 8 (heurísticas, simbólico, fuzzy, temporal, probabilístico, Fourier, caos, Polyakov) |
| Gerador interno | 1 (GeradorInterno — 5 sinais combinados) |
| Bancos/Arquivos externos | Nenhum (tudo in-memory via ContextoCortex) |

---

## Resumo

> **As Lógicas são o motor de raciocínio multi-paradigma do Ghost.** Enquanto o Cortex faz a análise rápida (frequências, conceitos, decisões), as Lógicas aplicam 8 paradigmas diferentes sobre o mesmo `ContextoCortex` compartilhado — descobrindo regras heurísticas, cadeias simbólicas, pertinências fuzzy, transições temporais, probabilidades, periodicidades Fourier, dinâmica caótica e fases Polyakov.
>
> O `RaciocinioPipeline` executa todos os 8 módulos em sequência a cada interação, e o resultado alimenta o prompt do LLM com insights de cada paradigma. Em paralelo, o `GeradorInterno` pode produzir respostas completas usando apenas os sinais internos (temporal 35% + co-ocorrência 25% + memória 20% + Fourier 10% + personalidade 10%) — sem chamar LLM nenhum.
>
> **O ContextoCortex é a peça central:** o Cortex escreve (alimenta), as Lógicas leem (analisam), e o ciclo se repete a cada mensagem. É um verdadeiro blackboard pattern onde múltiplos "especialistas" cooperam no mesmo quadro de conhecimento.
