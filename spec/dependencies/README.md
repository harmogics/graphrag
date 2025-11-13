# GraphRAG Dependencies Documentation

## Overview

This directory contains comprehensive documentation of all external libraries and packages used in GraphRAG, organized by their role in information processing. Understanding these dependencies is crucial for:

- **System architecture**: How components interact
- **Performance optimization**: Bottlenecks and trade-offs
- **Deployment planning**: Infrastructure requirements
- **Troubleshooting**: Debugging dependency-related issues
- **Alternatives evaluation**: When to consider different libraries

## Document Structure

### [01-llm-and-language-processing.md](01-llm-and-language-processing.md)

**Semantic Awareness Layer**

Covers dependencies that enable GraphRAG to understand and generate natural language:

**Core LLM Infrastructure**:
- **fnllm** (`0.2.3`): Unified LLM interface abstraction
- **openai** (`^1.57.0`): OpenAI SDK for chat completions and embeddings
- **tiktoken** (`^0.8.0`): BPE tokenizer for token counting and budgeting
- **json-repair** (`^0.30.3`): Fix malformed JSON from LLM outputs

**Natural Language Processing**:
- **nltk** (`3.9.1`): Sentence splitting, tokenization
- **spacy** (`^3.8.4`): Named Entity Recognition, POS tagging
- **textblob** (`^0.18.0`): Sentiment analysis, spelling correction

**Key Insights**:
- LLM awareness is **engineered** through prompting (few-shot, gleaning, RGC structure)
- Token budget management critical for cost/performance
- Embedding model choice affects downstream quality (1536D vs 3072D)

---

### [02-graph-and-vector-storage.md](02-graph-and-vector-storage.md)

**Structural Organization Layer**

Covers dependencies for graph construction, community detection, and vector similarity search:

**Graph Processing**:
- **networkx** (`^3.4.2`): Graph creation, manipulation, algorithms
- **graspologic** (`^3.4.1`): Hierarchical Leiden clustering for communities
- **umap-learn** (`^0.5.6`): Dimensionality reduction for visualization

**Vector Storage**:
- **lancedb** (`^0.17.0`): Embedded vector database (development/small deployments)
- **azure-search-documents** (`^11.5.2`): Cloud vector search (production/large deployments)

**Key Insights**:
- Graph provides **semantic structure** (entities, relationships, communities)
- Vector layer enables **semantic search** (similarity matching)
- Hierarchical communities support multi-scale queries (Global vs Local)
- Choice between LanceDB (embedded) and Azure AI Search (cloud) depends on scale/budget

---

### [03-data-processing-and-infrastructure.md](03-data-processing-and-infrastructure.md)

**Operational Foundation Layer**

Covers dependencies for data manipulation, configuration, async I/O, and cloud integration:

**Data Processing**:
- **pandas** (`^2.2.3`): DataFrame operations for entities, relationships
- **pyarrow** (`^15.0.0`): Parquet I/O for compressed columnar storage
- **numpy** (`^1.25.2`): Numerical computing for embedding operations

**Configuration**:
- **pydantic** (`^2.10.3`): Type-safe configuration validation
- **pyyaml** (`^6.0.2`): YAML config file parsing
- **environs** (`^11.0.0`): Environment variable parsing

**Azure Cloud**:
- **azure-identity** (`^1.19.0`): AAD authentication
- **azure-storage-blob** (`^12.24.0`): Blob storage for documents/results
- **azure-cosmos** (`^4.9.0`): NoSQL database for metadata

**CLI Interface**:
- **typer** (`^0.15.1`): CLI framework with type hints
- **rich** (`^13.9.4`): Beautiful terminal formatting
- **tqdm** (`^4.67.1`): Progress bars

**Async I/O**:
- **aiofiles** (`^24.1.0`): Non-blocking file operations

**Key Insights**:
- Parquet format provides **10-20x compression** over CSV with faster I/O
- Pydantic catches configuration errors **at load time**, not runtime
- Azure integration enables **cloud-scale deployments** with managed infrastructure
- Async I/O crucial for **concurrent operations** (parallel LLM calls, file loading)

---

## Dependency Categories by Information Processing Aspect

### 1. Semantic Understanding

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| nltk | Sentence splitting | Text preprocessing |
| spacy | NER, POS tagging | Entity candidate generation |
| textblob | Sentiment, correction | Text quality analysis |
| fnllm | LLM interaction | Entity extraction, Q&A |
| openai | Embeddings, chat | Encoding, reasoning |
| tiktoken | Token counting | Budget management |
| json-repair | JSON parsing | Output validation |

### 2. Structural Organization

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| networkx | Graph construction | Entity graph building |
| graspologic | Community detection | Hierarchical clustering |
| lancedb | Vector storage | Embedding persistence |
| azure-search-documents | Vector search | Similarity matching |
| umap-learn | Dimensionality reduction | Visualization |

### 3. Data Operations

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| pandas | DataFrame ops | Entity/relationship aggregation |
| numpy | Numerical computing | Vector similarity |
| pyarrow | Parquet I/O | Data persistence |

### 4. Configuration & Infrastructure

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| pydantic | Config validation | System initialization |
| pyyaml | YAML parsing | Config loading |
| environs | Env var parsing | Runtime configuration |
| azure-identity | Authentication | Cloud access |
| azure-storage-blob | Blob storage | Data I/O |
| azure-cosmos | NoSQL storage | Metadata persistence |

### 5. User Interface

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| typer | CLI framework | Command parsing |
| rich | Terminal formatting | Output display |
| tqdm | Progress tracking | User feedback |

### 6. Concurrency

| Dependency | Role | Processing Stage |
|------------|------|------------------|
| aiofiles | Async file I/O | Concurrent file ops |
| asyncio (stdlib) | Event loop | Parallel LLM calls |

---

## Complete Dependency Map

```
┌─────────────────────────────────────────────────────────────┐
│                         USER (CLI)                           │
└───────────────────────┬─────────────────────────────────────┘
                        │
                ┌───────┴───────┐
                │  typer, rich  │ (CLI Interface)
                └───────┬───────┘
                        │
        ┌───────────────┼───────────────┐
        │                               │
┌───────▼────────┐             ┌───────▼────────┐
│ Configuration  │             │  Data Sources  │
│ pydantic       │             │  aiofiles      │
│ pyyaml         │             │  azure-blob    │
│ environs       │             └───────┬────────┘
└───────┬────────┘                     │
        │                               │
        └───────────────┬───────────────┘
                        │
        ┌───────────────▼────────────────┐
        │    Text Preprocessing          │
        │    nltk (sentences)            │
        │    spacy (NER)                 │
        │    textblob (sentiment)        │
        └───────────────┬────────────────┘
                        │
        ┌───────────────▼────────────────┐
        │    Semantic Encoding           │
        │    fnllm (LLM interface)       │
        │    openai (embeddings, chat)   │
        │    tiktoken (token counting)   │
        │    json-repair (parsing)       │
        └───────────────┬────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │                                │
┌───────▼────────┐             ┌────────▼────────┐
│ Entity Graph   │             │ Text Embeddings │
│ networkx       │             │ numpy (arrays)  │
│ graspologic    │             └────────┬────────┘
│ (clustering)   │                      │
└───────┬────────┘             ┌────────▼────────┐
        │                      │ Vector Storage  │
        │                      │ lancedb / azure │
        │                      └────────┬────────┘
        │                               │
┌───────▼───────────────────────────────▼────────┐
│         Query Processing                       │
│         - Global Search (map-reduce)           │
│         - Local Search (entity-centric)        │
│         - DRIFT Search (multi-hop)             │
└───────────────────┬────────────────────────────┘
                    │
        ┌───────────▼────────────┐
        │  Data Persistence      │
        │  pandas (DataFrames)   │
        │  pyarrow (Parquet)     │
        │  azure-cosmos (NoSQL)  │
        └────────────────────────┘
```

---

## Version Compatibility Matrix

| GraphRAG | Python | fnllm | openai | networkx | graspologic | lancedb | pandas |
|----------|--------|-------|--------|----------|-------------|---------|--------|
| 2.1.0    | 3.10-3.12 | 0.2.3 | 1.57+ | 3.4+ | 3.4+ | 0.17+ | 2.2+ |
| 2.0.x    | 3.10-3.12 | 0.2.x | 1.50+ | 3.3+ | 3.4+ | 0.16+ | 2.1+ |
| 1.x      | 3.10-3.11 | N/A | 1.30+ | 3.2+ | 3.3+ | N/A | 2.0+ |

**Breaking Changes**:
- **fnllm 0.2.0**: Introduced unified interface (replaced direct OpenAI calls)
- **openai 1.0.0**: Major API redesign (async methods, new structure)
- **pandas 2.0.0**: PyArrow backend, nullable dtypes by default
- **lancedb 0.17.0**: New index types (IVF_PQ optimization)

---

## Installation Patterns

### Minimal Installation (Development)

```bash
# Core dependencies only
pip install fnllm==0.2.3 openai==1.57.0 tiktoken==0.8.0 \
            networkx==3.4.2 pandas==2.2.3 numpy==1.25.2
```

**Use case**: Local development, small datasets (<10K entities)

---

### Standard Installation (Production - Local)

```bash
# Install from pyproject.toml
poetry install

# Or with pip
pip install graphrag[local]
```

**Includes**:
- All core dependencies
- LanceDB for vector storage
- Local file storage only
- No Azure dependencies

---

### Cloud Installation (Production - Azure)

```bash
# Full Azure integration
pip install graphrag[azure]
```

**Includes**:
- All core dependencies
- Azure SDK packages (identity, blob, cosmos, search)
- Cloud-scale vector search
- Managed infrastructure

---

### Development Installation

```bash
# Include dev tools
poetry install --with dev
```

**Additional dev dependencies**:
- pytest, pytest-asyncio (testing)
- ruff, pyright (linting, type checking)
- mkdocs-material (documentation)
- jupyter (notebooks)

---

## Performance Characteristics

### Computational Bottlenecks

**1. LLM API Calls** (slowest):
- Latency: 500ms - 5s per call
- Mitigation: Parallel calls via `asyncio.gather`, caching

**2. Vector Similarity Search**:
- Latency: <10ms with index, ~1s without (brute force)
- Mitigation: Build ANN index (HNSW, IVF_PQ)

**3. Community Detection** (Leiden):
- Time: O(N log N) for N nodes
- Mitigation: Use largest connected component, prune weak edges

**4. Embedding Generation**:
- Throughput: ~1K texts/second (batch API)
- Mitigation: Batch requests (100-1000 per call)

**5. Parquet I/O**:
- Speed: 10x faster than CSV
- Mitigation: Use pyarrow engine, snappy compression

### Memory Usage

```
Component                  Memory per Item
-----------------------------------------
Entity (in DataFrame)      ~500 bytes
Relationship (in DataFrame) ~300 bytes
Text Unit (with embedding) ~8KB (1536D float32)
networkx Graph             ~150 bytes/node + 50 bytes/edge
LanceDB Index              ~6KB per 1536D vector
```

**Example**:
```
100K entities, 500K relationships, 1M text units:
- Entities: 50MB
- Relationships: 150MB
- Text embeddings: 8GB
- Graph: 40MB
- Total: ~8.3GB
```

---

## Troubleshooting Guide

### Common Dependency Issues

**Issue**: `ModuleNotFoundError: No module named 'fnllm'`

**Solution**:
```bash
pip install fnllm[azure,openai]==0.2.3
```

---

**Issue**: `ImportError: cannot import name 'hierarchical_leiden' from 'graspologic.partition'`

**Solution**:
```bash
# graspologic requires future package
pip install future==1.0.0
pip install graspologic==3.4.1
```

---

**Issue**: spaCy model not found

**Solution**:
```python
import spacy.cli
spacy.cli.download("en_core_web_sm")
```

---

**Issue**: NLTK data not found

**Solution**:
```python
import nltk
nltk.download('punkt')
nltk.download('averaged_perceptron_tagger')
```

---

**Issue**: LanceDB vector dimension mismatch

**Solution**:
```python
# Ensure query and stored embeddings have same dimensions
assert query_embedding.shape[0] == 1536  # text-embedding-3-small
assert all(emb.shape[0] == 1536 for emb in stored_embeddings)
```

---

**Issue**: Pandas `SettingWithCopyWarning`

**Solution**:
```python
# Bad: Chained assignment
df[df['type'] == 'PERSON']['score'] = 1.0  # Warning!

# Good: Use .loc
df.loc[df['type'] == 'PERSON', 'score'] = 1.0
```

---

## Alternative Dependencies

### When to Consider Alternatives

**LLM Providers**:
- **Current**: OpenAI (GPT-4, GPT-3.5)
- **Alternative**: Anthropic Claude, Google Gemini, local Llama
- **Trade-off**: Quality vs cost vs privacy

**Vector Stores**:
- **Current**: LanceDB (local), Azure AI Search (cloud)
- **Alternative**: Pinecone, Weaviate, Milvus, Qdrant
- **Trade-off**: Features vs cost vs scalability

**Graph Libraries**:
- **Current**: networkx (pure Python)
- **Alternative**: igraph (C-based, faster), Neo4j (database)
- **Trade-off**: Ease of use vs performance vs features

**Data Processing**:
- **Current**: pandas (in-memory)
- **Alternative**: Dask (distributed), Polars (faster), Spark (big data)
- **Trade-off**: Simplicity vs scale vs performance

---

## Dependency Update Strategy

### Versioning Policy

**Major updates** (e.g., 2.x → 3.x):
- Test thoroughly in dev environment
- Check for breaking API changes
- Update code if needed
- Roll out in stages

**Minor updates** (e.g., 2.2 → 2.3):
- Review changelog for new features/fixes
- Update in dev, test core functionality
- Deploy to production after validation

**Patch updates** (e.g., 2.2.1 → 2.2.2):
- Apply quickly (usually bug fixes)
- Minimal testing required
- Deploy after smoke tests

### Security Updates

**Critical vulnerabilities**:
- Apply immediately (even if breaking)
- Test in isolated environment
- Hotfix deploy if needed

**Moderate vulnerabilities**:
- Schedule update within 1 week
- Include in next regular deployment

**Low vulnerabilities**:
- Address in next scheduled update

---

## Cost Optimization

### Reducing Dependency Costs

**1. LLM Costs** (largest):
```
Optimization               Savings
----------------------------------
Use GPT-3.5 vs GPT-4       10x cheaper
Reduce max_tokens          Linear reduction
Enable response caching    50-90% (repeat queries)
Batch API calls            No savings, but faster
```

**2. Embedding Costs**:
```
Use text-embedding-3-small  6x cheaper than large
Batch embeddings (100+)     More efficient
Cache embeddings            Avoid re-encoding
```

**3. Vector Storage**:
```
LanceDB (local)            Free (storage costs only)
Azure AI Search S1         ~$250/month
Quantization (8-bit)       4x storage reduction
```

**4. Azure Infrastructure**:
```
Use Spot VMs               70-90% cheaper
Auto-scale on demand       Pay only when active
Choose cheaper regions     Up to 40% savings
```

---

## Future Dependency Roadmap

### Planned Additions

**1. Local LLM Support**:
- **llama-cpp-python**: Run quantized Llama models locally
- **vllm**: High-throughput LLM serving
- **Reason**: Reduce API costs, improve privacy

**2. Advanced Vector Search**:
- **hnswlib**: Faster HNSW implementation than LanceDB
- **Reason**: 2-3x faster queries at scale

**3. Streaming Processing**:
- **Kafka client**: Real-time document ingestion
- **Reason**: Support continuous indexing

**4. Multi-modal**:
- **transformers**: Vision-language models
- **Reason**: Handle images, diagrams alongside text

### Planned Removals

**1. Redundant NLP**:
- Consider removing textblob (functionality covered by spaCy)
- Consolidate to spaCy for all NLP tasks

**2. Legacy Support**:
- Drop Python 3.10 support when 3.13 is stable
- Migrate to modern pandas (PyArrow-backed)

---

## Contributing

### Adding New Dependencies

**Checklist**:
1. **Evaluate necessity**: Can existing dependency handle this?
2. **Check license**: MIT, Apache 2.0, BSD preferred
3. **Check maintenance**: Active development, recent commits?
4. **Check alternatives**: Why this library vs others?
5. **Measure impact**: Size, dependencies, performance?
6. **Update documentation**: Add to appropriate spec/dependencies file
7. **Update pyproject.toml**: Pin version with `^` for minor updates

**Example**:
```toml
[tool.poetry.dependencies]
new-library = "^1.2.3"  # Allows 1.2.3 - 1.x.x (not 2.0.0)
```

---

## Conclusion

GraphRAG's dependencies form a carefully curated stack that balances:

- **Functionality**: Rich features for LLM interaction, graph processing, vector search
- **Performance**: Optimized libraries (numpy, pyarrow, graspologic)
- **Flexibility**: Support for local development and cloud-scale deployment
- **Maintainability**: Well-documented, actively maintained libraries
- **Cost**: Mix of open-source (networkx, pandas) and managed services (Azure)

**Key Takeaways**:

1. **Three-layer architecture**: Semantic (LLM) → Structural (Graph) → Operational (Data)
2. **Hybrid deployment**: Local (LanceDB) for dev, Cloud (Azure) for production
3. **Performance-critical paths**: LLM calls, vector search, community detection
4. **Cost optimization**: Model choice, caching, batching, quantization
5. **Future-proof**: Modular design allows swapping dependencies (fnllm abstraction)

Understanding these dependencies is essential for **effective GraphRAG deployment**, from choosing the right configuration for your scale to optimizing performance and costs.

---

## Quick Reference

| Need | Use | Why |
|------|-----|-----|
| Call LLM | fnllm + openai | Unified interface, flexible |
| Embed text | openai.embeddings | 1536D vectors, $0.02/1M tokens |
| Count tokens | tiktoken | Exact OpenAI token counting |
| Build graph | networkx | Flexible, pure Python |
| Find communities | graspologic | Hierarchical Leiden |
| Store vectors | lancedb (dev) / azure-search (prod) | Embedded vs cloud-scale |
| Process data | pandas + pyarrow | DataFrames + efficient I/O |
| Parse config | pydantic + pyyaml | Type-safe validation |
| Async I/O | aiofiles + asyncio | Non-blocking operations |
| CLI interface | typer + rich | Type-safe + beautiful output |

**For more details**, see individual dependency documentation files.
