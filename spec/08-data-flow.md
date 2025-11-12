# Поток данных через GraphRAG Pipeline

## Обзор потока данных

Этот документ описывает полный поток данных через GraphRAG pipeline от входных документов до финальных выходных структур, пригодных для query execution.

## Схема полного потока

```
┌────────────────────────────────────────────────────────────────┐
│                      INPUT DATA                                │
│                                                                │
│  documents/                                                    │
│  ├── doc1.txt                                                  │
│  ├── doc2.pdf                                                  │
│  └── doc3.md                                                   │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 1: Document Loading & Chunking                          │
│                                                                │
│ Workflows:                                                     │
│ • create_base_documents    → documents.parquet                 │
│ • create_base_text_units   → text_units.parquet (базовые)     │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • documents.parquet (id, text, title, metadata)               │
│ • text_units.parquet (id, text, n_tokens, document_ids)       │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 2: Entity & Relationship Extraction (LLM)               │
│                                                                │
│ Workflow:                                                      │
│ • extract_graph            → entities + relationships          │
│                                                                │
│ LLM Agent: Extraction Agent (GPT-4)                           │
│ • Анализирует каждый text_unit                                │
│ • Извлекает entities (name, type, description)                │
│ • Извлекает relationships (source, target, description)       │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • entities.parquet (id, title, type, description, ...)        │
│ • relationships.parquet (id, source, target, description, ...) │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 3: Graph Construction & Finalization                    │
│                                                                │
│ Workflows:                                                     │
│ • finalize_graph           → NetworkX Graph + метрики          │
│ • finalize_entities        → добавить node_degree, frequency  │
│ • finalize_relationships   → добавить combined_degree         │
│                                                                │
│ Процессы:                                                      │
│ • Создание NetworkX графа из entities/relationships           │
│ • Вычисление степеней узлов (node_degree)                     │
│ • Вычисление частоты упоминаний (node_frequency)              │
│ • Layout: позиционирование узлов (node_x, node_y)             │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • entities.parquet (обновленные с метриками)                  │
│ • relationships.parquet (обновленные)                         │
│ • graph (NetworkX Graph в памяти)                             │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 4: Community Detection                                  │
│                                                                │
│ Workflow:                                                      │
│ • create_communities       → communities (иерархические)       │
│                                                                │
│ Алгоритм: Leiden Clustering                                   │
│ • Иерархическая кластеризация графа                           │
│ • Создание multi-level структуры (level 0, 1, 2, ...)        │
│ • Агрегация entity_ids, relationship_ids, text_unit_ids       │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • communities.parquet                                          │
│   (id, level, parent, children, entity_ids, ...)              │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 5: Text Units Finalization                              │
│                                                                │
│ Workflow:                                                      │
│ • create_final_text_units  → text_units (с entity/rel IDs)    │
│                                                                │
│ Процесс:                                                       │
│ • Добавить entity_ids к text_units                            │
│ • Добавить relationship_ids к text_units                      │
│ • Добавить covariate_ids (если claims enabled)                │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • text_units.parquet (финальная версия)                       │
│   (id, text, entity_ids, relationship_ids, ...)               │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 6: Community Reports Generation (LLM)                   │
│                                                                │
│ Workflow:                                                      │
│ • create_community_reports → community_reports                 │
│                                                                │
│ LLM Agent: Community Report Agent (GPT-4)                     │
│ • Для каждого сообщества:                                     │
│   - Построить контекст (entities + relationships)             │
│   - Генерировать структурированный отчет                      │
│   - Title, Summary, Findings, Rating                          │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • community_reports.parquet                                    │
│   (id, title, summary, findings, full_content, ...)           │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 7: Embeddings Generation                                │
│                                                                │
│ Workflow:                                                      │
│ • generate_text_embeddings → embeddings/*.parquet              │
│                                                                │
│ Embedding Model: text-embedding-3-small                       │
│ • Для каждого текстового поля:                                │
│   - documents.text                                             │
│   - text_units.text                                            │
│   - entities.title + description                              │
│   - relationships.description                                 │
│   - community_reports.full_content                            │
│ • Создание векторных представлений (1536 dimensions)          │
├────────────────────────────────────────────────────────────────┤
│ Data:                                                          │
│ • embeddings/document_text_embedding.parquet                   │
│ • embeddings/text_unit_text_embedding.parquet                  │
│ • embeddings/entity_description_embedding.parquet              │
│ • embeddings/community_full_content_embedding.parquet          │
│ • ... (и другие)                                               │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│ STAGE 8: Vector Store Upload (опционально)                    │
│                                                                │
│ Процесс:                                                       │
│ • Загрузка embeddings в vector store (LanceDB, Azure, etc.)   │
│ • Создание индексов для быстрого поиска                       │
├────────────────────────────────────────────────────────────────┤
│ Output:                                                        │
│ • Vector Store (queryable)                                     │
└────────────────────────────────────────────────────────────────┘
                            ↓
┌────────────────────────────────────────────────────────────────┐
│                   FINAL OUTPUT                                 │
│                                                                │
│ output/                                                        │
│ ├── documents.parquet                                          │
│ ├── text_units.parquet                                         │
│ ├── entities.parquet                                           │
│ ├── relationships.parquet                                      │
│ ├── communities.parquet                                        │
│ ├── community_reports.parquet                                  │
│ └── embeddings/                                                │
│     ├── document_text_embedding.parquet                        │
│     ├── text_unit_text_embedding.parquet                       │
│     ├── entity_description_embedding.parquet                   │
│     └── community_full_content_embedding.parquet               │
│                                                                │
│ + Vector Store (если configured)                              │
└────────────────────────────────────────────────────────────────┘
```

## Детальный поток данных по этапам

### Этап 1: Input → Text Units

```python
# Входные данные
documents/ (множество файлов)
    → Loader (CSVLoader, TextLoader, etc.)
    → pd.DataFrame

documents = pd.DataFrame({
    "id": ["doc_001", "doc_002", ...],
    "text": ["Full document text...", ...],
    "title": ["Document Title", ...],
    "metadata": [{...}, ...]
})

    → TokenTextSplitter(chunk_size=1200, overlap=100)
    → text_units

text_units = pd.DataFrame({
    "id": ["unit_001", "unit_002", ...],
    "text": ["Chunk text...", ...],
    "n_tokens": [1187, 1195, ...],
    "document_ids": [["doc_001"], ["doc_001"], ...]
})
```

### Этап 2: Text Units → Entities & Relationships

```python
# Для каждого text_unit:
text_unit = "GraphRAG is a framework..."

    → Extraction Agent (LLM)
    → Промпт: extract_graph_prompt + text_unit
    → LLM Response: structured entities + relationships

entities_raw = [
    {"name": "GraphRAG", "type": "organization", "description": "..."},
    {"name": "Knowledge Graph", "type": "concept", "description": "..."},
    ...
]

relationships_raw = [
    {"source": "GraphRAG", "target": "Knowledge Graph",
     "description": "...", "strength": 9},
    ...
]

    → Агрегация из всех text_units
    → Группировка по entity name
    → Summarization Agent (для множественных описаний)

entities = pd.DataFrame({
    "id": ["entity_001", ...],
    "title": ["GraphRAG", ...],
    "type": ["organization", ...],
    "description": ["Суммаризированное описание", ...],
    "text_unit_ids": [["unit_001", "unit_015", ...], ...]
})

relationships = pd.DataFrame({
    "id": ["rel_001", ...],
    "source": ["GraphRAG", ...],
    "target": ["Knowledge Graph", ...],
    "description": ["...", ...],
    "weight": [0.111, ...],  # 1/strength
    "text_unit_ids": [["unit_001"], ...]
})
```

### Этап 3: Entities & Relationships → Graph

```python
# Построение графа
graph = nx.Graph()

for entity in entities:
    graph.add_node(entity["title"], **entity)

for rel in relationships:
    graph.add_edge(rel["source"], rel["target"], **rel)

# Вычисление метрик
node_degrees = dict(graph.degree())
node_frequencies = entities["text_unit_ids"].apply(len)
layout = nx.spring_layout(graph)

# Обновление entities
entities["node_degree"] = entities["title"].map(node_degrees)
entities["node_frequency"] = node_frequencies
entities["node_x"] = entities["title"].map(lambda t: layout[t][0])
entities["node_y"] = entities["title"].map(lambda t: layout[t][1])

# Обновление relationships
relationships["source_degree"] = relationships["source"].map(node_degrees)
relationships["target_degree"] = relationships["target"].map(node_degrees)
relationships["combined_degree"] = (
    relationships["source_degree"] + relationships["target_degree"]
)
```

### Этап 4: Graph → Communities

```python
# Leiden clustering
from igraph import Graph as IGraph

igraph = IGraph.from_networkx(graph)
leiden_result = igraph.community_leiden(objective_function="modularity")

# Построение иерархии
communities_level_0 = []
for i, community_nodes in enumerate(leiden_result):
    communities_level_0.append({
        "id": f"community_{i}",
        "level": 0,
        "entity_ids": community_nodes,
        "parent": None,
        "children": []
    })

# Рекурсивно создать higher levels
# ... (см. 04-community-detection.md)

communities = pd.DataFrame({
    "id": [...],
    "level": [0, 0, 0, 1, 1, 2],
    "entity_ids": [[...], [...], ...],
    "relationship_ids": [[...], ...],
    "text_unit_ids": [[...], ...],
    "parent": [None, None, None, "community_50", ...],
    "children": [[], [], [], ["community_0", "community_1"], ...]
})
```

### Этап 5: Communities → Community Reports

```python
# Для каждого сообщества
for community in communities:
    # Построить контекст
    community_entities = entities[
        entities["id"].isin(community["entity_ids"])
    ]
    community_relationships = relationships[
        relationships["id"].isin(community["relationship_ids"])
    ]

    context = build_community_context(
        community_entities,
        community_relationships
    )

    # LLM call
    report_prompt = COMMUNITY_REPORT_PROMPT.format(input_text=context)
    report_json = await llm_call(report_prompt)

    # Парсинг
    report = json.loads(report_json)

    # Сохранить
    community_reports.append({
        "id": community["id"],
        "title": report["title"],
        "summary": report["summary"],
        "rating": report["rating"],
        "findings": report["findings"],
        "full_content": create_full_content(report)
    })

community_reports = pd.DataFrame(community_reports)
```

### Этап 6: All Data → Embeddings

```python
# Для каждого поля, которое нужно vectorize:

# 1. Document embeddings
doc_texts = documents["text"].tolist()
doc_embeddings = await create_embeddings(doc_texts, model="text-embedding-3-small")

embeddings_doc = pd.DataFrame({
    "id": documents["id"],
    "embedding": doc_embeddings
})

# 2. Text unit embeddings
unit_texts = text_units["text"].tolist()
unit_embeddings = await create_embeddings(unit_texts)

embeddings_unit = pd.DataFrame({
    "id": text_units["id"],
    "embedding": unit_embeddings
})

# 3. Entity embeddings (title + description)
entity_texts = (entities["title"] + " " + entities["description"]).tolist()
entity_embeddings = await create_embeddings(entity_texts)

embeddings_entity = pd.DataFrame({
    "id": entities["id"],
    "embedding": entity_embeddings
})

# 4. Community report embeddings (full_content)
report_texts = community_reports["full_content"].tolist()
report_embeddings = await create_embeddings(report_texts)

embeddings_community = pd.DataFrame({
    "id": community_reports["id"],
    "embedding": report_embeddings
})

# ... и так далее для всех полей
```

## Трансформации данных

### Документы → Text Units

```
1 Document (5000 tokens)
    ↓ Chunking (size=1200, overlap=100)
    ↓
4-5 Text Units (~1200 tokens each)
```

### Text Units → Entities

```
1000 Text Units
    ↓ LLM Extraction (avg 5 entities per unit)
    ↓
5000 Raw Entities
    ↓ Deduplication + Summarization
    ↓
2000 Unique Entities
```

### Entities → Communities

```
2000 Entities + 8000 Relationships
    ↓ Graph Construction
    ↓
NetworkX Graph (2000 nodes, 8000 edges)
    ↓ Leiden Clustering
    ↓
Communities:
- Level 0: 150 communities (avg 13 entities)
- Level 1: 30 communities (avg 67 entities)
- Level 2: 5 communities (avg 400 entities)
```

### Communities → Reports

```
150 + 30 + 5 = 185 Communities
    ↓ LLM Report Generation
    ↓
185 Community Reports
    ↓ Embedding
    ↓
185 Report Embeddings
```

## Статистика типичного запуска

### Пример: 100 документов

```python
STATISTICS = {
    "input": {
        "documents": 100,
        "avg_tokens_per_doc": 5000,
        "total_tokens": 500000
    },
    "stage_1_chunking": {
        "text_units": 420,
        "avg_tokens_per_unit": 1187
    },
    "stage_2_extraction": {
        "llm_calls": 420,
        "entities_raw": 2100,
        "entities_unique": 850,
        "relationships": 3400
    },
    "stage_3_graph": {
        "graph_nodes": 850,
        "graph_edges": 3400,
        "graph_density": 0.0094
    },
    "stage_4_communities": {
        "level_0": 65,
        "level_1": 12,
        "level_2": 2,
        "total": 79
    },
    "stage_5_reports": {
        "llm_calls": 79,
        "avg_report_length": 1500  # tokens
    },
    "stage_6_embeddings": {
        "documents": 100,
        "text_units": 420,
        "entities": 850,
        "relationships": 3400,
        "community_reports": 79,
        "total_embeddings": 4849
    },
    "costs": {
        "extraction_llm": "$35",
        "summarization_llm": "$2",
        "reports_llm": "$18",
        "embeddings": "$1.50",
        "total": "$56.50"
    },
    "time": {
        "chunking": "10 sec",
        "extraction": "~28 min",
        "graph_construction": "5 sec",
        "community_detection": "8 sec",
        "reports": "~5 min",
        "embeddings": "45 sec",
        "total": "~35 min"
    }
}
```

## Зависимости между этапами

### Критический путь

```
documents
    ↓ (обязательно)
text_units
    ↓ (обязательно)
entities + relationships (через LLM)
    ↓ (обязательно)
graph
    ↓ (обязательно)
communities
    ↓ (обязательно для query quality)
community_reports (через LLM)
    ↓ (обязательно для retrieval)
embeddings
    ↓ (опционально)
vector_store
```

### Параллелизация

```
# Этапы, которые можно распараллелить:

1. Extraction (по text_units)
   - Каждый text_unit обрабатывается независимо
   - Параллелизм: concurrent_requests (3-5)

2. Summarization (по entities)
   - Каждая entity суммаризируется независимо
   - Параллелизм: concurrent_requests

3. Community Reports (по communities)
   - Каждое сообщество обрабатывается независимо (на одном level)
   - Параллелизм: concurrent_requests

4. Embeddings (по batches)
   - Батчи обрабатываются последовательно, но внутри батча параллельно
```

## Форматы данных

### Parquet Tables Schema

Все данные хранятся в Apache Parquet формате:

```python
# Преимущества Parquet:
BENEFITS = {
    "compression": "Эффективное сжатие (gzip, snappy)",
    "columnar": "Столбцовое хранение для быстрых запросов",
    "typed": "Строгая типизация данных",
    "compatible": "Совместимость с pandas, spark, duckdb",
    "performance": "Быстрое чтение/запись"
}

# Типичные размеры файлов (для 100 документов):
FILE_SIZES = {
    "documents.parquet": "2.5 MB",
    "text_units.parquet": "8.0 MB",
    "entities.parquet": "3.2 MB",
    "relationships.parquet": "5.5 MB",
    "communities.parquet": "1.8 MB",
    "community_reports.parquet": "4.5 MB",
    "embeddings/*.parquet": "45 MB",  # Самые большие
    "total": "~70 MB"
}
```

## Следующий раздел

→ [Конфигурация системы](09-configuration.md)
