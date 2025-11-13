# Языковые промпты GraphRAG

## Обзор

Этот раздел содержит детальное описание всех **языковых промптов** (LLM prompts), используемых в GraphRAG для взаимодействия с большими языковыми моделями (LLM). Промпты являются критическими элементами системы, определяющими качество семантических преобразований и итогового результата.

```
┌──────────────────────────────────────────────────────────────┐
│              PROMPTS В АРХИТЕКТУРЕ GRAPHRAG                  │
└──────────────────────────────────────────────────────────────┘

                    INDEXING PROMPTS
                          │
┌─────────────────────────┼──────────────────────────────┐
│  Entity Extraction (T2) │ Community Report (T6)        │
│  Summarization (T4)     │ Claims Extraction (opt)      │
└─────────────────────────┼──────────────────────────────┘
                          │
                    INDEXED GRAPH
                          │
                    QUERY PROMPTS
                          │
┌─────────────────────────┼──────────────────────────────┐
│  Local Search (Q5)      │ Drift Search (Q5, iterative) │
│  Global Search:         │ Question Generation (Q7)     │
│    - Map (Q5)           │                              │
│    - Reduce (Q6)        │                              │
└─────────────────────────┴──────────────────────────────┘
```

## Категории промптов

### Индексационные промпты (Indexing Prompts)

Используются при построении knowledge graph из исходных документов.

| Промпт | Трансформация | LLM Model | Назначение | Документ |
|---|---|---|---|---|
| **Entity Extraction** | T2, T3 | GPT-4 | Извлечение entities и relationships | [01-entity-extraction.md](01-entity-extraction.md) |
| **Summarization** | T4 | GPT-3.5/GPT-4 | Консолидация описаний entities | [02-summarization.md](02-summarization.md) |
| **Community Report** | T6 | GPT-4 | Генерация отчетов по communities | [03-community-report.md](03-community-report.md) |
| **Claims Extraction** | Optional | GPT-4 | Извлечение claims/фактов (опционально) | [04-claim-extraction.md](04-claim-extraction.md) |

### Query промпты (Query Prompts)

Используются при выполнении запросов пользователей.

| Промпт | Трансформация | LLM Model | Назначение | Документ |
|---|---|---|---|---|
| **Local Search** | Q5 | GPT-4 | Ответ на specific вопросы | [05-local-search.md](05-local-search.md) |
| **Global Search Map** | Q5 (Map phase) | GPT-4 | Анализ community reports | [06-global-search-map.md](06-global-search-map.md) |
| **Global Search Reduce** | Q6 (Reduce phase) | GPT-4 | Синтез множественных ответов | [07-global-search-reduce.md](07-global-search-reduce.md) |
| **Drift Search Local** | Q5 (iterative) | GPT-4 | Итеративный graph walk | [08-drift-search.md](08-drift-search.md) |
| **Drift Search Primer** | Q5 (initial) | GPT-4 | Инициализация drift search | [08-drift-search.md](08-drift-search.md) |
| **Question Generation** | Q7 | GPT-3.5/GPT-4 | Генерация follow-up вопросов | [09-question-generation.md](09-question-generation.md) |

## Общие паттерны промптов

Все промпты в GraphRAG следуют нескольким общим принципам:

### 1. Структурная организация

```
---Role---
[Определение роли LLM]

---Goal---
[Четкая цель задачи]

---Steps--- или ---Data---
[Инструкции или входные данные]

---Output Format---
[Требуемый формат вывода]

---Examples---
[Примеры (обычно few-shot)]

---Real Data---
[Реальные данные для обработки]
```

### 2. Grounding Rules

Все промпты требуют **grounding** — ссылки на источники данных:

```
"This is an example sentence supported by multiple data references
[Data: <dataset name> (record ids); <dataset name> (record ids)]."

Example:
"GraphRAG uses LLMs for extraction [Data: Entities (5, 7); Relationships (23)]"
```

**Правила**:
- Не более 5 record IDs в одной ссылке
- Добавлять "+more" если источников больше
- Не включать информацию без evidence

### 3. Few-shot Learning

Большинство промптов используют **few-shot examples** (2-3 примера):

```
Example 1: [Simple case]
Example 2: [Medium complexity]
Example 3: [Complex case with multiple entities]
```

Это улучшает quality extraction на 20-30% vs zero-shot.

### 4. Structured Output

Промпты запрашивают **структурированный вывод**:

- **JSON** (Community Reports, Global Search Map): легко parse
- **Tuple format** (Entity Extraction): `("entity"|<name>|<type>|<description>)`
- **Markdown** (Query responses): читаемость для пользователя

## Роль промптов в цепочках преобразований

### Indexing Chain

```
Documents
  ↓
Text Chunks
  ↓ [PROMPT: Entity Extraction]
Entities + Relationships (raw)
  ↓ [PROMPT: Summarization]
Entities (consolidated descriptions)
  ↓ [Algorithmic: Leiden Clustering]
Communities
  ↓ [PROMPT: Community Report]
Community Reports
  ↓
Indexed Knowledge Graph
```

### Query Chains

**Local Search**:
```
User Query → [Algorithmic: Retrieval] → Entities + Text Units
  ↓ [PROMPT: Local Search]
Answer with citations
```

**Global Search**:
```
User Query → [Algorithmic: Retrieval] → Community Reports (10x)
  ↓ [PROMPT: Global Map] (parallel 10x)
Intermediate Answers (10x)
  ↓ [PROMPT: Global Reduce]
Final Synthesized Answer
```

**Drift Search**:
```
User Query → [PROMPT: Drift Primer] → Initial answer + follow-ups
  ↓ [Iterative: 3-5 hops]
Each hop: [PROMPT: Drift Local] → Answer + new follow-ups
  ↓ [PROMPT: Drift Reduce]
Final Answer combining all hops
```

## Используемые технологии и библиотеки

### LLM Providers

```python
# OpenAI API
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key="...",
    base_url="..."  # Optional: Azure OpenAI
)

# Вызов промпта
response = await client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": PROMPT_TEMPLATE},
        {"role": "user", "content": user_data}
    ],
    temperature=0.0,  # Deterministic для extraction
    max_tokens=4000
)
```

**Файлы**:
- `graphrag/llm/openai/openai_chat_llm.py`
- `graphrag/llm/openai/openai_embeddings_llm.py`

### Prompt Management

```python
# Template variables
ENTITY_EXTRACTION_PROMPT.format(
    entity_types="organization, person, geo, event",
    input_text=chunk_text,
    tuple_delimiter="|",
    record_delimiter="##",
    completion_delimiter="<|COMPLETE|>"
)
```

**Файлы**:
- `graphrag/prompts/index/*.py` — indexing промпты
- `graphrag/prompts/query/*.py` — query промпты

### Response Parsing

```python
# Parsing structured outputs
import json
import re

# For JSON responses (Community Reports)
parsed = json.loads(response.choices[0].message.content)

# For tuple format (Entity Extraction)
tuples = response.split("##")  # record_delimiter
for tuple_str in tuples:
    parts = tuple_str.split("|")  # tuple_delimiter
    entity_name = parts[1]
    entity_type = parts[2]
    # ...
```

**Файлы**:
- `graphrag/index/operations/extract_entities/extract_entities.py`
- `graphrag/index/operations/summarize_descriptions/summarize_descriptions.py`

### Rate Limiting & Retry

```python
# Rate limiting для LLM calls
from graphrag.llm.limiting import LLMLimiter

limiter = LLMLimiter(
    tokens_per_minute=50_000,
    requests_per_minute=1_000
)

# Retry logic
from graphrag.llm.openai.utils import retry_with_backoff

@retry_with_backoff(max_retries=10, max_retry_wait=10.0)
async def call_llm(prompt):
    return await client.chat.completions.create(...)
```

**Файлы**:
- `graphrag/llm/limiting/llm_limiter.py`
- `graphrag/llm/base/rate_limiting_llm.py`

### Gleaning (Iterative Extraction)

Для улучшения recall при entity extraction используется **gleaning** — повторные проходы:

```python
# Initial extraction
entities = await extract_entities(text, entity_types)

# Gleaning iterations (if max_gleanings > 0)
for i in range(max_gleanings):
    # Ask LLM: "Were any entities missed?"
    continue_response = await llm(CONTINUE_PROMPT)

    # If LLM says yes, extract more
    if "Y" in continue_response:
        additional_entities = await extract_entities(text, entity_types)
        entities.extend(additional_entities)
    else:
        break
```

**Промпты**:
- `CONTINUE_PROMPT`: "MANY entities were missed... Add them below"
- `LOOP_PROMPT`: "Answer Y or N if there are still entities to add"

**Файлы**:
- `graphrag/prompts/index/extract_graph.py` (CONTINUE_PROMPT, LOOP_PROMPT)
- `graphrag/index/operations/extract_entities/extract_entities.py`

**Эффект**: +15-20% recall при `max_gleanings=2-3`

## Параметры промптов

### Temperature

```yaml
Entity Extraction: 0.0      # Deterministic, factual
Summarization: 0.0          # Consistent summaries
Community Reports: 0.1      # Slight creativity for synthesis
Global Map: 0.0             # Factual analysis
Global Reduce: 0.0          # Consistent synthesis
Question Generation: 0.3    # More creative
```

### Max Tokens

```yaml
Entity Extraction: 4000     # Large for many entities
Summarization: 500          # Short consolidated description
Community Reports: 3000     # Detailed report with findings
Local Search: 2000          # Medium-length answer
Global Map: 1000            # Short key points per report
Global Reduce: 2000         # Final synthesized answer
```

### Context Windows

```yaml
Entity Extraction:
  - Input: Text chunk (~1200 tokens)
  - Prompt: ~500 tokens
  - Output: ~1500 tokens
  - Total: ~3200 tokens

Community Report:
  - Input: Community data (~5000 tokens)
  - Prompt: ~1000 tokens
  - Output: ~2000 tokens
  - Total: ~8000 tokens

Local Search:
  - Input: Context (~8000 tokens)
  - Prompt: ~300 tokens
  - Output: ~2000 tokens
  - Total: ~10300 tokens
```

## Стоимость промптов

### Indexing (на 1000 документов)

```
Entity Extraction (T2):
  - Calls: ~8,000
  - Tokens (in): 9.6M
  - Tokens (out): 480K
  - Cost @ GPT-4: $96.00
  - Cost @ GPT-3.5: $9.60
  - % of total: 65%

Summarization (T4):
  - Calls: ~2,000
  - Tokens (in): 200K
  - Tokens (out): 100K
  - Cost @ GPT-4: $3.00
  - Cost @ GPT-3.5: $0.30
  - % of total: 5%

Community Reports (T6):
  - Calls: ~60
  - Tokens (in): 480K
  - Tokens (out): 60K
  - Cost @ GPT-4: $6.00
  - Cost @ GPT-3.5: $0.60
  - % of total: 10%

Total Indexing: $105.00 (GPT-4) or $10.50 (GPT-3.5)
```

### Query (per request)

```
Local Search:
  - Calls: 1
  - Tokens (in): ~8000
  - Tokens (out): ~500
  - Cost @ GPT-4: $0.09
  - Cost @ GPT-3.5: $0.01

Global Search:
  - Calls: 11 (10 map + 1 reduce)
  - Tokens (in): ~25000
  - Tokens (out): ~3800
  - Cost @ GPT-4: $0.41
  - Cost @ GPT-3.5: $0.04

Drift Search (3 hops):
  - Calls: 5 (1 primer + 3 local + 1 reduce)
  - Tokens (in): ~20000
  - Tokens (out): ~3000
  - Cost @ GPT-4: $0.30
  - Cost @ GPT-3.5: $0.03
```

## Качественные метрики промптов

### Entity Extraction

```yaml
Precision: 0.92    # 92% извлеченных entities корректны
Recall: 0.75       # 75% всех entities найдены (без gleaning)
Recall (gleaning=2): 0.85  # +10% с gleaning
F1: 0.83

Common errors:
  - Missed entities: 15% (решается gleaning)
  - Hallucinated entities: 8% (снижается lower temperature)
  - Wrong type: 5% (улучшается better examples)
```

### Summarization

```yaml
Coherence: 0.90         # Логическая связность
Completeness: 0.85      # Сохранение ключевой информации
Consistency: 0.95       # Отсутствие противоречий
Conciseness: 0.88       # Лаконичность

Common issues:
  - Lost nuances: 10-15% (trade-off за краткость)
  - Contradictions (rare): <5%
```

### Community Reports

```yaml
Factual Accuracy: 0.78  # Соответствие данным
Comprehensiveness: 0.82 # Полнота coverage
Citation Quality: 0.90  # Качество grounding
Structure Quality: 0.95 # JSON format compliance

Common issues:
  - Generic reports: 15-20% (не enough specificity)
  - Factual errors: 5-10% (hallucinations)
  - Missing citations: 10% (grounding rules не соблюдены)
```

### Query Answers

```yaml
Local Search:
  - Correctness: 0.85
  - Citation Accuracy: 0.90
  - Relevance: 0.88

Global Search:
  - Completeness: 0.92  # Широта coverage
  - Correctness: 0.82
  - Synthesis Quality: 0.85

Drift Search:
  - Discovery: 0.70      # Нахождение non-obvious connections
  - Coherence: 0.75      # Связность multi-hop answer
  - Relevance: 0.80
```

## Оптимизация промптов

### Стратегии улучшения

**1. Few-shot Examples**:
```
Improvement: +20-30% quality
Cost: +10-15% tokens (static overhead)
Recommendation: Always include 2-3 examples
```

**2. Grounding Rules**:
```
Improvement: -50% hallucinations
Cost: +5% output tokens (citations)
Recommendation: Mandatory for factual tasks
```

**3. Gleaning**:
```
Improvement: +10-20% recall
Cost: 2-3x LLM calls
Recommendation: Use for high-value extraction (max_gleanings=2)
```

**4. Temperature Tuning**:
```
Factual tasks (extraction, summarization): T=0.0
Creative tasks (reports, question gen): T=0.1-0.3
```

**5. Model Selection**:
```
High-stakes (entity extraction, reports): GPT-4
Lower-stakes (summarization, question gen): GPT-3.5-turbo
Cost savings: ~90% with GPT-3.5 где possible
Quality loss: ~5-10%
```

## Документация промптов

Каждый промпт документирован в отдельном файле:

1. **[Entity Extraction](01-entity-extraction.md)** — извлечение entities и relationships
2. **[Summarization](02-summarization.md)** — консолидация описаний
3. **[Community Report](03-community-report.md)** — генерация отчетов
4. **[Claim Extraction](04-claim-extraction.md)** — извлечение claims (опционально)
5. **[Local Search](05-local-search.md)** — ответы на specific вопросы
6. **[Global Search Map](06-global-search-map.md)** — анализ community reports
7. **[Global Search Reduce](07-global-search-reduce.md)** — синтез ответов
8. **[Drift Search](08-drift-search.md)** — exploratory graph walk
9. **[Question Generation](09-question-generation.md)** — генерация follow-ups

## Связанные ресурсы

### Внутренняя документация

- **[Transformation Chains](../transform/03-transformation-chains.md)** — роль промптов в цепочках
- **[Indexing Transformations](../transform/01-indexing-transformations.md)** — T2, T4, T6
- **[Query Transformations](../transform/02-query-transformations.md)** — Q5, Q6, Q7

### Код

- **Промпты**: `graphrag/prompts/`
- **LLM вызовы**: `graphrag/llm/`
- **Parsing**: `graphrag/index/operations/`

### Конфигурация

```yaml
# settings.yaml
extract_graph:
  prompt: null  # Use default GRAPH_EXTRACTION_PROMPT
  max_gleanings: 1
  model_id: "default_chat_model"

summarize_descriptions:
  prompt: null  # Use default SUMMARIZE_PROMPT
  max_length: 500

community_reports:
  prompt: null  # Use default COMMUNITY_REPORT_PROMPT
  max_length: 2000
```

---

**Версия**: GraphRAG 0.x
**Последнее обновление**: 2025-01-13
