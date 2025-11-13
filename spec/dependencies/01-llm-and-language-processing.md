# LLM and Language Processing Dependencies

## Overview

This document describes external dependencies used for Large Language Model (LLM) interaction, text embedding, tokenization, and natural language processing in GraphRAG. These libraries form the **semantic awareness layer**, enabling the system to understand, encode, and generate natural language.

## Core LLM Infrastructure

### fnllm

**Version**: `0.2.3` (with `azure` and `openai` extras)

**Purpose**: Unified LLM interface and management layer

**Role in GraphRAG**:
- **Primary LLM abstraction**: Provides consistent API across different LLM providers (OpenAI, Azure OpenAI)
- **Model lifecycle management**: Handles model initialization, configuration, and connection pooling
- **Chat completion**: Enables conversational interactions for entity extraction, query answering, and summarization
- **Streaming support**: Allows real-time token-by-token response generation

**Architecture Integration**:
```
GraphRAG Layer                 fnllm Layer                 Provider Layer
-------------                 -----------                 --------------
ChatModel Protocol     →      fnllm Abstraction    →      OpenAI API
                                                    →      Azure OpenAI API
```

**Key Features Used**:
- **Provider abstraction**: Switch between OpenAI/Azure without code changes
- **Async support**: Enables parallel LLM calls (map-reduce pattern)
- **Error handling**: Automatic retries, rate limiting, timeout management
- **Cost tracking**: Token usage monitoring for LLM calls

**Code Locations**:
- `graphrag/language_model/providers/fnllm/models.py` - Model implementations
- `graphrag/language_model/providers/fnllm/cache.py` - Response caching
- `graphrag/language_model/providers/fnllm/events.py` - Event callbacks
- `graphrag/language_model/manager.py` - Central model manager

**Usage Example**:
```python
from graphrag.language_model.manager import ModelManager
from graphrag.config.models.language_model_config import LanguageModelConfig

llm_config = LanguageModelConfig(
    type="openai_chat",
    model="gpt-4-turbo",
    max_tokens=4000
)

model = ModelManager().get_or_create_chat_model(
    name="entity_extraction",
    model_type=llm_config.type,
    config=llm_config
)

response = await model.achat(prompt="Extract entities from text...")
```

**Performance Characteristics**:
- **Latency**: 500ms - 5s per call (model-dependent)
- **Throughput**: Limited by API rate limits (RPM, TPM)
- **Cost**: $0.01 - $0.06 per 1K tokens (model-dependent)
- **Concurrency**: Supports hundreds of parallel async calls

**Alternatives Considered**:
- **LangChain**: Too heavyweight, opinionated abstractions
- **Direct SDK**: Less flexible, requires provider-specific code
- **fnllm chosen**: Lightweight, flexible, maintains control

---

### openai

**Version**: `^1.57.0`

**Purpose**: Official OpenAI Python SDK

**Role in GraphRAG**:
- **Direct API access**: Lower-level control when fnllm abstraction insufficient
- **Embedding generation**: Create text embeddings via `text-embedding-3-small/large`
- **Function calling**: Structured output extraction (JSON mode)
- **Model listing**: Query available models and capabilities

**Architecture Integration**:
```
fnllm (abstraction)
    |
    v
openai SDK (implementation)
    |
    v
OpenAI/Azure REST API
```

**Key Features Used**:
- **Chat completions**: Primary interface for GPT-3.5/GPT-4 models
- **Embeddings API**: Convert text to 1536D or 3072D vectors
- **Streaming**: Real-time token generation for better UX
- **JSON mode**: Force LLM to output valid JSON (entity extraction)
- **Async client**: Native async/await support for concurrency

**Code Locations**:
- `graphrag/language_model/providers/fnllm/models.py` - Wraps OpenAI client
- `graphrag/index/operations/embed_text.py` - Uses embedding API

**Embedding Model Used**:
```python
# Default embedding configuration
embedding_model = "text-embedding-3-small"
dimensions = 1536
cost_per_1M_tokens = $0.02
```

**Why this embedding model**:
- **Balance**: Good quality/cost trade-off
- **Dimensionality**: 1536D sufficient for semantic similarity
- **Speed**: Fast inference (<100ms per batch)
- **Stability**: Consistent embeddings (no model drift)

**Usage Example**:
```python
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=api_key)

# Generate embedding
response = await client.embeddings.create(
    model="text-embedding-3-small",
    input="Query text to embed"
)
query_vector = response.data[0].embedding  # 1536D vector
```

---

### tiktoken

**Version**: `^0.8.0`

**Purpose**: BPE tokenizer for OpenAI models

**Role in GraphRAG**:
- **Token counting**: Calculate prompt/completion tokens before API calls
- **Budget management**: Ensure prompts fit within context window (8K/128K tokens)
- **Cost estimation**: Predict API costs based on token usage
- **Chunking decisions**: Determine optimal text unit sizes

**Architecture Integration**:
```
Text Input → tiktoken (tokenize) → Token Count → Budget Check → LLM Call
```

**Key Features Used**:
- **Encoding**: `cl100k_base` (GPT-3.5/4), `p50k_base` (older models)
- **Fast tokenization**: Rust-backed implementation (~10K tokens/ms)
- **Exact counts**: Matches OpenAI's token counting precisely
- **Special tokens**: Handles system message tokens, function call overhead

**Code Locations**:
- `graphrag/query/llm/text_utils.py` - Token counting utilities
- `graphrag/index/operations/chunk_text.py` - Text chunking based on tokens

**Usage Example**:
```python
import tiktoken

encoding = tiktoken.get_encoding("cl100k_base")

# Count tokens in prompt
text = "Extract all entities from this document..."
token_count = len(encoding.encode(text))

# Estimate cost
cost_per_1k_tokens = 0.01  # GPT-4 Turbo
estimated_cost = (token_count / 1000) * cost_per_1k_tokens
```

**Context Window Management**:
```python
def pack_into_context(texts, max_tokens=8000):
    """Pack texts into context window with token budget."""
    encoding = tiktoken.get_encoding("cl100k_base")
    packed = []
    current_tokens = 0

    for text in texts:
        text_tokens = len(encoding.encode(text))
        if current_tokens + text_tokens > max_tokens:
            break  # Would exceed budget
        packed.append(text)
        current_tokens += text_tokens

    return packed, current_tokens
```

**Performance Characteristics**:
- **Tokenization speed**: ~10M tokens/second
- **Memory overhead**: Minimal (encoding cached)
- **Accuracy**: 100% match with OpenAI counting

---

### json-repair

**Version**: `^0.30.3`

**Purpose**: Fix malformed JSON from LLM outputs

**Role in GraphRAG**:
- **Robust parsing**: Recover from LLM JSON syntax errors
- **Error tolerance**: Handle incomplete/truncated JSON (streaming)
- **Validation**: Ensure entity extraction outputs are parseable

**Architecture Integration**:
```
LLM Output (JSON string) → json-repair (fix) → json.loads() → Structured Data
```

**Why Needed**:
LLMs sometimes generate invalid JSON:
- Missing closing braces: `{"entity": "Alice"`
- Trailing commas: `{"entities": [1, 2, 3,]}`
- Unquoted keys: `{entity: "value"}`
- Embedded newlines in strings

**Code Locations**:
- `graphrag/query/llm/text_utils.py` - `try_parse_json_object()` function
- `graphrag/index/operations/extract_graph/graph_extractor.py` - Entity extraction parsing

**Usage Example**:
```python
from json_repair import repair_json
import json

# LLM output (malformed)
llm_output = '{"points": [{"description": "Point 1", "score": 90,]}'

# Repair and parse
repaired = repair_json(llm_output)
data = json.loads(repaired)  # Success!
```

**Fallback Strategy**:
```python
def try_parse_json_object(json_str: str):
    """Try parsing JSON with progressive fallback."""
    # Try 1: Direct parse
    try:
        return json.loads(json_str)
    except json.JSONDecodeError:
        pass

    # Try 2: Repair and parse
    try:
        repaired = repair_json(json_str)
        return json.loads(repaired)
    except Exception:
        pass

    # Try 3: Return empty dict (fail gracefully)
    return {}
```

---

## Natural Language Processing

### nltk

**Version**: `3.9.1`

**Purpose**: Natural Language Toolkit for text processing

**Role in GraphRAG**:
- **Sentence splitting**: Divide text into sentence boundaries
- **Text tokenization**: Word-level tokenization (not BPE)
- **Language detection**: Identify text language
- **Stop words**: Filter common words

**Architecture Integration**:
```
Raw Text → nltk (sentence split) → Sentences → Chunk by token count → Text Units
```

**Key Features Used**:
- **Punkt sentence tokenizer**: Robust sentence boundary detection
- **Word tokenizers**: Extract words for statistics
- **Corpus utilities**: Access linguistic resources

**Code Locations**:
- `graphrag/index/operations/chunk_text.py` - Sentence-aware chunking
- Text preprocessing pipelines

**Usage Example**:
```python
import nltk

# Download required data
nltk.download('punkt')

text = "GraphRAG is a system. It uses LLMs. Entities are extracted."
sentences = nltk.sent_tokenize(text)
# ['GraphRAG is a system.', 'It uses LLMs.', 'Entities are extracted.']
```

**Sentence-Aware Chunking**:
```python
def chunk_text_with_sentences(text, max_tokens=300):
    """Chunk text respecting sentence boundaries."""
    sentences = nltk.sent_tokenize(text)
    chunks = []
    current_chunk = []
    current_tokens = 0

    for sentence in sentences:
        sent_tokens = count_tokens(sentence)

        if current_tokens + sent_tokens > max_tokens and current_chunk:
            # Flush current chunk
            chunks.append(' '.join(current_chunk))
            current_chunk = [sentence]
            current_tokens = sent_tokens
        else:
            current_chunk.append(sentence)
            current_tokens += sent_tokens

    if current_chunk:
        chunks.append(' '.join(current_chunk))

    return chunks
```

**Why Sentence-Aware Chunking**:
- **Semantic coherence**: Sentences are natural semantic units
- **Better embeddings**: Complete sentences embed better than fragments
- **Improved extraction**: Entity extraction more accurate on complete sentences

---

### spacy

**Version**: `^3.8.4`

**Purpose**: Industrial-strength NLP library

**Role in GraphRAG**:
- **Named Entity Recognition (NER)**: Identify persons, organizations, locations
- **Part-of-speech tagging**: Identify nouns, verbs for entity candidates
- **Dependency parsing**: Understand grammatical structure
- **Linguistic features**: Extract linguistic attributes

**Architecture Integration**:
```
Text → spacy NLP pipeline → {entities, POS tags, dependencies} → Entity candidates
```

**Key Features Used**:
- **Pre-trained models**: `en_core_web_sm/md/lg` for English
- **Entity recognition**: Detect PERSON, ORG, GPE, EVENT entities
- **Tokenization**: Linguistic tokenization (not BPE)
- **Lemmatization**: Normalize words to base forms

**Code Locations**:
- `graphrag/index/operations/extract_graph/` - May augment LLM extraction
- Text preprocessing for better LLM prompts

**Usage Example**:
```python
import spacy

nlp = spacy.load("en_core_web_sm")

text = "Microsoft was founded by Bill Gates in Seattle."
doc = nlp(text)

# Extract named entities
entities = [(ent.text, ent.label_) for ent in doc.ents]
# [('Microsoft', 'ORG'), ('Bill Gates', 'PERSON'), ('Seattle', 'GPE')]

# POS tagging
pos_tags = [(token.text, token.pos_) for token in doc]
# [('Microsoft', 'PROPN'), ('was', 'AUX'), ('founded', 'VERB'), ...]
```

**Role in Entity Extraction**:

**Option 1: Standalone NER** (not primary in GraphRAG):
```python
def extract_entities_with_spacy(text):
    """Use spaCy for entity extraction."""
    doc = nlp(text)
    entities = []
    for ent in doc.ents:
        entities.append({
            "name": ent.text,
            "type": ent.label_,
            "start": ent.start_char,
            "end": ent.end_char
        })
    return entities
```

**Option 2: Hybrid LLM + spaCy** (more likely):
```python
def hybrid_entity_extraction(text):
    """Combine LLM and spaCy extraction."""
    # Step 1: spaCy extracts entity candidates
    doc = nlp(text)
    candidates = [ent.text for ent in doc.ents]

    # Step 2: LLM validates and enriches
    prompt = f"""
    The following entities were detected: {candidates}

    For each entity, provide:
    - Refined name (resolve coreferences)
    - Accurate type (PERSON, ORG, CONCEPT, EVENT)
    - Comprehensive description
    """

    llm_response = await llm.achat(prompt)
    return parse_llm_entities(llm_response)
```

**Performance Characteristics**:
- **Processing speed**: ~10K tokens/second (model-dependent)
- **Accuracy**: F1 ~85% for NER on standard benchmarks
- **Memory**: 50-500MB (model size)

---

### textblob

**Version**: `^0.18.0.post0`

**Purpose**: Simplified NLP library for text processing

**Role in GraphRAG**:
- **Sentiment analysis**: Analyze text sentiment (positive/negative/neutral)
- **Text correction**: Fix spelling errors
- **N-gram extraction**: Extract multi-word phrases
- **Text statistics**: Calculate readability scores

**Architecture Integration**:
```
Text → TextBlob → {sentiment, corrected_text, noun_phrases} → Metadata
```

**Key Features Used**:
- **Sentiment polarity**: -1 (negative) to +1 (positive)
- **Noun phrase extraction**: Identify candidate entities
- **Spelling correction**: Clean noisy text
- **Simple API**: Easy-to-use interface

**Usage Example**:
```python
from textblob import TextBlob

text = "GraphRAG is an excellent system for knowledge extraction."
blob = TextBlob(text)

# Sentiment analysis
sentiment = blob.sentiment.polarity  # 0.75 (positive)

# Noun phrases (entity candidates)
noun_phrases = blob.noun_phrases
# ['graphrag', 'excellent system', 'knowledge extraction']

# Spelling correction
noisy_text = "Grapphrag is an excelent systm"
corrected = TextBlob(noisy_text).correct()
# "Graphrag is an excellent system"
```

**Potential Use Cases** (may not be currently active):
- **Quality filtering**: Remove low-quality text units (extreme sentiment, errors)
- **Entity candidate generation**: Use noun phrases as entity seeds
- **Text normalization**: Correct OCR errors before processing

---

## Dependency Relationships

### Interaction Diagram

```
                    User Query / Source Text
                            |
                            v
    +-------------------+---+---+-------------------+
    |                   |       |                   |
    v                   v       v                   v
nltk              textblob    spacy            tiktoken
(sentences)      (sentiment)  (NER)         (token count)
    |                   |       |                   |
    +-------------------+-------+-------------------+
                            |
                            v
                      Text Units
                   (semantically coherent)
                            |
                            v
                    +-------+-------+
                    |               |
                    v               v
                openai          fnllm
              (embeddings)     (chat)
                    |               |
                    v               v
            Query Vectors    Entity Extraction
                                    |
                                    v
                              json-repair
                            (parse output)
```

### Functional Layering

**Layer 1: Pre-processing**
- `nltk`: Sentence splitting
- `spacy`: Linguistic analysis
- `textblob`: Text cleaning, sentiment

**Layer 2: Tokenization & Budgeting**
- `tiktoken`: Token counting, budget management

**Layer 3: LLM Interaction**
- `fnllm`: Unified LLM interface
- `openai`: SDK implementation
- `json-repair`: Output parsing

**Layer 4: Semantic Encoding**
- `openai`: Text embeddings (vectors)

---

## Configuration and Best Practices

### LLM Model Selection

**Entity Extraction**:
- **Model**: GPT-4 Turbo / GPT-4o
- **Reason**: High accuracy for structured extraction
- **Max tokens**: 1000-2000 (entity lists can be long)
- **Temperature**: 0.0 (deterministic)

**Query Answering** (Map phase):
- **Model**: GPT-3.5 Turbo (cost optimization) or GPT-4 (quality)
- **Max tokens**: 1000 (key points)
- **Temperature**: 0.0

**Query Answering** (Reduce phase):
- **Model**: GPT-4 Turbo (synthesis requires reasoning)
- **Max tokens**: 2000 (comprehensive answer)
- **Temperature**: 0.0

### Embedding Configuration

**Text Embeddings**:
```python
model = "text-embedding-3-small"
dimensions = 1536
batch_size = 100  # Embed in batches for efficiency
```

**Query Embeddings**:
```python
# Same model as text for consistency
model = "text-embedding-3-small"
dimensions = 1536
# Single query per call (real-time)
```

**Why same model for query and text**:
- **Cosine similarity**: Only valid if vectors in same space
- **Consistency**: Training distribution matches

### Token Budget Management

**Context Windows**:
```python
# By model
GPT_35_TURBO = 16_385  # tokens
GPT_4_TURBO = 128_000  # tokens
GPT_4O = 128_000  # tokens

# Allocation strategy
SYSTEM_PROMPT = 500  # Fixed overhead
QUERY = 100  # User question
DATA_CONTEXT = max_tokens - SYSTEM_PROMPT - QUERY - 500  # Buffer
```

**Chunking Strategy**:
```python
# Text unit size
OPTIMAL_CHUNK_SIZE = 300  # tokens
MIN_CHUNK_SIZE = 100
MAX_CHUNK_SIZE = 600

# Community report size
REPORT_MAX_TOKENS = 1000  # Fit ~6 reports in 8K context
```

---

## Performance and Cost Optimization

### LLM Call Optimization

**Batching**:
```python
# Good: Parallel map calls
results = await asyncio.gather(*[
    llm.achat(prompt, context=chunk)
    for chunk in chunks
])

# Bad: Sequential calls
results = []
for chunk in chunks:
    result = await llm.achat(prompt, context=chunk)
    results.append(result)
```

**Caching**:
```python
# fnllm supports caching
cache = PipelineCache()

model = ModelManager().get_or_create_chat_model(
    ...,
    cache=cache  # Enable response caching
)

# Identical prompts return cached responses (no API call)
```

**Rate Limiting**:
```python
# Semaphore to limit concurrency
semaphore = asyncio.Semaphore(10)  # Max 10 concurrent calls

async def call_with_limit(prompt):
    async with semaphore:
        return await llm.achat(prompt)
```

### Embedding Optimization

**Batch Embeddings**:
```python
# Good: Batch API call
texts = [unit.text for unit in text_units]
embeddings = await embed_texts_batch(texts, batch_size=100)

# Bad: Individual calls
embeddings = []
for text in texts:
    emb = await embed_text(text)
    embeddings.append(emb)
```

**Cost Comparison**:
```
text-embedding-3-small: $0.02 / 1M tokens
text-embedding-3-large: $0.13 / 1M tokens

For 100K text units (avg 300 tokens each):
- Total tokens: 30M
- small: $0.60
- large: $3.90

Typical choice: small (6.5x cheaper, minimal quality loss)
```

---

## Troubleshooting

### Common Issues

**Issue 1: JSON Parsing Failures**
```
Error: json.JSONDecodeError: Expecting ',' delimiter
```

**Solution**: json-repair handles most cases, but verify prompt examples:
```python
# Add explicit JSON formatting instructions
prompt += "\n\nEnsure output is valid JSON. No trailing commas!"
```

**Issue 2: Token Limit Exceeded**
```
Error: This model's maximum context length is 8192 tokens
```

**Solution**: Pre-count tokens with tiktoken:
```python
total_tokens = count_tokens(system_prompt) + count_tokens(data)
if total_tokens > max_tokens:
    # Truncate data
    data = truncate_to_token_budget(data, max_tokens - system_tokens)
```

**Issue 3: Rate Limiting**
```
Error: Rate limit reached for requests
```

**Solution**: Implement exponential backoff:
```python
# fnllm handles this automatically
# But can configure retry strategy
model_config.max_retries = 5
model_config.retry_delay = 2.0  # seconds
```

**Issue 4: spaCy Model Not Found**
```
Error: Can't find model 'en_core_web_sm'
```

**Solution**: Download model first:
```python
import spacy.cli
spacy.cli.download("en_core_web_sm")
```

---

## Future Considerations

### Planned Improvements

**1. Multi-Modal Embeddings**:
- Use vision-language models for image understanding
- Embed diagrams, charts alongside text
- Unified multi-modal vector space

**2. Faster Local LLMs**:
- Integrate llama.cpp for on-premises deployment
- Use quantized models (GGUF format)
- Reduce OpenAI API costs

**3. Advanced NLP**:
- Coreference resolution (link pronouns to entities)
- Relation extraction (beyond entity extraction)
- Event extraction (temporal knowledge)

### Alternative Dependencies

**LLM Frameworks**:
- **Anthropic SDK**: Support Claude models
- **Google AI SDK**: Gemini integration
- **Hugging Face Transformers**: Local model hosting

**Embedding Models**:
- **sentence-transformers**: Local embedding models
- **Cohere**: Alternative embedding API
- **Instructor embeddings**: Domain-specific fine-tuning

**NLP Tools**:
- **Stanza**: Stanford NLP alternative to spaCy
- **AllenNLP**: Research-focused NLP toolkit
- **Flair**: State-of-the-art NER models

---

## Conclusion

The LLM and language processing dependencies form GraphRAG's **cognitive layer**, enabling:
- **Understanding**: NLP tools parse and structure text
- **Reasoning**: LLMs extract entities, answer questions, synthesize knowledge
- **Encoding**: Embeddings convert semantics to geometric representations
- **Efficiency**: Tokenizers manage costs and context budgets

These libraries work in concert to transform raw text into structured semantic knowledge, with fnllm and OpenAI at the core, supported by robust NLP utilities (nltk, spaCy, textblob) and essential tooling (tiktoken, json-repair).

**Key Takeaway**: GraphRAG's effectiveness depends on carefully orchestrating these dependencies—choosing the right models, managing token budgets, optimizing API calls, and handling edge cases gracefully.
