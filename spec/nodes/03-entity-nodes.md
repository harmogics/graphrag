# Entity Nodes (Узлы сущностей)

## Обзор

**Entity Nodes** — это ключевые концепты, извлеченные из текста: организации, персоны, места, события, технологии. Entities являются **центральными узлами графа знаний** и основой для навигации и поиска.

## Структура Entity Node

### Атрибуты

```python
class Entity:
    # Идентификация
    id: str                         # Уникальный ID (SHA512 hash)
    human_readable_id: str          # "entity_0001"
    title: str                      # Название entity (normalized, capitalized)

    # Классификация
    type: str                       # Тип: organization, person, geo, event, etc.

    # Описание
    description: str                # Суммаризированное описание (LLM-generated)

    # Связи к источникам
    text_unit_ids: list[str]        # Text units где упоминается

    # Метрики графа
    node_degree: int                # Количество связей (relationships)
    node_frequency: int             # Количество упоминаний (text_units)

    # Позиция для визуализации
    node_x: float                   # X-координата
    node_y: float                   # Y-координата
```

**Файл данных**: `output/entities.parquet`

## Типы entities

### Стандартные типы

```python
STANDARD_ENTITY_TYPES = {
    "organization": {
        "description": "Компании, организации, институты",
        "examples": ["Microsoft", "OpenAI", "MIT"]
    },
    "person": {
        "description": "Имена людей",
        "examples": ["John Doe", "Marie Curie"]
    },
    "geo": {
        "description": "Географические локации",
        "examples": ["New York", "Europe", "Pacific Ocean"]
    },
    "event": {
        "description": "События, инциденты, встречи",
        "examples": ["World War II", "AI Summit 2024"]
    }
}
```

### Кастомные типы

```python
# Можно добавить domain-specific типы
CUSTOM_ENTITY_TYPES = {
    "technology": "Технологии и инструменты",
    "product": "Продукты и сервисы",
    "concept": "Абстрактные концепты",
    "method": "Методы и подходы",
    "metric": "Метрики и измерения"
}

# В конфигурации:
extract_graph:
  entity_types:
    - organization
    - person
    - geo
    - event
    - technology  # Кастомный тип
    - product     # Кастомный тип
```

## Роль в архитектуре

### Позиция в графе

```
┌─────────────────────────────────────────────┐
│          TEXT UNIT NODES                    │
│  Текстовые фрагменты                        │
└──────────────────┬──────────────────────────┘
                   │ LLM Extraction
                   ↓
┌─────────────────────────────────────────────┐
│       ENTITY NODES (Вы здесь)               │
│  • Центральные узлы графа знаний            │
│  • Ключ для навигации                       │
│  • Основа для Local Search                  │
└──────────────────┬──────────────────────────┘
                   │ Relationships
                   ↓
┌─────────────────────────────────────────────┐
│       RELATIONSHIP EDGES                    │
│  Связи между entities                      │
└──────────────────┬──────────────────────────┘
                   │ Clustering
                   ↓
┌─────────────────────────────────────────────┐
│          COMMUNITY NODES                    │
│  Тематические кластеры entities            │
└─────────────────────────────────────────────┘
```

### Связи

```
Entity
   ├─► mentioned_in → Text Unit (M:N)
   │
   ├─► related_to → Entity (M:N через Relationships)
   │
   ├─► member_of → Community (M:1 на каждом level)
   │
   └─► subject_of → Covariate/Claim (1:N)
```

## Извлечение entities

### Extraction Agent

```python
# Extraction происходит для каждого text_unit
def extract_entities_from_text_unit(
    text_unit: str,
    entity_types: list[str],
    llm: ChatModel
) -> list[Entity]:
    """
    Извлекает entities из text_unit с помощью LLM
    """
    # Промпт для extraction
    prompt = f"""
    Extract all entities from the following text.
    Entity types: {', '.join(entity_types)}

    Text:
    {text_unit}

    Format each entity as:
    ("entity"<|>NAME<|>TYPE<|>DESCRIPTION)
    """

    # LLM call
    response = await llm(prompt)

    # Парсинг
    entities = parse_extraction_response(response)

    return entities
```

**Промпт**: `graphrag/prompts/index/extract_graph.py:6`

### Deduplication и суммаризация

После extraction из всех text_units, entities дедуплицируются:

```python
def deduplicate_entities(
    raw_entities: list[Entity]
) -> list[Entity]:
    """
    Объединяет одинаковые entities из разных text_units
    """
    # Группировать по normalized title
    grouped = defaultdict(list)

    for entity in raw_entities:
        key = normalize_entity_name(entity.title)
        grouped[key].append(entity)

    # Для каждой группы, объединить
    deduplicated = []

    for entity_name, entities in grouped.items():
        # Объединить text_unit_ids
        all_text_units = []
        for e in entities:
            all_text_units.extend(e.text_unit_ids)

        # Собрать все описания для суммаризации
        descriptions = [e.description for e in entities]

        # LLM суммаризация (если > 1 описание)
        if len(descriptions) > 1:
            final_description = summarize_descriptions(
                entity_name,
                descriptions
            )
        else:
            final_description = descriptions[0]

        # Создать финальную entity
        deduplicated_entity = Entity(
            id=generate_entity_id(entity_name),
            title=entity_name,
            type=entities[0].type,  # Все должны быть одного типа
            description=final_description,
            text_unit_ids=list(set(all_text_units))
        )

        deduplicated.append(deduplicated_entity)

    return deduplicated
```

## Метрики entities

### Node Degree (степень узла)

**Определение**: Количество relationships, инцидентных entity.

```python
def compute_node_degrees(
    entities: pd.DataFrame,
    relationships: pd.DataFrame
) -> pd.DataFrame:
    """
    Вычисляет node_degree для каждой entity
    """
    degree_counts = defaultdict(int)

    for _, rel in relationships.iterrows():
        degree_counts[rel["source"]] += 1
        degree_counts[rel["target"]] += 1

    entities["node_degree"] = entities["title"].map(degree_counts).fillna(0)

    return entities
```

**Интерпретация**:
- **Высокий degree (>20)**: Hub entity, центральный концепт
- **Средний degree (5-20)**: Важная, но не центральная entity
- **Низкий degree (1-4)**: Периферийная entity

### Node Frequency (частота упоминаний)

**Определение**: Количество text_units, где упоминается entity.

```python
entities["node_frequency"] = entities["text_unit_ids"].apply(len)
```

**Интерпретация**:
- **Высокая frequency (>50)**: Основная тема документов
- **Средняя frequency (10-50)**: Важный концепт
- **Низкая frequency (1-9)**: Второстепенный или редкий

### Комбинированная важность

```python
def compute_entity_importance(entities: pd.DataFrame) -> pd.DataFrame:
    """
    Вычисляет importance score для entities
    """
    # Нормализовать метрики
    max_degree = entities["node_degree"].max()
    max_frequency = entities["node_frequency"].max()

    normalized_degree = entities["node_degree"] / max_degree
    normalized_frequency = entities["node_frequency"] / max_frequency

    # Комбинированный score (можно настроить веса)
    entities["importance"] = (
        0.6 * normalized_degree +
        0.4 * normalized_frequency
    )

    return entities.sort_values("importance", ascending=False)
```

## Использование в Query Execution

### Роль в Local Search (primary)

Entities — отправная точка для Local Search:

```python
def local_search_entity_centric(
    query: str,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    text_units: pd.DataFrame,
    entity_embeddings: pd.DataFrame
) -> str:
    """
    Local Search с фокусом на entities
    """
    # 1. Найти релевантные entities (semantic search)
    relevant_entities = semantic_search(
        query,
        entity_embeddings,
        top_k=20
    )

    # 2. Расширить набор entities через relationships
    expanded_entities = expand_entities_via_relationships(
        relevant_entities,
        relationships,
        entities,
        max_hops=2
    )

    # 3. Собрать text_units для всех entities
    context_text_units = get_text_units_for_entities(
        expanded_entities,
        text_units
    )

    # 4. Ранжировать text_units
    ranked_units = rank_text_units(
        query,
        context_text_units,
        text_unit_embeddings
    )

    # 5. Построить контекст (топ 10 text_units)
    context = build_context(ranked_units[:10])

    # 6. LLM генерирует ответ
    answer = llm_generate(
        query,
        context,
        system_prompt="Answer using entities and their relationships."
    )

    return answer
```

### Entity-based Navigation

```python
def navigate_from_entity(
    start_entity_id: str,
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    max_depth: int = 3
) -> dict:
    """
    Навигация по графу от стартовой entity
    """
    visited = set()
    result = {
        "entities": [],
        "relationships": [],
        "depth_map": {}
    }

    queue = [(start_entity_id, 0)]  # (entity_id, depth)

    while queue:
        current_id, depth = queue.pop(0)

        if current_id in visited or depth > max_depth:
            continue

        visited.add(current_id)

        # Добавить entity
        entity = entities[entities["id"] == current_id].iloc[0]
        result["entities"].append(entity)
        result["depth_map"][current_id] = depth

        # Найти связанные entities
        related_rels = relationships[
            (relationships["source"] == entity["title"]) |
            (relationships["target"] == entity["title"])
        ]

        for _, rel in related_rels.iterrows():
            result["relationships"].append(rel)

            # Добавить связанную entity в очередь
            next_entity_title = (
                rel["target"] if rel["source"] == entity["title"]
                else rel["source"]
            )
            next_entity = entities[entities["title"] == next_entity_title]

            if len(next_entity) > 0:
                next_id = next_entity.iloc[0]["id"]
                if next_id not in visited:
                    queue.append((next_id, depth + 1))

    return result
```

### Hub Entity Detection

```python
def find_hub_entities(
    entities: pd.DataFrame,
    top_k: int = 10
) -> pd.DataFrame:
    """
    Находит hub entities (высокий node_degree)
    """
    hubs = entities.nlargest(top_k, "node_degree")

    # Добавить дополнительные метрики
    hubs["centrality_score"] = compute_betweenness_centrality(hubs, relationships)

    return hubs[["title", "type", "node_degree", "node_frequency", "centrality_score"]]
```

## Entity Embeddings

### Два типа embeddings

```python
# 1. Entity Title Embedding (только название)
entity_title_embeddings = create_embeddings(
    entities["title"].tolist(),
    model="text-embedding-3-small"
)

# 2. Entity Description Embedding (название + описание)
entity_texts = (entities["title"] + " " + entities["description"]).tolist()
entity_description_embeddings = create_embeddings(
    entity_texts,
    model="text-embedding-3-small"
)

# Description embedding более информативен для retrieval
```

### Semantic Entity Search

```python
def search_entities_semantic(
    query: str,
    entities: pd.DataFrame,
    entity_embeddings: pd.DataFrame,
    entity_type: str | None = None,
    top_k: int = 10
) -> pd.DataFrame:
    """
    Семантический поиск entities
    """
    # 1. Фильтр по типу (опционально)
    if entity_type:
        filtered_entities = entities[entities["type"] == entity_type]
        filtered_embeddings = entity_embeddings[
            entity_embeddings["id"].isin(filtered_entities["id"])
        ]
    else:
        filtered_entities = entities
        filtered_embeddings = entity_embeddings

    # 2. Query embedding
    query_emb = create_embedding(query)

    # 3. Cosine similarity
    similarities = filtered_embeddings["embedding"].apply(
        lambda emb: cosine_similarity(query_emb, emb)
    )

    # 4. Top-K
    top_indices = similarities.nlargest(top_k).index
    top_entity_ids = filtered_embeddings.loc[top_indices, "id"]

    # 5. Результаты
    results = filtered_entities[
        filtered_entities["id"].isin(top_entity_ids)
    ].copy()

    results["similarity"] = results["id"].map(
        dict(zip(
            filtered_embeddings.loc[top_indices, "id"],
            similarities.loc[top_indices]
        ))
    )

    return results.sort_values("similarity", ascending=False)
```

## Entity Co-occurrence

### Анализ совместных упоминаний

```python
def compute_entity_cooccurrence(
    entities: pd.DataFrame,
    text_units: pd.DataFrame
) -> pd.DataFrame:
    """
    Вычисляет совместные упоминания entities в text_units
    """
    cooccurrence_matrix = defaultdict(lambda: defaultdict(int))

    # Для каждого text_unit
    for _, unit in text_units.iterrows():
        unit_entities = unit["entity_ids"]

        # Для каждой пары entities в text_unit
        for i, entity1 in enumerate(unit_entities):
            for entity2 in unit_entities[i+1:]:
                # Увеличить счетчик co-occurrence
                key = tuple(sorted([entity1, entity2]))
                cooccurrence_matrix[key[0]][key[1]] += 1

    # Преобразовать в DataFrame
    cooccurrence_data = []
    for entity1, entity2_counts in cooccurrence_matrix.items():
        for entity2, count in entity2_counts.items():
            cooccurrence_data.append({
                "entity1": entity1,
                "entity2": entity2,
                "cooccurrence_count": count
            })

    return pd.DataFrame(cooccurrence_data)
```

### Использование co-occurrence для query expansion

```python
def expand_query_entities(
    query_entities: list[str],
    cooccurrence: pd.DataFrame,
    threshold: int = 5
) -> list[str]:
    """
    Расширяет набор entities через co-occurrence
    """
    expanded = set(query_entities)

    for entity in query_entities:
        # Найти co-occurring entities
        related = cooccurrence[
            ((cooccurrence["entity1"] == entity) |
             (cooccurrence["entity2"] == entity)) &
            (cooccurrence["cooccurrence_count"] >= threshold)
        ]

        for _, row in related.iterrows():
            other_entity = (
                row["entity2"] if row["entity1"] == entity
                else row["entity1"]
            )
            expanded.add(other_entity)

    return list(expanded)
```

## Entity Type Analysis

### Распределение по типам

```python
def analyze_entity_types(entities: pd.DataFrame) -> dict:
    """
    Анализирует распределение entities по типам
    """
    type_stats = {}

    for entity_type in entities["type"].unique():
        type_entities = entities[entities["type"] == entity_type]

        type_stats[entity_type] = {
            "count": len(type_entities),
            "avg_degree": type_entities["node_degree"].mean(),
            "avg_frequency": type_entities["node_frequency"].mean(),
            "top_entities": type_entities.nlargest(5, "node_degree")["title"].tolist()
        }

    return type_stats
```

### Type-specific Search

```python
def search_by_entity_type(
    query: str,
    entity_type: str,
    entities: pd.DataFrame,
    entity_embeddings: pd.DataFrame
) -> pd.DataFrame:
    """
    Поиск entities конкретного типа

    Examples:
        # Найти организации, связанные с AI
        orgs = search_by_entity_type("AI", "organization", entities, embeddings)

        # Найти людей, работающих с GraphRAG
        people = search_by_entity_type("GraphRAG", "person", entities, embeddings)
    """
    return search_entities_semantic(
        query,
        entities,
        entity_embeddings,
        entity_type=entity_type,
        top_k=10
    )
```

## Визуализация entities

### Entity Network Visualization

```python
def visualize_entity_network(
    entities: pd.DataFrame,
    relationships: pd.DataFrame,
    central_entity: str | None = None,
    max_entities: int = 50
) -> None:
    """
    Визуализирует сеть entities

    Args:
        central_entity: Если указано, показать окрестность этой entity
        max_entities: Максимальное количество entities для отображения
    """
    import networkx as nx
    import matplotlib.pyplot as plt

    # Создать граф
    G = nx.Graph()

    if central_entity:
        # Найти окрестность
        neighborhood = navigate_from_entity(
            central_entity,
            entities,
            relationships,
            max_depth=2
        )
        relevant_entities = neighborhood["entities"]
        relevant_relationships = neighborhood["relationships"]
    else:
        # Топ entities по importance
        relevant_entities = compute_entity_importance(entities).head(max_entities)
        relevant_relationships = relationships[
            relationships["source"].isin(relevant_entities["title"]) &
            relationships["target"].isin(relevant_entities["title"])
        ]

    # Добавить узлы
    for entity in relevant_entities:
        G.add_node(
            entity["title"],
            type=entity["type"],
            degree=entity["node_degree"]
        )

    # Добавить ребра
    for _, rel in relevant_relationships.iterrows():
        G.add_edge(rel["source"], rel["target"], weight=rel["weight"])

    # Визуализировать
    pos = nx.spring_layout(G)

    # Размер узлов по degree
    node_sizes = [G.nodes[node]["degree"] * 100 for node in G.nodes()]

    # Цвет по типу
    type_colors = {
        "organization": "blue",
        "person": "red",
        "geo": "green",
        "event": "orange"
    }
    node_colors = [
        type_colors.get(G.nodes[node]["type"], "gray")
        for node in G.nodes()
    ]

    nx.draw(
        G, pos,
        node_size=node_sizes,
        node_color=node_colors,
        with_labels=True,
        font_size=8
    )

    plt.show()
```

## Best Practices

### 1. Нормализация entity names

```python
def normalize_entity_name(name: str) -> str:
    """
    Нормализует название entity для de-duplication
    """
    # Capitalize
    name = name.title()

    # Удалить лишние пробелы
    name = ' '.join(name.split())

    # Удалить специальные символы
    name = re.sub(r'[^\w\s-]', '', name)

    return name
```

### 2. Entity Disambiguation

```python
def disambiguate_entities(
    entities: pd.DataFrame,
    context: str
) -> pd.DataFrame:
    """
    Различает entities с одинаковыми названиями
    """
    # Группировать entities по названию
    grouped = entities.groupby("title")

    disambiguated = []

    for title, group in grouped:
        if len(group) > 1:
            # Есть дубликаты, нужна disambiguation
            # Использовать контекст и тип для различения
            for i, (_, entity) in enumerate(group.iterrows()):
                entity["title"] = f"{title} ({entity['type']})"
                disambiguated.append(entity)
        else:
            disambiguated.extend(group.to_dict('records'))

    return pd.DataFrame(disambiguated)
```

## Следующие разделы

- **[← Text Unit Nodes](02-text-unit-nodes.md)**
- **[Relationship Edges →](04-relationship-edges.md)**
- **[Query Execution →](09-query-execution.md)**
