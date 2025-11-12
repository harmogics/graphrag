# Обнаружение сообществ (Community Detection)

## Обзор этапа

**Community Detection** — это процесс выявления тематических кластеров (communities) в графе знаний. Сообщества представляют собой группы тесно связанных сущностей, которые образуют семантически связные тематические области.

GraphRAG использует **иерархический алгоритм Leiden clustering** для создания многоуровневой структуры сообществ от детальных до абстрактных тем.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/create_communities.py`
- **Clustering**: `graphrag/index/operations/cluster_graph.py`
- **Community Creation**: `graphrag/index/operations/create_communities.py`
- **Конфигурация**: `graphrag/config/models/cluster_graph_config.py`

## Процесс обнаружения сообществ

```
INPUT: Граф G (entities + relationships)
    ↓
┌─────────────────────────────────────────┐
│  1. Извлечение LCC (опционально)        │
│     Largest Connected Component         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  2. Leiden Clustering (иерархический)   │
│     - Level 0: детальные сообщества     │
│     - Level 1: средние сообщества       │
│     - Level N: абстрактные темы         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  3. Построение иерархии parent-child    │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  4. Агрегация entity_ids по сообществам │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  5. Агрегация relationship_ids          │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  6. Агрегация text_unit_ids             │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  7. Вычисление размера сообществ        │
└─────────────────────────────────────────┘
    ↓
OUTPUT: communities.parquet (иерархическая структура)
```

## Алгоритм Leiden Clustering

### Что такое Leiden Algorithm?

**Leiden algorithm** — это современный алгоритм обнаружения сообществ в графах, который:
- Максимизирует **modularity** (плотность связей внутри сообществ vs между)
- Создает **иерархическую структуру** сообществ
- Быстрее и точнее классического Louvain algorithm

### Принцип работы

```
1. Local Move Phase:
   - Каждый узел пытается перейти в соседнее сообщество
   - Переход происходит, если увеличивает modularity

2. Refinement Phase (отличие от Louvain):
   - Разбиение сообществ на подмножества
   - Проверка, что все узлы действительно хорошо соединены

3. Aggregation Phase:
   - Создание нового графа, где сообщества → узлы
   - Повторение процесса для следующего уровня

4. Итерация до сходимости
```

### Реализация в GraphRAG

```python
def cluster_graph(
    graph: nx.Graph,
    max_cluster_size: int = 10,
    use_lcc: bool = True,
    seed: int | None = None
) -> Communities:
    """
    Применяет иерархический Leiden clustering

    Args:
        graph: NetworkX граф
        max_cluster_size: Максимальный размер одного кластера
        use_lcc: Использовать только Largest Connected Component
        seed: Random seed для воспроизводимости

    Returns:
        Communities: Иерархическая структура сообществ
    """
    # 1. Опционально: извлечь LCC
    if use_lcc:
        graph = get_largest_connected_component(graph)

    # 2. Преобразовать в igraph (Leiden реализован в igraph)
    import igraph as ig
    g = ig.Graph.from_networkx(graph)

    # 3. Применить Leiden clustering
    communities_result = g.community_leiden(
        objective_function="modularity",
        weights="weight",
        n_iterations=2,  # Количество итераций уточнения
        seed=seed
    )

    # 4. Построить иерархию
    hierarchy = build_community_hierarchy(
        graph,
        communities_result,
        max_cluster_size=max_cluster_size
    )

    return hierarchy
```

**Файл**: `graphrag/index/operations/cluster_graph.py:35`

## Конфигурация clustering

### ClusterGraphConfig параметры

```python
class ClusterGraphConfig:
    # Максимальный размер одного сообщества
    max_cluster_size: int = 10

    # Использовать только LCC
    use_lcc: bool = True

    # Random seed для воспроизводимости
    seed: int | None = None

    # Стратегия clustering (опционально)
    strategy: dict | None = None
```

**Файл**: `graphrag/config/models/cluster_graph_config.py:15`

### Влияние max_cluster_size

```python
# max_cluster_size = 5: много маленьких, детальных сообществ
# → Высокая гранулярность, больше уровней иерархии

# max_cluster_size = 20: меньше, но более крупных сообществ
# → Низкая гранулярность, меньше уровней

# Рекомендация: 10 (баланс между детальностью и обзором)
```

## Иерархическая структура сообществ

### Понятие уровней (Levels)

GraphRAG создает **многоуровневую иерархию** сообществ:

```
Level 0 (Bottom): Детальные микротемы
    Community 0: [Entity1, Entity2, Entity3]
    Community 1: [Entity4, Entity5]
    Community 2: [Entity6, Entity7, Entity8, Entity9]
    ...

Level 1 (Middle): Средние темы
    Community 10: [Community 0, Community 1]  → объединяет 0 и 1
    Community 11: [Community 2]
    ...

Level 2 (Top): Абстрактные макротемы
    Community 20: [Community 10, Community 11]  → объединяет все
```

### Отношения Parent-Child

```python
class Community:
    id: str                     # Уникальный ID сообщества
    level: int                  # Уровень в иерархии (0, 1, 2, ...)
    entity_ids: list[str]       # Сущности в этом сообществе
    parent: str | None          # ID родительского сообщества
    children: list[str]         # IDs дочерних сообществ

# Пример:
Community(
    id="community_10",
    level=1,
    entity_ids=["entity_1", "entity_2", "entity_4", "entity_5"],
    parent="community_20",
    children=["community_0", "community_1"]
)
```

### Построение иерархии

```python
def build_community_hierarchy(
    graph: nx.Graph,
    leiden_result: list[list[str]],  # Сообщества уровня 0
    max_cluster_size: int
) -> list[Community]:
    """
    Строит многоуровневую иерархию из результатов Leiden

    Process:
    1. Level 0: Базовые сообщества из Leiden
    2. Для каждого уровня:
       - Если есть сообщества > max_cluster_size:
         → Разбить на подсообщества
       - Построить граф сообществ (сообщества = узлы)
       - Применить Leiden снова
       - Создать parent-child связи
    3. Повторять до тех пор, пока не останется 1 сообщество на top level
    """
    all_communities = []
    current_level = 0
    current_communities = leiden_result

    while len(current_communities) > 1:
        # Создать Community объекты для текущего уровня
        level_communities = []
        for i, entity_ids in enumerate(current_communities):
            community = Community(
                id=f"community_{len(all_communities) + i}",
                level=current_level,
                entity_ids=entity_ids,
                parent=None,  # Заполнится на следующем уровне
                children=[]
            )
            level_communities.append(community)

        all_communities.extend(level_communities)

        # Построить граф следующего уровня
        next_graph = build_community_graph(graph, level_communities)

        # Применить Leiden к графу сообществ
        next_leiden = leiden_clustering(next_graph)

        # Установить parent-child связи
        for parent_idx, child_communities in enumerate(next_leiden):
            parent_id = f"community_{len(all_communities) + parent_idx}"
            for child_community in child_communities:
                child_community.parent = parent_id

        current_communities = next_leiden
        current_level += 1

    return all_communities
```

**Файл**: `graphrag/index/operations/cluster_graph.py:120`

## Агрегация данных в сообществах

### Агрегация entity_ids

Для каждого сообщества собираются все сущности, принадлежащие этому кластеру:

```python
def aggregate_entities_in_communities(
    communities: list[Community],
    entities: pd.DataFrame
) -> pd.DataFrame:
    """
    Для каждого сообщества находит все entity_ids
    """
    community_data = []

    for community in communities:
        # Level 0: entity_ids напрямую из clustering
        if community.level == 0:
            entity_ids = community.entity_ids

        # Higher levels: собрать entity_ids из всех дочерних сообществ
        else:
            entity_ids = []
            for child_id in community.children:
                child_entities = communities_df.loc[
                    communities_df["id"] == child_id,
                    "entity_ids"
                ].values[0]
                entity_ids.extend(child_entities)

        community_data.append({
            "id": community.id,
            "level": community.level,
            "entity_ids": entity_ids,
            "parent": community.parent,
            "children": community.children
        })

    return pd.DataFrame(community_data)
```

**Файл**: `graphrag/index/operations/create_communities.py:45`

### Агрегация relationship_ids

Отношения включаются в сообщество, если **обе** сущности (source и target) принадлежат этому сообществу:

```python
def aggregate_relationships_in_communities(
    communities: pd.DataFrame,
    relationships: pd.DataFrame
) -> pd.DataFrame:
    """
    Находит relationship_ids для каждого сообщества
    """
    for idx, community in communities.iterrows():
        entity_ids_set = set(community["entity_ids"])

        # Найти отношения, где обе стороны в сообществе
        community_relationships = relationships[
            relationships["source"].isin(entity_ids_set) &
            relationships["target"].isin(entity_ids_set)
        ]

        relationship_ids = community_relationships["id"].tolist()
        communities.at[idx, "relationship_ids"] = relationship_ids

    return communities
```

### Агрегация text_unit_ids

Text units агрегируются через сущности:

```python
def aggregate_text_units_in_communities(
    communities: pd.DataFrame,
    entities: pd.DataFrame
) -> pd.DataFrame:
    """
    Находит text_unit_ids для каждого сообщества через entities
    """
    for idx, community in communities.iterrows():
        # Найти все entities в сообществе
        community_entities = entities[
            entities["id"].isin(community["entity_ids"])
        ]

        # Собрать все text_unit_ids из этих entities
        text_unit_ids = set()
        for entity_text_units in community_entities["text_unit_ids"]:
            text_unit_ids.update(entity_text_units)

        communities.at[idx, "text_unit_ids"] = list(text_unit_ids)

    return communities
```

## Вычисление размера сообщества

### Size метрика

**Размер сообщества** — это количество сущностей в нём:

```python
communities["size"] = communities["entity_ids"].apply(len)
```

### Интерпретация размера

- **Маленькие сообщества (1-5 entities)**: Очень специфичные микротемы
- **Средние сообщества (6-15 entities)**: Хорошо определенные темы
- **Большие сообщества (16+ entities)**: Широкие или абстрактные темы

### Размер по уровням

```
Level 0: avg_size = 8   (детальные темы)
Level 1: avg_size = 25  (средние темы)
Level 2: avg_size = 100 (макротемы)
```

## Период (Period) в сообществах

### Концепция периода

**Period** — это временной или тематический диапазон, к которому относится сообщество.

```python
class Community:
    period: str | None  # Например: "2024-Q1", "historical", "technical"
```

### Использование периодов

- **Временная сегментация**: Разделение сообществ по временным периодам
- **Тематическая сегментация**: Разделение по доменам (техника, история, политика)
- **Опционально**: Не обязательно для базового использования

## Финальная схема communities.parquet

```python
COMMUNITIES_FINAL_COLUMNS = [
    "id",                   # str: уникальный ID сообщества
    "human_readable_id",    # str: "community_0001", "community_0002", ...
    "community",            # int: номер сообщества внутри уровня
    "level",                # int: уровень в иерархии (0, 1, 2, ...)
    "parent",               # str | None: ID родительского сообщества
    "children",             # list[str]: IDs дочерних сообществ
    "entity_ids",           # list[str]: сущности в сообществе
    "relationship_ids",     # list[str]: отношения внутри сообщества
    "text_unit_ids",        # list[str]: text_units, связанные с сообществом
    "period",               # str | None: временной/тематический период
    "size",                 # int: количество сущностей
]
```

**Файл схемы**: `graphrag/data_model/schemas.py:85`

## Примеры сообществ

### Пример 1: Техническая тема (Level 0)

```python
Community(
    id="community_0012",
    human_readable_id="community_0012",
    level=0,
    parent="community_0105",
    children=[],
    entity_ids=[
        "entity_0034",  # "GraphRAG"
        "entity_0045",  # "Knowledge Graph"
        "entity_0067",  # "Entity Extraction"
        "entity_0089",  # "LLM"
        "entity_0123"   # "RAG"
    ],
    relationship_ids=[
        "rel_0234", "rel_0245", "rel_0267"
    ],
    text_unit_ids=[
        "text_unit_001", "text_unit_015", "text_unit_032"
    ],
    size=5
)
```

### Пример 2: Широкая тема (Level 1)

```python
Community(
    id="community_0105",
    human_readable_id="community_0105",
    level=1,
    parent="community_0201",
    children=["community_0012", "community_0013", "community_0014"],
    entity_ids=[
        # Все entities из дочерних сообществ
        "entity_0034", "entity_0045", ..., "entity_0234"
    ],
    size=42
)
```

## Анализ сообществ

### Статистика по уровням

```python
def analyze_community_levels(communities: pd.DataFrame):
    """
    Анализирует распределение сообществ по уровням
    """
    stats_by_level = communities.groupby("level").agg({
        "id": "count",
        "size": ["mean", "min", "max"],
        "entity_ids": lambda x: sum(len(ids) for ids in x)
    })

    return stats_by_level

# Пример вывода:
#        count  size_mean  size_min  size_max  total_entities
# level
# 0        145       7.8         2        15            1131
# 1         28      40.4        18        82            1131
# 2          5     226.2       150       350            1131
```

### Поиск самых крупных сообществ

```python
def find_largest_communities(
    communities: pd.DataFrame,
    level: int,
    top_k: int = 5
) -> pd.DataFrame:
    """
    Находит самые крупные сообщества на заданном уровне
    """
    level_communities = communities[communities["level"] == level]
    largest = level_communities.nlargest(top_k, "size")
    return largest
```

### Визуализация иерархии

```python
def visualize_community_hierarchy(communities: pd.DataFrame):
    """
    Создает древовидную визуализацию иерархии сообществ
    """
    import networkx as nx
    import matplotlib.pyplot as plt

    # Построить дерево
    tree = nx.DiGraph()

    for _, community in communities.iterrows():
        tree.add_node(
            community["id"],
            level=community["level"],
            size=community["size"]
        )
        if community["parent"]:
            tree.add_edge(community["parent"], community["id"])

    # Нарисовать
    pos = nx.spring_layout(tree)
    nx.draw(tree, pos, with_labels=True, node_size=300)
    plt.show()
```

## Качество кластеризации

### Метрики качества

```python
def evaluate_clustering_quality(
    graph: nx.Graph,
    communities: pd.DataFrame
) -> dict:
    """
    Вычисляет метрики качества кластеризации
    """
    # Modularity: [0, 1], higher = better
    modularity = compute_modularity(graph, communities)

    # Coverage: доля ребер внутри сообществ
    coverage = compute_coverage(graph, communities)

    # Performance: комбинированная метрика
    performance = compute_performance(graph, communities)

    return {
        "modularity": modularity,
        "coverage": coverage,
        "performance": performance
    }

# Пример:
# {
#     "modularity": 0.78,    # Очень хорошо (> 0.7)
#     "coverage": 0.92,      # Отлично (> 0.9)
#     "performance": 0.85    # Хорошо
# }
```

### Оптимальное количество сообществ

```python
# Нет единого ответа, зависит от:
# - Размера графа
# - Плотности связей
# - Желаемой гранулярности

# Общее правило:
# num_communities ≈ sqrt(num_nodes)

# Для графа с 2000 узлами:
# Оптимально: ~45 сообществ на level 0
```

## Следующий этап

После создания сообществ:
- **create_community_reports**: Генерация человекочитаемых отчетов о каждом сообществе с использованием LLM

→ [Переход к генерации отчетов о сообществах](05-community-reports.md)
