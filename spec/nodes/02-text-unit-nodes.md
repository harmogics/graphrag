# Text Unit Nodes (Узлы текстовых фрагментов)

## Обзор

**Text Unit Nodes** — это текстовые фрагменты (chunks), полученные после разбиения документов. Они являются **единицей обработки** для LLM и ключевым компонентом для Local Search.

## Структура Text Unit Node

### Атрибуты

```python
class TextUnit:
    # Идентификация
    id: str                         # Уникальный ID (SHA512 hash)
    human_readable_id: str          # "text_unit_0001"

    # Содержимое
    text: str                       # Текст chunk
    n_tokens: int                   # Количество токенов

    # Связи назад (к источникам)
    document_ids: list[str]         # Исходные документы

    # Связи вперед (к извлеченным данным)
    entity_ids: list[str]           # Entities в этом text_unit
    relationship_ids: list[str]     # Relationships в этом text_unit
    covariate_ids: list[str]        # Claims в этом text_unit

    # Метаданные
    metadata: dict | None           # Дополнительная информация
```

**Файлы данных**:
- `output/text_units.parquet` (финальная версия)
- `cache/base_text_units.parquet` (промежуточная)

## Роль в архитектуре

### Позиция в иерархии

```
┌─────────────────────────────────────────────┐
│          DOCUMENT NODES                     │
│  Полные документы                           │
└──────────────────┬──────────────────────────┘
                   │ Chunking (size=1200, overlap=100)
                   ↓
┌─────────────────────────────────────────────┐
│       TEXT UNIT NODES (Вы здесь)            │
│  • Управляемые фрагменты для LLM            │
│  • Единица extraction                       │
│  • Контекстная единица для retrieval        │
└──────────────────┬──────────────────────────┘
                   │ LLM Extraction
                   ↓
┌─────────────────────────────────────────────┐
│          ENTITY & RELATIONSHIP NODES        │
│  Структурированные данные                   │
└─────────────────────────────────────────────┘
```

### Связи с другими узлами

```
Text Unit
   ├─► belongs_to → Document (N:1 или N:M через chunking overlap)
   │
   ├─► contains_entity → Entity (N:M)
   │
   ├─► mentions_relationship → Relationship (N:M)
   │
   └─► has_claim → Covariate (N:M)
```

## Создание Text Unit Nodes

### Workflow: create_base_text_units

```python
# graphrag/index/workflows/create_base_text_units.py

async def create_base_text_units(
    config: GraphRagConfig,
    context: PipelineRunContext
) -> pd.DataFrame:
    """
    Разбивает документы на text_units

    Process:
    1. Загрузить documents
    2. Для каждого документа:
       - Применить TokenTextSplitter
       - Создать text_units с overlap
       - Генерировать SHA512 ID
       - Подсчитать токены
    3. Сохранить base_text_units
    """

    documents = await context.storage.get("documents")

    text_units = []
    for _, doc in documents.iterrows():
        # Chunking
        chunks = chunk_text(
            doc["text"],
            chunk_size=config.chunking.size,
            overlap=config.chunking.overlap
        )

        # Создать text_units
        for i, chunk in enumerate(chunks):
            unit = TextUnit(
                id=generate_chunk_id(chunk, doc["id"], i),
                text=chunk,
                n_tokens=count_tokens(chunk),
                document_ids=[doc["id"]]
            )
            text_units.append(unit)

    df = pd.DataFrame(text_units)
    await context.storage.set("text_units", df)
    return df
```

### Финализация text_units

После extraction, text_units обновляются с entity/relationship IDs:

```python
# graphrag/index/workflows/create_final_text_units.py

async def create_final_text_units(
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    base_text_units: pd.DataFrame
) -> pd.DataFrame:
    """
    Добавляет entity_ids и relationship_ids к text_units
    """
    final_units = base_text_units.copy()

    # Для каждого text_unit
    for idx, unit in final_units.iterrows():
        unit_id = unit["id"]

        # Найти entities, упомянутые в этом text_unit
        unit_entities = entities[
            entities["text_unit_ids"].apply(
                lambda ids: unit_id in ids
            )
        ]
        final_units.at[idx, "entity_ids"] = unit_entities["id"].tolist()

        # Найти relationships
        unit_relationships = relationships[
            relationships["text_unit_ids"].apply(
                lambda ids: unit_id in ids
            )
        ]
        final_units.at[idx, "relationship_ids"] = unit_relationships["id"].tolist()

    return final_units
```

## Семантическое перекрытие (Overlap)

### Почему overlap критичен

```
Document: "... A B C D E F G H I J K L ..."

С overlap=0:
Chunk 1: [A B C D]
Chunk 2: [E F G H]
Chunk 3: [I J K L]
❌ Сущность на границе (D-E) может быть потеряна

С overlap=2:
Chunk 1: [A B C D]
Chunk 2:     [C D E F G H]
Chunk 3:         [G H I J K L]
✅ Сущности на границах сохранены в обоих chunks
```

### Влияние на граф

```python
# Пример: Entity "GraphRAG" упоминается на границе chunks

# Text Unit 1 (конец):
"...and this is where GraphRAG comes in."

# Text Unit 2 (начало) - с overlap:
"GraphRAG comes in. GraphRAG uses LLMs to extract..."

# Результат extraction:
# • Text Unit 1 → Entity "GraphRAG" (неполное упоминание)
# • Text Unit 2 → Entity "GraphRAG" + описание
# → После deduplication: одна entity с полным контекстом
```

## Использование в Query Execution

### Роль в Local Search (primary)

Text Units — основной источник контекста для детальных вопросов:

```python
def local_search(
    query: str,
    entities: pd.DataFrame,
    text_units: pd.DataFrame,
    embeddings: pd.DataFrame
) -> str:
    """
    Local Search использует text_units для контекста
    """
    # 1. Найти релевантные entities
    relevant_entities = semantic_search(
        query,
        entity_embeddings,
        top_k=20
    )

    # 2. Собрать text_unit_ids из этих entities
    text_unit_ids = set()
    for entity in relevant_entities:
        text_unit_ids.update(entity["text_unit_ids"])

    # 3. Получить text_units
    context_units = text_units[text_units["id"].isin(text_unit_ids)]

    # 4. Ранжировать text_units по релевантности
    ranked_units = rank_text_units_by_relevance(
        query,
        context_units,
        text_unit_embeddings
    )

    # 5. Построить контекст из топ text_units
    context = build_context_from_text_units(ranked_units[:10])

    # 6. LLM генерирует ответ
    answer = llm_generate(
        query,
        context,
        system_prompt="Answer based on the provided text units."
    )

    return answer
```

### Роль в Drift Search

Text Units используются для исследовательского поиска:

```python
def drift_search(
    query: str,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    text_units: pd.DataFrame,
    max_hops: int = 3
) -> dict:
    """
    Drift Search: исследовательский поиск с graph walk
    """
    visited_entities = set()
    collected_units = []

    # Начать с релевантных entities
    current_entities = find_relevant_entities(query, entities, top_k=5)

    for hop in range(max_hops):
        # Собрать text_units текущих entities
        for entity in current_entities:
            if entity["id"] not in visited_entities:
                visited_entities.add(entity["id"])

                # Получить text_units этой entity
                entity_units = text_units[
                    text_units["entity_ids"].apply(
                        lambda ids: entity["id"] in ids
                    )
                ]
                collected_units.extend(entity_units.to_dict('records'))

        # Найти связанные entities через relationships
        current_entities = find_related_entities(
            current_entities,
            relationships,
            entities
        )

    # Построить answer с follow-up вопросами
    context = build_context_from_text_units(collected_units)
    answer = llm_generate_with_followups(query, context)

    return {
        "answer": answer["text"],
        "follow_up_questions": answer["questions"],
        "num_text_units_used": len(collected_units),
        "num_entities_visited": len(visited_entities)
    }
```

## Text Unit Embeddings

### Создание embeddings

```python
# Text Unit embeddings для semantic search
text_unit_embeddings = create_embeddings(
    text_units["text"].tolist(),
    model="text-embedding-3-small",
    batch_size=500
)

# Сохранить
embeddings_df = pd.DataFrame({
    "id": text_units["id"],
    "embedding": text_unit_embeddings
})

embeddings_df.to_parquet("embeddings.text_unit_text_embedding.parquet")
```

### Семантический поиск по text_units

```python
def semantic_search_text_units(
    query: str,
    text_units: pd.DataFrame,
    text_unit_embeddings: pd.DataFrame,
    top_k: int = 10
) -> pd.DataFrame:
    """
    Прямой семантический поиск по text_units
    """
    # 1. Embedding query
    query_emb = create_embedding(query)

    # 2. Cosine similarity
    similarities = text_unit_embeddings["embedding"].apply(
        lambda emb: cosine_similarity(query_emb, emb)
    )

    # 3. Top-K
    top_indices = similarities.nlargest(top_k).index
    top_unit_ids = text_unit_embeddings.loc[top_indices, "id"]

    # 4. Результаты с scores
    results = text_units[text_units["id"].isin(top_unit_ids)].copy()
    results["score"] = results["id"].map(
        dict(zip(
            text_unit_embeddings.loc[top_indices, "id"],
            similarities.loc[top_indices]
        ))
    )

    return results.sort_values("score", ascending=False)
```

## Ранжирование text_units

### Комбинированное ранжирование

```python
def rank_text_units_hybrid(
    query: str,
    text_units: pd.DataFrame,
    entities: pd.DataFrame,
    text_unit_embeddings: pd.DataFrame,
    alpha: float = 0.7
) -> pd.DataFrame:
    """
    Hybrid ranking: semantic similarity + entity relevance
    """
    # 1. Semantic score
    semantic_scores = compute_semantic_scores(
        query,
        text_units,
        text_unit_embeddings
    )

    # 2. Entity relevance score
    entity_scores = compute_entity_relevance(
        query,
        text_units,
        entities
    )

    # 3. Combine
    final_scores = (
        alpha * semantic_scores +
        (1 - alpha) * entity_scores
    )

    # 4. Sort
    text_units["score"] = final_scores
    return text_units.sort_values("score", ascending=False)
```

### Ранжирование по метрикам entities

```python
def rank_by_entity_importance(
    text_units: pd.DataFrame,
    entities: pd.DataFrame
) -> pd.DataFrame:
    """
    Ранжирует text_units по важности содержащихся entities
    """
    scores = []

    for _, unit in text_units.iterrows():
        # Получить entities в этом text_unit
        unit_entities = entities[
            entities["id"].isin(unit["entity_ids"])
        ]

        # Вычислить importance score
        if len(unit_entities) > 0:
            # Средний node_degree entities
            avg_degree = unit_entities["node_degree"].mean()
            # Количество entities
            num_entities = len(unit_entities)

            score = avg_degree * num_entities
        else:
            score = 0

        scores.append(score)

    text_units["importance_score"] = scores
    return text_units.sort_values("importance_score", ascending=False)
```

## Статистика text_units

### Анализ коллекции

```python
def analyze_text_units(
    text_units: pd.DataFrame,
    entities: pd.DataFrame
) -> dict:
    """
    Анализирует коллекцию text_units
    """
    stats = {
        "total_units": len(text_units),
        "total_tokens": text_units["n_tokens"].sum(),
        "avg_tokens": text_units["n_tokens"].mean(),
        "min_tokens": text_units["n_tokens"].min(),
        "max_tokens": text_units["n_tokens"].max(),

        # Entity coverage
        "units_with_entities": len(
            text_units[text_units["entity_ids"].apply(len) > 0]
        ),
        "avg_entities_per_unit": text_units["entity_ids"].apply(len).mean(),

        # Relationship coverage
        "units_with_relationships": len(
            text_units[text_units["relationship_ids"].apply(len) > 0]
        ),
        "avg_relationships_per_unit": text_units["relationship_ids"].apply(len).mean(),

        # Distribution
        "token_distribution": {
            "< 500": len(text_units[text_units["n_tokens"] < 500]),
            "500-1000": len(text_units[
                (text_units["n_tokens"] >= 500) &
                (text_units["n_tokens"] < 1000)
            ]),
            "1000-1500": len(text_units[
                (text_units["n_tokens"] >= 1000) &
                (text_units["n_tokens"] < 1500)
            ]),
            "> 1500": len(text_units[text_units["n_tokens"] >= 1500])
        }
    }

    return stats
```

## Оптимизация использования text_units

### Кэширование контекста

```python
class TextUnitContextCache:
    """Кэш для часто используемых text_units"""

    def __init__(self):
        self.cache = {}

    def get_context(
        self,
        text_unit_ids: list[str],
        text_units: pd.DataFrame
    ) -> str:
        """
        Получает контекст для text_units с кэшированием
        """
        # Создать cache key
        cache_key = "_".join(sorted(text_unit_ids))

        if cache_key in self.cache:
            return self.cache[cache_key]

        # Построить контекст
        units = text_units[text_units["id"].isin(text_unit_ids)]
        context = "\n\n".join([
            f"[Unit {i+1}]\n{unit['text']}"
            for i, (_, unit) in enumerate(units.iterrows())
        ])

        # Кэшировать
        self.cache[cache_key] = context

        return context
```

### Де дупликация при overlap

```python
def deduplicate_overlapping_units(
    text_units: list[dict]
) -> list[dict]:
    """
    Удаляет дублированный контент из overlapping text_units
    """
    if len(text_units) <= 1:
        return text_units

    deduplicated = [text_units[0]]

    for i in range(1, len(text_units)):
        current = text_units[i]
        previous = deduplicated[-1]

        # Найти overlap
        overlap = find_text_overlap(previous["text"], current["text"])

        if overlap > 0:
            # Удалить overlapping часть из current
            current_text = remove_overlap(current["text"], overlap)
            current["text"] = current_text

        deduplicated.append(current)

    return deduplicated
```

## Визуализация text_units

### Text Unit Explorer

```python
def visualize_text_unit_coverage(
    text_unit: dict,
    entities: pd.DataFrame,
    relationships: pd.DataFrame
) -> str:
    """
    Визуализирует text_unit с аннотациями
    """
    text = text_unit["text"]

    # Получить entities в этом text_unit
    unit_entities = entities[
        entities["id"].isin(text_unit["entity_ids"])
    ]

    # Выделить entities в тексте
    annotated_text = text
    for _, entity in unit_entities.iterrows():
        entity_name = entity["title"]
        # Обернуть в markdown bold
        annotated_text = annotated_text.replace(
            entity_name,
            f"**{entity_name}** [{entity['type']}]"
        )

    # Добавить информацию
    output = f"""
# Text Unit: {text_unit['human_readable_id']}

**Tokens**: {text_unit['n_tokens']}
**Entities**: {len(text_unit['entity_ids'])}
**Relationships**: {len(text_unit['relationship_ids'])}

## Text:
{annotated_text}

## Extracted Entities:
"""

    for _, entity in unit_entities.iterrows():
        output += f"- **{entity['title']}** ({entity['type']}): {entity['description'][:100]}...\n"

    return output
```

## Best Practices

### 1. Размер chunks

```python
# Рекомендации по размеру
CHUNK_SIZE_RECOMMENDATIONS = {
    "scientific_papers": 1500,    # Больше для технического контекста
    "news_articles": 1000,        # Стандартный размер
    "social_media": 500,          # Меньше для коротких постов
    "legal_documents": 2000,      # Больше для юридического контекста
    "code_documentation": 1200    # Стандартный для технической документации
}

# Overlap обычно 8-10% от chunk_size
overlap = int(chunk_size * 0.08)
```

### 2. Обработка длинных text_units

```python
def handle_long_text_units(
    text_units: pd.DataFrame,
    max_tokens: int = 2000
) -> pd.DataFrame:
    """
    Разбивает слишком длинные text_units
    """
    processed_units = []

    for _, unit in text_units.iterrows():
        if unit["n_tokens"] > max_tokens:
            # Разбить на подчасти
            sub_chunks = split_long_text_unit(
                unit["text"],
                max_tokens=max_tokens
            )

            for i, chunk in enumerate(sub_chunks):
                new_unit = unit.copy()
                new_unit["id"] = f"{unit['id']}_part{i}"
                new_unit["text"] = chunk
                new_unit["n_tokens"] = count_tokens(chunk)
                processed_units.append(new_unit)
        else:
            processed_units.append(unit)

    return pd.DataFrame(processed_units)
```

## Следующие разделы

- **[← Document Nodes](01-document-nodes.md)**: Исходные документы
- **[Entity Nodes →](03-entity-nodes.md)**: Извлеченные сущности
- **[Query Execution →](09-query-execution.md)**: Использование в поиске
