# Large Language Model Algorithms

## Overview

GraphRAG extensively uses **Large Language Models (LLMs)** as core ML algorithms for knowledge extraction, summarization, and question answering. LLMs are **transformer-based neural networks** trained on massive text corpora that can understand and generate human language.

## ML Algorithm Classification

**Type**: Deep Learning - Transformer Neural Networks
**Training**: Pre-trained models (external) + optional fine-tuning
**Inference**: Cloud API calls (OpenAI, Azure OpenAI)
**Implementation**: External (third-party)

---

## LLM Algorithms Used in GraphRAG

### 1. Entity and Relationship Extraction

**Purpose**: Extract structured knowledge graph elements from unstructured text

**ML Algorithm**: Few-shot learning with GPT-4/GPT-3.5-turbo

**How it works**:
```
Input: Text chunk (200-600 tokens)
Prompt: "Extract entities and relationships from the following text..."
         + Few-shot examples (2-5 demonstrations)
LLM: Processes via 96-layer transformer (GPT-4)
Output: JSON with entities [{title, type, description}] and relationships
```

**Neural architecture** (GPT-4 - simplified):
- **Layers**: ~96 transformer layers
- **Parameters**: ~1.76 trillion (estimated, not officially confirmed)
- **Attention**: Multi-head self-attention (learns contextual relationships)
- **Training**: Next-token prediction on ~13 trillion tokens

**Code**: `graphrag/index/operations/extract_graph/extract_graph.py:27-135`

**Prompt structure**:
```yaml
system_prompt: |
  You are an expert at extracting entities and relationships from text.
  Extract the following entity types: {entity_types}
  Format the output as JSON: {format_spec}

few_shot_examples:
  - input: "Microsoft founded OpenAI in 2015..."
    output: |
      {
        "entities": [
          {"title": "MICROSOFT", "type": "organization", "description": "..."},
          {"title": "OPENAI", "type": "organization", "description": "..."}
        ],
        "relationships": [
          {"source": "MICROSOFT", "target": "OPENAI", "description": "founded"}
        ]
      }

user_text: "{actual_chunk_to_process}"
```

**ML parameters**:
```yaml
model: gpt-4-turbo-preview
temperature: 0.0      # Deterministic output
max_tokens: 4000      # Allow detailed extractions
top_p: 1.0           # Full probability distribution
```

**External dependency**:
- **Package**: `openai` (Python SDK)
- **Provider**: OpenAI API or Azure OpenAI Service
- **Model**: GPT-4, GPT-3.5-turbo, GPT-4o

---

### 2. Description Summarization

**Purpose**: Condense multiple entity/relationship descriptions into single coherent summary

**ML Algorithm**: Abstractive summarization with LLM

**How it works**:
```
Input: List of descriptions ["desc1", "desc2", ...]
Prompt: "Summarize the following descriptions into a single coherent description..."
LLM: Identifies common themes, removes redundancy, generates summary
Output: Single consolidated description
```

**Code**: `graphrag/index/operations/summarize_descriptions/summarize_descriptions.py:23-150`

**Example**:
```
Input descriptions:
1. "Microsoft is a technology company"
2. "Microsoft founded in 1975 by Bill Gates"
3. "Microsoft develops Windows and Office"

LLM summary:
"Microsoft is a technology company founded in 1975 by Bill Gates that develops
products including Windows and Office."
```

**ML task type**: Seq2Seq (sequence-to-sequence) generation
- **Input sequence**: Concatenated descriptions (tokens)
- **Output sequence**: Summarized description (tokens)
- **Mechanism**: Encoder-decoder attention + autoregressive generation

---

### 3. Community Report Generation

**Purpose**: Generate human-readable summaries of graph communities

**ML Algorithm**: Multi-document summarization with structured output

**Code**: Referenced in community detection workflows

**Prompt structure**:
```
Given the following entities and relationships in a community:

Entities:
- MICROSOFT (organization): Technology company founded in 1975...
- OPENAI (organization): AI research lab...
- GPT-4 (product): Large language model...

Relationships:
- MICROSOFT → OPENAI: Partnership and investment
- OPENAI → GPT-4: Developed

Generate a comprehensive summary of this community.
```

**LLM output**:
```markdown
# Technology Partnership Community

This community represents Microsoft's strategic investments in AI technology,
centered around their partnership with OpenAI. Key elements include:

- Microsoft's multi-billion dollar investment in OpenAI
- OpenAI's development of GPT-4 and other AI models
- Integration of OpenAI technology into Microsoft products

The community illustrates the convergence of cloud computing and AI research.
```

**ML parameters**:
```yaml
model: gpt-4-turbo-preview
temperature: 0.2      # Slightly creative but consistent
max_tokens: 2000      # Detailed summaries
```

---

### 4. Query Understanding and Answer Generation

**Purpose**: Generate natural language answers to user questions using retrieved context

**ML Algorithm**: Retrieval-augmented generation (RAG)

**Code**: `graphrag/query/structured_search/local_search/search.py:33-141`

**Flow**:
```
1. Query → Embedding (neural network)
2. Retrieve relevant context (vector search)
3. Construct prompt with context
4. LLM generates answer
```

**Prompt template**:
```
System: You are a helpful assistant answering questions based on provided context.

Context:
{retrieved_entities}
{retrieved_relationships}
{retrieved_text_chunks}

User question: {query}

Generate a comprehensive answer based ONLY on the provided context.
```

**ML components**:
- **Understanding**: LLM interprets query intent
- **Reasoning**: LLM synthesizes information from multiple context pieces
- **Generation**: LLM produces fluent, coherent answer

**Answer quality depends on**:
- Context relevance (retrieval quality)
- LLM reasoning capability (model size)
- Prompt design (instruction clarity)

---

## ML Model Architectures

### Transformer Architecture (Underlying all LLMs)

**Core components**:

1. **Self-Attention**:
   ```
   Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) V

   Where:
   - Q, K, V: Query, Key, Value matrices
   - d_k: Dimension of keys
   ```
   **Purpose**: Model relationships between all tokens in input

2. **Multi-Head Attention**:
   ```
   MultiHead(Q, K, V) = Concat(head_1, ..., head_h) W^O
   head_i = Attention(Q W_i^Q, K W_i^K, V W_i^V)
   ```
   **Purpose**: Capture different types of relationships (syntax, semantics, etc.)

3. **Feed-Forward Network**:
   ```
   FFN(x) = max(0, xW_1 + b_1) W_2 + b_2
   ```
   **Purpose**: Non-linear transformation of attention outputs

4. **Layer Normalization** + **Residual Connections**:
   ```
   x_out = LayerNorm(x + Sublayer(x))
   ```
   **Purpose**: Stabilize training, enable deep networks (96+ layers)

**Full Transformer Block**:
```
Input Tokens
    ↓
Token Embeddings + Positional Encoding
    ↓
┌─────────────────────────────────┐
│  Transformer Layer 1            │
│  - Multi-Head Self-Attention    │
│  - Add & Norm                   │
│  - Feed-Forward Network         │
│  - Add & Norm                   │
└─────────────────────────────────┘
    ↓
┌─────────────────────────────────┐
│  Transformer Layer 2            │
│  ...                            │
└─────────────────────────────────┘
    ↓
    ... (96 layers for GPT-4)
    ↓
Output Logits (vocabulary probabilities)
```

---

## External Dependencies (Packages)

### 1. openai (Python SDK)

**Purpose**: Interface to OpenAI API

**Package**: `openai==1.x`

**Usage in GraphRAG**:
```python
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

# Chat completion
response = client.chat.completions.create(
    model="gpt-4-turbo-preview",
    messages=[
        {"role": "system", "content": "You are an expert..."},
        {"role": "user", "content": "Extract entities from..."}
    ],
    temperature=0.0,
    max_tokens=4000
)

# Text embedding
embedding = client.embeddings.create(
    model="text-embedding-ada-002",
    input="Text to embed"
)
```

**Key features**:
- Automatic retry with exponential backoff
- Streaming support for long responses
- Rate limit handling

---

### 2. fnllm (Unified LLM Interface)

**Purpose**: Abstraction layer over multiple LLM providers

**Package**: `fnllm` (Microsoft, used internally by GraphRAG)

**From** `spec/dependencies/01-llm-and-language-processing.md`:

**Why fnllm?**
- **Provider abstraction**: Same code for OpenAI, Azure, Anthropic
- **Automatic retries**: Built-in error handling
- **Caching**: Avoid duplicate API calls
- **Token counting**: Predict costs before requests

**Code**: `graphrag/language_model/providers/fnllm/`

**Usage**:
```python
from graphrag.language_model.manager import ModelManager

model = ModelManager().get_or_create_chat_model(
    name="entity_extraction",
    model_type="openai_chat",  # or "azure_openai_chat"
    config=llm_config,
    callbacks=callbacks,
    cache=cache
)

# Unified interface regardless of provider
response = await model.achat(messages=[...])
```

---

### 3. Azure OpenAI Service

**Purpose**: Enterprise-grade OpenAI models with Azure infrastructure

**Package**: `openai` (same SDK, different endpoint)

**Configuration**:
```yaml
llm:
  type: azure_openai_chat
  api_base: https://your-resource.openai.azure.com
  api_version: "2024-02-01"
  deployment_name: gpt-4-turbo
  api_key: ${AZURE_OPENAI_API_KEY}
```

**Advantages**:
- **Data residency**: Keep data in specific Azure regions
- **Enterprise SLA**: 99.9% uptime guarantee
- **Private networking**: VNet integration
- **Cost management**: Quota controls, budget alerts

---

## ML Training Details

### Pre-training (OpenAI)

**GraphRAG does NOT train models** - uses pre-trained models from OpenAI

**Training details** (general GPT-4 info, not GraphRAG-specific):

1. **Data**: ~13 trillion tokens (web text, books, code)
2. **Objective**: Next-token prediction (language modeling)
   ```
   Maximize: P(token_t | token_1, ..., token_{t-1})
   ```
3. **Compute**: Thousands of GPUs/TPUs for months
4. **Cost**: Estimated $100M+ for GPT-4 training

**Training is external** - GraphRAG only does **inference** (API calls)

---

### Fine-tuning (Optional)

**GraphRAG supports** (but doesn't require) fine-tuned models

**Use case**: Domain-specific entity extraction

**Process** (outside GraphRAG):
```python
# 1. Collect training data
training_data = [
    {"messages": [
        {"role": "system", "content": "Extract entities..."},
        {"role": "user", "content": "Text about proteins..."},
        {"role": "assistant", "content": '{"entities": [...]}'}
    ]},
    ...
]

# 2. Fine-tune via OpenAI API
openai.FineTuningJob.create(
    training_file="training_data.jsonl",
    model="gpt-3.5-turbo",
    hyperparameters={"n_epochs": 3}
)

# 3. Use fine-tuned model in GraphRAG
# Configure model name to fine-tuned model ID
```

**Benefits**:
- 10-30% better extraction quality for specialized domains
- Lower inference cost (can use smaller base model)

---

## Performance Characteristics

### Latency

**Typical API response times** (GraphRAG measurements):

| Operation | Model | Tokens | Latency | Cost per 1M tokens |
|-----------|-------|--------|---------|-------------------|
| Entity extraction | GPT-4-turbo | 500 input, 1000 output | 3-5 sec | $30 |
| Summarization | GPT-4-turbo | 200 input, 100 output | 1-2 sec | $30 |
| Answer generation | GPT-4-turbo | 2000 input, 500 output | 4-6 sec | $30 |
| Entity extraction | GPT-3.5-turbo | 500 input, 1000 output | 1-2 sec | $1.50 |

**Bottleneck**: Network latency + model inference time (dominated by inference)

---

### Throughput

**Parallel processing** (GraphRAG implementation):
```python
# Process 100 chunks concurrently
semaphore = asyncio.Semaphore(10)  # 10 concurrent requests

async def extract_entities(chunk):
    async with semaphore:
        return await llm.achat(messages=[...])

tasks = [extract_entities(chunk) for chunk in chunks]
results = await asyncio.gather(*tasks)
```

**Throughput**: ~10-50 extractions/second (limited by API rate limits)

---

### Cost

**GraphRAG typical costs** (10K document corpus):

| Operation | Tokens | Cost (GPT-4) | Cost (GPT-3.5) |
|-----------|--------|--------------|----------------|
| Entity extraction | 100M | $3,000 | $150 |
| Description summarization | 20M | $600 | $30 |
| Community reports | 10M | $300 | $15 |
| **Total indexing** | **130M** | **$3,900** | **$195** |
| Query (per answer) | 5K | $0.15 | $0.0075 |

**Cost optimization**:
1. Use GPT-3.5-turbo instead of GPT-4 (20x cheaper, 80% quality)
2. Cache LLM responses (avoid re-extraction on updates)
3. Reduce prompt size (fewer examples, shorter instructions)

---

## Integration with GraphRAG Pipeline

### Indexing Flow

```
Documents → Chunks → [LLM: Entity Extraction] → Entities
                                ↓
                         [LLM: Summarization] → Consolidated Descriptions
                                ↓
                         Graph Construction → Communities
                                ↓
                         [LLM: Community Reports] → Summaries
```

### Query Flow

```
User Query → Embed Query → Vector Search → Retrieve Context
                                              ↓
                                        [LLM: Answer Generation] → Response
```

---

## Limitations and Considerations

### 1. Hallucination

**Problem**: LLMs sometimes generate false information

**Mitigation in GraphRAG**:
- **Retrieval-augmented**: Provide factual context from corpus
- **Temperature=0**: Reduce randomness in extraction
- **Few-shot examples**: Guide model to expected output format

---

### 2. Context Length Limits

**Problem**: LLMs have max context window (8K-128K tokens)

**Mitigation**:
- **Chunking**: Process documents in segments (see `spec/vectors/02-chunking-strategies.md`)
- **Summarization**: Condense context before querying

---

### 3. API Cost

**Problem**: LLM inference is expensive ($3K-$4K per 10K docs)

**Mitigation**:
- **Model selection**: Use GPT-3.5 where possible
- **Caching**: Avoid re-processing unchanged content
- **Batch processing**: Reduce API overhead

---

## Future Enhancements

### 1. Local LLM Inference

**Current**: Cloud API only (OpenAI, Azure)

**Proposed**: Support local models (Llama 2, Mistral)
```python
# Use local model via Ollama
llm_config:
  type: ollama
  model: llama2:70b
  base_url: http://localhost:11434
```

**Benefits**: No API cost, data privacy

---

### 2. Multi-Model Ensemble

**Current**: Single model for all tasks

**Proposed**: Task-specific models
```python
extraction_model: gpt-4-turbo      # High quality
summarization_model: gpt-3.5-turbo # Cheaper, sufficient
answer_model: gpt-4o               # Fast, conversational
```

**Benefits**: Cost-quality trade-off optimization

---

## Conclusion

Large Language Models are the **core ML technology** in GraphRAG, enabling:
- **Knowledge extraction**: Transform unstructured text → structured graph
- **Summarization**: Condense multi-source information
- **Question answering**: Generate natural language responses

**Key characteristics**:
- **External implementation**: OpenAI API (pre-trained models)
- **Transformer architecture**: 96-layer neural network (GPT-4)
- **Inference-only**: GraphRAG calls API, doesn't train models
- **Cost**: $3K-$4K per 10K documents (GPT-4), $150-$200 (GPT-3.5)

**Dependencies**:
- `openai` (Python SDK)
- `fnllm` (unified interface)
- Azure OpenAI Service (optional, enterprise deployment)

**Integration**: Powers extraction, summarization, and generation throughout GraphRAG pipeline
