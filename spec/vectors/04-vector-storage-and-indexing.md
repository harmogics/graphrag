# Vector Storage and Indexing

## Overview

Vector storage provides **persistent, scalable storage** for embeddings with **fast similarity search** capabilities. GraphRAG supports multiple vector stores (LanceDB, Azure AI Search, Cosmos DB) that handle embedding persistence, index construction, and ANN query execution.

## Conceptual Foundation

### Vector Database vs. Traditional Database

**Traditional database**:
- **Query**: Exact match (`WHERE name = 'Microsoft'`)
- **Index**: B-tree, hash index
- **Complexity**: O(log N) for point queries

**Vector database**:
- **Query**: Similarity search (`k nearest neighbors to query_embedding`)
- **Index**: HNSW graph, IVF clustering
- **Complexity**: O(log N) for ANN, O(N) for exact

**Why specialized storage?**
- **Efficiency**: ANN indexes optimized for high-dimensional vectors
- **Scale**: Handle millions of 1536-dim vectors (6+ GB raw)
- **Performance**: Sub-millisecond search latency

---

## Vector Store Implementations

### 1. LanceDB (Embedded)

**Architecture**: Local file-based vector database

**Code**: `lancedb.py:21-154`

```python
class LanceDBVectorStore(BaseVectorStore):
    """LanceDB vector storage implementation."""

    def connect(self, **kwargs: Any) -> Any:
        """Connect to vector storage."""
        self.db_connection = lancedb.connect(kwargs["db_uri"])  # Local directory
        if self.collection_name in self.db_connection.table_names():
            self.document_collection = self.db_connection.open_table(self.collection_name)

    def load_documents(
        self, documents: list[VectorStoreDocument], overwrite: bool = True
    ) -> None:
        """Load documents into vector storage."""
        # Convert to PyArrow table
        data = [
            {
                "id": document.id,
                "text": document.text,
                "vector": document.vector,  # 1536-dim float list
                "attributes": json.dumps(document.attributes),
            }
            for document in documents
        ]

        schema = pa.schema([
            pa.field("id", pa.string()),
            pa.field("text", pa.string()),
            pa.field("vector", pa.list_(pa.float64())),  # Vector column
            pa.field("attributes", pa.string()),
        ])

        if overwrite:
            self.document_collection = self.db_connection.create_table(
                self.collection_name, data=data, mode="overwrite"
            )
        else:
            # Append to existing table
            self.document_collection = self.db_connection.open_table(self.collection_name)
            self.document_collection.add(data)
```

**Storage format**: **Apache Arrow** (columnar, zero-copy)

**Indexing**: Automatic ANN index creation on vector column

**Features**:
- **Embedded**: No separate server (runs in-process)
- **File-based**: Data stored in `.lance` files (efficient columnar format)
- **Fast ingest**: Batched writes via PyArrow
- **Automatic indexing**: ANN index built incrementally

**Use cases**:
- **Local development**: No cloud dependencies
- **Small-medium corpora**: < 10M vectors
- **Research/prototyping**: Fast iteration

**Storage size** (10K entities, 1536-dim embeddings):
```
Raw embeddings:  10K × 1536 × 4 bytes = 61.4 MB
LanceDB .lance:  ~65 MB (includes metadata, index)
```

---

### 2. Azure AI Search (Cloud)

**Architecture**: Fully managed cloud vector search service

**Code**: `azure_ai_search.py:36-154`

```python
class AzureAISearchVectorStore(BaseVectorStore):
    """Azure AI Search vector storage implementation."""

    def connect(self, **kwargs: Any) -> Any:
        """Connect to AI search vector storage."""
        url = kwargs["url"]
        api_key = kwargs.get("api_key")
        self.vector_size = kwargs.get("vector_size", DEFAULT_VECTOR_SIZE)

        self.db_connection = SearchClient(
            endpoint=url,
            index_name=self.collection_name,
            credential=AzureKeyCredential(api_key),
        )
        self.index_client = SearchIndexClient(endpoint=url, credential=...)

    def load_documents(self, documents: list[VectorStoreDocument], overwrite: bool = True) -> None:
        """Load documents into Azure AI Search index."""
        if overwrite:
            # Create index with HNSW configuration
            vector_search = VectorSearch(
                algorithms=[
                    HnswAlgorithmConfiguration(
                        name="HnswAlg",
                        parameters=HnswParameters(
                            metric=VectorSearchAlgorithmMetric.COSINE,  # Cosine similarity
                            m=4,              # Connections per node
                            ef_construction=400,  # Build-time breadth
                            ef_search=500,   # Query-time breadth
                        ),
                    )
                ],
                profiles=[
                    VectorSearchProfile(
                        name="vectorSearchProfile",
                        algorithm_configuration_name="HnswAlg",
                    )
                ],
            )

            # Define index schema
            index = SearchIndex(
                name=self.collection_name,
                fields=[
                    SimpleField(name="id", type=SearchFieldDataType.String, key=True),
                    SearchableField(name="text", type=SearchFieldDataType.String),
                    SearchField(
                        name="vector",
                        type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
                        vector_search_dimensions=self.vector_size,  # 1536
                        vector_search_profile_name="vectorSearchProfile",
                    ),
                ],
                vector_search=vector_search,
            )

            self.index_client.create_or_update_index(index)

        # Upload documents
        self.db_connection.upload_documents(documents=[
            {
                "id": doc.id,
                "text": doc.text,
                "vector": doc.vector,
                **doc.attributes,
            }
            for doc in documents
        ])
```

**Storage**: Cloud-managed (Azure infrastructure)

**Indexing**: **HNSW** (Hierarchical Navigable Small World)

**Features**:
- **Managed service**: No infrastructure management
- **Scalability**: Handle billions of vectors
- **High availability**: Built-in replication
- **Security**: Azure AD integration, encryption at rest
- **Geo-distribution**: Deploy across regions

**Use cases**:
- **Production deployments**: Enterprise-scale applications
- **Large corpora**: > 10M vectors
- **Global applications**: Multi-region deployment

**Cost** (Azure pricing, 2024):
```
Storage: ~$0.50 per GB/month
Queries: ~$5 per 1M queries
Index: 10M vectors × 1536-dim = ~61 GB → ~$30/month storage
```

---

### 3. Cosmos DB (Azure, NoSQL + Vector)

**Architecture**: Hybrid NoSQL database with vector search

**Features**:
- **Document + vector**: Store JSON documents with embedded vectors
- **Global distribution**: Multi-region replication
- **Change feed**: Real-time updates
- **DiskANN**: Microsoft's ANN algorithm (alternative to HNSW)

**Use case**: Applications needing both document storage and vector search

---

## Index Construction Algorithms

### HNSW (Hierarchical Navigable Small World)

**Concept**: Multi-layer graph where each layer is a "navigable small world"

**Structure**:
```
Layer 2 (coarse, sparse):  A ←→ H ←→ Z
                            ↓     ↓
Layer 1 (medium):          A ←→ D ←→ H ←→ T ←→ Z
                            ↓     ↓     ↓     ↓     ↓
Layer 0 (fine, dense):     A-B-C-D-E-F-G-H-...-Z (all vectors)
```

**Build algorithm**:
1. **Insert vector**: Start at top layer
2. **Navigate to nearest neighbor** in current layer
3. **Descend**: Move to layer below
4. **Repeat** until reaching layer 0
5. **Connect**: Add edges to k nearest neighbors in each layer

**Parameters**:
- **m**: Max connections per node (default: 16)
  - Higher = better search, more memory
- **ef_construction**: Build-time search breadth (default: 200)
  - Higher = better index quality, slower build

**Query algorithm**:
1. **Start at top layer** (entry point)
2. **Greedy search**: Move to nearest neighbor
3. **Descend** when no closer neighbors found
4. **Repeat** until layer 0
5. **Return k-NN** from layer 0

**Complexity**:
- **Build**: O(N × log N × D)
- **Query**: O(log N × D)

**Memory overhead**:
```
Graph structure: ~2x embedding size
10M vectors × 1536-dim × 4 bytes = 61.4 MB (raw)
HNSW index: ~120 MB (includes graph + vectors)
```

---

## Data Model

### VectorStoreDocument

**Code**: `base.py:15-26`

```python
@dataclass
class VectorStoreDocument:
    """A document stored in vector storage."""

    id: str | int             # Unique identifier
    text: str | None          # Original text (optional)
    vector: list[float] | None  # Embedding (1536-dim)
    attributes: dict[str, Any] = field(default_factory=dict)  # Metadata
```

**Example**:
```python
VectorStoreDocument(
    id="entity_MICROSOFT",
    text="Microsoft is a technology company founded in 1975...",
    vector=[0.023, -0.015, ..., 0.042],  # 1536 floats
    attributes={
        "title": "MICROSOFT",
        "type": "organization",
        "degree": 120,
    }
)
```

**Storage mapping**:
```
LanceDB table schema:
  id: string
  text: string
  vector: list[float64] (1536 elements)
  attributes: string (JSON-encoded)

Azure AI Search index:
  id: string (key)
  text: searchable string
  vector: Collection(Single) with vector_search_dimensions=1536
  title, type, degree: additional fields
```

---

### VectorStoreSearchResult

**Code**: `base.py:29-37`

```python
@dataclass
class VectorStoreSearchResult:
    """A vector storage search result."""

    document: VectorStoreDocument  # Retrieved document
    score: float                   # Similarity score [-1, 1], higher = more similar
```

**Example**:
```python
result = VectorStoreSearchResult(
    document=VectorStoreDocument(
        id="entity_OPENAI",
        text="OpenAI is an AI research lab...",
        vector=[...],
    ),
    score=0.91  # High cosine similarity
)
```

---

## Integration with GraphRAG Pipeline

### Indexing Flow

**Code**: `embed_text.py:127-209`

```python
async def _text_embed_with_vector_store(
    input: pd.DataFrame,
    embed_column: str,
    strategy: dict[str, Any],
    vector_store: BaseVectorStore,
    vector_store_config: dict,
    id_column: str = "id",
    title_column: str | None = None,
):
    """Embed text and store in vector database."""
    insert_batch_size = vector_store_config.get("batch_size", 500)
    overwrite = vector_store_config.get("overwrite", True)

    all_results = []
    i = 0

    # Process in batches
    while insert_batch_size * i < input.shape[0]:
        batch = input.iloc[insert_batch_size * i : insert_batch_size * (i + 1)]
        texts = batch[embed_column].to_numpy().tolist()
        titles = batch[title_column].to_numpy().tolist()
        ids = batch[id_column].to_numpy().tolist()

        # Embed batch
        result = await strategy_exec(texts, callbacks, cache, strategy_config)
        embeddings = result.embeddings

        # Create VectorStoreDocument objects
        documents = [
            VectorStoreDocument(
                id=doc_id,
                text=doc_text,
                vector=doc_vector.tolist() if isinstance(doc_vector, np.ndarray) else doc_vector,
                attributes={"title": doc_title},
            )
            for doc_id, doc_text, doc_title, doc_vector in zip(ids, texts, titles, embeddings)
        ]

        # Store in vector database
        vector_store.load_documents(documents, overwrite=(overwrite and i == 0))

        all_results.extend(embeddings)
        i += 1

    return all_results
```

**Flow**:
```
Entities DataFrame → Embed descriptions → Create VectorStoreDocuments → Store in LanceDB/Azure
```

**Collections** (separate indexes):
- `entity_description_embeddings`: Entity descriptions
- `text_unit_embeddings`: Text chunk embeddings
- `community_report_embeddings`: Community summary embeddings

---

## Connection to Graph Algorithms

### Graph Metadata in Vector Store

**Integration**: Store graph-derived attributes with embeddings

```python
# After computing graph centrality
entity = {
    "id": "MICROSOFT",
    "description": "Technology company...",
    "degree": 120,        # From spec/graph/02-centrality-and-ranking.md
    "community_id": 5,    # From spec/graph/01-community-detection.md
}

# Embed and store with metadata
vector_store.load_documents([
    VectorStoreDocument(
        id=entity["id"],
        text=entity["description"],
        vector=embed_text(entity["description"]),
        attributes={
            "degree": entity["degree"],        # Graph centrality
            "community_id": entity["community_id"],  # Clustering result
        }
    )
])
```

**Query-time use**: Filter by graph properties
```python
# Retrieve only high-centrality entities
vector_store.filter_by_attribute("degree > 50")
results = vector_store.similarity_search_by_vector(query_emb, k=10)
```

---

## Performance Characteristics

### Storage Size

| Data | Count | Size per Item | Total Size |
|------|-------|---------------|------------|
| Entities | 50K | 1536 × 4B + metadata (~6.5KB) | ~325 MB |
| Text units | 100K | 1536 × 4B (~6KB) | ~600 MB |
| Community reports | 500 | 1536 × 4B (~6KB) | ~3 MB |
| **Total** | **150.5K** | - | **~928 MB** |

**With HNSW index**: ~1.8 GB (2x for graph structure)

---

### Ingestion Performance

**LanceDB** (local, 10K documents):
```
Batch size: 500
Batches: 20
Time per batch: ~200ms (embedding) + ~50ms (storage)
Total: ~5 seconds (2K docs/sec)
```

**Azure AI Search** (cloud, 10K documents):
```
Batch size: 500
Batches: 20
Time per batch: ~200ms (embedding) + ~300ms (upload + index)
Total: ~10 seconds (1K docs/sec)
```

**Bottleneck**: Embedding API calls, not storage

---

### Query Performance

| Store | Index Type | 10K vectors | 1M vectors | 100M vectors |
|-------|-----------|-------------|------------|--------------|
| LanceDB | Auto ANN | 5ms | 20ms | N/A (memory limit) |
| Azure AI Search | HNSW | 10ms | 15ms | 50ms |
| Brute-force | None | 50ms | 5s | 500s |

**Recommendation**: Use ANN for corpora > 10K vectors

---

## Connection to Dependencies

### lancedb: Embedded Vector DB

**From** `spec/dependencies/02-graph-and-vector-storage.md`:

**Why LanceDB?**
- **Embedded**: No separate server (vs. Elasticsearch, Milvus)
- **Fast**: Rust-based, Apache Arrow format
- **Disk-efficient**: Columnar storage, 10x smaller than JSON

**Example**:
```python
import lancedb

db = lancedb.connect("./lancedb")  # Local directory
table = db.create_table("entities", data=[
    {"id": "MICROSOFT", "vector": [...], "text": "..."}
])

# Query
results = table.search([0.23, ...]).limit(10).to_list()
```

---

### Azure SDK: Cloud Integration

**From** `spec/dependencies/03-data-processing-and-infrastructure.md`:

**Azure AI Search integration**:
```python
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient

# Create index
index_client = SearchIndexClient(endpoint=url, credential=credential)
index_client.create_or_update_index(index_definition)

# Upload documents
search_client = SearchClient(endpoint=url, index_name=name, credential=credential)
search_client.upload_documents(documents)

# Search
results = search_client.search(
    search_text=None,
    vector_queries=[VectorizedQuery(vector=query_emb, k_nearest_neighbors=10)]
)
```

---

## Troubleshooting

### Problem: Vector Store Out of Memory

**Symptom**: `MemoryError` with LanceDB on large corpus

**Diagnosis**: Entire index loaded in memory

**Solution**:
1. **Switch to cloud store** (Azure AI Search handles memory externally)

2. **Shard data** (split into multiple tables):
   ```python
   # Shard by first letter
   for letter in "ABCDEFGHIJKLMNOPQRSTUVWXYZ":
       table_name = f"entities_{letter}"
       entities_subset = [e for e in entities if e.id.startswith(letter)]
       db.create_table(table_name, data=entities_subset)
   ```

3. **Use disk-based storage** (LanceDB supports memory-mapped files)

---

### Problem: Slow Index Build

**Symptom**: Hours to build HNSW index for 1M vectors

**Diagnosis**: Suboptimal ef_construction parameter

**Solution**:
1. **Reduce ef_construction**:
   ```yaml
   ef_construction: 100  # Down from 400 (faster build, slightly lower quality)
   ```

2. **Build incrementally** (add vectors in batches, index updates automatically)

---

## Conclusion

Vector storage provides the **persistent, scalable foundation** for GraphRAG's semantic retrieval capabilities. By using specialized vector databases (LanceDB, Azure AI Search) with ANN indexes (HNSW), GraphRAG achieves:
- **Fast similarity search**: O(log N) queries via graph-based indexes
- **Scalable storage**: Handle millions of high-dimensional vectors
- **Flexible deployment**: Embedded (LanceDB) for local, cloud (Azure) for production
- **Integration with graph**: Store centrality, communities as metadata for hybrid retrieval

**Key principles**:
- **Columnar storage**: Apache Arrow for efficient vector serialization
- **HNSW indexing**: Multi-layer navigable graph for ANN
- **Batched ingestion**: 500-doc batches optimize throughput
- **Metadata enrichment**: Store graph attributes alongside embeddings

**Key files**:
- `base.py:15-86` — VectorStore interface and data models
- `lancedb.py:21-154` — LanceDB embedded implementation
- `azure_ai_search.py:36-154` — Azure cloud implementation
