# Semantic Patterns

## Обзор

**Semantic Patterns** — это паттерны обработки семантической информации, используемые для трансформации, навигации и поиска в пространстве значений.

## Основные Semantic Patterns

1. **Chunking Strategy Pattern** - стратегии разбиения текста
2. **Embedding Pattern** - векторное представление семантики
3. **Graph Construction Pattern** - построение семантического графа
4. **Context Building Pattern** - формирование семантического контекста
5. **Semantic Retrieval Pattern** - поиск по семантическому сходству

---

## 1. Chunking Strategy Pattern

### Определение

**Chunking Strategy** — паттерн для разбиения непрерывного текста на семантически когерентные фрагменты с сохранением контекста.

### Strategy Interface

```python
# graphrag/index/operations/chunk_text/typing.py

@dataclass
class TextChunk:
    """Output of chunking strategy."""
    text_chunk: str
    source_doc_indices: list[int]

# Strategy protocol
def chunking_strategy(
    input: list[str],
    config: ChunkingConfig,
    tick: ProgressTicker
) -> Iterable[TextChunk]:
    """Interface for chunking strategies."""
    ...
```

### Strategy 1: Token-based Chunking

```python
# graphrag/index/operations/chunk_text/strategies.py

def run_tokens(
    input: list[str],
    config: ChunkingConfig,
    tick: ProgressTicker,
) -> Iterable[TextChunk]:
    """Chunks text using token count with overlap."""
    tokens_per_chunk = config.size  # 1200 default
    chunk_overlap = config.overlap  # 100 default
    encoding_name = config.encoding_model  # cl100k_base

    # Get tiktoken encoder/decoder
    encode, decode = get_encoding_fn(encoding_name)

    # Split texts on tokens with overlap
    return split_multiple_texts_on_tokens(
        input,
        Tokenizer(
            chunk_overlap=chunk_overlap,
            tokens_per_chunk=tokens_per_chunk,
            encode=encode,
            decode=decode,
        ),
        tick,
    )

# Core algorithm (simplified)
def split_text_on_tokens(text: str, tokenizer: Tokenizer) -> Iterable[str]:
    """Sliding window over tokens."""
    tokens = tokenizer.encode(text)
    chunks = []

    start_idx = 0
    while start_idx < len(tokens):
        # Extract chunk
        end_idx = min(start_idx + tokenizer.tokens_per_chunk, len(tokens))
        chunk_tokens = tokens[start_idx:end_idx]

        # Decode back to text
        chunk_text = tokenizer.decode(chunk_tokens)
        chunks.append(chunk_text)

        # Move window with overlap
        start_idx += (tokenizer.tokens_per_chunk - tokenizer.chunk_overlap)

    return chunks
```

#### Характеристики Token-based

| Характеристика | Значение |
|---|---|
| Размер chunk | 1200 tokens (default) |
| Overlap | 100 tokens (8.3%) |
| Контекст preservation | 95-98% |
| Граница | Token-based (может разрезать слова) |
| Предсказуемость | Высокая (фиксированный размер) |

### Strategy 2: Sentence-based Chunking

```python
def run_sentences(
    input: list[str],
    _config: ChunkingConfig,
    tick: ProgressTicker
) -> Iterable[TextChunk]:
    """Chunks text by sentence boundaries."""
    for doc_idx, text in enumerate(input):
        # Use NLTK for sentence tokenization
        sentences = nltk.sent_tokenize(text)

        for sentence in sentences:
            yield TextChunk(
                text_chunk=sentence,
                source_doc_indices=[doc_idx],
            )
        tick(1)
```

#### Характеристики Sentence-based

| Характеристика | Значение |
|---|---|
| Размер chunk | Variable (sentence length) |
| Overlap | None |
| Контекст preservation | 85-90% (lower, no overlap) |
| Граница | Sentence boundary (natural) |
| Предсказуемость | Low (variable size) |

### Selection Criteria

```python
# graphrag/config/models/chunking_config.py

class ChunkStrategy(str, Enum):
    """Available chunking strategies."""
    TOKENS = "tokens"      # Default: predictable, good overlap
    SENTENCES = "sentences"  # Natural boundaries

# Config
class ChunkingConfig:
    strategy: ChunkStrategy = ChunkStrategy.TOKENS
    size: int = 1200  # tokens
    overlap: int = 100  # tokens
    encoding_model: str = "cl100k_base"
```

**Используйте Token-based**:
- ✅ Когда нужен предсказуемый размер
- ✅ Для LLM с token limits
- ✅ Когда overlap критичен для контекста

**Используйте Sentence-based**:
- ✅ Для естественных границ
- ✅ Когда семантическая целостность важнее размера
- ✅ Для анализа на уровне предложений

---

## 2. Embedding Pattern

### Определение

**Embedding Pattern** — преобразование текста или концептов в плотное векторное представление, где семантически близкие объекты находятся близко в пространстве.

### Pattern Structure

```python
class EmbeddingPattern:
    """Text → Dense Vector transformation."""

    def __init__(self, embedding_model: EmbeddingModel):
        self._model = embedding_model

    async def embed_text(self, text: str) -> np.ndarray:
        """Transform text to dense vector."""
        # Normalize text
        text = self._normalize(text)

        # Call embedding model
        vector = await self._model.embed(text)

        # Normalize vector (optional)
        vector = vector / np.linalg.norm(vector)

        return vector  # Shape: (dimensions,) e.g., (1536,)

    async def embed_batch(self, texts: list[str]) -> np.ndarray:
        """Batch embedding for efficiency."""
        vectors = await self._model.embed_batch(texts)
        return vectors  # Shape: (batch_size, dimensions)
```

### Реализация в GraphRAG

```python
# graphrag/index/operations/embed_text/embed_text.py

async def embed_text(
    text_units: pd.DataFrame,
    callbacks: WorkflowCallbacks,
    cache: PipelineCache,
    text_embedder: EmbeddingModel,
    embedding_name: str,
) -> pd.DataFrame:
    """Embed text units."""
    texts_to_embed = text_units["text"].tolist()

    # Batch embedding
    embeddings = await text_embedder.embed_batch(texts_to_embed)

    # Add to dataframe
    text_units[embedding_name] = embeddings.tolist()

    return text_units
```

### Semantic Properties

```python
# Similarity in embedding space
def cosine_similarity(v1: np.ndarray, v2: np.ndarray) -> float:
    """Semantic similarity via cosine distance."""
    return np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))

# Example
text1 = "GraphRAG is a knowledge graph system"
text2 = "Knowledge graphs organize information"
text3 = "The weather is sunny today"

embedding1 = await embed_text(text1)
embedding2 = await embed_text(text2)
embedding3 = await embed_text(text3)

sim_12 = cosine_similarity(embedding1, embedding2)  # ~0.85 (high)
sim_13 = cosine_similarity(embedding1, embedding3)  # ~0.12 (low)
```

### Embedding Applications

| Application | Pattern | Code Location |
|---|---|---|
| **Text Unit Embedding** | Embed chunks for retrieval | `operations/embed_text/` |
| **Entity Embedding** | Embed entity descriptions | `operations/embed_text/` |
| **Community Embedding** | Embed community reports | `operations/embed_text/` |
| **Query Embedding** | Embed queries for search | `query/context_builder/` |

### Optimization

```python
# Batch processing for efficiency
BATCH_SIZE = 100

async def embed_large_corpus(texts: list[str]) -> list[np.ndarray]:
    """Embed with batching."""
    embeddings = []

    for i in range(0, len(texts), BATCH_SIZE):
        batch = texts[i:i+BATCH_SIZE]
        batch_embeddings = await embed_batch(batch)
        embeddings.extend(batch_embeddings)

    return embeddings
```

---

## 3. Graph Construction Pattern

### Определение

**Graph Construction** — паттерн построения семантического графа из текста через извлечение сущностей и связей.

### Pipeline

```
Text
  ↓ (T2) Entity Extraction
Entities
  ↓ (T3) Relationship Extraction
Entities + Relationships
  ↓ (Build Graph)
NetworkX Graph
  ↓ (T4) Consolidation
Consolidated Graph
```

### Implementation

```python
# graphrag/index/operations/extract_graph/extract_graph.py

async def extract_graph(
    text_units: pd.DataFrame,
    callbacks: WorkflowCallbacks,
    cache: PipelineCache,
    graph_extractor: GraphExtractor,
    **kwargs
) -> pd.DataFrame:
    """Extract graph from text units."""

    # Извлечение для каждого text unit
    results = []
    for idx, row in text_units.iterrows():
        text = row["text"]

        # Extract entities and relationships (using LLM)
        extraction_result = await graph_extractor([text])

        # extraction_result.output is a NetworkX graph
        graph = extraction_result.output

        results.append({
            "graph": graph,
            "entities": list(graph.nodes()),
            "relationships": list(graph.edges()),
        })

    return pd.DataFrame(results)
```

### Graph Data Structure

```python
import networkx as nx

# NetworkX graph representation
graph = nx.Graph()

# Add entity nodes
graph.add_node(
    "GRAPHRAG",
    type="TECHNOLOGY",
    description="Knowledge graph RAG system",
)

graph.add_node(
    "MICROSOFT RESEARCH",
    type="ORGANIZATION",
    description="Research division of Microsoft",
)

# Add relationship edges
graph.add_edge(
    "GRAPHRAG",
    "MICROSOFT RESEARCH",
    relationship="developed_by",
    description="GraphRAG was developed by Microsoft Research",
    strength=9,
)
```

### Graph Operations

```python
# Consolidation: Merge similar entities
def consolidate_graph(graphs: list[nx.Graph]) -> nx.Graph:
    """Merge multiple graphs into one."""
    consolidated = nx.Graph()

    for graph in graphs:
        for node in graph.nodes():
            if node in consolidated:
                # Merge descriptions
                consolidated.nodes[node]["description"] += "\n" + graph.nodes[node]["description"]
            else:
                # Add new node
                consolidated.add_node(node, **graph.nodes[node])

        for u, v, data in graph.edges(data=True):
            if consolidated.has_edge(u, v):
                # Merge relationships
                consolidated[u][v]["strength"] = max(
                    consolidated[u][v].get("strength", 0),
                    data.get("strength", 0)
                )
            else:
                # Add new edge
                consolidated.add_edge(u, v, **data)

    return consolidated
```

---

## 4. Context Building Pattern

### Определение

**Context Building** — паттерн формирования релевантного семантического контекста для ответа на запрос.

### Builder Interface

```python
# graphrag/query/context_builder/builders.py

class LocalContextBuilder(ABC):
    """Interface for building search context."""

    @abstractmethod
    def build_context(
        self,
        query: str,
        conversation_history: ConversationHistory | None = None,
        **kwargs,
    ) -> ContextBuilderResult:
        """Build context for query."""
```

### Implementation: LocalSearchMixedContext

```python
# graphrag/query/structured_search/local_search/mixed_context.py

class LocalSearchMixedContext(LocalContextBuilder):
    """Build context from entities, relationships, text units."""

    def __init__(
        self,
        entities: list[Entity],
        relationships: list[Relationship],
        text_units: list[TextUnit],
        entity_text_embeddings: BaseVectorStore,
        text_embedder: EmbeddingModel,
        token_encoder: tiktoken.Encoding,
    ):
        self.entities = entities
        self.relationships = relationships
        self.text_units = text_units
        self.entity_embeddings = entity_text_embeddings
        self.text_embedder = text_embedder
        self.token_encoder = token_encoder

    def build_context(
        self, query: str, **kwargs
    ) -> ContextBuilderResult:
        """Multi-step context building."""

        # ===== STEP 1: Entity Extraction from Query =====
        query_entities = self._extract_entities_from_query(query)

        # ===== STEP 2: Semantic Retrieval via Embeddings =====
        query_embedding = self.text_embedder.embed(query)
        similar_entities = self.entity_embeddings.similarity_search(
            query_embedding, k=kwargs.get("top_k_mapped_entities", 30)
        )

        # ===== STEP 3: Graph Expansion =====
        # Get relationships for retrieved entities
        related_relationships = self._get_relationships(similar_entities)

        # Expand to neighboring entities
        expanded_entities = self._expand_entities(
            similar_entities, related_relationships
        )

        # ===== STEP 4: Text Unit Retrieval =====
        # Get original text units for entities
        text_units = self._get_text_units_for_entities(expanded_entities)

        # ===== STEP 5: Context Formatting =====
        context = self._format_context(
            entities=expanded_entities,
            relationships=related_relationships,
            text_units=text_units,
            max_tokens=kwargs.get("max_tokens", 8000),
        )

        return ContextBuilderResult(
            context_chunks=context,
            context_records={
                "entities": pd.DataFrame(expanded_entities),
                "relationships": pd.DataFrame(related_relationships),
                "text_units": pd.DataFrame(text_units),
            },
        )
```

### Context Format Example

```
# Entities
- GRAPHRAG (TECHNOLOGY): Knowledge graph RAG system
- MICROSOFT RESEARCH (ORGANIZATION): Research division of Microsoft
- RAG (CONCEPT): Retrieval-Augmented Generation

# Relationships
- GRAPHRAG developed_by MICROSOFT RESEARCH (strength: 9)
- GRAPHRAG implements RAG (strength: 8)

# Text Units
[1] "GraphRAG is a knowledge graph approach to RAG developed by
     Microsoft Research in 2023..."
[2] "The system uses community detection to organize entities into
     hierarchical topics..."
```

---

## 5. Semantic Retrieval Pattern

### Определение

**Semantic Retrieval** — поиск релевантных объектов на основе семантического сходства в векторном пространстве.

### Pattern Structure

```python
class SemanticRetrieval:
    """Retrieval based on semantic similarity."""

    def __init__(
        self,
        vector_store: BaseVectorStore,
        embedding_model: EmbeddingModel,
    ):
        self._vector_store = vector_store
        self._embedding_model = embedding_model

    async def search(
        self,
        query: str,
        top_k: int = 10,
        filters: dict | None = None,
    ) -> list[ScoredResult]:
        """Semantic similarity search."""

        # 1. Embed query
        query_embedding = await self._embedding_model.embed(query)

        # 2. Vector search
        results = await self._vector_store.similarity_search(
            query_embedding,
            k=top_k,
            filters=filters,
        )

        # 3. Re-rank (optional)
        results = self._rerank(query, results)

        return results
```

### Vector Store Interface

```python
# graphrag/vector_stores/base.py

class BaseVectorStore(ABC):
    """Base interface for vector stores."""

    @abstractmethod
    async def similarity_search(
        self,
        query_embedding: np.ndarray,
        k: int = 10,
        filters: dict | None = None,
    ) -> list[tuple[str, float]]:
        """Search by vector similarity."""

    @abstractmethod
    async def add_embeddings(
        self,
        texts: list[str],
        embeddings: list[np.ndarray],
        metadata: list[dict] | None = None,
    ) -> None:
        """Add embeddings to store."""
```

### Implementations

```python
# graphrag/vector_stores/lancedb.py
class LanceDBVectorStore(BaseVectorStore):
    """LanceDB implementation (local, fast)."""

# graphrag/vector_stores/azure_ai_search.py
class AzureAISearchVectorStore(BaseVectorStore):
    """Azure AI Search implementation (cloud, scalable)."""
```

### Hybrid Retrieval

```python
def hybrid_retrieval(
    query: str,
    vector_store: BaseVectorStore,
    entity_index: dict,
    top_k: int = 30,
) -> list[Entity]:
    """Combine semantic + keyword retrieval."""

    # Semantic retrieval
    semantic_results = vector_store.similarity_search(query, k=top_k)

    # Keyword matching (exact entity names in query)
    keyword_results = [
        entity for entity in entity_index.values()
        if entity.name.lower() in query.lower()
    ]

    # Combine and deduplicate
    combined = list(set(semantic_results + keyword_results))

    # Re-rank by combined score
    ranked = rank_by_hybrid_score(combined, query)

    return ranked[:top_k]
```

---

## Semantic Pattern Composition

### Example: Complete Indexing Pipeline

```python
async def semantic_indexing_pipeline(documents: list[str]) -> KnowledgeGraph:
    """Complete semantic processing pipeline."""

    # 1. CHUNKING PATTERN
    chunks = chunking_strategy(
        documents,
        config=ChunkingConfig(strategy="tokens", size=1200, overlap=100)
    )

    # 2. GRAPH CONSTRUCTION PATTERN
    graphs = []
    for chunk in chunks:
        graph = await extract_graph(chunk)  # LLM extraction
        graphs.append(graph)

    # Consolidate graphs
    consolidated_graph = consolidate_graphs(graphs)

    # 3. EMBEDDING PATTERN
    # Embed entities
    entity_descriptions = [node["description"] for node in consolidated_graph.nodes()]
    entity_embeddings = await embed_batch(entity_descriptions)

    # Embed chunks
    chunk_texts = [chunk.text for chunk in chunks]
    chunk_embeddings = await embed_batch(chunk_texts)

    # 4. INDEXING
    knowledge_graph = KnowledgeGraph(
        graph=consolidated_graph,
        entity_embeddings=entity_embeddings,
        chunk_embeddings=chunk_embeddings,
    )

    return knowledge_graph
```

### Example: Query Execution

```python
async def semantic_query(
    query: str,
    knowledge_graph: KnowledgeGraph,
) -> Answer:
    """Complete query execution using semantic patterns."""

    # 1. EMBEDDING PATTERN
    query_embedding = await embed_text(query)

    # 2. SEMANTIC RETRIEVAL PATTERN
    relevant_entities = await semantic_retrieval(
        query_embedding,
        knowledge_graph.entity_embeddings,
        top_k=30
    )

    # 3. CONTEXT BUILDING PATTERN
    context = build_context(
        query=query,
        entities=relevant_entities,
        graph=knowledge_graph.graph,
    )

    # 4. ANSWER GENERATION (Agent Pattern)
    answer = await generate_answer(
        query=query,
        context=context,
    )

    return answer
```

---

**Related**:
- [Agent Patterns](01-agent-patterns.md)
- [Architectural Patterns](03-architectural-patterns.md)
- [Code-Level Patterns](04-code-level-patterns.md)
