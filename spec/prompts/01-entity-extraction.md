# Entity and Relationship Extraction Prompt

## Обзор

**Entity Extraction Prompt** — это **самый критичный промпт** в GraphRAG, отвечающий за извлечение entities (сущностей) и relationships (отношений) из текстовых chunks. Этот промпт выполняет семантическое преобразование **T2** (Entity Extraction) и **T3** (Relationship Extraction) одновременно.

```
┌──────────────────────────────────────────────────────────────┐
│         ENTITY EXTRACTION В PIPELINE GRAPHRAG                │
└──────────────────────────────────────────────────────────────┘

Text Chunk (1200 tokens)
  "GraphRAG is a knowledge graph construction system developed
   by Microsoft Research. It uses LLMs like GPT-4 to extract
   entities and relationships from text..."
            ↓
      [ENTITY EXTRACTION PROMPT]
      Model: GPT-4, Temperature: 0.0
            ↓
Extracted Output (tuple format):
  ("entity"|GRAPHRAG|ORGANIZATION|Knowledge graph construction system)##
  ("entity"|MICROSOFT RESEARCH|ORGANIZATION|Research division of Microsoft)##
  ("entity"|GPT-4|TECHNOLOGY|Large language model)##
  ("relationship"|GRAPHRAG|MICROSOFT RESEARCH|Developed by|9)##
  ("relationship"|GRAPHRAG|GPT-4|Uses for extraction|8)
```

## Характеристики промпта

| Характеристика | Значение |
|---|---|
| **Тип** | Few-shot extraction (3 examples) |
| **Трансформация** | T2 (Entities) + T3 (Relationships) |
| **LLM Model** | GPT-4 (default), GPT-3.5-turbo (cost optimization) |
| **Temperature** | 0.0 (deterministic) |
| **Max Tokens** | 4000 (output) |
| **Стоимость** | ~$96 на 1000 docs @ GPT-4 |
| **% от total indexing cost** | 65% |
| **Gleaning Support** | Yes (iterative refinement) |

## Полный текст промпта

### Основной промпт

```
-Goal-
Given a text document that is potentially relevant to this activity and a
list of entity types, identify all entities of those types from the text
and all relationships among the identified entities.

-Steps-
1. Identify all entities. For each identified entity, extract the following
   information:
   - entity_name: Name of the entity, capitalized
   - entity_type: One of the following types: [{entity_types}]
   - entity_description: Comprehensive description of the entity's
     attributes and activities
   Format each entity as:
   ("entity"{tuple_delimiter}<entity_name>{tuple_delimiter}<entity_type>
   {tuple_delimiter}<entity_description>)

2. From the entities identified in step 1, identify all pairs of
   (source_entity, target_entity) that are *clearly related* to each other.
   For each pair of related entities, extract the following information:
   - source_entity: name of the source entity, as identified in step 1
   - target_entity: name of the target entity, as identified in step 1
   - relationship_description: explanation as to why you think the source
     entity and the target entity are related to each other
   - relationship_strength: a numeric score indicating strength of the
     relationship between the source entity and target entity
   Format each relationship as:
   ("relationship"{tuple_delimiter}<source_entity>{tuple_delimiter}
   <target_entity>{tuple_delimiter}<relationship_description>
   {tuple_delimiter}<relationship_strength>)

3. Return output in English as a single list of all the entities and
   relationships identified in steps 1 and 2. Use **{record_delimiter}**
   as the list delimiter.

4. When finished, output {completion_delimiter}
```

### Примеры (Few-shot Learning)

**Example 1** (Simple - Finance/Politics):
```
Entity_types: ORGANIZATION,PERSON

Text:
The Verdantis's Central Institution is scheduled to meet on Monday and
Thursday, with the institution planning to release its latest policy
decision on Thursday at 1:30 p.m. PDT, followed by a press conference
where Central Institution Chair Martin Smith will take questions. Investors
expect the Market Strategy Committee to hold its benchmark interest rate
steady in a range of 3.5%-3.75%.

Output:
("entity"|CENTRAL INSTITUTION|ORGANIZATION|The Central Institution is the
Federal Reserve of Verdantis, which is setting interest rates on Monday
and Thursday)##
("entity"|MARTIN SMITH|PERSON|Martin Smith is the chair of the Central
Institution)##
("entity"|MARKET STRATEGY COMMITTEE|ORGANIZATION|The Central Institution
committee makes key decisions about interest rates and the growth of
Verdantis's money supply)##
("relationship"|MARTIN SMITH|CENTRAL INSTITUTION|Martin Smith is the Chair
of the Central Institution and will answer questions at a press conference|9)
<|COMPLETE|>
```

**Example 2** (Business/Tech):
```
Entity_types: ORGANIZATION

Text:
TechGlobal's (TG) stock skyrocketed in its opening day on the Global
Exchange Thursday. But IPO experts warn that the semiconductor corporation's
debut on the public markets isn't indicative of how other newly listed
companies may perform. TechGlobal, a formerly public company, was taken
private by Vision Holdings in 2014. The well-established chip designer says
it powers 85% of premium smartphones.

Output:
("entity"|TECHGLOBAL|ORGANIZATION|TechGlobal is a stock now listed on the
Global Exchange which powers 85% of premium smartphones)##
("entity"|VISION HOLDINGS|ORGANIZATION|Vision Holdings is a firm that
previously owned TechGlobal)##
("relationship"|TECHGLOBAL|VISION HOLDINGS|Vision Holdings formerly owned
TechGlobal from 2014 until present|5)
<|COMPLETE|>
```

**Example 3** (Complex - Geopolitics with multiple entities):
```
Entity_types: ORGANIZATION,GEO,PERSON

Text:
Five Aurelians jailed for 8 years in Firuzabad and widely regarded as
hostages are on their way home to Aurelia. The swap orchestrated by Quintara
was finalized when $8bn of Firuzi funds were transferred to financial
institutions in Krohaara, the capital of Quintara. The exchange initiated
in Firuzabad's capital, Tiruzia, led to the four men and one woman, who are
also Firuzi nationals, boarding a chartered flight to Krohaara. They were
welcomed by senior Aurelian officials and are now on their way to Aurelia's
capital, Cashion. The Aurelians include 39-year-old businessman Samuel
Namara, who has been held in Tiruzia's Alhamia Prison, as well as journalist
Durke Bataglani, 59, and environmentalist Meggie Tazbah, 53, who also holds
Bratinas nationality.

Output:
("entity"|FIRUZABAD|GEO|Firuzabad held Aurelians as hostages)##
("entity"|AURELIA|GEO|Country seeking to release hostages)##
("entity"|QUINTARA|GEO|Country that negotiated a swap of money in exchange
for hostages)##
("entity"|TIRUZIA|GEO|Capital of Firuzabad where the Aurelians were being
held)##
("entity"|KROHAARA|GEO|Capital city in Quintara)##
("entity"|CASHION|GEO|Capital city in Aurelia)##
("entity"|SAMUEL NAMARA|PERSON|Aurelian who spent time in Tiruzia's Alhamia
Prison)##
("entity"|ALHAMIA PRISON|GEO|Prison in Tiruzia)##
("entity"|DURKE BATAGLANI|PERSON|Aurelian journalist who was held hostage)##
("entity"|MEGGIE TAZBAH|PERSON|Bratinas national and environmentalist who
was held hostage)##
("relationship"|FIRUZABAD|AURELIA|Firuzabad negotiated a hostage exchange
with Aurelia|2)##
("relationship"|QUINTARA|AURELIA|Quintara brokered the hostage exchange
between Firuzabad and Aurelia|2)##
("relationship"|QUINTARA|FIRUZABAD|Quintara brokered the hostage exchange
between Firuzabad and Aurelia|2)##
("relationship"|SAMUEL NAMARA|ALHAMIA PRISON|Samuel Namara was a prisoner
at Alhamia prison|8)##
("relationship"|SAMUEL NAMARA|MEGGIE TAZBAH|Samuel Namara and Meggie Tazbah
were exchanged in the same hostage release|2)##
("relationship"|SAMUEL NAMARA|DURKE BATAGLANI|Samuel Namara and Durke
Bataglani were exchanged in the same hostage release|2)##
("relationship"|MEGGIE TAZBAH|DURKE BATAGLANI|Meggie Tazbah and Durke
Bataglani were exchanged in the same hostage release|2)##
("relationship"|SAMUEL NAMARA|FIRUZABAD|Samuel Namara was a hostage in
Firuzabad|2)##
("relationship"|MEGGIE TAZBAH|FIRUZABAD|Meggie Tazbah was a hostage in
Firuzabad|2)##
("relationship"|DURKE BATAGLANI|FIRUZABAD|Durke Bataglani was a hostage in
Firuzabad|2)
<|COMPLETE|>
```

### Gleaning Prompts

**CONTINUE_PROMPT** (для iterative refinement):
```
MANY entities and relationships were missed in the last extraction.
Remember to ONLY emit entities that match any of the previously extracted
types. Add them below using the same format:
```

**LOOP_PROMPT** (проверка завершения):
```
It appears some entities and relationships may have still been missed.
Answer Y or N if there are still entities or relationships that need to
be added.
```

## Параметры промпта

### Template Variables

```python
entity_types: str = "organization, person, geo, event"
  # Типы entities для extraction
  # Default: ["organization", "person", "geo", "event"]
  # Настраиваемые через config

input_text: str = chunk.text
  # Text chunk для обработки (~1200 tokens)

tuple_delimiter: str = "|"
  # Разделитель полей внутри tuple

record_delimiter: str = "##"
  # Разделитель между records (entities/relationships)

completion_delimiter: str = "<|COMPLETE|>"
  # Маркер завершения output
```

### LLM Parameters

```yaml
model: gpt-4                    # Primary model
fallback_model: gpt-3.5-turbo   # Cost optimization option
temperature: 0.0                # Deterministic extraction
max_tokens: 4000                # Large output capacity
top_p: 1.0                      # No nucleus sampling
frequency_penalty: 0.0          # No repetition penalty
presence_penalty: 0.0           # No novelty penalty
```

### Gleaning Configuration

```yaml
max_gleanings: 1                # Number of refinement iterations
  # 0: No gleaning (faster, lower recall)
  # 1: One refinement pass (default, balanced)
  # 2-3: Multiple passes (best recall, higher cost)
```

**Effect of gleaning**:
```
max_gleanings=0: Recall 0.75, Cost 1x
max_gleanings=1: Recall 0.85, Cost 2x
max_gleanings=2: Recall 0.90, Cost 3x
```

## Роль в цепочке преобразований

### Позиция в Indexing Chain

```
Documents
  ↓ T1: Text Chunking
Text Chunks (~1200 tokens each)
  ↓
  ├─→ [T2: ENTITY EXTRACTION PROMPT] ──→ Entities (raw)
  │                                         ↓
  └─→ [T3: RELATIONSHIP EXTRACTION] ──→ Relationships
            (combined в одном промпте)       ↓
                                          Graph (E+R)
                                             ↓
                                    T4: Summarization
                                             ↓
                                    Consolidated Entities
```

### Входные данные

**Input**:
```python
chunk: TextChunk = {
    "text_chunk": "GraphRAG is a knowledge graph...",  # ~1200 tokens
    "n_tokens": 1200,
    "source_doc_indices": [5],
    "chunk_id": "doc_5_chunk_12"
}

entity_types: List[str] = [
    "organization",
    "person",
    "geo",
    "event"
]
```

**Семантические характеристики input**:
- Chunk size: Optimal 1000-1500 tokens (balance context/cost)
- Entity types: Определяют focus extraction
- Language: Промпт на English, но извлекает из любого языка

### Выходные данные

**Output (raw tuple format)**:
```
("entity"|GRAPHRAG|ORGANIZATION|Knowledge graph construction system...)##
("entity"|MICROSOFT RESEARCH|ORGANIZATION|Research division of Microsoft...)##
("entity"|GPT-4|TECHNOLOGY|Large language model used for entity extraction)##
("relationship"|GRAPHRAG|MICROSOFT RESEARCH|Developed by Microsoft Research division|9)##
("relationship"|GRAPHRAG|GPT-4|Uses GPT-4 for entity and relationship extraction|8)
<|COMPLETE|>
```

**Parsed structured output**:
```python
entities: List[Entity] = [
    {
        "name": "GRAPHRAG",
        "type": "ORGANIZATION",
        "description": "Knowledge graph construction system...",
        "source_chunk_id": "doc_5_chunk_12"
    },
    {
        "name": "MICROSOFT RESEARCH",
        "type": "ORGANIZATION",
        "description": "Research division of Microsoft...",
        "source_chunk_id": "doc_5_chunk_12"
    },
    # ...
]

relationships: List[Relationship] = [
    {
        "source": "GRAPHRAG",
        "target": "MICROSOFT RESEARCH",
        "description": "Developed by Microsoft Research division",
        "strength": 9,
        "source_chunk_id": "doc_5_chunk_12"
    },
    # ...
]
```

### Downstream использование

**T4: Summarization** (следующий этап):
```
Entities (multiple descriptions from different chunks):
  GRAPHRAG: ["Knowledge graph construction system...",
             "Uses LLMs for extraction...",
             "Developed by Microsoft Research..."]
        ↓ T4: SUMMARIZATION PROMPT
Consolidated Entity:
  GRAPHRAG: "Knowledge graph construction system developed by Microsoft
             Research that uses LLMs like GPT-4 for entity extraction..."
```

**T5: Community Detection**:
```
Graph: Entities + Relationships
        ↓ Leiden Algorithm
Communities: Groups of related entities
```

## Семантические характеристики

| Характеристика | Значение | Описание |
|---|---|---|
| **Semantic Preservation** | 70-85% | Концепты из текста → entities |
| **Information Density** | 60% | Абстракция от full text к concepts |
| **Precision** | 0.92 | 92% извлеченных entities корректны |
| **Recall** | 0.75 (no gleaning) | 75% entities найдены |
| **Recall** | 0.85 (gleaning=1) | +10% с одним gleaning pass |
| **F1 Score** | 0.83 | Harmonic mean P/R |
| **Hallucination Rate** | 0.08 | 8% entities не из текста |
| **Type Accuracy** | 0.95 | 95% correct entity type |

### Качественные аспекты

**Что сохраняется**:
✅ Key concepts и named entities
✅ Relationships между entities
✅ Contextual descriptions
✅ Attribute information

**Что теряется**:
❌ Non-entity information (background context)
❌ Exact phrasing (абстрагировано)
❌ Некоторые implicit relationships
❌ Fine-grained details не относящиеся к entities

**Пример semantic loss**:
```
Input text (300 tokens):
"GraphRAG is an innovative knowledge graph construction system developed
by Microsoft Research. It represents a significant advancement in the field
of information extraction, utilizing state-of-the-art large language models
such as GPT-4. The system is designed to extract entities and relationships
from unstructured text with high accuracy. Many researchers consider it a
breakthrough in RAG technology..."

Extracted entities (60 tokens equivalent):
- GRAPHRAG (ORGANIZATION): Knowledge graph construction system by MS Research
- MICROSOFT RESEARCH (ORGANIZATION): Research division of Microsoft
- GPT-4 (TECHNOLOGY): LLM used for extraction

Relationship (20 tokens):
- GRAPHRAG --[developed by]--> MICROSOFT RESEARCH (strength: 9)
- GRAPHRAG --[uses]--> GPT-4 (strength: 8)

Preservation: ~80 tokens / 300 tokens = 26% token-wise
              BUT captures 85% of key semantic concepts
```

## Обработка ошибок и граничных случаев

### Common Extraction Errors

**1. Missed Entities (15-25% без gleaning)**:
```
Text: "John Smith, CEO of Acme Corp, announced the merger."

Missed: "John Smith" (если LLM focus на organization)

Mitigation:
  - Gleaning: "Were any PERSON entities missed?"
  - Better examples с PERSON entities
```

**2. Hallucinated Entities (8%)**:
```
Text: "The company uses advanced AI technology."

Hallucinated: "ADVANCED AI" (слишком generic, не named entity)

Mitigation:
  - Temperature=0.0 (reduce creativity)
  - Explicit instruction: "Identify NAMED entities, not generic terms"
```

**3. Wrong Entity Type (5%)**:
```
Text: "Microsoft Research published a paper."

Extracted: MICROSOFT RESEARCH (PERSON) ← WRONG, should be ORGANIZATION

Mitigation:
  - More examples с ambiguous names
  - Contextual clues в description
```

**4. Incorrect Relationship Strength (10-15%)**:
```
Text: "GraphRAG uses GPT-4 extensively."

Extracted: strength=5 ← Understated (should be 8-9 based on "extensively")

Mitigation:
  - Calibration examples
  - Explicit rubric: "1-3: weak, 4-6: moderate, 7-10: strong"
```

### Edge Cases

**Multi-word entities с special characters**:
```
Input: "AT&T's 5G network"
Correct: ("entity"|AT&T|ORGANIZATION|Telecommunications company)
         ("entity"|5G NETWORK|TECHNOLOGY|Fifth generation mobile network)
```

**Ambiguous entity types**:
```
Input: "Paris Climate Agreement"
Ambiguous: PARIS - GEO or part of event name?
Correct: ("entity"|PARIS CLIMATE AGREEMENT|EVENT|International treaty...)
```

**Nested entities**:
```
Input: "Microsoft Research Asia"
Options:
  1. Single entity: MICROSOFT RESEARCH ASIA (ORGANIZATION)
  2. Nested: MICROSOFT (ORGANIZATION) + RESEARCH ASIA (ORGANIZATION)
Prompt favors: Single entity (more specific)
```

## Связи с другими промптами

### Sequential Dependencies

```
ENTITY EXTRACTION (T2/T3)
        ↓ provides entities
SUMMARIZATION PROMPT (T4)
        ↓ consolidates descriptions
COMMUNITY REPORT PROMPT (T6)
        ↓ uses consolidated entities
```

### Data Flow

```
Entity Extraction Output:
  Entity: GRAPHRAG
  Descriptions (from multiple chunks): [desc1, desc2, desc3]
        ↓
Summarization Input:
  Entity: GRAPHRAG
  Description List: [desc1, desc2, desc3]
        ↓
Summarization Output:
  Consolidated Description: "GraphRAG is a knowledge graph construction
  system developed by Microsoft Research that uses LLMs..."
        ↓
Community Report Input:
  Entities: [GRAPHRAG, LLM, KNOWLEDGE GRAPH, ...]
  Relationships: [GRAPHRAG--uses-->LLM, ...]
```

## Используемые библиотеки и инструменты

### LLM Client

```python
# graphrag/llm/openai/openai_chat_llm.py
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=config.api_key)

response = await client.chat.completions.create(
    model="gpt-4",
    messages=[
        {"role": "system", "content": GRAPH_EXTRACTION_PROMPT},
        {"role": "user", "content": formatted_input}
    ],
    temperature=0.0,
    max_tokens=4000
)
```

**Файлы**:
- `graphrag/llm/openai/openai_chat_llm.py`
- `graphrag/llm/base/base_llm.py`

### Prompt Formatting

```python
# graphrag/prompts/index/extract_graph.py
formatted_prompt = GRAPH_EXTRACTION_PROMPT.format(
    entity_types=", ".join(entity_types),
    input_text=chunk.text_chunk,
    tuple_delimiter="|",
    record_delimiter="##",
    completion_delimiter="<|COMPLETE|>"
)
```

### Response Parsing

```python
# graphrag/index/operations/extract_entities/extract_entities.py
import re

def parse_extraction_response(response: str) -> tuple:
    # Split by record delimiter
    records = response.split("##")

    entities = []
    relationships = []

    for record in records:
        if "<|COMPLETE|>" in record:
            continue

        # Parse tuple format: ("entity"|NAME|TYPE|DESC)
        match = re.match(
            r'\("(\w+)"\|([^|]+)\|([^|]+)\|([^)]+)\)',
            record.strip()
        )

        if match:
            record_type = match.group(1)
            if record_type == "entity":
                entities.append({
                    "name": match.group(2),
                    "type": match.group(3),
                    "description": match.group(4)
                })
            elif record_type == "relationship":
                # Parse relationship (4 fields)
                # ...

    return entities, relationships
```

**Файлы**:
- `graphrag/index/operations/extract_entities/extract_entities.py`
- `graphrag/index/operations/extract_entities/typing.py`

### Gleaning Implementation

```python
# graphrag/index/operations/extract_entities/extract_entities.py
async def extract_entities_with_gleaning(
    chunk: TextChunk,
    entity_types: List[str],
    max_gleanings: int = 1
) -> tuple:
    # Initial extraction
    entities, relationships = await extract_entities(chunk, entity_types)

    # Gleaning iterations
    for i in range(max_gleanings):
        # Ask: Were entities missed?
        continue_response = await llm(
            prompt=CONTINUE_PROMPT,
            previous_output=entities + relationships
        )

        # Check if more entities found
        additional_entities, additional_rels = parse_extraction_response(
            continue_response
        )

        if len(additional_entities) == 0:
            break

        entities.extend(additional_entities)
        relationships.extend(additional_rels)

    return deduplicate(entities), deduplicate(relationships)
```

**Файлы**:
- `graphrag/prompts/index/extract_graph.py` (CONTINUE_PROMPT, LOOP_PROMPT)

### Rate Limiting

```python
# graphrag/llm/limiting/llm_limiter.py
from graphrag.llm.limiting import LLMLimiter

limiter = LLMLimiter(
    tokens_per_minute=50_000,
    requests_per_minute=1_000
)

# Wraps LLM calls
async def rate_limited_extraction(chunk):
    async with limiter:
        return await extract_entities(chunk)
```

**Файлы**:
- `graphrag/llm/limiting/llm_limiter.py`
- `graphrag/llm/base/rate_limiting_llm.py`

## Оптимизация и Best Practices

### Оптимизация стоимости

**1. Model Selection**:
```yaml
# High quality (default)
model: gpt-4
cost_per_1k_docs: $96

# Cost-optimized
model: gpt-3.5-turbo
cost_per_1k_docs: $9.60
quality_loss: ~10-15%
```

**2. Gleaning Trade-off**:
```yaml
# Fast (no gleaning)
max_gleanings: 0
recall: 0.75
cost: 1x

# Balanced (default)
max_gleanings: 1
recall: 0.85
cost: 2x

# High recall
max_gleanings: 2
recall: 0.90
cost: 3x
```

**3. Batch Processing**:
```python
# Process chunks in parallel (respecting rate limits)
async def batch_extract(chunks, batch_size=25):
    tasks = [extract_entities(chunk) for chunk in chunks]
    results = await asyncio.gather(*tasks)
    return results
```

### Оптимизация качества

**1. Custom Entity Types**:
```python
# Domain-specific types (scientific papers)
entity_types = [
    "method",           # Algorithms, techniques
    "metric",           # Evaluation metrics
    "dataset",          # Data sources
    "finding",          # Research findings
    "researcher"        # Authors, scientists
]
```

**2. Domain-Specific Examples**:
```python
# Add domain examples to few-shot
CUSTOM_EXAMPLE = """
Entity_types: METHOD,METRIC,DATASET
Text:
We evaluated BERT on the SQuAD dataset using F1 score and exact match metrics.

Output:
("entity"|BERT|METHOD|Bidirectional transformer model)##
("entity"|SQUAD|DATASET|Question answering dataset)##
("entity"|F1 SCORE|METRIC|Harmonic mean of precision and recall)##
("entity"|EXACT MATCH|METRIC|Binary accuracy metric)##
("relationship"|BERT|SQUAD|Evaluated on|9)
<|COMPLETE|>
"""
```

**3. Post-Processing Filters**:
```python
def filter_low_quality_entities(entities):
    return [
        e for e in entities
        if len(e.name) > 2                    # Not too short
        and len(e.description) > 10           # Has meaningful description
        and not is_generic_term(e.name)       # Not "company", "person", etc.
    ]
```

## Метрики производительности

### Latency

```
Per chunk (1200 tokens):
  - LLM call: 2-4 sec (GPT-4)
  - Parsing: <100ms
  - Total: ~2.5 sec

With gleaning (max_gleanings=1):
  - Initial call: 2.5 sec
  - Gleaning call: 2.5 sec
  - Total: ~5 sec

Throughput (25 concurrent):
  - Without gleaning: ~10 chunks/sec
  - With gleaning: ~5 chunks/sec
```

### Cost Breakdown (1000 documents → ~8000 chunks)

```
Model: GPT-4
  - Input tokens: 9.6M (~1200 per chunk)
  - Output tokens: 480K (~60 per chunk)
  - Cost: $96.00

Model: GPT-3.5-turbo
  - Input tokens: 9.6M
  - Output tokens: 480K
  - Cost: $9.60

Savings with GPT-3.5: 90% cost reduction
Quality impact: -10-15% F1 score
```

## Связанные ресурсы

### Документация

- **[Indexing Transformations](../transform/01-indexing-transformations.md)** — T2 & T3 детали
- **[Transformation Chains](../transform/03-transformation-chains.md)** — роль в pipeline
- **[Summarization Prompt](02-summarization.md)** — следующий этап

### Код

- **Промпт**: `graphrag/prompts/index/extract_graph.py`
- **Execution**: `graphrag/index/operations/extract_entities/extract_entities.py`
- **Parsing**: `graphrag/index/operations/extract_entities/typing.py`
- **LLM calls**: `graphrag/llm/openai/openai_chat_llm.py`

### Конфигурация

```yaml
# settings.yaml
extract_graph:
  prompt: null  # Use default GRAPH_EXTRACTION_PROMPT
  entity_types:
    - organization
    - person
    - geo
    - event
  max_gleanings: 1
  strategy: null
  encoding_model: cl100k_base
  model_id: "default_chat_model"
```

---

**Тип преобразования**: T2 (Entity Extraction) + T3 (Relationship Extraction)
**Критичность**: ⭐⭐⭐⭐⭐ (Highest - фундамент knowledge graph)
**Стоимость**: 65% от total indexing cost
