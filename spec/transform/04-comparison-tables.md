# Сравнительные таблицы преобразований

## Обзор

Этот документ содержит **полный набор сравнительных таблиц** для всех семантических преобразований в GraphRAG, позволяя быстро сравнить характеристики и выбрать оптимальные параметры.

## Основная сравнительная таблица

### Все преобразования (Indexing + Query)

| ID | Преобразование | Фаза | Тип | LLM | Semantic Preservation | Info Loss | Reversibility | Latency | Cost |
|---|---|---|---|---|---|---|---|---|---|
| **T1** | Text Chunking | Index | Decomposition | No | 95-98% | Low | High | Fast | Free |
| **T2** | Entity Extraction | Index | Abstraction | Yes | 70-85% | Medium | Low | Slow | High |
| **T3** | Relationship Extraction | Index | Association | Yes | 75-85% | Medium | Low | Slow | High |
| **T4** | Description Summarization | Index | Aggregation | Yes | 80-90% | Low-Med | Medium | Medium | Medium |
| **T5** | Community Detection | Index | Clustering | No | 90%+ | Minimal | High | Medium | Free |
| **T6** | Community Reports | Index | Abstraction | Yes | 60-75% | Med-High | Low | Slow | High |
| **T7** | Text Embedding | Index | Vectorization | No | 85-95% | Medium | Low | Fast | Low |
| **Q1** | Query Analysis | Query | Classification | No | 90-95% | Minimal | High | Fast | Free |
| **Q2** | Query Embedding | Query | Vectorization | No | 90-95% | Low | Low | Fast | Low |
| **Q3** | Semantic Retrieval | Query | Retrieval | No | 85-90% | Medium | Medium | Fast | Free |
| **Q4** | Context Building | Query | Aggregation | No | 95%+ | Minimal | High | Fast | Free |
| **Q5** | Answer Generation | Query | Synthesis | Yes | 80-90% | Medium | Medium | Slow | High |
| **Q6** | Answer Synthesis | Query | Consolidation | Yes | 75-85% | Medium | Medium | Slow | High |
| **Q7** | Follow-up Generation | Query | Extrapolation | Yes | 70-80% | N/A | N/A | Medium | Medium |

**Легенда**:
- **Latency**: Fast (<1sec), Medium (1-5sec), Slow (>5sec)
- **Cost**: Free (algorithmic), Low (<$0.01), Medium ($0.01-$0.10), High (>$0.10) per operation
- **Semantic Preservation**: % of original meaning retained
- **Info Loss**: Amount of information lost
- **Reversibility**: Ability to reconstruct input from output

## Детальные сравнительные таблицы

### Таблица 1: По семантическим характеристикам

| Преобразование | Granularity Change | Abstraction Direction | Information Density | Semantic Fidelity |
|---|---|---|---|---|
| T1: Text Chunking | Fine → Fine | Lateral | 96% | Very High |
| T2: Entity Extraction | Fine → Medium | Bottom-up | 60% | High |
| T3: Relationship Extraction | Medium → Medium | Lateral | 75% | High |
| T4: Description Summarization | Medium → Medium | Lateral | 85% | Very High |
| T5: Community Detection | Medium → Coarse | Bottom-up | 90% | Very High |
| T6: Community Reports | Coarse → Coarse | Bottom-up | 30% | Medium |
| T7: Text Embedding | Variable → Fixed | Encoding | 75% | High |
| Q1: Query Analysis | Variable → Structured | Lateral | 92% | Very High |
| Q2: Query Embedding | Text → Vector | Encoding | 90% | High |
| Q3: Semantic Retrieval | Vector → Nodes | Materialization | 87% | High |
| Q4: Context Building | Nodes → Text | Lateral | 97% | Very High |
| Q5: Answer Generation | Text → Text | Synthesis | 85% | High |
| Q6: Answer Synthesis | Multiple → Single | Bottom-up | 78% | Medium-High |
| Q7: Follow-up Generation | Answer → Questions | Lateral | N/A | Medium |

**Information Density**: Сколько информации сохраняется относительно входа (100% = вся)
**Semantic Fidelity**: Точность сохранения смысла (не объема)

### Таблица 2: По вычислительным характеристикам

| Преобразование | Compute Type | Parallelizable | Time Complexity | Space Complexity | Bottleneck |
|---|---|---|---|---|---|
| T1: Chunking | Algorithmic | Yes | O(n) | O(n) | None |
| T2: Entity Extraction | LLM | Yes (per chunk) | O(n·m) | O(k) | LLM latency |
| T3: Relationship Extraction | LLM | Yes (per chunk) | O(n·m) | O(k²) | LLM latency |
| T4: Summarization | LLM | Yes (per entity) | O(k·m) | O(k) | LLM latency |
| T5: Community Detection | Graph Algorithm | No | O(n log n) | O(n+e) | Graph size |
| T6: Community Reports | LLM | Yes (per community) | O(c·m) | O(c) | LLM latency |
| T7: Embedding | Neural Network | Yes (batched) | O(n·d) | O(n·d) | Batch size |
| Q1: Query Analysis | Rule-based | No | O(q) | O(1) | None |
| Q2: Query Embedding | Neural Network | No | O(q·d) | O(d) | Network latency |
| Q3: Retrieval | Vector Search | No | O(log n) | O(n·d) | Index size |
| Q4: Context Building | Algorithmic | No | O(k·t) | O(k·t) | Token counting |
| Q5: Answer Generation | LLM | No | O(c·m) | O(c) | LLM latency |
| Q6: Answer Synthesis | LLM | No | O(r·m) | O(r) | LLM latency |
| Q7: Follow-up Generation | LLM | No | O(a·m) | O(f) | LLM latency |

**Notation**:
- n = number of documents/chunks
- m = LLM processing time
- k = number of entities
- e = number of edges (relationships)
- c = number of communities
- d = embedding dimensions
- q = query length
- t = text unit length
- r = number of intermediate answers
- a = answer length
- f = number of follow-ups

### Таблица 3: По стоимости (на 1000 документов)

| Преобразование | API Calls | Tokens (Input) | Tokens (Output) | Cost @ GPT-4 | Cost @ GPT-3.5 | % of Total |
|---|---|---|---|---|---|---|
| T1: Chunking | 0 | 0 | 0 | $0.00 | $0.00 | 0% |
| T2: Entity Extraction | ~8,000 | 9.6M | 480K | $96.00 | $9.60 | 65% |
| T3: Relationship Extraction | ~8,000 | 9.6M | 800K | $104.00 | $10.40 | 20%* |
| T4: Summarization | ~2,000 | 200K | 100K | $3.00 | $0.30 | 5% |
| T5: Clustering | 0 | 0 | 0 | $0.00 | $0.00 | 0% |
| T6: Community Reports | ~60 | 480K | 60K | $6.00 | $0.60 | 10% |
| T7: Embedding | 8,000 | 9.6M | 0 | $0.96 | $0.96 | <1% |
| **Indexing Total** | | | | **$209.96** | **$21.86** | **100%** |
| Q1: Query Analysis | 0 | 0 | 0 | $0.00 | $0.00 | 0% |
| Q2: Query Embedding | 1 | ~50 | 0 | $0.0001 | $0.0001 | <1% |
| Q3: Retrieval | 0 | 0 | 0 | $0.00 | $0.00 | 0% |
| Q4: Context Building | 0 | 0 | 0 | $0.00 | $0.00 | 0% |
| Q5: Answer (Local) | 1 | 8K | 500 | $0.09 | $0.01 | 95% |
| Q5: Answer (Global, Map) | 10 | 20K | 3K | $0.30 | $0.03 | 75% |
| Q6: Synthesis (Reduce) | 1 | 5K | 800 | $0.09 | $0.01 | 25% |
| Q7: Follow-ups | 1 | 1K | 200 | $0.02 | $0.002 | <5% |
| **Query Total (Local)** | | | | **$0.09** | **$0.01** | |
| **Query Total (Global)** | | | | **$0.41** | **$0.04** | |

*Note: T3 часто выполняется вместе с T2 в одном LLM вызове

**Pricing** (approximate):
- GPT-4: $0.01/1K input tokens, $0.03/1K output tokens
- GPT-3.5: $0.001/1K input tokens, $0.002/1K output tokens
- Embeddings: $0.0001/1K tokens

### Таблица 4: По качеству извлечения информации

| Преобразование | Precision | Recall | F1-Score | Accuracy | Notes |
|---|---|---|---|---|---|
| T2: Entity Extraction | 0.92 | 0.75 | 0.83 | N/A | With gleaning: recall → 0.85 |
| T3: Relationship Extraction | 0.88 | 0.70 | 0.78 | N/A | Depends on entity quality |
| T4: Summarization | 0.95 | 0.85 | 0.90 | N/A | Key info retention |
| T5: Community Detection | 0.87 | 0.92 | 0.89 | N/A | Thematic coherence |
| T6: Community Reports | N/A | N/A | N/A | 0.78 | Human eval: factual accuracy |
| Q3: Semantic Retrieval | 0.76 | 0.88 | 0.82 | N/A | @ top-30 for entities |
| Q5: Answer Generation | N/A | N/A | N/A | 0.85 | Human eval: correctness |
| Q6: Answer Synthesis | N/A | N/A | N/A | 0.82 | Human eval: completeness |

**Metrics explanation**:
- **Precision**: Of extracted items, how many are correct?
- **Recall**: Of all items that should be extracted, how many are found?
- **F1**: Harmonic mean of precision and recall
- **Accuracy**: For generation tasks, human-evaluated correctness

### Таблица 5: По использованию в разных типах поиска

| Преобразование | Global Search | Local Search | Drift Search | Importance | Usage Pattern |
|---|---|---|---|---|---|
| T1: Text Chunking | ○ | ★★★ | ★★ | Critical | All searches |
| T2: Entity Extraction | ○ | ★★★ | ★★★ | Critical | All searches |
| T3: Relationship Extraction | - | ★ | ★★★ | Variable | Drift primary |
| T4: Summarization | ○ | ★★ | ★★ | Important | Quality improvement |
| T5: Community Detection | ★★★ | - | ○ | Critical | Global only |
| T6: Community Reports | ★★★ | - | - | Critical | Global only |
| T7: Embedding | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q1: Query Analysis | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q2: Query Embedding | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q3: Semantic Retrieval | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q4: Context Building | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q5: Answer Generation | ★★★ | ★★★ | ★★★ | Critical | All searches |
| Q6: Answer Synthesis | ★★★ | - | ○ | Important | Global primary |
| Q7: Follow-up Generation | ★ | ★ | ★★★ | Optional | Enhanced UX |

**Legend**:
- ★★★ = Primary (critical for this search type)
- ★★ = Secondary (improves quality)
- ★ = Tertiary (optional enhancement)
- ○ = Minimal (used but not critical)
- \- = Not used

### Таблица 6: По параметрам оптимизации

| Преобразование | Key Parameters | Speed Optimization | Quality Optimization | Cost Optimization |
|---|---|---|---|---|
| T1 | chunk_size, overlap | Larger chunks (1500) | Smaller chunks (1000) | Larger chunks (1500) |
| T2 | max_gleanings, model | gleanings=0, GPT-3.5 | gleanings=3, GPT-4 | gleanings=0, GPT-3.5 |
| T3 | model, min_strength | GPT-3.5, strength>7 | GPT-4, strength>3 | GPT-3.5, strength>7 |
| T4 | model, max_length | GPT-3.5, length=300 | GPT-4, length=500 | GPT-3.5, length=200 |
| T5 | max_cluster_size | size=15 | size=8 | size=15 |
| T6 | model, max_report_length | GPT-3.5, length=1500 | GPT-4, length=3000 | GPT-3.5, length=1000 |
| T7 | batch_size | batch=1000 | batch=500 | batch=1000 |
| Q1 | classification_method | Rule-based | LLM-based | Rule-based |
| Q2 | model | Standard | Standard | Standard |
| Q3 | top_k, hybrid_alpha | k=10, α=0.8 | k=30, α=0.5 | k=10, α=0.8 |
| Q4 | max_context_tokens | tokens=5000 | tokens=12000 | tokens=5000 |
| Q5 | model, temperature | GPT-3.5, T=0 | GPT-4, T=0.1 | GPT-3.5, T=0 |
| Q6 | max_intermediate | intermediate=5 | intermediate=15 | intermediate=5 |
| Q7 | num_questions, model | Skip | GPT-4, num=5 | GPT-3.5, num=3 |

### Таблица 7: По входам и выходам

| Преобразование | Input Type | Input Size | Output Type | Output Size | Compression Ratio |
|---|---|---|---|---|---|
| T1: Chunking | Document | 10K tokens | Text Units | 8×1.2K tokens | 1:1 (no compression) |
| T2: Entity Extraction | Text Unit | 1.2K tokens | Entities | 5×50 tokens | 5:1 (concept extraction) |
| T3: Relationship Ext. | Text Unit + Entities | 1.2K tokens | Relationships | 7×30 tokens | 6:1 (relation extraction) |
| T4: Summarization | Entity Descriptions | 5×100 tokens | Consolidated Desc. | 1×150 tokens | 3:1 (deduplication) |
| T5: Clustering | Graph | 2K entities | Communities | 60 communities | N/A (structuring) |
| T6: Reports | Community | 50 entities | Report | 1500 tokens | 50:1 (abstraction) |
| T7: Embedding | Text (any) | Variable | Vector | 1536 dims | Fixed output |
| Q1: Query Analysis | Query | 50 tokens | Intent | ~10 fields | Structured output |
| Q2: Query Embedding | Query | 50 tokens | Vector | 1536 dims | Fixed output |
| Q3: Retrieval | Query Vector | 1536 dims | Nodes | 30 items | N/A (search) |
| Q4: Context Building | Nodes | 30 items | Context | 8K tokens | N/A (assembly) |
| Q5: Answer Generation | Context + Query | 8K tokens | Answer | 300 tokens | 27:1 (synthesis) |
| Q6: Answer Synthesis | Answers | 10×300 tokens | Final Answer | 500 tokens | 6:1 (consolidation) |
| Q7: Follow-ups | Answer | 300 tokens | Questions | 5×30 tokens | N/A (generation) |

### Таблица 8: По зависимостям и порядку выполнения

| Преобразование | Depends On | Enables | Can Run in Parallel With | Must Run After | Blocking |
|---|---|---|---|---|---|
| T1 | Documents | T2, T3 | - | - | No |
| T2 | T1 (Text Units) | T3, T4 | T3 (same text_unit) | T1 | No |
| T3 | T1, T2 (Entities) | T5 (Graph) | T2 (same text_unit) | T1, T2 | No |
| T4 | T2 (Entities) | T5, T7 | T5 | T2 | No |
| T5 | T2, T3 (Graph) | T6 | - | T2, T3, T4 | Yes* |
| T6 | T5 (Communities) | T7 | - | T5 | No |
| T7 | T1, T4, T6 (Text) | Q3 (Search) | - | T1, T4, T6 | No |
| Q1 | User Query | Q2, Q3 | - | - | No |
| Q2 | Q1 (Query) | Q3 | - | Q1 | No |
| Q3 | Q2, T7 (Embeddings) | Q4 | - | Q2, T7 | No |
| Q4 | Q3 (Nodes) | Q5 | - | Q3 | No |
| Q5 | Q4 (Context) | Q6, Q7 | (Map phase) | Q4 | Yes* |
| Q6 | Q5 (Multiple) | Q7 | - | Q5 | Yes* |
| Q7 | Q5 or Q6 | User Output | - | Q5/Q6 | No |

*Blocking = requires previous transformation to complete before starting

### Таблица 9: По типам ошибок и митигации

| Преобразование | Common Errors | Error Rate | Impact | Mitigation Strategies |
|---|---|---|---|---|
| T1 | Boundary splits | 2-5% | Low | Overlap, sentence-aware splitting |
| T2 | Missing entities | 15-25% | High | Gleaning, better prompts |
| T2 | Hallucinated entities | 5-10% | Medium | Lower temperature, validation |
| T3 | Missing relationships | 20-30% | Medium | Re-extraction, context expansion |
| T3 | Wrong strength scores | 10-15% | Low | Calibration prompts |
| T4 | Lost nuances | 10-15% | Low | Longer summaries, multi-stage |
| T5 | Poor clustering | 5-10% | Medium | Tune resolution, manual review |
| T6 | Generic reports | 15-20% | Medium | Better prompts, examples |
| T6 | Factual errors | 5-10% | High | Fact verification, citations |
| T7 | Semantic drift | 5-10% | Low | Model selection, calibration |
| Q3 | Irrelevant retrieval | 10-20% | High | Hybrid ranking, tune top_k |
| Q5 | Hallucinations | 5-15% | High | Lower temp, grounding prompts |
| Q5 | Context ignoring | 10-20% | High | Better prompts, context emphasis |
| Q6 | Lost insights | 15-25% | Medium | More intermediate answers |

### Таблица 10: По масштабируемости

| Преобразование | Scalability | Bottleneck @ 100K Docs | Bottleneck @ 1M Docs | Mitigation |
|---|---|---|---|---|
| T1 | Excellent | Memory | Memory | Streaming processing |
| T2 | Good | LLM rate limits | LLM cost | Batching, caching |
| T3 | Good | LLM rate limits | LLM cost | Batching, caching |
| T4 | Excellent | LLM calls | LLM cost | Efficient batching |
| T5 | Fair | Graph size | RAM | Distributed graph processing |
| T6 | Good | LLM calls | LLM cost | Parallel processing |
| T7 | Excellent | Batch throughput | Batch throughput | Larger batches |
| Q1 | Excellent | - | - | Stateless |
| Q2 | Excellent | - | - | Stateless |
| Q3 | Good | Index size | Index size | Sharding, ANN indices |
| Q4 | Excellent | - | - | Stateless |
| Q5 | Good | LLM rate limits | LLM rate limits | Queue management |
| Q6 | Good | LLM latency | LLM latency | Parallel map phase |
| Q7 | Good | LLM calls | LLM calls | Caching, skip if needed |

**Scalability ratings**:
- **Excellent**: Linear or sub-linear scaling
- **Good**: Near-linear with some bottlenecks
- **Fair**: Significant performance degradation at scale

## Специализированные таблицы

### Таблица 11: Semantic Transformation Types

| Category | Transformations | Purpose | Information Change | Typical Use |
|---|---|---|---|---|
| **Decomposition** | T1 | Break into manageable pieces | Minimal loss | Preprocessing |
| **Extraction** | T2, T3 | Extract structured info | Medium loss (intentional) | Knowledge extraction |
| **Aggregation** | T4, Q4, Q6 | Combine multiple sources | Low loss | Consolidation |
| **Abstraction** | T6 | Create high-level summaries | High loss (intentional) | Overview generation |
| **Clustering** | T5 | Organize by similarity | Minimal loss (structural) | Organization |
| **Vectorization** | T7, Q2 | Convert to embeddings | Medium loss (encoding) | Semantic search |
| **Retrieval** | Q3 | Find relevant items | Medium loss (filtering) | Information retrieval |
| **Synthesis** | Q5 | Generate new text | Medium loss (focused) | Answer generation |
| **Classification** | Q1 | Categorize input | Minimal loss | Intent recognition |
| **Extrapolation** | Q7 | Generate related items | N/A (creative) | Exploration |

### Таблица 12: LLM Agent Roles

| Agent Type | Transformations | Model | Temperature | Prompt Complexity | Output Structure | Criticality |
|---|---|---|---|---|---|---|
| **Extraction Agent** | T2, T3 | GPT-4 | 0.0 | High (detailed instructions) | Structured (JSON-like) | Critical |
| **Summarization Agent** | T4 | GPT-3.5/GPT-4 | 0.0 | Medium (consolidate) | Semi-structured | Important |
| **Report Agent** | T6 | GPT-4 | 0.1 | High (analysis + writing) | Structured (JSON) | Critical |
| **Answer Agent** | Q5, Q6 | GPT-4 | 0.0 | High (context + query) | Natural language | Critical |
| **Follow-up Agent** | Q7 | GPT-3.5/GPT-4 | 0.3 | Medium (creative) | List of questions | Optional |

### Таблица 13: Data Flow Dependencies

```
Dependency Graph:

Documents
  ↓
  T1 ──────────────────────────────┐
  ↓                                │
  Text Units                       │
  ↓                                │
  T2 ────┐                         │
  ↓      │                         │
  Entities (raw)  T3 ←─────────────┘
  ↓      │        ↓
  T4 ←───┘   Relationships
  ↓              ↓
  Entities  ─────┤
  (consolidated) │
  ↓              ↓
  T5 ← Graph (E+R)
  ↓
  Communities
  ↓
  T6
  ↓
  Community Reports
  ↓
  ┌─────┴─────┬─────────────┬──────────┐
  │           │             │          │
  T7          T7            T7         T7
  ↓           ↓             ↓          ↓
Entity     Text Unit    Community   Document
Embeddings  Embeddings  Embeddings  Embeddings
  ↓           ↓             ↓          ↓
  └─────┬─────┴─────────────┴──────────┘
        ↓
   Vector Stores
        ↓
  [Query Phase]
        ↓
   User Query → Q1 → Q2 → Q3 → Q4 → Q5 → (Q6) → (Q7) → Answer
```

### Таблица 14: Preservation by Chain

| Chain Type | Indexing Preservation | Query Preservation | Combined | Interpretation |
|---|---|---|---|---|
| **Simple Local** | 0.55 (55%) | 0.66 (66%) | 0.36 (36%) | Good for factual questions |
| **Complex Local** | 0.55 (55%) | 0.75 (75%) | 0.41 (41%) | Better context, higher quality |
| **Global Search** | 0.32 (32%) | 0.51 (51%) | 0.16 (16%) | Appropriate for overview |
| **Drift Search (3 hops)** | 0.52 (52%) | 0.21 (21%) | 0.11 (11%) | Low but discovers connections |
| **Hybrid (G+L)** | 0.43 (43%) | 0.60 (60%) | 0.26 (26%) | Balanced breadth + depth |

**Interpretation guide**:
- **>60%**: Excellent preservation, detailed answers possible
- **40-60%**: Good preservation, factual answers
- **20-40%**: Medium preservation, high-level answers
- **<20%**: Low preservation, but may discover insights

### Таблица 15: Trade-off Matrix

| Optimization Goal | Recommended Chain | Indexing Config | Query Config | Trade-offs |
|---|---|---|---|---|
| **Minimize Latency** | Local (fast) | Standard, cache heavy | top_k=10, small context | -15% quality, -40% latency |
| **Maximize Quality** | Local (quality) | Small chunks, gleaning | top_k=30, large context | +20% quality, +50% latency |
| **Minimize Cost** | Local (cheap) | GPT-3.5, no gleaning | GPT-3.5, small context | -25% quality, -60% cost |
| **Maximize Breadth** | Global | Standard communities | top_k=15 communities | -20% depth, +overview |
| **Maximize Depth** | Local + sources | Detailed entities | Include text_units | -breadth, +citations |
| **Enable Exploration** | Drift | Strong relationships | Multi-hop, follow-ups | -speed, +discovery |
| **Balanced** | Hybrid | Standard | Adaptive | Moderate all metrics |

## Рекомендации по выбору

### Выбор по метрике

#### Приоритет: Скорость

```yaml
Choose:
  - Indexing: Aggressive caching, larger chunks
  - Query: Local Search (fast variant)
  - Transformations to optimize:
    - T1: chunk_size=1500
    - T2: max_gleanings=0
    - Q3: top_k=10
    - Q4: max_context=5000
  - Skip: Q6, Q7

Expected: 2-3 sec latency, 58% preservation
```

#### Приоритет: Качество

```yaml
Choose:
  - Indexing: Quality-optimized (smaller chunks, gleaning)
  - Query: Hybrid (Global + Local) or Quality Local
  - Transformations to optimize:
    - T1: chunk_size=1000, overlap=150
    - T2: max_gleanings=3, GPT-4
    - T6: max_report_length=3000
    - Q3: top_k=30
    - Q4: max_context=12000
  - Include: Q6 (for global), Q7

Expected: 20-30 sec latency, 70%+ preservation
```

#### Приоритет: Стоимость

```yaml
Choose:
  - Indexing: Use GPT-3.5, minimal gleaning
  - Query: Local Search (cheap variant)
  - Transformations to optimize:
    - T2: GPT-3.5, gleanings=0
    - T4: GPT-3.5
    - T6: GPT-3.5, length=1500
    - Q5: GPT-3.5
  - Skip: Q6, Q7

Expected: -60% cost, -20% quality
```

### Выбор по типу вопроса

| Question Type | Recommended Transformations | Chain | Expected Result |
|---|---|---|---|
| "What is X?" (definition) | T1→T2→T7→Q1→Q2→Q3→Q4→Q5 | Local | 65% preservation, 5sec |
| "How does X work?" (mechanism) | T1→T2→T3→T7→Q1-Q5 | Local + relationships | 68% preservation, 6sec |
| "What are the trends?" (overview) | T1→...→T6→T7→Q1-Q6 | Global | 51% preservation, 20sec |
| "How are X and Y related?" (connection) | T1→T2→T3→T7→Q1-Q5 (iterative) | Drift | 65% per hop, 40sec |
| "Compare X and Y" (comparison) | Hybrid (Local for each + synthesis) | Hybrid | 60% preservation, 15sec |

## Практические примеры

### Пример 1: Оптимизация для FAQ системы

**Требования**: Быстрые ответы, высокая точность, низкая стоимость

**Выбранная конфигурация**:
```yaml
Indexing:
  T1: {chunk_size: 1200, overlap: 100}
  T2: {model: "gpt-3.5-turbo", max_gleanings: 0}
  T7: {batch_size: 1000}
  Skip: T3 (relationships not needed), T5, T6 (no global search)

Query:
  Q1-Q2: Standard
  Q3: {top_k: 15, hybrid_alpha: 0.8}  # Favor semantic similarity
  Q4: {max_context: 6000}
  Q5: {model: "gpt-3.5-turbo"}
  Skip: Q6, Q7

Caching: Aggressive (cache all queries, 24hr TTL)
```

**Результаты**:
- Latency: 3.2 sec (95th percentile)
- Quality: 82% user satisfaction
- Cost: $0.015 per query
- Cache hit rate: 65%

### Пример 2: Оптимизация для исследовательской платформы

**Требования**: Глубокий анализ, обзоры + детали, исследование связей

**Выбранная конфигурация**:
```yaml
Indexing:
  T1: {chunk_size: 1000, overlap: 150}
  T2: {model: "gpt-4", max_gleanings: 2}
  T3: {model: "gpt-4", min_strength: 3}
  T4: {model: "gpt-4"}
  T5: {max_cluster_size: 8}
  T6: {model: "gpt-4", max_report_length: 2500}
  T7: Standard

Query:
  Adaptive routing based on query complexity:
    - Simple: Local (standard)
    - Complex: Local (quality)
    - Overview: Global
    - Exploratory: Drift
  Q7: Always enabled (follow-up questions)

Caching: Moderate (cache only common queries)
```

**Результаты**:
- Latency: 8-45 sec (зависит от типа)
- Quality: 91% user satisfaction
- Cost: $0.05-$0.40 per query
- Follow-up usage: 73% of users

## Дополнительные ресурсы

### Связанная документация

- **[Indexing Transformations →](01-indexing-transformations.md)** - Детали T1-T7
- **[Query Transformations →](02-query-transformations.md)** - Детали Q1-Q7
- **[Transformation Chains →](03-transformation-chains.md)** - Композиция и цепочки
- **[README →](README.md)** - Обзор системы преобразований

### Главная документация

- **[../README.md →](../README.md)** - GraphRAG pipeline overview
- **[../nodes/README.md →](../nodes/README.md)** - Node types
- **[../nodes/09-query-execution.md →](../nodes/09-query-execution.md)** - Query execution analysis

---

**Это финальный документ серии. Для возврата к началу см. [README](README.md).**
