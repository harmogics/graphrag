# Извлечение сущностей и отношений (Entity & Relationship Extraction)

## Обзор этапа

Извлечение сущностей и отношений — это ключевой этап pipeline, где **языковые модели (LLM)** анализируют каждый текстовый фрагмент (text_unit) и идентифицируют:
- **Entities (сущности)**: Ключевые концепты, объекты, персоны, организации, места, события
- **Relationships (отношения)**: Связи между сущностями с описанием и силой связи

Этот процесс преобразует неструктурированный текст в **структурированный граф знаний**.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/extract_graph.py`
- **Операция**: `graphrag/index/operations/extract_graph/extract_graph.py`
- **Graph Extractor**: `graphrag/index/operations/extract_graph/graph_extractor.py`
- **Промпт**: `graphrag/prompts/index/extract_graph.py`
- **Конфигурация**: `graphrag/config/models/extract_graph_config.py`

## Процесс извлечения

```
INPUT: text_units.parquet
    ↓
┌─────────────────────────────────────────┐
│  1. Загрузка text_units                 │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  2. Для каждого text_unit:              │
│     ┌─────────────────────────────────┐ │
│     │ a. Формирование промпта         │ │
│     │ b. LLM call (Extraction)        │ │
│     │ c. Парсинг результата           │ │
│     │ d. Gleaning (опционально)       │ │
│     └─────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  3. Агрегация всех извлеченных данных   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  4. Группировка по entity names         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  5. Суммаризация описаний (LLM)         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  6. Создание финальных таблиц           │
└─────────────────────────────────────────┘
    ↓
OUTPUT: entities.parquet + relationships.parquet
```

## Роль LLM Агента извлечения (Extraction Agent)

### Функция агента

**Extraction Agent** — это специализированный LLM-агент, который анализирует текст и извлекает структурированную информацию о сущностях и их связях.

### Конфигурация агента

```python
class ExtractGraphConfig:
    # Модель LLM для извлечения
    model_id: str = DEFAULT_CHAT_MODEL_ID  # Обычно GPT-4

    # Типы сущностей для извлечения
    entity_types: list[str] = [
        "organization",
        "person",
        "geo",
        "event"
    ]

    # Путь к кастомному промпту (опционально)
    prompt: str | None = None

    # Количество итераций "gleaning" для улучшения результата
    max_gleanings: int = 0

    # Override стратегии извлечения
    strategy: dict | None = None

    # Модель для токенизации
    encoding_model: str = "cl100k_base"
```

**Файл**: `graphrag/config/models/extract_graph_config.py:17`

### Промпт агента извлечения

Агент использует структурированный промпт из `graphrag/prompts/index/extract_graph.py`:

```
-Goal-
Given a text document that is potentially relevant to this activity and
a list of entity types, identify all entities of those types from the
text and all relationships among the identified entities.

-Steps-
1. Identify all entities. For each identified entity, extract:
   - entity_name: Name of the entity, capitalized
   - entity_type: One of the following types: [organization, person, geo, event]
   - entity_description: Comprehensive description of the entity's attributes and activities
   Format: ("entity"<|><entity_name><|><entity_type><|><entity_description>)

2. From the entities identified in step 1, identify all pairs of
   (source_entity, target_entity) that are *clearly related* to each other.
   For each pair extract:
   - source_entity: name of the source entity
   - target_entity: name of the target entity
   - relationship_description: explanation of the relationship
   - relationship_strength: numeric score (1-10) indicating strength
   Format: ("relationship"<|><source><|><target><|><description><|><strength>)

3. Return output in English as a single list using **##** as delimiter.

4. When finished, output <|COMPLETE|>
```

**Файл**: `graphrag/prompts/index/extract_graph.py:6`

### Пример работы агента

**Входной текст**:
```
The Verdantis's Central Institution is scheduled to meet on Monday and Thursday,
with the institution planning to release its latest policy decision on Thursday
at 1:30 p.m. PDT, followed by a press conference where Central Institution Chair
Martin Smith will take questions.
```

**Выход агента**:
```
("entity"<|>CENTRAL INSTITUTION<|>ORGANIZATION<|>The Central Institution is the
Federal Reserve of Verdantis, which is setting interest rates on Monday and Thursday)
##
("entity"<|>MARTIN SMITH<|>PERSON<|>Martin Smith is the chair of the Central Institution)
##
("entity"<|>MARKET STRATEGY COMMITTEE<|>ORGANIZATION<|>The Central Institution
committee makes key decisions about interest rates and the growth of Verdantis's money supply)
##
("relationship"<|>MARTIN SMITH<|>CENTRAL INSTITUTION<|>Martin Smith is the Chair
of the Central Institution and will answer questions at a press conference<|>9)
<|COMPLETE|>
```

## Процесс Gleaning (Доочистка)

### Что такое Gleaning?

**Gleaning** — это итеративный процесс улучшения результатов extraction, где агент несколько раз проходит по тексту, чтобы найти пропущенные сущности и отношения.

### Механизм работы

```python
# Псевдокод gleaning процесса

extraction_results = []

# Первичная экстракция
result = llm_call(extraction_prompt, text)
extraction_results.append(result)

# Gleaning iterations
for i in range(max_gleanings):
    # Промпт: "Найди пропущенные сущности и отношения"
    continue_prompt = CONTINUE_PROMPT + format_existing_results(extraction_results)

    additional_result = llm_call(continue_prompt, text)

    # Проверка, найдено ли что-то новое
    loop_result = llm_call(LOOP_PROMPT, text)  # Ответ: Y/N

    if loop_result == "N":
        break  # Больше нечего извлекать

    extraction_results.append(additional_result)

# Объединить все результаты
final_entities, final_relationships = merge_results(extraction_results)
```

### Промпты для Gleaning

**CONTINUE_PROMPT**:
```
MANY entities and relationships were missed in the last extraction.
Remember to ONLY emit entities that match any of the previously extracted types.
Add them below using the same format:
```

**LOOP_PROMPT**:
```
It appears some entities and relationships may have still been missed.
Answer Y or N if there are still entities or relationships that need to be added.
```

**Файл**: `graphrag/prompts/index/extract_graph.py:128`

### Когда использовать Gleaning?

- **max_gleanings = 0** (по умолчанию): Быстрое извлечение, одна итерация
- **max_gleanings = 1-2**: Умеренное улучшение качества (+20-30% recall)
- **max_gleanings = 3+**: Максимальное качество, но высокая стоимость

**Рекомендация**: Для большинства случаев достаточно max_gleanings = 0 или 1.

## Парсинг результатов LLM

### Формат вывода

LLM возвращает строку в специальном формате с разделителями:

```
("entity"<|>NAME<|>TYPE<|>DESCRIPTION)
##
("relationship"<|>SOURCE<|>TARGET<|>DESCRIPTION<|>STRENGTH)
##
...
<|COMPLETE|>
```

### Парсер

```python
def parse_extraction_result(
    result: str,
    tuple_delimiter: str = "<|>",
    record_delimiter: str = "##"
) -> tuple[list[Entity], list[Relationship]]:
    """
    Парсит вывод LLM и преобразует в структурированные данные
    """
    entities = []
    relationships = []

    # Разделить по record_delimiter
    records = result.split(record_delimiter)

    for record in records:
        record = record.strip()

        # Пропустить completion delimiter
        if "<|COMPLETE|>" in record:
            continue

        # Парсинг entity
        if record.startswith('("entity"'):
            # Извлечь поля: name, type, description
            parts = record.split(tuple_delimiter)
            entity = Entity(
                name=parts[1].strip(),
                type=parts[2].strip(),
                description=parts[3].strip().rstrip(')')
            )
            entities.append(entity)

        # Парсинг relationship
        elif record.startswith('("relationship"'):
            # Извлечь поля: source, target, description, strength
            parts = record.split(tuple_delimiter)
            relationship = Relationship(
                source=parts[1].strip(),
                target=parts[2].strip(),
                description=parts[3].strip(),
                strength=int(parts[4].strip().rstrip(')'))
            )
            relationships.append(relationship)

    return entities, relationships
```

**Файл реализации**: `graphrag/index/operations/extract_graph/graph_extractor.py:150`

## Агрегация и суммаризация

### Проблема множественных описаний

После извлечения из всех text_units одна и та же сущность может быть упомянута многократно с разными описаниями:

```
Entity: "GraphRAG"
Описание 1: "GraphRAG is a knowledge graph construction framework"
Описание 2: "GraphRAG uses LLMs to extract entities and relationships"
Описание 3: "GraphRAG supports hierarchical community detection"
```

### Роль Summarization Agent

**Summarization Agent** — это второй LLM-агент, который объединяет множественные описания в одно краткое и полное.

### Процесс суммаризации

```python
async def summarize_descriptions(
    entities: pd.DataFrame,
    summarization_strategy: dict,
    cache: PipelineCache
) -> pd.DataFrame:
    """
    Для каждой уникальной сущности:
    1. Собрать все её описания из разных text_units
    2. Вызвать LLM для создания единого краткого описания
    3. Заменить множественные описания одним
    """

    grouped = entities.groupby("entity_name")

    for entity_name, group in grouped:
        descriptions = group["description"].tolist()

        if len(descriptions) == 1:
            # Если только одно описание, оставить как есть
            continue

        # Объединить описания для LLM
        combined = "\n".join(descriptions)

        # Промпт для суммаризации
        prompt = f"""
        The following are descriptions of the same entity "{entity_name}".
        Please create a single, comprehensive, and concise description that
        captures all the key information:

        {combined}

        Output only the final consolidated description.
        """

        # LLM call
        summary = await llm_call(prompt, cache_key=entity_name)

        # Обновить все записи этой сущности
        entities.loc[entities["entity_name"] == entity_name, "description"] = summary

    return entities
```

### Промпт суммаризации

```python
# graphrag/prompts/index/summarize_descriptions.py

SUMMARIZE_PROMPT = """
You are a helpful assistant responsible for generating a comprehensive summary
of the data provided below.
Given one or two entities, and a list of descriptions, all related to the same
entity or group of entities.
Please concatenate all of these into a single, comprehensive description.
Make sure to include information collected from all the descriptions.
If the provided descriptions are contradictory, please resolve the contradictions
and provide a single, coherent summary.
Make sure it is written in third person, and include the entity names so we have
the full context.

#############################
Entity: {entity_name}
Descriptions: {description_list}
#############################
Output:
"""
```

**Файл**: `graphrag/prompts/index/summarize_descriptions.py:6`

## Структуры данных выхода

### Entity Structure

```python
class Entity:
    id: str                     # SHA512 hash (entity_name)
    human_readable_id: str      # Например: "entity_001"
    title: str                  # Название сущности (capitalized)
    type: str                   # Тип: organization, person, geo, event
    description: str            # Суммаризированное описание
    text_unit_ids: list[str]    # IDs text_units, где упомянута

    # Добавляется на этапе graph construction:
    node_frequency: int         # Количество упоминаний
    node_degree: int            # Степень узла в графе
    node_x: float               # X-координата в layout
    node_y: float               # Y-координата в layout
```

### Relationship Structure

```python
class Relationship:
    id: str                     # SHA512 hash (source + target + description)
    human_readable_id: str      # Например: "rel_001"
    source: str                 # Entity title источника
    target: str                 # Entity title цели
    description: str            # Описание отношения
    weight: float               # 1.0 / strength (для графа)
    text_unit_ids: list[str]    # IDs text_units, где упомянуто

    # Добавляется на этапе graph construction:
    combined_degree: int        # Сумма степеней source и target
    source_degree: int          # Степень source entity
    target_degree: int          # Степень target entity
```

### Схемы Parquet таблиц

**entities.parquet**:
```python
COLUMNS = [
    "id",                 # str
    "human_readable_id",  # str
    "title",              # str (entity name)
    "type",               # str
    "description",        # str
    "text_unit_ids",      # list[str]
]
```

**relationships.parquet**:
```python
COLUMNS = [
    "id",                 # str
    "human_readable_id",  # str
    "source",             # str (entity title)
    "target",             # str (entity title)
    "description",        # str
    "weight",             # float
    "text_unit_ids",      # list[str]
]
```

**Файл схем**: `graphrag/data_model/schemas.py`

## Обработка ошибок и кэширование

### LLM Error Handling

```python
async def extract_with_retry(
    text: str,
    extractor: GraphExtractor,
    max_retries: int = 3
) -> GraphExtractionResult:
    """
    Извлечение с повторными попытками при ошибках
    """
    for attempt in range(max_retries):
        try:
            result = await extractor([text])
            return result
        except Exception as e:
            log.warning(f"Extraction failed (attempt {attempt + 1}): {e}")
            if attempt == max_retries - 1:
                # Последняя попытка, вернуть пустой результат
                return GraphExtractionResult(
                    output=nx.Graph(),
                    source_docs={}
                )
            await asyncio.sleep(2 ** attempt)  # Exponential backoff
```

### LLM Response Caching

Все LLM-вызовы кэшируются для идемпотентности:

```python
# Ключ кэша: hash(prompt + model_config + input_text)
cache_key = hashlib.sha256(
    f"{prompt}{model_id}{temperature}{input_text}".encode()
).hexdigest()

# Проверить кэш
cached_result = await cache.get(cache_key)
if cached_result:
    return cached_result

# Вызвать LLM
result = await llm_call(prompt, input_text)

# Сохранить в кэш
await cache.set(cache_key, result)
```

**Преимущества**:
- Повторное выполнение не создает дополнительных расходов
- Восстановление после сбоев без потери прогресса
- Детерминированность результатов

**Файл кэша**: `graphrag/cache/pipeline_cache.py`

## Асинхронная обработка

### Параллелизм extraction

```python
async def extract_graph_parallel(
    text_units: pd.DataFrame,
    extractor: GraphExtractor,
    num_threads: int = 4
) -> tuple[pd.DataFrame, pd.DataFrame]:
    """
    Параллельная обработка text_units
    """
    # Разбить на батчи
    batches = [text_units[i::num_threads] for i in range(num_threads)]

    # Асинхронно обработать каждый батч
    tasks = [
        extract_batch(batch, extractor)
        for batch in batches
    ]

    results = await asyncio.gather(*tasks)

    # Объединить результаты
    all_entities = pd.concat([r[0] for r in results])
    all_relationships = pd.concat([r[1] for r in results])

    return all_entities, all_relationships
```

### Конфигурация параллелизма

```python
# В LanguageModelConfig
class LanguageModelConfig:
    concurrent_requests: int = 3  # Количество одновременных LLM-запросов
    async_mode: AsyncType = AsyncIO  # AsyncIO | Threads
```

## Примеры кастомизации

### Кастомные типы сущностей

```python
# config.yaml
extract_graph:
  entity_types:
    - organization
    - person
    - geo
    - event
    - product        # Кастомный тип
    - technology     # Кастомный тип
    - concept        # Кастомный тип
```

### Кастомный промпт

```python
# custom_extraction_prompt.txt

-Goal-
Extract entities and relationships with focus on technical terminology.

-Steps-
1. Identify technical terms, concepts, and technologies
2. Extract relationships with emphasis on "uses", "implements", "extends"
3. Include version numbers and technical specifications

[... rest of prompt ...]
```

```yaml
# config.yaml
extract_graph:
  prompt: "./custom_extraction_prompt.txt"
```

### Многоязычная обработка

```python
# Промпт для русского языка
GRAPH_EXTRACTION_PROMPT_RU = """
-Цель-
Извлечь из текста все сущности и отношения между ними.

-Шаги-
1. Идентифицировать все сущности...
[...]
"""
```

## Метрики качества extraction

### Оценка результатов

```python
extraction_stats = {
    "total_text_units": 1000,
    "total_entities_extracted": 5420,
    "total_relationships_extracted": 8750,
    "unique_entities": 2340,
    "avg_entities_per_text_unit": 5.42,
    "avg_relationships_per_text_unit": 8.75,
    "entity_types_distribution": {
        "organization": 890,
        "person": 1120,
        "geo": 230,
        "event": 100
    },
    "avg_entity_description_length": 87,  # tokens
    "avg_relationship_strength": 6.2,
    "llm_tokens_used": 3_400_000,
    "llm_cost_usd": 68.00
}
```

## Следующий этап

После извлечения сущностей и отношений:
- **finalize_graph**: Финализация графа, вычисление степеней узлов
- **create_communities**: Построение графа и выявление тематических сообществ

→ [Переход к построению графа знаний](03-graph-construction.md)
