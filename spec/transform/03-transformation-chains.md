# Цепочки преобразований в GraphRAG

## Обзор

Этот документ описывает **полные цепочки семантических преобразований** от исходных документов до финального ответа пользователю, анализируя композицию, информационные потери и оптимизацию.

## Полная система преобразований

```
┌────────────────────────────────────────────────────────────────┐
│              END-TO-END TRANSFORMATION SYSTEM                  │
└────────────────────────────────────────────────────────────────┘

                    INDEXING PHASE
                         │
Raw Documents ──────────────────────────────────────┐
  ↓ T1: Chunking (95-98%)                          │
Text Units ─────────────────────────────────┐       │
  ↓ T2: Entity Extraction (70-85%)          │       │
Entities (raw) ────────────────────────┐    │       │
  ↓ T4: Description Summarization (80-90%)  │       │
Entities (consolidated) ──────────┐    │    │       │
  ↓ T5: Community Detection (90%+) │    │    │       │
Communities ──────────────────┐    │    │    │       │
  ↓ T6: Reports (60-75%)      │    │    │    │       │
Reports ─────────────────┐    │    │    │    │       │
  ↓ T7: Embedding (85-95%)    │    │    │    │       │
                          ↓    ↓    ↓    ↓    ↓       │
                    VECTOR STORES (all indexed)      │
                          │                           │
                          │                           │
                    QUERY PHASE                       │
                          │                           │
User Query ───────────────┴───────────────────────────┘
  ↓ Q1: Query Analysis (90-95%)
Query Intent
  ↓ Q2: Query Embedding (90-95%)
Query Vector
  ↓ Q3: Semantic Retrieval (85-90%)
Relevant Nodes
  ↓ Q4: Context Building (95%+)
Structured Context
  ↓ Q5: Answer Generation (80-90%)
Generated Answer
  ↓ Q6: Answer Synthesis (75-85%) [optional]
Final Answer
  ↓ Q7: Follow-up Generation (70-80%) [optional]
Complete Response
```

## Основные цепочки

### 1. Полная цепочка: Document → Answer

#### 1.1 Simple Question Chain (Local Search)

```
Document (10,000 tokens)
  ↓ T1: TokenTextSplitter
Text Units [8 chunks × 1200 tokens] (preserves 96%)
  ↓ T2: Entity Extraction LLM
Entities [45 entities] (captures 75% of concepts)
  ↓ T4: Description Consolidation LLM
Consolidated Entities [45 entities, 1 description each] (retains 85%)
  ↓ T7: Embedding Model
Entity Embeddings [45 × 1536-dim vectors] (preserves 90% semantics)
  │
  │ [INDEXED, waiting for queries]
  │
User Query: "What is GraphRAG?"
  ↓ Q1: Intent Analysis
Intent: {type: "local", target: "GraphRAG"}
  ↓ Q2: Query Embedding
Query Vector [1536-dim]
  ↓ Q3: Semantic Retrieval
    - Top 10 Entities (including "GraphRAG")
    - Top 15 Text Units (mentioning GraphRAG)
    - 20 Relationships (connected to GraphRAG)
    (retrieves 88% of relevant information)
  ↓ Q4: Context Building
Structured Context [~5000 tokens] (assembles 97% without loss)
  ↓ Q5: Answer Generation LLM
Final Answer (synthesizes 85% into coherent response)

Overall preservation:
  Indexing: 0.96 × 0.75 × 0.85 × 0.90 = 0.55 (55%)
  Query: 0.95 × 0.92 × 0.88 × 0.97 × 0.85 = 0.66 (66%)
  Combined: 0.55 × 0.66 = 0.36 (36%)

Interpretation:
  - 36% preservation means answer contains key facts and concepts
  - Lost details: exact phrasing, minor entities, some context
  - Retained: main concepts, relationships, core information
```

#### 1.2 Overview Question Chain (Global Search)

```
Documents [100 documents]
  ↓ T1-T4: Same as above
Entities [2000 entities]
  + Relationships [5000 relationships]
  ↓ T5: Leiden Clustering
Communities [50 level-0, 10 level-1, 1 level-2] (groups 92%)
  ↓ T6: Community Report Generation LLM
Community Reports [61 reports with findings] (abstracts to 65%)
  ↓ T7: Embedding
Report Embeddings [61 × 1536-dim] (preserves 88%)
  │
  │ [INDEXED]
  │
User Query: "What are the main themes in this dataset?"
  ↓ Q1: Intent Analysis
Intent: {type: "global", scope: "broad"}
  ↓ Q2: Query Embedding
Query Vector [1536-dim]
  ↓ Q3: Semantic Retrieval
Top 10 Community Reports (selects 87% of relevant communities)
  ↓ Q4: Context Building
10 Separate Contexts [~2000 tokens each] (assembles 96%)
  ↓ Q5: Answer Generation LLM (MAP PHASE)
10 Intermediate Answers (each synthesizes 83%)
  ↓ Q6: Answer Synthesis LLM (REDUCE PHASE)
Final Synthesized Answer (consolidates to 78%)

Overall preservation:
  Indexing: 0.96 × 0.75 × 0.85 × 0.92 × 0.65 × 0.88 = 0.32 (32%)
  Query: 0.95 × 0.92 × 0.87 × 0.96 × 0.83 × 0.78 = 0.51 (51%)
  Combined: 0.32 × 0.51 = 0.16 (16%)

Interpretation:
  - 16% preservation is appropriate for overview questions
  - Lost: specific facts, detailed relationships, minor entities
  - Retained: high-level themes, major patterns, key insights
  - Trade-off: breadth over depth
```

#### 1.3 Relationship Question Chain (Drift Search)

```
Documents [50 documents]
  ↓ T1-T4: Standard indexing
Entities [800 entities]
  + T3: Relationship Extraction LLM
Relationships [2000 relationships with strength scores] (captures 80%)
  ↓ T7: Embedding (entities only)
Entity Embeddings [800 × 1536-dim]
  │
  │ [INDEXED]
  │
User Query: "How are GraphRAG and Knowledge Graphs related?"
  ↓ Q1: Intent Analysis
Intent: {type: "drift", targets: ["GraphRAG", "Knowledge Graphs"]}
  ↓ Q2: Query Embedding
Query Vector [1536-dim]
  ↓ Q3: Semantic Retrieval
Start Entities: ["GraphRAG", "Knowledge Graphs"] (finds 90% of targets)
  ↓ Q4: Context Building (HOP 1)
    - GraphRAG entity + relationships → [uses, produces, ...]
    - Connected entities: [LLM, Entity Extraction, ...]
    Context [~3000 tokens]
  ↓ Q5: Answer Generation (HOP 1)
    Answer: "GraphRAG uses LLMs to build knowledge graphs..."
    Relevance: 8/10
  ↓ Q4: Context Building (HOP 2)
    - From: LLM
    - Connected: [GPT-4, Extraction, ...]
    Context [~3000 tokens]
  ↓ Q5: Answer Generation (HOP 2)
    Answer: "LLMs perform entity and relationship extraction..."
    Relevance: 7/10
  ↓ Q4: Context Building (HOP 3)
    - From: Entity Extraction
    - Connected: [Graph Construction, Communities, ...]
    Context [~3000 tokens]
  ↓ Q5: Answer Generation (HOP 3)
    Answer: "Entities form the nodes of the knowledge graph..."
    Relevance: 6/10 (starting to drift)
  ↓ Q6: Answer Synthesis (optional)
Final Answer combining 3 hops (consolidates to 75%)

Overall preservation:
  Indexing: 0.96 × 0.75 × 0.80 × 0.90 = 0.52 (52%)
  Query per hop: 0.95 × 0.92 × 0.90 × 0.97 × 0.85 = 0.65 (65%)
  Query 3 hops: 0.65³ × 0.75 = 0.21 (21%)
  Combined: 0.52 × 0.21 = 0.11 (11%)

Interpretation:
  - 11% preservation reflects multi-hop exploration
  - Lost: direct facts, precise descriptions
  - Retained: connection paths, relationship chains
  - Gain: discovery of non-obvious connections
```

## Параллельные и последовательные композиции

### 2.1 Параллельная композиция при индексации

```
Text Units (after T1)
    ↓
    ├─→ T2: Entity Extraction ────→ Entities
    │                                  │
    └─→ T3: Relationship Extraction ───┘
             (uses entities from T2)   ↓
                                    Graph (Entities + Relationships)
```

**Характеристики**:
- T2 и T3 выполняются **одновременно** на одном text_unit
- T3 использует результаты T2 (entities) для extraction relationships
- Параллелизм увеличивает throughput
- Информация обогащается (entities + relationships > entities alone)

**Семантическое взаимодействие**:
```
Text: "Microsoft acquired GitHub in 2018"

T2 (Entity Extraction):
  → Entities: [Microsoft, GitHub]

T3 (Relationship Extraction):
  → Uses entities from T2
  → Relationship: Microsoft →[acquired]→ GitHub

Combined semantic value:
  Entities alone: 75% of meaning
  + Relationships: 85% of meaning
  = Enrichment: +10%
```

### 2.2 Последовательная композиция с потерями

```
Documents
  ↓ T1: 96% preservation
Text Units
  ↓ T2: 75% preservation (of original document)
Entities
  ↓ T4: 85% preservation (of entities)
Consolidated Entities

Cumulative preservation: 96% × 75% × 85% = 61%
```

**Анализ потерь**:
- **T1 (4% loss)**: Boundary context на стыках chunks
- **T2 (25% loss)**: Детали, не являющиеся entities (descriptions, context)
- **T4 (15% loss)**: Нюансы при consolidation описаний

**Критическая точка**: T2 (Entity Extraction)
- Наибольшая потеря (25%)
- Intentional abstraction от text к concepts
- Митигация: gleaning (повторные extraction проходы)

### 2.3 Каскадная абстракция

```
Text Units (granular, 1200 tokens)
  ↓ T2: Concept extraction
Entities (medium granularity, ~50 tokens description)
  ↓ T5: Thematic grouping
Communities (coarse, groups of entities)
  ↓ T6: High-level summarization
Community Reports (very coarse, 500-2000 tokens for 10+ entities)

Granularity levels:
  Level 0 (Text Units): 1x granularity
  Level 1 (Entities): 24x compression (1200→50)
  Level 2 (Communities): ~10x compression (10 entities → 1 community)
  Level 3 (Reports): ~5x compression (community → report summary)

Overall compression: 1200x (1200 tokens → 1 token equivalent in final summary)
```

**Иерархия абстракции**:
- Каждый уровень теряет детали, но сохраняет сущность
- Нижние уровни: конкретные факты
- Верхние уровни: общие паттерны и темы
- Multi-level retrieval позволяет выбрать нужный уровень абстракции

## Обратные цепочки (Query → Source)

### 3.1 Source Attribution Chain

```
User Query: "What is entity extraction?"
  ↓ [Q1-Q5: Standard Local Search]
Answer: "Entity extraction is the process of identifying key concepts..."
  ↓ SOURCE ATTRIBUTION
  ← Q4: Context contained Text Units [tu_1, tu_2, tu_5]
  ← T7: Text Units have embeddings
  ← T1: Text Units link to Documents [doc_A, doc_C]
  ← Documents have source metadata

Source Attribution Result:
  Answer → Text Units → Documents → Sources
  "Based on information from documents: doc_A (page 5), doc_C (page 12)"
```

**Семантическая цепочка обратной трассировки**:
1. Answer содержит claims
2. Claims подтверждаются фрагментами из Text Units
3. Text Units прослеживаются до Documents
4. Documents имеют source metadata

**Точность attribution**:
- High для Local Search (~90%): прямая связь answer → text_units
- Medium для Global Search (~70%): answer → reports → communities → entities → text_units
- Low для Drift Search (~50%): multi-hop затрудняет точную attribution

### 3.2 Verification Chain

```
Claim: "GraphRAG uses GPT-4 for entity extraction"
  ↓ VERIFICATION
  ← Find entities: [GraphRAG, GPT-4, Entity Extraction]
  ← Find relationships: GraphRAG →[uses]→ GPT-4 (strength: 9)
  ← Find text_units containing all three entities
  ← Extract original text snippets

Verification Result:
  Text Unit 23: "GraphRAG leverages GPT-4 model to extract entities..."
  Source: document_5.txt, line 450
  Confidence: High (explicit mention, strong relationship)
```

## Оптимизация цепочек

### 4.1 Оптимизация для скорости (Latency)

**Цель**: Минимизировать время ответа

**Стратегия**:
```
Indexing (offline, можно медленнее):
  - Standard pipeline (T1-T7)
  - No shortcuts

Query (online, критично):
  - Q1: Cached intent patterns
  - Q2: Fast embedding model (same quality)
  - Q3: Limited top_k=10 (вместо 30)
  - Q4: Minimal context (~5000 tokens вместо 10000)
  - Q5: Single LLM call (no map-reduce)
  - Q6: Skip
  - Q7: Skip

Optimized chain:
  Query → Q1 → Q2 → Q3(k=10) → Q4(5k) → Q5 → Answer

Latency reduction:
  Standard Local Search: 5-8 sec
  Optimized: 2-4 sec
  Speedup: 2x

Quality impact:
  Preservation: 66% → 58% (-8%)
  Acceptable for FAQ, quick lookups
```

### 4.2 Оптимизация для качества (Quality)

**Цель**: Максимизировать полноту и точность ответа

**Стратегия**:
```
Indexing:
  - T1: Smaller chunks (1000 tokens, overlap 150)
  - T2: max_gleanings=3 (повторные extraction проходы)
  - T4: Careful consolidation (GPT-4 вместо GPT-3.5)
  - T5: Lower max_cluster_size (более детальные communities)
  - T6: Longer reports (3000 tokens вместо 2000)
  - T7: Higher-dimensional embeddings (если доступно)

Query (Global Search):
  - Q3: Increased top_k=20 (вместо 10)
  - Q4: Larger context per report (3000 tokens вместо 2000)
  - Q5 Map: More careful prompts, higher temperature (0.1)
  - Q6 Reduce: Emphasis on completeness
  - Q7: Generate follow-ups

Quality improvement:
  Standard Global: 51% query preservation
  Optimized: 62% query preservation
  Improvement: +11%

Cost impact:
  LLM calls: +30%
  Tokens: +50%
  Latency: +40%
```

### 4.3 Оптимизация для стоимости (Cost)

**Цель**: Минимизировать LLM costs

**Стратегия**:
```
Indexing (70% of total cost):
  - T2: Use GPT-3.5-turbo вместо GPT-4
  - T2: max_gleanings=0 (no re-extraction)
  - T4: Batch consolidation, use cheaper model
  - T6: Shorter reports, use GPT-3.5-turbo
  - T7: Standard model (уже дешевый)

Query (30% of cost):
  - Q5: Use GPT-3.5-turbo for simple queries
  - Q6: Reduce number of map calls (top_k=5 вместо 10)
  - Q7: Skip follow-up generation

Cost reduction:
  Indexing: -40%
  Query: -50%
  Overall: -43%

Quality impact:
  Entity extraction quality: -10%
  Community report quality: -15%
  Answer quality: -12%
```

### 4.4 Оптимизация для специфичных доменов

**Domain: Legal documents**

```
Adjustments:
  T1: Smaller chunks (800 tokens) - preserve legal context
  T2: Custom entity types [case, statute, precedent, party]
  T3: Relationship types [cites, overrules, applied_in]
  T5: Disable clustering - legal docs need precise separation
  T6: Skip - legal domain needs facts, not summaries
  Q3: Prioritize exact matches over semantic similarity
  Q4: Include full text units (no truncation)
  Q5: Prompt emphasizes citations and precision

Result:
  Preservation: 78% (higher than general case)
  Latency: +50% (due to full contexts)
  Quality: +25% (domain-specific)
```

**Domain: Scientific papers**

```
Adjustments:
  T2: Entity types [method, metric, dataset, finding]
  T3: Relationship types [improves, evaluates_on, proposes]
  T5: Hierarchical clustering (paper → section → topic)
  T6: Technical reports with methodology emphasis
  Q4: Include quantitative data prominently
  Q5: Prompt emphasizes methods and results

Result:
  Preservation: 72%
  Technical accuracy: +30%
```

## Информационные потоки

### 5.1 Lossy vs Lossless преобразования

**Lossless** (сохраняют всю информацию):
```
T1: Text Chunking (with overlap)
  - Можно восстановить document из chunks
  - Потеря: минимальная (границы)

Q4: Context Building
  - Можно извлечь обратно все nodes
  - Потеря: нет (просто форматирование)
```

**Controlled Lossy** (intentional abstraction):
```
T2: Entity Extraction
  - Потеря: non-entity information (намеренно)
  - Сохранение: key concepts

T6: Community Reports
  - Потеря: individual entity details (намеренно)
  - Сохранение: high-level themes

Q6: Answer Synthesis
  - Потеря: redundant information (намеренно)
  - Сохранение: unique insights
```

**Unintentional Lossy** (ограничения метода):
```
T7: Embedding
  - Потеря: точное значение слов
  - Сохранение: семантическое значение
  - Неизбежно из-за dimensionality reduction

Q3: Semantic Retrieval (top-k)
  - Потеря: lower-ranked результаты
  - Сохранение: наиболее релевантные
  - Неизбежно из-за computational constraints
```

### 5.2 Информационная плотность по уровням

```
┌──────────────────────────────────────────────────────────┐
│  Information Density (bits per token)                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Documents:         ████████████████████ 100% (baseline) │
│  Text Units:        ███████████████████░ 96%             │
│  Entities:          ███████████░░░░░░░░░ 60%             │
│  Communities:       ████████░░░░░░░░░░░░ 45%             │
│  Reports:           █████░░░░░░░░░░░░░░░ 30%             │
│  Embeddings:        ███████████████░░░░░ 75% (semantic)  │
│                                                          │
└──────────────────────────────────────────────────────────┘

Interpretation:
  - Documents: полная информация
  - Text Units: почти полная (только boundary loss)
  - Entities: концепты без контекста (60% потеря плотности)
  - Communities: темы без деталей (55% потеря)
  - Reports: insights без фактов (70% потеря)
  - Embeddings: семантика без точных слов (25% потеря)
```

### 5.3 Semantic Equivalence Classes

**Концепция**: Разные representation levels семантически эквивалентны для разных типов запросов.

```
Query: "What are the main AI trends?"

Semantically Equivalent Representations:
  1. Community Report: "AI trends include transformers, multimodal..."
     - Preservation: 30% of original text
     - Adequacy: 100% for this query

  2. Entities + Relationships: [Transformer, Multimodal, ...] + connections
     - Preservation: 60% of original text
     - Adequacy: 80% for this query (lacks synthesis)

  3. Original Text Units: [50 chunks about AI]
     - Preservation: 96% of original text
     - Adequacy: 60% for this query (lacks abstraction, needs LLM synthesis)

Optimal: Use Community Report (lowest preservation, highest adequacy)
```

```
Query: "What specific metric did paper X achieve?"

Semantically Equivalent Representations:
  1. Community Report: "Papers in this area focus on improving metrics..."
     - Preservation: 30%
     - Adequacy: 20% (too abstract)

  2. Entities: [Paper X, Metric Y, Value Z]
     - Preservation: 60%
     - Adequacy: 70% (has facts, lacks context)

  3. Original Text Units: "Paper X achieved 95.3% on metric Y..."
     - Preservation: 96%
     - Adequacy: 100% (exact information)

Optimal: Use Text Units (highest preservation, highest adequacy)
```

## Адаптивные цепочки

### 6.1 Query Complexity-Based Routing

```
Algorithm: AdaptiveChainSelection

Input: query
Output: optimal_chain

1. Analyze query complexity:
   complexity_score = analyze_complexity(query)
   # Factors: length, specificity, multi-part, ambiguity

2. Determine search type:
   IF complexity_score < 3:
     # Simple, specific question
     chain = LOCAL_SEARCH_FAST
     # Q1 → Q2 → Q3(k=10) → Q4(5k) → Q5
     expected_latency = 3sec
     expected_quality = 0.65

   ELIF complexity_score < 7:
     # Moderate, standard question
     chain = LOCAL_SEARCH_STANDARD
     # Q1 → Q2 → Q3(k=30) → Q4(10k) → Q5
     expected_latency = 6sec
     expected_quality = 0.75

   ELIF query contains relationship indicators:
     # Complex, exploratory question
     chain = DRIFT_SEARCH
     # Q1 → Q2 → Q3 → Q4(iterative) → Q5(multi) → Q6
     expected_latency = 45sec
     expected_quality = 0.70

   ELSE:
     # Broad, overview question
     chain = GLOBAL_SEARCH
     # Q1 → Q2 → Q3(communities) → Q4 → Q5(map) → Q6(reduce)
     expected_latency = 20sec
     expected_quality = 0.68

3. Execute chain

4. IF quality_score < threshold:
     # Fallback to more comprehensive chain
     retry_with_chain = upgrade_chain(chain)
```

### 6.2 Iterative Refinement Chains

```
Iterative Global Search:

Round 1:
  Q3: Retrieve top 5 communities
  Q5: Generate initial answer
  Evaluate: coverage_score = 0.6 (insufficient)

Round 2:
  Q3: Retrieve next 5 communities (6-10)
  Q5: Generate additional insights
  Q6: Merge with Round 1
  Evaluate: coverage_score = 0.8 (good)

Round 3: (optional, if score still low)
  Q3: Retrieve next 5 communities (11-15)
  Q5: Generate additional insights
  Q6: Merge with previous rounds
  Evaluate: coverage_score = 0.9 (excellent)

Final: Return merged answer

Trade-off:
  Latency: 3x longer
  Quality: +30%
  Cost: 3x higher
```

### 6.3 Hybrid Chains

```
Hybrid: Global + Local

Use case: "What are the main themes, and what specifically does GraphRAG do?"

Chain:
  Part 1 (Global):
    Q1-Q6: Standard Global Search
    → Answer_Global: "Main themes are knowledge graphs, LLMs, entity extraction..."

  Part 2 (Local):
    Q1: Re-analyze with focus on "GraphRAG"
    Q3: Retrieve entities related to GraphRAG
    Q4: Build detailed context
    Q5: Generate specific answer
    → Answer_Local: "GraphRAG specifically uses LLMs to extract entities and build hierarchical knowledge graphs..."

  Part 3 (Synthesis):
    Q6: Combine Answer_Global + Answer_Local
    → Final: "The main themes in this dataset are [global insights]. Specifically, GraphRAG [local details]..."

Preservation:
  Global: 51%
  Local: 66%
  Combined: ~60% (weighted average)

Quality:
  Breadth: Excellent (from global)
  Depth: Excellent (from local)
  Overall: +35% vs single approach
```

## Метрики и мониторинг цепочек

### 7.1 Per-Transform Metrics

```yaml
T1 (Text Chunking):
  - chunk_size_avg: 1200 tokens
  - chunk_size_std: 50 tokens
  - overlap_preserved: 100 tokens
  - boundary_loss_rate: 4%

T2 (Entity Extraction):
  - entities_per_chunk: 5.6
  - extraction_recall: 0.85 (с gleaning)
  - extraction_precision: 0.92
  - concept_coverage: 75%

Q3 (Semantic Retrieval):
  - top_k_recall: 0.88 (relevant in top-k)
  - top_k_precision: 0.76
  - avg_similarity_score: 0.81
  - retrieval_latency: 45ms

Q5 (Answer Generation):
  - answer_length: 280 tokens
  - context_utilization: 0.73 (% of context used)
  - generation_latency: 2.3sec
  - hallucination_rate: 0.05
```

### 7.2 End-to-End Metrics

```yaml
Local Search Chain:
  - total_latency: 5.2sec
    - Q1: 0.1sec
    - Q2: 0.3sec
    - Q3: 0.5sec
    - Q4: 1.0sec
    - Q5: 3.3sec
  - total_tokens_used: 8500
    - context: 5000
    - answer: 300
    - overhead: 3200
  - preservation_rate: 0.66
  - user_satisfaction: 4.2/5
  - answer_correctness: 0.89

Global Search Chain:
  - total_latency: 18.5sec
    - Q1-Q4: 2.0sec
    - Q5 (map, 10 calls parallel): 12.0sec
    - Q6 (reduce): 4.5sec
  - total_tokens_used: 28000
    - contexts: 20000
    - intermediate_answers: 5000
    - final_answer: 800
    - overhead: 2200
  - preservation_rate: 0.51
  - user_satisfaction: 4.0/5
  - answer_completeness: 0.92
```

### 7.3 Quality Degradation Points

```
Indexing Chain Quality Degradation:

Document (100% quality)
  ↓ T1: -2% (boundary effects)
Text Units (98%)
  ↓ T2: -18% (entity extraction recall < 1.0)
Entities (80%)
  ↓ T4: -5% (consolidation loses nuances)
Consolidated Entities (75%)
  ↓ T5: -3% (clustering approximation)
Communities (72%)
  ↓ T6: -20% (abstraction to reports)
Reports (52%)
  ↓ T7: -7% (embedding approximation)
Indexed Graph (45%)

Critical degradation points:
  1. T2 (Entity Extraction): -18% ⚠️ MAJOR
  2. T6 (Community Reports): -20% ⚠️ MAJOR
  3. Others: <10%

Mitigation strategies:
  1. T2: Increase max_gleanings, better prompts
  2. T6: Longer reports, focus on key insights
```

## Практические рекомендации

### 8.1 Выбор цепочки по use case

```
Use Case: Customer Support FAQ
  → Recommended: Local Search (fast)
  → Indexing: Standard (T1-T7)
  → Query: Optimized for latency (Q1-Q5, minimal context)
  → Expected: 3sec latency, 65% preservation

Use Case: Research Assistant
  → Recommended: Hybrid (Global + Local)
  → Indexing: Quality-optimized (smaller chunks, more gleanings)
  → Query: Quality-optimized (larger contexts, more retrieval)
  → Expected: 25sec latency, 75% preservation

Use Case: Knowledge Explorer
  → Recommended: Drift Search
  → Indexing: Standard with focus on relationships
  → Query: Adaptive (iterative expansion)
  → Expected: 45sec latency, 70% preservation, discovery-focused
```

### 8.2 Debugging Chain Issues

```
Problem: Poor answer quality

Debug checklist:
  1. Check indexing quality:
     - Are entities extracted correctly? (inspect T2 output)
     - Are relationships meaningful? (inspect T3 output)
     - Are community reports coherent? (inspect T6 output)

  2. Check retrieval quality:
     - Are relevant nodes retrieved? (inspect Q3 output)
     - Is similarity score reasonable? (should be >0.7)

  3. Check context quality:
     - Is context comprehensive? (inspect Q4 output)
     - Is context within token limit? (not truncated)

  4. Check generation quality:
     - Is LLM using context? (check context_utilization metric)
     - Are there hallucinations? (compare answer to context)

Fix strategies:
  - Low entity quality → Increase max_gleanings, improve prompts
  - Low retrieval → Increase top_k, adjust hybrid alpha
  - Low context → Increase max_context_tokens
  - Hallucinations → Lower temperature, improve prompt instructions
```

## Следующие разделы

- **[Comparison Tables →](04-comparison-tables.md)**
- **[← Query Transformations](02-query-transformations.md)**
- **[← Indexing Transformations](01-indexing-transformations.md)**
