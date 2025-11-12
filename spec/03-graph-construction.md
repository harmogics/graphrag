# Построение графа знаний (Graph Construction)

## Обзор этапа

После извлечения сущностей и отношений GraphRAG строит **граф знаний** (Knowledge Graph) — структурированное представление данных, где:
- **Узлы (nodes)** = Сущности (entities)
- **Ребра (edges)** = Отношения (relationships)

Этот этап включает создание графовой структуры, вычисление метрик узлов и ребер, и подготовку данных для кластеризации.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/finalize_graph.py`
- **Graph Creation**: `graphrag/index/operations/create_graph.py`
- **Layout Computation**: `graphrag/index/operations/layout_graph.py`
- **Degree Computation**: `graphrag/index/operations/compute_degree.py`
- **Entity Finalization**: `graphrag/index/operations/finalize_entities.py`
- **Relationship Finalization**: `graphrag/index/operations/finalize_relationships.py`

## Процесс построения графа

```
INPUT: entities.parquet + relationships.parquet
    ↓
┌─────────────────────────────────────────┐
│  1. Создание NetworkX графа             │
│     - Узлы из entities                  │
│     - Ребра из relationships            │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  2. Вычисление степеней узлов           │
│     - node_degree для каждой entity     │
│     - combined_degree для relationships │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  3. Вычисление частоты упоминаний       │
│     - node_frequency (по text_units)    │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  4. Layout: позиционирование узлов      │
│     - Вычисление (x, y) координат       │
│     - Использование spring layout       │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  5. Обрезка слабых связей (опционально) │
│     - Удаление low-weight edges         │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  6. Финализация entities и relationships│
│     - Добавление вычисленных метрик     │
└─────────────────────────────────────────┘
    ↓
OUTPUT: Финализированный граф G + обновленные parquet
```

## Создание NetworkX графа

### Структура графа

GraphRAG использует библиотеку **NetworkX** для представления графа знаний:

```python
import networkx as nx

def create_graph(
    entities: pd.DataFrame,
    relationships: pd.DataFrame
) -> nx.Graph:
    """
    Создает неориентированный граф из entities и relationships

    Args:
        entities: DataFrame с колонками [id, title, type, description, ...]
        relationships: DataFrame с колонками [source, target, weight, ...]

    Returns:
        NetworkX Graph объект
    """
    G = nx.Graph()

    # Добавить узлы (entities)
    for _, entity in entities.iterrows():
        G.add_node(
            entity["title"],  # Имя узла = entity title
            **{
                "id": entity["id"],
                "type": entity["type"],
                "description": entity["description"],
                "text_unit_ids": entity["text_unit_ids"]
            }
        )

    # Добавить ребра (relationships)
    for _, rel in relationships.iterrows():
        # Проверить, что обе сущности существуют
        if rel["source"] in G and rel["target"] in G:
            G.add_edge(
                rel["source"],
                rel["target"],
                weight=rel["weight"],
                description=rel["description"],
                id=rel["id"]
            )

    return G
```

**Файл**: `graphrag/index/operations/create_graph.py:23`

### Веса ребер

Веса ребер вычисляются из strength (1-10), указанного LLM:

```python
def compute_edge_weight(strength: int) -> float:
    """
    Преобразует strength (1-10) в weight для графа

    Higher strength → Lower weight (для алгоритмов shortest path)
    """
    return 1.0 / max(strength, 1)

# Пример:
# strength = 10 → weight = 0.1  (очень сильная связь)
# strength = 5  → weight = 0.2  (средняя связь)
# strength = 1  → weight = 1.0  (слабая связь)
```

**Почему инверсия?**
- В алгоритмах на графах (Dijkstra, clustering) меньший вес = более близкая связь
- LLM указывает strength (больше = сильнее), но для графа нужен weight (меньше = сильнее)

## Вычисление степеней узлов (Node Degrees)

### Node Degree

**Степень узла (node degree)** — это количество ребер, инцидентных узлу:

```python
def compute_node_degrees(graph: nx.Graph) -> dict[str, int]:
    """
    Вычисляет degree для каждого узла в графе

    Returns:
        {node_name: degree}
    """
    return dict(graph.degree())

# Пример:
# Entity "GraphRAG":
#   - связана с 15 другими сущностями
#   → node_degree = 15
```

**Файл**: `graphrag/index/operations/compute_degree.py:15`

### Интерпретация степени

- **Высокий degree (20+)**: Центральная сущность, "хаб" в графе знаний
  - Пример: "Microsoft", "AI", "Machine Learning"

- **Средний degree (5-20)**: Важные, но не центральные концепты
  - Пример: "Graph Neural Networks", "Entity Extraction"

- **Низкий degree (1-4)**: Периферийные или специфичные сущности
  - Пример: Конкретные даты, малоизвестные персоны

### Combined Degree для ребер

Для каждого relationship вычисляется **combined_degree** — сумма степеней обоих узлов:

```python
def compute_edge_combined_degree(
    relationships: pd.DataFrame,
    node_degrees: dict[str, int]
) -> pd.DataFrame:
    """
    Добавляет combined_degree к relationships
    """
    relationships["source_degree"] = relationships["source"].map(node_degrees)
    relationships["target_degree"] = relationships["target"].map(node_degrees)
    relationships["combined_degree"] = (
        relationships["source_degree"] + relationships["target_degree"]
    )

    return relationships
```

**Использование**: Ребра между высокостепенными узлами — наиболее важные связи.

**Файл**: `graphrag/index/operations/compute_edge_combined_degree.py:20`

## Вычисление частоты упоминаний (Node Frequency)

### Node Frequency

**Частота узла (node_frequency)** — количество text_units, где упоминается сущность:

```python
def compute_node_frequency(entities: pd.DataFrame) -> pd.DataFrame:
    """
    Вычисляет частоту упоминаний для каждой сущности
    """
    entities["node_frequency"] = entities["text_unit_ids"].apply(len)
    return entities

# Пример:
# Entity "GraphRAG"
#   - упомянута в 42 text_units
#   → node_frequency = 42
```

### Интерпретация частоты

- **Высокая частота (50+)**: Основная тема документов
- **Средняя частота (10-50)**: Важный, часто упоминаемый концепт
- **Низкая частота (1-9)**: Второстепенный или редкий концепт

### Отличие от degree

- **node_frequency**: Сколько раз упоминается в тексте
- **node_degree**: Сколько других сущностей связано с ней

Высокая частота + низкий degree = "много говорят, но мало связей"
Низкая частота + высокий degree = "редко упоминается, но связана с многими"

## Layout: позиционирование узлов

### Цель layout

**Graph layout** — это алгоритм вычисления (x, y) координат для каждого узла графа для визуализации.

### Force-directed layout (Spring layout)

GraphRAG использует **spring layout** (force-directed) из NetworkX:

```python
def layout_graph(
    graph: nx.Graph,
    layout_config: dict | None = None
) -> dict[str, tuple[float, float]]:
    """
    Вычисляет координаты узлов для визуализации

    Args:
        graph: NetworkX граф
        layout_config: Параметры layout (опционально)

    Returns:
        {node_name: (x, y)}
    """
    # Spring layout: узлы отталкиваются, связанные притягиваются
    positions = nx.spring_layout(
        graph,
        k=1.0,           # Оптимальное расстояние между узлами
        iterations=50,   # Количество итераций оптимизации
        seed=42          # Для воспроизводимости
    )

    return positions
```

**Файл**: `graphrag/index/operations/layout_graph.py:18`

### Альтернативные layout алгоритмы

```python
# Kamada-Kawai layout (более точный, медленнее)
positions = nx.kamada_kawai_layout(graph)

# Spectral layout (быстрый, использует собственные векторы)
positions = nx.spectral_layout(graph)

# Circular layout (узлы в кругу)
positions = nx.circular_layout(graph)
```

### Применение координат

```python
def apply_layout_to_entities(
    entities: pd.DataFrame,
    layout: dict[str, tuple[float, float]]
) -> pd.DataFrame:
    """
    Добавляет (x, y) координаты к entities
    """
    entities["node_x"] = entities["title"].map(lambda t: layout.get(t, (0, 0))[0])
    entities["node_y"] = entities["title"].map(lambda t: layout.get(t, (0, 0))[1])

    return entities
```

## Обрезка графа (Graph Pruning)

### Цель pruning

В **Fast Mode** GraphRAG обрезает граф, удаляя слабые связи для ускорения обработки.

### Алгоритм pruning

```python
def prune_graph(
    graph: nx.Graph,
    min_weight_threshold: float = 0.5,
    min_degree: int = 1
) -> nx.Graph:
    """
    Удаляет слабые ребра и изолированные узлы

    Args:
        graph: Исходный граф
        min_weight_threshold: Минимальный weight ребра (ребра с weight > threshold удаляются)
        min_degree: Минимальная степень узла (узлы с меньшей степенью удаляются)

    Returns:
        Обрезанный граф
    """
    G = graph.copy()

    # 1. Удалить слабые ребра
    edges_to_remove = [
        (u, v)
        for u, v, data in G.edges(data=True)
        if data.get("weight", 1.0) > min_weight_threshold
    ]
    G.remove_edges_from(edges_to_remove)

    # 2. Удалить изолированные узлы
    nodes_to_remove = [
        node
        for node, degree in dict(G.degree()).items()
        if degree < min_degree
    ]
    G.remove_nodes_from(nodes_to_remove)

    return G
```

**Файл**: `graphrag/index/operations/prune_graph.py:25`

### Когда использовать pruning?

- **Standard Mode**: Не используется (сохраняет все связи)
- **Fast Mode**: Pruning включен для ускорения clustering

**Компромисс**:
- ✅ Быстрее кластеризация
- ❌ Потеря слабых, но потенциально важных связей

## Использование Largest Connected Component (LCC)

### Что такое LCC?

**Largest Connected Component (LCC)** — это самый большой связный подграф в графе.

```
Исходный граф:
   [A]---[B]       [E]---[F]
    |     |         |
   [C]---[D]       [G]

LCC:
   [A]---[B]
    |     |
   [C]---[D]
```

### Зачем нужен LCC?

Многие алгоритмы (Leiden clustering, Node2Vec) работают только на связных графах. LCC позволяет:
1. Избежать ошибок в алгоритмах
2. Сосредоточиться на основной структуре
3. Игнорировать изолированные компоненты

### Извлечение LCC

```python
def get_largest_connected_component(graph: nx.Graph) -> nx.Graph:
    """
    Извлекает largest connected component из графа
    """
    if len(graph) == 0:
        return graph

    # Найти все связные компоненты
    components = list(nx.connected_components(graph))

    # Выбрать самый большой
    largest = max(components, key=len)

    # Создать subgraph
    return graph.subgraph(largest).copy()
```

**Файл**: `graphrag/index/operations/create_graph.py:75`

## Финализация entities и relationships

### Финализация entities

После вычисления всех метрик entities обновляются:

```python
def finalize_entities(
    entities: pd.DataFrame,
    graph: nx.Graph,
    layout: dict[str, tuple[float, float]]
) -> pd.DataFrame:
    """
    Добавляет финальные метрики к entities
    """
    # Вычислить degree
    node_degrees = dict(graph.degree())
    entities["node_degree"] = entities["title"].map(node_degrees).fillna(0)

    # Вычислить frequency
    entities["node_frequency"] = entities["text_unit_ids"].apply(len)

    # Добавить layout координаты
    entities["node_x"] = entities["title"].map(lambda t: layout.get(t, (0.0, 0.0))[0])
    entities["node_y"] = entities["title"].map(lambda t: layout.get(t, (0.0, 0.0))[1])

    # Добавить human_readable_id
    entities["human_readable_id"] = [f"entity_{i:04d}" for i in range(len(entities))]

    return entities
```

**Файл**: `graphrag/index/operations/finalize_entities.py:30`

### Финализация relationships

```python
def finalize_relationships(
    relationships: pd.DataFrame,
    node_degrees: dict[str, int]
) -> pd.DataFrame:
    """
    Добавляет финальные метрики к relationships
    """
    # Добавить degrees
    relationships["source_degree"] = relationships["source"].map(node_degrees).fillna(0)
    relationships["target_degree"] = relationships["target"].map(node_degrees).fillna(0)
    relationships["combined_degree"] = (
        relationships["source_degree"] + relationships["target_degree"]
    )

    # Добавить human_readable_id
    relationships["human_readable_id"] = [
        f"rel_{i:04d}" for i in range(len(relationships))
    ]

    return relationships
```

**Файл**: `graphrag/index/operations/finalize_relationships.py:25`

## Финальные схемы данных

### Финальная схема entities.parquet

```python
ENTITIES_FINAL_COLUMNS = [
    "id",                  # str: SHA512 hash
    "human_readable_id",   # str: "entity_0001", "entity_0002", ...
    "title",               # str: название сущности (capitalized)
    "type",                # str: organization, person, geo, event
    "description",         # str: суммаризированное описание
    "text_unit_ids",       # list[str]: IDs text_units, где упомянута
    "node_frequency",      # int: количество упоминаний в text_units
    "node_degree",         # int: степень узла в графе
    "node_x",              # float: X-координата для визуализации
    "node_y",              # float: Y-координата для визуализации
]
```

### Финальная схема relationships.parquet

```python
RELATIONSHIPS_FINAL_COLUMNS = [
    "id",                  # str: SHA512 hash
    "human_readable_id",   # str: "rel_0001", "rel_0002", ...
    "source",              # str: entity title источника
    "target",              # str: entity title цели
    "description",         # str: описание отношения
    "weight",              # float: 1.0 / strength
    "combined_degree",     # int: source_degree + target_degree
    "text_unit_ids",       # list[str]: IDs text_units
    "source_degree",       # int: степень source entity
    "target_degree",       # int: степень target entity
]
```

**Файл схем**: `graphrag/data_model/schemas.py:45`

## Анализ графа знаний

### Статистика графа

```python
def compute_graph_statistics(graph: nx.Graph) -> dict:
    """
    Вычисляет основные статистики графа
    """
    return {
        "num_nodes": graph.number_of_nodes(),
        "num_edges": graph.number_of_edges(),
        "density": nx.density(graph),
        "avg_degree": sum(dict(graph.degree()).values()) / graph.number_of_nodes(),
        "avg_clustering": nx.average_clustering(graph),
        "num_connected_components": nx.number_connected_components(graph),
        "diameter": nx.diameter(graph) if nx.is_connected(graph) else None
    }

# Пример вывода:
{
    "num_nodes": 2340,
    "num_edges": 8750,
    "density": 0.0032,
    "avg_degree": 7.48,
    "avg_clustering": 0.23,
    "num_connected_components": 12,
    "diameter": 18
}
```

### Центральные узлы (Hub Detection)

```python
def find_hub_nodes(graph: nx.Graph, top_k: int = 10) -> list[str]:
    """
    Находит наиболее центральные узлы по различным метрикам
    """
    # Degree centrality
    degree_centrality = nx.degree_centrality(graph)

    # Betweenness centrality (важность для путей)
    betweenness_centrality = nx.betweenness_centrality(graph)

    # PageRank
    pagerank = nx.pagerank(graph)

    # Топ узлы по degree
    top_by_degree = sorted(
        degree_centrality.items(),
        key=lambda x: x[1],
        reverse=True
    )[:top_k]

    return [node for node, _ in top_by_degree]

# Пример:
# Top hubs: ["AI", "Machine Learning", "GraphRAG", "Microsoft", "LLM"]
```

### Обнаружение мостов (Bridge Detection)

```python
def find_bridge_edges(graph: nx.Graph) -> list[tuple[str, str]]:
    """
    Находит ребра-мосты (удаление которых разделяет граф)
    """
    return list(nx.bridges(graph))

# Пример:
# Bridges: [("GraphRAG", "RAG"), ("AI", "NLP")]
# → Критические связи между кластерами
```

## Визуализация графа

### Экспорт для визуализации

```python
def export_graph_for_viz(
    graph: nx.Graph,
    entities: pd.DataFrame,
    output_path: str
):
    """
    Экспортирует граф в формат для визуализации (JSON, GEXF, GraphML)
    """
    # Добавить атрибуты узлов из entities
    for _, entity in entities.iterrows():
        node = entity["title"]
        if node in graph:
            graph.nodes[node].update({
                "type": entity["type"],
                "description": entity["description"],
                "frequency": entity["node_frequency"],
                "x": entity["node_x"],
                "y": entity["node_y"]
            })

    # Экспорт в GEXF (для Gephi)
    nx.write_gexf(graph, f"{output_path}/graph.gexf")

    # Экспорт в GraphML (для yEd, Cytoscape)
    nx.write_graphml(graph, f"{output_path}/graph.graphml")

    # Экспорт в JSON (для D3.js, vis.js)
    from networkx.readwrite import json_graph
    data = json_graph.node_link_data(graph)
    with open(f"{output_path}/graph.json", "w") as f:
        json.dump(data, f)
```

## Примеры использования

### Поиск путей между сущностями

```python
# Найти кратчайший путь между двумя сущностями
path = nx.shortest_path(graph, source="GraphRAG", target="Knowledge Graph")
print(path)
# ["GraphRAG", "RAG", "Retrieval", "Knowledge Base", "Knowledge Graph"]
```

### Фильтрация по типу сущностей

```python
# Создать подграф только с person и organization
person_org_nodes = [
    node for node, data in graph.nodes(data=True)
    if data.get("type") in ["person", "organization"]
]
subgraph = graph.subgraph(person_org_nodes)
```

### Анализ окрестности узла

```python
# Найти всех соседей узла (1-hop neighborhood)
neighbors = list(graph.neighbors("GraphRAG"))

# Найти 2-hop neighborhood
two_hop = set()
for neighbor in neighbors:
    two_hop.update(graph.neighbors(neighbor))
```

## Следующий этап

После построения и финализации графа:
- **create_communities**: Иерархическая кластеризация графа (Leiden algorithm)
- **create_community_reports**: Генерация описаний найденных сообществ

→ [Переход к обнаружению сообществ](04-community-detection.md)
