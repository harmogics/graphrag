# Embeddings и векторизация (Embeddings & Vectorization)

## Обзор этапа

**Embedding Generation** — это финальный этап pipeline, который преобразует текстовые данные в **векторные представления** для семантического поиска и retrieval. GraphRAG создает embeddings для всех основных компонентов системы, обеспечивая эффективный поиск по содержимому.

## Архитектура компонента

### Ключевые файлы
- **Workflow**: `graphrag/index/workflows/generate_text_embeddings.py`
- **Embed Operation**: `graphrag/index/operations/embed_text/embed_text.py`
- **Graph Embeddings**: `graphrag/index/operations/embed_graph/embed_graph.py`
- **Конфигурация**: `graphrag/config/models/text_embedding_config.py`
- **Embeddings Spec**: `graphrag/config/embeddings.py`

## Типы embeddings в GraphRAG

### 1. Text Embeddings (основные)

GraphRAG создает embeddings для следующих компонентов:

```python
# graphrag/config/embeddings.py

EMBEDDED_FIELDS = {
    "document_text_embedding",              # documents.text
    "text_unit_text_embedding",             # text_units.text
    "entity_title_embedding",               # entities.title
    "entity_description_embedding",         # entities.title + description
    "relationship_description_embedding",   # relationships.description
    "community_title_embedding",            # community_reports.title
    "community_summary_embedding",          # community_reports.summary
    "community_full_content_embedding",     # community_reports.full_content
}
```

**Файл**: `graphrag/config/embeddings.py:15`

### 2. Graph Embeddings (опциональные)

Node embeddings на основе **Node2Vec** алгоритма для структурного представления графа.

## Процесс генерации embeddings

```
INPUT: Финализированные parquet таблицы
    ↓
┌─────────────────────────────────────────┐
│  1. Для каждого embedding field:        │
│     ┌─────────────────────────────────┐ │
│     │ a. Загрузить source DataFrame   │ │
│     └─────────────────────────────────┘ │
│     ┌─────────────────────────────────┐ │
│     │ b. Извлечь текстовую колонку    │ │
│     └─────────────────────────────────┘ │
│     ┌─────────────────────────────────┐ │
│     │ c. Батчирование текстов         │ │
│     │    - По batch_size              │ │
│     │    - По batch_max_tokens        │ │
│     └─────────────────────────────────┘ │
│     ┌─────────────────────────────────┐ │
│     │ d. Embedding LLM call           │ │
│     │    (text-embedding-3-small)     │ │
│     └─────────────────────────────────┘ │
│     ┌─────────────────────────────────┐ │
│     │ e. Кэширование результатов      │ │
│     └─────────────────────────────────┘ │
│     ┌─────────────────────────────────┐ │
│     │ f. Сохранение embeddings        │ │
│     └─────────────────────────────────┘ │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  2. Опционально: Graph embeddings       │
│     (Node2Vec)                          │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  3. Загрузка в Vector Store             │
│     (если configured)                   │
└─────────────────────────────────────────┘
    ↓
OUTPUT: embeddings/*.parquet + Vector Store
```

## Конфигурация Text Embeddings

### TextEmbeddingConfig параметры

```python
class TextEmbeddingConfig:
    # Модель embedding
    model_id: str = DEFAULT_EMBEDDING_MODEL_ID  # "text-embedding-3-small"

    # Vector store для хранения
    vector_store_id: str = DEFAULT_VECTOR_STORE_ID

    # Batch size для API calls
    batch_size: int = 500

    # Максимальное количество токенов в батче
    batch_max_tokens: int = 10000

    # Какие embeddings создавать
    target: TextEmbeddingTarget = "all"  # all | required | selected | none

    # Список конкретных embeddings (если target = "selected")
    names: list[str] = []

    # Override стратегии
    strategy: dict | None = None

    # Модель токенизации
    encoding_model: str = "cl100k_base"
```

**Файл**: `graphrag/config/models/text_embedding_config.py:20`

### TextEmbeddingTarget опции

```python
class TextEmbeddingTarget(str, Enum):
    ALL = "all"              # Все доступные embeddings
    REQUIRED = "required"    # Только обязательные (для query)
    SELECTED = "selected"    # Только указанные в names
    NONE = "none"            # Не создавать embeddings
```

## Embedding Models

### OpenAI Embedding Models

GraphRAG поддерживает следующие модели OpenAI:

```python
# text-embedding-3-small (рекомендуется)
{
    "model": "text-embedding-3-small",
    "dimensions": 1536,
    "cost_per_1M_tokens": $0.02,
    "max_input_tokens": 8191
}

# text-embedding-3-large (высокое качество)
{
    "model": "text-embedding-3-large",
    "dimensions": 3072,
    "cost_per_1M_tokens": $0.13,
    "max_input_tokens": 8191
}

# text-embedding-ada-002 (legacy)
{
    "model": "text-embedding-ada-002",
    "dimensions": 1536,
    "cost_per_1M_tokens": $0.10,
    "max_input_tokens": 8191
}
```

### Azure OpenAI Embeddings

```python
# Конфигурация для Azure
class LanguageModelConfig:
    type: ModelType = "azure_openai_embedding"
    api_base: str = "https://YOUR_RESOURCE.openai.azure.com/"
    api_version: str = "2024-02-15-preview"
    deployment_name: str = "text-embedding-3-small"
    api_key: str = "YOUR_API_KEY"
```

## Процесс embedding

### Батчирование текстов

```python
async def batch_texts_for_embedding(
    texts: list[str],
    batch_size: int = 500,
    batch_max_tokens: int = 10000,
    encoding_model: str = "cl100k_base"
) -> list[list[str]]:
    """
    Группирует тексты в батчи для эффективной обработки

    Args:
        texts: Список текстов для embedding
        batch_size: Максимальное количество текстов в батче
        batch_max_tokens: Максимальное количество токенов в батче
        encoding_model: Модель токенизации

    Returns:
        Список батчей
    """
    import tiktoken

    encoding = tiktoken.get_encoding(encoding_model)
    batches = []
    current_batch = []
    current_tokens = 0

    for text in texts:
        text_tokens = len(encoding.encode(text))

        # Проверить, умещается ли в текущий батч
        if (len(current_batch) >= batch_size or
            current_tokens + text_tokens > batch_max_tokens):
            # Сохранить текущий батч и начать новый
            if current_batch:
                batches.append(current_batch)
            current_batch = [text]
            current_tokens = text_tokens
        else:
            current_batch.append(text)
            current_tokens += text_tokens

    # Добавить последний батч
    if current_batch:
        batches.append(current_batch)

    return batches
```

**Файл**: `graphrag/index/operations/embed_text/embed_text.py:65`

### Embedding LLM Call

```python
async def create_embeddings(
    texts: list[str],
    embedding_model: str,
    api_key: str
) -> list[list[float]]:
    """
    Создает embeddings для списка текстов

    Args:
        texts: Список текстов
        embedding_model: Название модели
        api_key: API ключ

    Returns:
        Список векторов embeddings
    """
    import openai

    openai.api_key = api_key

    response = await openai.Embedding.acreate(
        model=embedding_model,
        input=texts
    )

    # Извлечь embeddings в правильном порядке
    embeddings = [item["embedding"] for item in response["data"]]

    return embeddings
```

### Кэширование embeddings

```python
async def cached_embed_text(
    text: str,
    embedding_model: str,
    cache: PipelineCache
) -> list[float]:
    """
    Создает embedding с кэшированием
    """
    # Ключ кэша: hash(text + model)
    cache_key = hashlib.sha256(
        f"{text}_{embedding_model}".encode()
    ).hexdigest()

    # Проверить кэш
    cached_embedding = await cache.get(cache_key)
    if cached_embedding:
        return cached_embedding

    # Создать embedding
    embedding = await create_embeddings([text], embedding_model)

    # Сохранить в кэш
    await cache.set(cache_key, embedding[0])

    return embedding[0]
```

## Детальные embeddings

### 1. Document Text Embeddings

```python
# Embeddings для полного текста документов
# Используется для: document-level retrieval

INPUT: documents.parquet["text"]
OUTPUT: embeddings.document_text_embedding.parquet

SCHEMA:
{
    "id": str,              # document.id
    "embedding": list[float]  # vector (1536 dims)
}
```

### 2. Text Unit Embeddings

```python
# Embeddings для text chunks
# Используется для: chunk-level retrieval, local search

INPUT: text_units.parquet["text"]
OUTPUT: embeddings.text_unit_text_embedding.parquet

SCHEMA:
{
    "id": str,              # text_unit.id
    "embedding": list[float]
}
```

### 3. Entity Embeddings

```python
# Два типа entity embeddings:

# a) Entity Title Embedding (только название)
INPUT: entities.parquet["title"]
OUTPUT: embeddings.entity_title_embedding.parquet

# b) Entity Description Embedding (название + описание)
INPUT: entities.parquet["title"] + " " + entities.parquet["description"]
OUTPUT: embeddings.entity_description_embedding.parquet

# Entity description embedding более информативен для retrieval
```

### 4. Relationship Embeddings

```python
# Embeddings для отношений
# Используется для: поиск похожих отношений

INPUT: relationships.parquet["source"] + " -> " + relationships.parquet["target"]
       + ": " + relationships.parquet["description"]
OUTPUT: embeddings.relationship_description_embedding.parquet
```

### 5. Community Report Embeddings

```python
# Три типа community embeddings:

# a) Title Embedding (краткий)
INPUT: community_reports.parquet["title"]
OUTPUT: embeddings.community_title_embedding.parquet

# b) Summary Embedding (средний)
INPUT: community_reports.parquet["summary"]
OUTPUT: embeddings.community_summary_embedding.parquet

# c) Full Content Embedding (полный, рекомендуется)
INPUT: community_reports.parquet["full_content"]
OUTPUT: embeddings.community_full_content_embedding.parquet

# Full content embedding наиболее полезен для global search
```

## Graph Embeddings (Node2Vec)

### Концепция Node2Vec

**Node2Vec** — это алгоритм, который создает embeddings для узлов графа на основе структуры связей.

### Принцип работы

```
1. Random Walks:
   Для каждого узла генерируются случайные пути (walks) по графу

2. Word2Vec Training:
   Walks рассматриваются как "sentences", узлы как "words"
   Обучается Word2Vec модель

3. Node Embeddings:
   Для каждого узла получаем embedding vector
```

### Конфигурация Node2Vec

```python
class EmbedGraphConfig:
    enabled: bool = False           # По умолчанию выключено
    dimensions: int = 1536          # Размерность векторов
    num_walks: int = 10             # Random walks per node
    walk_length: int = 40           # Длина каждого walk
    window_size: int = 2            # Context window для word2vec
    iterations: int = 3             # Training iterations
    random_seed: int = 597832       # Для воспроизводимости
    use_lcc: bool = True            # Использовать только LCC
```

**Файл**: `graphrag/config/models/embed_graph_config.py:15`

### Применение Node2Vec

```python
def embed_graph(
    graph: nx.Graph,
    config: EmbedGraphConfig
) -> dict[str, list[float]]:
    """
    Создает embeddings для узлов графа с помощью Node2Vec

    Args:
        graph: NetworkX граф
        config: Конфигурация embeddings

    Returns:
        {node_name: embedding_vector}
    """
    from node2vec import Node2Vec

    # Опционально: использовать только LCC
    if config.use_lcc:
        graph = get_largest_connected_component(graph)

    # Инициализация Node2Vec
    node2vec = Node2Vec(
        graph,
        dimensions=config.dimensions,
        walk_length=config.walk_length,
        num_walks=config.num_walks,
        workers=4
    )

    # Обучение модели
    model = node2vec.fit(
        window=config.window_size,
        min_count=1,
        batch_words=4,
        epochs=config.iterations,
        seed=config.random_seed
    )

    # Извлечь embeddings
    embeddings = {}
    for node in graph.nodes():
        embeddings[node] = model.wv[node].tolist()

    return embeddings
```

**Файл**: `graphrag/index/operations/embed_graph/embed_graph.py:35`

### Использование graph embeddings

```python
# Graph embeddings полезны для:

# 1. Структурный поиск похожих сущностей
similar_entities = find_similar_by_graph_structure(
    entity="GraphRAG",
    graph_embeddings=embeddings
)

# 2. Комбинация с text embeddings
combined_similarity = (
    0.7 * cosine_similarity(text_emb1, text_emb2) +
    0.3 * cosine_similarity(graph_emb1, graph_emb2)
)

# 3. Визуализация графа
# Использовать embeddings для 2D projection (t-SNE, UMAP)
```

## Vector Store Integration

### Поддерживаемые Vector Stores

GraphRAG поддерживает следующие vector stores:

```python
class VectorStoreType(str, Enum):
    LANCEDB = "lancedb"         # Рекомендуется (локальный)
    AZURE_AI_SEARCH = "azure_ai_search"
    CHROMADB = "chromadb"
    QDRANT = "qdrant"
```

### Конфигурация Vector Store

```python
class VectorStoreConfig:
    type: VectorStoreType = VectorStoreType.LANCEDB
    container_name: str = "embeddings"
    overwrite: bool = True

    # Для Azure AI Search
    azure_search_endpoint: str | None = None
    azure_search_key: str | None = None

    # Для других провайдеров
    connection_string: str | None = None
```

**Файл**: `graphrag/config/models/vector_store_config.py:18`

### Загрузка в LanceDB

```python
async def upload_to_lancedb(
    embeddings: pd.DataFrame,
    collection_name: str,
    vector_store_path: str
):
    """
    Загружает embeddings в LanceDB

    Args:
        embeddings: DataFrame с колонками [id, embedding]
        collection_name: Название коллекции
        vector_store_path: Путь к БД
    """
    import lancedb

    # Подключиться к БД
    db = lancedb.connect(vector_store_path)

    # Создать или перезаписать таблицу
    table = db.create_table(
        collection_name,
        data=embeddings.to_dict('records'),
        mode="overwrite"
    )

    # Создать индекс для быстрого поиска
    table.create_index(
        metric="cosine",
        num_partitions=256,
        num_sub_vectors=96
    )
```

## Семантический поиск

### Поиск по embeddings

```python
async def semantic_search(
    query: str,
    embeddings: pd.DataFrame,
    embedding_model: str,
    top_k: int = 10
) -> list[tuple[str, float]]:
    """
    Выполняет семантический поиск по embeddings

    Args:
        query: Поисковый запрос
        embeddings: DataFrame с embeddings
        embedding_model: Модель для создания query embedding
        top_k: Количество результатов

    Returns:
        Список (id, similarity_score)
    """
    # Создать embedding для query
    query_embedding = await create_embeddings([query], embedding_model)

    # Вычислить cosine similarity со всеми embeddings
    similarities = embeddings.apply(
        lambda row: cosine_similarity(
            query_embedding[0],
            row["embedding"]
        ),
        axis=1
    )

    # Сортировать по similarity
    top_indices = similarities.nlargest(top_k).index
    results = [
        (embeddings.loc[idx, "id"], similarities.loc[idx])
        for idx in top_indices
    ]

    return results
```

### Cosine Similarity

```python
def cosine_similarity(vec1: list[float], vec2: list[float]) -> float:
    """
    Вычисляет cosine similarity между двумя векторами

    Returns:
        Similarity score [0, 1]
    """
    import numpy as np

    vec1 = np.array(vec1)
    vec2 = np.array(vec2)

    dot_product = np.dot(vec1, vec2)
    norm1 = np.linalg.norm(vec1)
    norm2 = np.linalg.norm(vec2)

    if norm1 == 0 or norm2 == 0:
        return 0.0

    return dot_product / (norm1 * norm2)
```

## Hybrid Search (Text + Graph)

### Комбинированный поиск

```python
async def hybrid_search(
    query: str,
    text_embeddings: pd.DataFrame,
    graph_embeddings: pd.DataFrame,
    alpha: float = 0.7,
    top_k: int = 10
) -> list[tuple[str, float]]:
    """
    Комбинирует text и graph embeddings для поиска

    Args:
        query: Поисковый запрос
        text_embeddings: Text embeddings
        graph_embeddings: Graph embeddings
        alpha: Вес text embeddings (1-alpha для graph)
        top_k: Количество результатов

    Returns:
        Список (entity_id, combined_score)
    """
    # Поиск по text embeddings
    text_results = await semantic_search(query, text_embeddings, top_k=top_k*2)

    # Для каждого результата, добавить graph similarity
    combined_results = []

    for entity_id, text_score in text_results:
        # Получить graph embedding
        graph_emb = graph_embeddings[graph_embeddings["id"] == entity_id]

        if len(graph_emb) > 0:
            # Вычислить graph similarity (к соседним узлам query entities)
            graph_score = compute_graph_similarity(entity_id, query)

            # Комбинировать scores
            combined_score = alpha * text_score + (1 - alpha) * graph_score
        else:
            combined_score = text_score

        combined_results.append((entity_id, combined_score))

    # Сортировать по combined score
    combined_results.sort(key=lambda x: x[1], reverse=True)

    return combined_results[:top_k]
```

## Оптимизация производительности

### Batch Size Tuning

```python
# Для OpenAI API:
# - Оптимальный batch_size: 500-1000 texts
# - Оптимальный batch_max_tokens: 8000-10000

# Для локальных моделей:
# - Зависит от GPU memory
# - batch_size: 32-128
```

### Параллелизация

```python
async def embed_texts_parallel(
    texts: list[str],
    embedding_model: str,
    num_workers: int = 4
) -> list[list[float]]:
    """
    Параллельная генерация embeddings
    """
    # Разбить на chunks для workers
    chunk_size = len(texts) // num_workers
    chunks = [texts[i:i+chunk_size] for i in range(0, len(texts), chunk_size)]

    # Параллельно обработать
    tasks = [
        create_embeddings(chunk, embedding_model)
        for chunk in chunks
    ]

    results = await asyncio.gather(*tasks)

    # Объединить результаты
    return [emb for result in results for emb in result]
```

### Кэширование и переиспользование

```python
# Embeddings кэшируются по (text + model):
# - Повторные запуски не пересоздают embeddings
# - Изменения в тексте требуют invalidation

# Стратегия обновления:
# 1. Создать новые embeddings только для новых/измененных текстов
# 2. Переиспользовать существующие embeddings
```

## Метрики качества embeddings

### Оценка embeddings

```python
def evaluate_embeddings_quality(
    embeddings: pd.DataFrame,
    ground_truth_pairs: list[tuple[str, str, float]]
) -> dict:
    """
    Оценивает качество embeddings на ground truth парах

    Args:
        embeddings: DataFrame с embeddings
        ground_truth_pairs: [(id1, id2, expected_similarity), ...]

    Returns:
        Метрики качества
    """
    predicted_similarities = []
    expected_similarities = []

    for id1, id2, expected in ground_truth_pairs:
        emb1 = embeddings[embeddings["id"] == id1]["embedding"].values[0]
        emb2 = embeddings[embeddings["id"] == id2]["embedding"].values[0]

        predicted = cosine_similarity(emb1, emb2)

        predicted_similarities.append(predicted)
        expected_similarities.append(expected)

    # Вычислить корреляцию
    from scipy.stats import spearmanr
    correlation, p_value = spearmanr(predicted_similarities, expected_similarities)

    return {
        "spearman_correlation": correlation,
        "p_value": p_value
    }
```

## Следующий этап

После создания embeddings:
- Система готова для **query execution** (global search, local search)
- Embeddings используются для **retrieval** релевантного контекста

→ [Переход к роли языковых агентов](07-llm-agents-roles.md)
