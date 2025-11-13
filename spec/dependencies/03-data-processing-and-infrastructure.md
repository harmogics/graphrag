# Data Processing and Infrastructure Dependencies

## Overview

This document describes dependencies used for data manipulation, serialization, configuration, async I/O, cloud storage, and CLI interactions in GraphRAG. These libraries form the **operational foundation**, enabling efficient data processing, configuration management, and infrastructure integration.

## Data Processing Libraries

### pandas

**Version**: `^2.2.3`

**Purpose**: Data analysis and manipulation library

**Role in GraphRAG**:
- **Dataframe operations**: Primary data structure for entities, relationships, text units
- **Data aggregation**: Group entities, merge relationships, compute statistics
- **CSV/Parquet I/O**: Read source documents, write processed results
- **Data transformation**: Filter, sort, pivot data throughout pipeline

**Architecture Integration**:
```
Raw Data (CSV/Parquet) → pandas DataFrames → Processing Operations → Output (CSV/Parquet)
```

**Key Features Used**:

**1. Entity Aggregation**:
```python
import pandas as pd

# Entities extracted from multiple text units
entities_df = pd.DataFrame([
    {'title': 'ALICE', 'type': 'PERSON', 'description': 'Engineer', 'source_id': 'doc_1'},
    {'title': 'ALICE', 'type': 'PERSON', 'description': 'Team lead', 'source_id': 'doc_2'},
    {'title': 'BOB', 'type': 'PERSON', 'description': 'Designer', 'source_id': 'doc_1'},
])

# Merge same entities from different sources
merged_entities = (
    entities_df.groupby(['title', 'type'], sort=False)
    .agg({
        'description': list,  # Collect all descriptions
        'source_id': list,    # Track all sources
        'source_id': 'count'  # Frequency = number of mentions
    })
    .rename(columns={'source_id': 'frequency'})
    .reset_index()
)

# Result:
# title   type    description                       source_id            frequency
# ALICE   PERSON  ['Engineer', 'Team lead']        ['doc_1', 'doc_2']   2
# BOB     PERSON  ['Designer']                     ['doc_1']            1
```

**2. Relationship Deduplication**:
```python
# Relationships may be extracted multiple times
relationships_df = pd.DataFrame([
    {'source': 'ALICE', 'target': 'BOB', 'description': 'colleagues', 'weight': 1.0},
    {'source': 'ALICE', 'target': 'BOB', 'description': 'work together', 'weight': 2.0},
])

# Merge by (source, target) pair
merged_relationships = (
    relationships_df.groupby(['source', 'target'], sort=False)
    .agg({
        'description': list,
        'weight': 'sum'  # Sum relationship weights
    })
    .reset_index()
)

# Result: Single relationship with combined weight
```

**3. Data Pipeline**:
```python
# Typical pandas pipeline
result = (
    df
    .query("type == 'PERSON'")           # Filter
    .sort_values('frequency', ascending=False)  # Sort
    .head(100)                           # Top-100
    .assign(rank=lambda x: range(1, len(x) + 1))  # Add rank column
)
```

**Code Locations**:
- `graphrag/index/operations/extract_graph/extract_graph.py` - Entity/relationship merging
- `graphrag/index/operations/*.py` - Most operations use pandas DataFrames
- `graphrag/query/context_builder/*.py` - Context data preparation

**Performance Characteristics**:
- **Speed**: 10-100M rows/second (operation-dependent)
- **Memory**: ~100 bytes per row (depends on column types)
- **Scalability**: Efficient up to ~10M rows in memory
- **Beyond**: Use Dask for larger datasets

**Why pandas**:
- **Industry standard**: Widely used, well-documented
- **Rich API**: Hundreds of data manipulation functions
- **Integration**: Works seamlessly with numpy, pyarrow, parquet
- **Performance**: C-optimized core operations

---

### pyarrow

**Version**: `^15.0.0`

**Purpose**: Apache Arrow columnar memory format and Parquet file support

**Role in GraphRAG**:
- **Parquet I/O**: Read/write compressed columnar data files
- **Zero-copy**: Fast data sharing between pandas and vector stores
- **Memory efficiency**: Columnar format reduces memory footprint
- **Interoperability**: Standard format across tools (Spark, DuckDB, etc.)

**Architecture Integration**:
```
Source Data (Parquet) → pyarrow → pandas DataFrames → Processing → pyarrow → Output (Parquet)
```

**Key Features Used**:

**1. Parquet Reading**:
```python
import pyarrow.parquet as pq
import pandas as pd

# Read parquet file (compressed columnar format)
table = pq.read_table('entities.parquet')

# Convert to pandas for processing
df = table.to_pandas()

# Parquet advantages:
# - 10x smaller than CSV (compression + columnar)
# - 5x faster to read (no parsing, binary format)
# - Preserves types (no int→float conversion issues)
```

**2. Parquet Writing**:
```python
# Write pandas DataFrame to parquet
df.to_parquet(
    'output/entities.parquet',
    engine='pyarrow',
    compression='snappy',  # Fast compression (vs gzip: slower but smaller)
    index=False
)

# File size comparison (100K entities):
# CSV: 50MB
# Parquet (snappy): 5MB  (10x reduction)
# Parquet (gzip): 3MB    (17x reduction, slower read)
```

**3. Schema Evolution**:
```python
# Parquet supports schema evolution
# Add new column without rewriting entire dataset
import pyarrow as pa

# Define schema with new column
schema = pa.schema([
    ('title', pa.string()),
    ('type', pa.string()),
    ('description', pa.string()),
    ('embedding', pa.list_(pa.float32()))  # New: embedding vector
])

# Write with explicit schema
pq.write_table(table, 'entities_v2.parquet', schema=schema)
```

**Code Locations**:
- `graphrag/index/storage/*.py` - Parquet file I/O
- `graphrag/index/config/storage.py` - Storage format configuration

**Why Parquet**:
- **Compression**: 10-20x smaller than CSV
- **Speed**: Faster read/write than CSV
- **Type safety**: Preserves data types (int, float, datetime, etc.)
- **Columnar**: Only read columns needed (not entire row)
- **Ecosystem**: Standard format for big data tools

**Performance Characteristics**:
```
Dataset: 1M entities with 1536D embeddings

CSV:
- File size: 9GB
- Write time: 5 minutes
- Read time: 3 minutes

Parquet (snappy):
- File size: 600MB  (15x smaller)
- Write time: 1 minute  (5x faster)
- Read time: 10 seconds  (18x faster)
```

---

### numpy

**Version**: `^1.25.2`

**Purpose**: Numerical computing library

**Role in GraphRAG**:
- **Array operations**: Efficient manipulation of embedding vectors
- **Linear algebra**: Vector similarity computations (dot product, cosine)
- **Statistics**: Compute means, std dev, percentiles
- **Performance**: C-optimized numerical operations

**Architecture Integration**:
```
Embeddings (lists) → numpy arrays → Vectorized operations → Results
```

**Key Features Used**:

**1. Vector Similarity**:
```python
import numpy as np

# Query and entity embeddings
query_vec = np.array([0.1, -0.2, 0.3, ..., 0.5])  # 1536D
entity_vecs = np.array([
    [0.12, -0.18, 0.31, ..., 0.48],  # Entity 1
    [0.05, -0.15, 0.25, ..., 0.52],  # Entity 2
    # ... thousands of entities
])

# Cosine similarity (vectorized)
def cosine_similarity_batch(query, entities):
    # Normalize vectors
    query_norm = query / np.linalg.norm(query)
    entities_norm = entities / np.linalg.norm(entities, axis=1, keepdims=True)

    # Dot product (cosine of angle)
    similarities = entities_norm @ query_norm  # Matrix-vector multiplication

    return similarities

sims = cosine_similarity_batch(query_vec, entity_vecs)
# Array of similarities: [0.85, 0.72, 0.91, ...]

# Top-k most similar
top_k_indices = np.argsort(sims)[-10:][::-1]  # Top-10, descending
```

**2. Embedding Statistics**:
```python
# Analyze embedding distribution
embeddings = np.array([emb for emb in all_embeddings])  # (N, 1536)

# Compute statistics
mean_vec = np.mean(embeddings, axis=0)       # Average embedding
std_vec = np.std(embeddings, axis=0)         # Standard deviation per dimension
norms = np.linalg.norm(embeddings, axis=1)   # Vector magnitudes

print(f"Mean norm: {np.mean(norms):.4f}")
print(f"Std norm: {np.std(norms):.4f}")
# Check if embeddings are normalized: mean_norm ≈ 1.0, std_norm ≈ 0
```

**3. Dimensionality Reduction Prep**:
```python
# Prepare data for UMAP, t-SNE
from sklearn.decomposition import PCA

# High-dimensional embeddings
embeddings = np.array(all_embeddings)  # (10000, 1536)

# PCA: reduce 1536D → 128D (preprocessing for UMAP)
pca = PCA(n_components=128)
embeddings_reduced = pca.fit_transform(embeddings)  # (10000, 128)

# Explained variance
print(f"Variance retained: {sum(pca.explained_variance_ratio_):.2%}")
# Typically 80-90% with 128 components
```

**Code Locations**:
- `graphrag/query/context_builder/entity_extraction.py` - Similarity computations
- `graphrag/index/operations/embed_graph/*.py` - Embedding normalization

**Performance**:
- **Vector ops**: 10-100x faster than pure Python
- **Memory**: Contiguous arrays (no Python object overhead)
- **Parallelism**: BLAS/LAPACK auto-parallelization

---

## Configuration and Environment

### pydantic

**Version**: `^2.10.3`

**Purpose**: Data validation and settings management using Python type hints

**Role in GraphRAG**:
- **Config validation**: Ensure configuration files have correct structure/types
- **Type safety**: Catch configuration errors at load time (not runtime)
- **Defaults**: Provide sensible default values
- **Documentation**: Auto-generate config schemas

**Architecture Integration**:
```
YAML/JSON config → pydantic models → Validated config objects → GraphRAG components
```

**Key Features Used**:

**1. Configuration Models**:
```python
from pydantic import BaseModel, Field, field_validator

class LLMConfig(BaseModel):
    """LLM configuration with validation."""
    type: str = "openai_chat"
    model: str = "gpt-4-turbo"
    max_tokens: int = Field(default=2000, ge=1, le=128000)  # Must be 1-128000
    temperature: float = Field(default=0.0, ge=0.0, le=2.0)  # Must be 0.0-2.0
    api_key: str = Field(default="", min_length=1)  # Required, non-empty

    @field_validator('type')
    @classmethod
    def validate_type(cls, v):
        allowed = ['openai_chat', 'azure_openai_chat']
        if v not in allowed:
            raise ValueError(f"type must be one of {allowed}, got {v}")
        return v

# Load and validate config
config = LLMConfig(
    model="gpt-4",
    max_tokens=4000,
    api_key="sk-..."
)

# Type error caught immediately:
try:
    bad_config = LLMConfig(max_tokens="many")  # TypeError!
except Exception as e:
    print(f"Validation error: {e}")
```

**2. Nested Configuration**:
```python
class EmbeddingConfig(BaseModel):
    model: str = "text-embedding-3-small"
    dimensions: int = 1536
    batch_size: int = 100

class GraphRAGConfig(BaseModel):
    llm: LLMConfig
    embedding: EmbeddingConfig
    max_cluster_size: int = 10
    chunk_size: int = 300

# Hierarchical validation
full_config = GraphRAGConfig(
    llm=LLMConfig(...),
    embedding=EmbeddingConfig(...)
)
```

**3. Environment Variable Integration**:
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    """Load settings from environment variables."""
    openai_api_key: str
    azure_endpoint: str = ""
    log_level: str = "INFO"

    class Config:
        env_prefix = "GRAPHRAG_"  # Look for GRAPHRAG_OPENAI_API_KEY, etc.
        env_file = ".env"         # Load from .env file

settings = Settings()  # Automatically loads from env vars
```

**Code Locations**:
- `graphrag/config/models/*.py` - All configuration model definitions
- `graphrag/config/config.py` - Main config loading logic

**Why pydantic**:
- **Type safety**: Catch errors early (config loading, not runtime)
- **Validation**: Rich validation rules (ranges, regex, custom)
- **IDE support**: Type hints enable autocomplete
- **JSON schema**: Auto-generate documentation from models

---

### pyyaml

**Version**: `^6.0.2`

**Purpose**: YAML parser and emitter

**Role in GraphRAG**:
- **Config files**: Primary format for GraphRAG configuration
- **Readable**: YAML more human-friendly than JSON
- **Comments**: Supports comments in config files
- **Complex structures**: Handle nested, multi-document configs

**Usage Example**:
```python
import yaml

# Load YAML config
with open('config.yaml', 'r') as f:
    raw_config = yaml.safe_load(f)

# Example config.yaml:
"""
llm:
  type: openai_chat
  model: gpt-4-turbo
  max_tokens: 2000
  temperature: 0.0

embedding:
  model: text-embedding-3-small
  dimensions: 1536

chunking:
  size: 300
  overlap: 100
"""

# Access values
llm_model = raw_config['llm']['model']  # 'gpt-4-turbo'
```

**Code Locations**:
- `graphrag/config/config.py` - YAML config loading
- User-facing config files: `settings.yaml`

---

### environs

**Version**: `^11.0.0`

**Purpose**: Environment variable parsing and validation

**Role in GraphRAG**:
- **12-factor app**: Configuration via environment variables
- **Type casting**: Parse env vars as int, bool, list, etc.
- **Validation**: Ensure required env vars are set
- **Defaults**: Fallback values if env var not set

**Usage Example**:
```python
from environs import Env

env = Env()
env.read_env()  # Load from .env file

# Parse typed environment variables
OPENAI_API_KEY = env.str("OPENAI_API_KEY")  # Required string
AZURE_ENDPOINT = env.str("AZURE_ENDPOINT", default="")  # Optional
MAX_WORKERS = env.int("MAX_WORKERS", default=4)  # Parse as int
ENABLE_CACHING = env.bool("ENABLE_CACHING", default=True)  # Parse as bool
ALLOWED_MODELS = env.list("ALLOWED_MODELS", default=["gpt-4", "gpt-3.5-turbo"])  # Parse as list
```

**Code Locations**:
- `graphrag/config/*.py` - Environment-based configuration loading

---

## Async I/O and Concurrency

### aiofiles

**Version**: `^24.1.0`

**Purpose**: Async file I/O operations

**Role in GraphRAG**:
- **Non-blocking file reads**: Don't block event loop during I/O
- **Concurrent file access**: Read multiple files simultaneously
- **Async/await integration**: Works with asyncio ecosystem

**Architecture Integration**:
```
Async Pipeline → aiofiles (read) → Process → aiofiles (write) → Output
```

**Usage Example**:
```python
import aiofiles
import asyncio

# Async file reading
async def load_document(file_path: str) -> str:
    async with aiofiles.open(file_path, 'r', encoding='utf-8') as f:
        content = await f.read()
    return content

# Concurrent document loading
async def load_documents_batch(file_paths: list[str]) -> list[str]:
    tasks = [load_document(path) for path in file_paths]
    documents = await asyncio.gather(*tasks)
    return documents

# Load 1000 files concurrently (vs sequential: 100x faster)
docs = await load_documents_batch(file_paths)
```

**Code Locations**:
- `graphrag/index/operations/*.py` - Async file I/O in indexing pipeline
- Document loading operations

**Why async file I/O**:
- **Throughput**: Load 100+ files concurrently
- **Responsiveness**: Don't block on slow disk I/O
- **Scalability**: Handle thousands of small files efficiently

---

## Azure Cloud Integration

### azure-identity

**Version**: `^1.19.0`

**Purpose**: Azure authentication and credential management

**Role in GraphRAG**:
- **AAD authentication**: Authenticate to Azure services
- **Managed identity**: Use Azure VM/container identity (no keys)
- **DefaultAzureCredential**: Auto-detect best auth method
- **Token management**: Handle token refresh automatically

**Usage Example**:
```python
from azure.identity import DefaultAzureCredential

# Tries multiple auth methods in order:
# 1. Environment variables (AZURE_CLIENT_ID, etc.)
# 2. Managed Identity (if running on Azure)
# 3. Azure CLI (if logged in)
# 4. Interactive browser (as fallback)
credential = DefaultAzureCredential()

# Use with Azure services
from azure.storage.blob import BlobServiceClient

blob_client = BlobServiceClient(
    account_url="https://account.blob.core.windows.net",
    credential=credential  # Automatic token management
)
```

**Code Locations**:
- `graphrag/index/storage/azure_blob_storage.py` - Azure Blob authentication
- `graphrag/vector_stores/azure_ai_search.py` - Azure Search authentication

**Why DefaultAzureCredential**:
- **Flexible**: Works in dev (CLI) and prod (Managed Identity)
- **Secure**: No hardcoded credentials
- **Automatic**: Handles token refresh, retry logic

---

### azure-storage-blob

**Version**: `^12.24.0`

**Purpose**: Azure Blob Storage SDK

**Role in GraphRAG**:
- **Cloud storage**: Store/retrieve documents, parquet files
- **Scalability**: Handle TB+ of data
- **Durability**: 99.999999999% (11 9's) durability
- **Integration**: Works with pandas, parquet

**Architecture Integration**:
```
Source Documents (Azure Blob) → Download → Process → Upload (Azure Blob) → Results
```

**Usage Example**:
```python
from azure.storage.blob import BlobServiceClient

# Connect to Azure Storage
blob_service = BlobServiceClient(
    account_url="https://account.blob.core.windows.net",
    credential=credential
)

# Upload file
container = blob_service.get_container_client("graphrag-data")
with open("entities.parquet", "rb") as data:
    container.upload_blob(name="output/entities.parquet", data=data, overwrite=True)

# Download file
blob_client = container.get_blob_client("input/documents.csv")
with open("documents.csv", "wb") as f:
    download_stream = blob_client.download_blob()
    f.write(download_stream.readall())

# List files
blobs = container.list_blobs(name_starts_with="output/")
for blob in blobs:
    print(f"File: {blob.name}, Size: {blob.size} bytes")
```

**Code Locations**:
- `graphrag/index/storage/azure_blob_storage.py` - Blob storage operations

**Performance Tips**:
```python
# Parallel upload for large files
blob_client.upload_blob(
    data,
    overwrite=True,
    max_concurrency=4  # Upload chunks in parallel
)

# Batch operations
from azure.storage.blob import BlobBatch

batch = container.get_blob_batch_client()
batch.delete_blobs(*blob_list)  # Delete multiple blobs efficiently
```

---

### azure-cosmos

**Version**: `^4.9.0`

**Purpose**: Azure Cosmos DB SDK (NoSQL database)

**Role in GraphRAG**:
- **Metadata storage**: Store entity metadata, provenance
- **Query history**: Track user queries and results
- **Configuration**: Store dynamic configuration
- **State management**: Persist pipeline state for resumption

**Usage Example**:
```python
from azure.cosmos import CosmosClient

# Connect to Cosmos DB
client = CosmosClient(url=cosmos_url, credential=credential)
database = client.get_database_client("graphrag")
container = database.get_container_client("entities")

# Insert entity metadata
entity_item = {
    'id': 'entity_001',
    'title': 'GRAPHRAG',
    'type': 'SYSTEM',
    'description': 'Knowledge extraction system',
    'extracted_at': '2024-01-15T10:30:00Z',
    'source_docs': ['doc_1', 'doc_2']
}
container.create_item(body=entity_item)

# Query entities
query = "SELECT * FROM c WHERE c.type = @type"
parameters = [{"name": "@type", "value": "PERSON"}]
items = list(container.query_items(
    query=query,
    parameters=parameters,
    enable_cross_partition_query=True
))
```

**Code Locations**:
- `graphrag/index/storage/azure_cosmos_storage.py` - Cosmos DB operations (if used)

**Why Cosmos DB**:
- **Global distribution**: Multi-region replication
- **Low latency**: <10ms reads/writes at P99
- **Flexible schema**: JSON documents, no rigid schema
- **Multi-model**: Supports Graph API, MongoDB API, etc.

---

## CLI and User Interface

### typer

**Version**: `^0.15.1`

**Purpose**: CLI framework built on Click

**Role in GraphRAG**:
- **Command-line interface**: `graphrag index`, `graphrag query`, etc.
- **Type hints**: Use Python type hints for argument validation
- **Help generation**: Auto-generate help text from docstrings
- **Subcommands**: Organize related commands

**Architecture Integration**:
```
User command → typer → Parse args → Execute GraphRAG operation
```

**Usage Example**:
```python
import typer
from typing import Optional

app = typer.Typer()

@app.command()
def index(
    config: str = typer.Option("settings.yaml", help="Configuration file path"),
    root: str = typer.Option(".", help="Project root directory"),
    verbose: bool = typer.Option(False, "--verbose", "-v", help="Verbose output"),
    resume: Optional[str] = typer.Option(None, help="Resume from timestamp"),
):
    """
    Build knowledge graph index from source documents.

    This command processes all documents in the input directory,
    extracts entities and relationships, and builds a searchable index.
    """
    if verbose:
        typer.echo("Verbose mode enabled")

    typer.echo(f"Loading config from {config}")
    # ... indexing logic ...
    typer.secho("✓ Indexing complete!", fg=typer.colors.GREEN, bold=True)

# CLI usage:
# $ graphrag index --config settings.yaml --verbose
# $ graphrag index --help  (auto-generated help)
```

**Code Locations**:
- `graphrag/cli/main.py` - Main CLI entry point
- `graphrag/cli/*.py` - Individual command implementations

**Why typer**:
- **Type-safe**: Uses Python type hints (no manual parsing)
- **User-friendly**: Beautiful help text, colors, progress bars
- **Composable**: Easy to add new commands, options
- **Testing**: CLI commands are just Python functions (easy to test)

---

### rich

**Version**: `^13.9.4`

**Purpose**: Rich text and beautiful formatting in terminal

**Role in GraphRAG**:
- **Progress bars**: Show indexing/query progress
- **Tables**: Display results in formatted tables
- **Syntax highlighting**: Color-code output for readability
- **Panels**: Organize output into visual sections

**Usage Example**:
```python
from rich.console import Console
from rich.table import Table
from rich.progress import Progress, SpinnerColumn, TextColumn

console = Console()

# Pretty-print data structures
console.print({"status": "success", "entities": 1234}, style="bold green")

# Tables
table = Table(title="Top Entities")
table.add_column("Entity", style="cyan")
table.add_column("Type", style="magenta")
table.add_column("Frequency", justify="right", style="green")

table.add_row("GRAPHRAG", "SYSTEM", "42")
table.add_row("MICROSOFT", "ORG", "38")
console.print(table)

# Progress bars
with Progress(
    SpinnerColumn(),
    TextColumn("[progress.description]{task.description}"),
) as progress:
    task = progress.add_task("Indexing documents...", total=1000)
    for i in range(1000):
        # ... process document ...
        progress.update(task, advance=1)
```

**Code Locations**:
- `graphrag/cli/*.py` - CLI output formatting
- Progress tracking in long-running operations

---

### tqdm

**Version**: `^4.67.1`

**Purpose**: Progress bar library

**Role in GraphRAG**:
- **Progress tracking**: Show completion status for loops
- **ETA estimation**: Estimate time remaining
- **Simple API**: Wrap any iterable with progress bar

**Usage Example**:
```python
from tqdm import tqdm
import asyncio

# Synchronous progress
documents = load_documents()
for doc in tqdm(documents, desc="Processing documents"):
    process_document(doc)

# Async progress
from tqdm.asyncio import tqdm_asyncio

async def process_docs_async(docs):
    tasks = [process_doc_async(doc) for doc in docs]
    results = await tqdm_asyncio.gather(*tasks, desc="Processing")
    return results
```

**Code Locations**:
- `graphrag/index/operations/*.py` - Progress tracking in operations
- `graphrag/query/structured_search/drift_search/search.py` - DRIFT query progress

---

## Development and Debugging

### devtools

**Version**: `^0.12.2`

**Purpose**: Python development tools for debugging

**Role in GraphRAG**:
- **Pretty printing**: Enhanced debug output
- **Performance profiling**: Identify bottlenecks
- **Object inspection**: Examine object state

**Usage Example**:
```python
from devtools import debug

# Enhanced print debugging
entity_graph = build_graph(entities, relationships)
debug(entity_graph)  # Shows detailed object structure

# Output:
# entity_graph: nx.Graph (
#     nodes=1234,
#     edges=5678,
#     density=0.0074,
#     connected_components=12
# )
```

**Code Locations**:
- Development/debugging sessions (not in production code)

---

## Dependency Interaction Diagram

```
                    User (CLI)
                        |
                        v
                   typer (parse)
                        |
            +-----------+-----------+
            |                       |
            v                       v
      rich (display)          tqdm (progress)
            |                       |
            +-----------+-----------+
                        |
                        v
                 Configuration
                        |
        +---------------+---------------+
        |                               |
        v                               v
pydantic (validate)              environs (env vars)
        |                               |
        +---------------+---------------+
                        |
                        v
                  GraphRAG Pipeline
                        |
        +---------------+---------------+
        |               |               |
        v               v               v
   pandas (data)   aiofiles (I/O)   numpy (math)
        |               |               |
        v               v               v
   pyarrow          Azure Blob      Embeddings
    (parquet)       (storage)       (vectors)
        |               |               |
        +---------------+---------------+
                        |
                        v
                     Output
```

---

## Performance Optimization

### Pandas Optimization

**1. Use Categorical Types**:
```python
# Bad: Store entity types as strings (8+ bytes each)
df['type'] = df['type'].astype(str)

# Good: Use categorical (saves 50-90% memory)
df['type'] = df['type'].astype('category')
# Only stores type once + indices (1-2 bytes each)
```

**2. Avoid Iterrows**:
```python
# Bad: Iterate rows (1000x slower)
for index, row in df.iterrows():
    process(row['value'])

# Good: Vectorized operations
df['result'] = df['value'].apply(process)

# Better: Numpy vectorized (if possible)
df['result'] = numpy_process(df['value'].values)
```

### Async Optimization

**1. Batch Async Operations**:
```python
# Good: Batch async calls
batch_size = 100
for i in range(0, len(items), batch_size):
    batch = items[i:i+batch_size]
    results = await asyncio.gather(*[process(item) for item in batch])
    save_results(results)
```

**2. Limit Concurrency**:
```python
# Prevent overwhelming resources
semaphore = asyncio.Semaphore(10)  # Max 10 concurrent

async def limited_process(item):
    async with semaphore:
        return await process(item)
```

---

## Conclusion

Data processing and infrastructure dependencies provide GraphRAG's **operational foundation**:

**Data Layer**:
- **pandas**: Flexible data manipulation
- **pyarrow**: Efficient columnar storage
- **numpy**: High-performance numerical computing

**Configuration Layer**:
- **pydantic**: Type-safe configuration
- **pyyaml**: Human-readable config files
- **environs**: Environment-based settings

**Cloud Layer**:
- **azure-identity**: Secure authentication
- **azure-storage-blob**: Scalable file storage
- **azure-cosmos**: NoSQL metadata storage

**Interface Layer**:
- **typer**: User-friendly CLI
- **rich**: Beautiful terminal output
- **tqdm**: Progress tracking

**Async Layer**:
- **aiofiles**: Non-blocking file I/O
- **asyncio**: Concurrent operations

Together, these libraries enable GraphRAG to efficiently process large datasets, integrate with cloud infrastructure, and provide a polished user experience—from configuration to execution to results presentation.
