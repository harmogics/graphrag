# Концептуальный процесс формирования Document Chunks

## Обзор

Chunking (разбиение на фрагменты) — это **фундаментальный процесс** в GraphRAG, который преобразует длинные документы в управляемые текстовые блоки оптимального размера для последующей обработки LLM-агентами. Это первое и критически важное семантическое преобразование в pipeline индексации.

**Концептуальная роль chunking**:
- Преобразует документы произвольной длины в фрагменты фиксированного размера
- Сохраняет семантическую целостность через overlap (перекрытие)
- Балансирует между детальностью контекста и вычислительной эффективностью
- Создает основу для всех последующих преобразований

```
┌─────────────────────────────────────────────────────────┐
│              CHUNKING В PIPELINE GRAPHRAG               │
└─────────────────────────────────────────────────────────┘

Raw Documents (variable length: 100-100,000+ tokens)
            ↓
      ┌─────────────┐
      │  CHUNKING   │  ← Первое преобразование
      └─────────────┘
            ↓
Text Units (fixed size: ~1200 tokens each with 100 overlap)
            ↓
  [Entity Extraction] → Entities
  [Relationship Extraction] → Relationships
  [Embedding] → Vectors
            ↓
    Knowledge Graph
```

## Концептуальные основы chunking

### Зачем нужен chunking?

**Проблема**: Документы имеют произвольную длину, часто превышающую:
- Контекстные окна LLM (4K-128K tokens в зависимости от модели)
- Оптимальный размер для quality extraction (LLM лучше обрабатывают короткие фрагменты)
- Эффективные размеры для embedding моделей (обычно 512-8192 tokens)

**Решение**: Разбиение на chunks — управляемые фрагменты с:
1. **Фиксированным размером** — предсказуемость обработки
2. **Семантическим overlap** — сохранение контекста на границах
3. **Прослеживаемостью** — связь chunk → source document

### Ключевые концепции

#### 1. Семантическая единица (Semantic Unit)

Chunk — это не просто произвольный набор токенов, а **семантическая единица** текста:

```
Плохой chunk (разрыв на середине предложения):
┌────────────────────────────────────────────────┐
│ "...the GraphRAG system uses large language   │
│  models to extract entities and relationsh-"  │ ← РАЗРЫВ
└────────────────────────────────────────────────┘

Хороший chunk (семантически целостный):
┌────────────────────────────────────────────────┐
│ "...the GraphRAG system uses large language   │
│  models to extract entities and relationships │
│  from text. These entities form the nodes..." │ ← ПОЛНЫЙ КОНТЕКСТ
└────────────────────────────────────────────────┘
```

**Принципы семантической целостности**:
- Chunk должен содержать полные мысли/предложения (когда возможно)
- Минимизация разрывов внутри концептуальных единиц
- Сохранение локального контекста для LLM

#### 2. Overlap (Перекрытие) как механизм сохранения контекста

**Концепция**: Соседние chunks перекрываются, чтобы сохранить информацию на границах.

```
Document: "ABCDEFGHIJKLMNOPQRSTUVWXYZ"

Без overlap:
┌─────────┐
│ ABCDEFG │  Chunk 1
└─────────┘
          ┌─────────┐
          │ HIJKLMN │  Chunk 2  ← Потеря контекста между G и H
          └─────────┘
                    ┌─────────┐
                    │ OPQRSTU │  Chunk 3
                    └─────────┘

С overlap = 2:
┌─────────┐
│ ABCDEFG │  Chunk 1
└─────────┘
      ┌─────────┐
      │ FGHIJKL │  Chunk 2  ← F,G повторяются (контекст сохранен)
      └─────────┘
            ┌─────────┐
            │ KLMNOPQ │  Chunk 3  ← K,L повторяются
            └─────────┘
```

**Семантическая роль overlap**:
- **Сохранение reference context**: Entities на границах видны в обоих chunks
- **Устранение boundary loss**: Предотвращение потери связей между chunks
- **Улучшение entity extraction**: LLM видит полный контекст вокруг entity

**Trade-off overlap**:
```
Overlap = 0:
  ✓ Минимальное дублирование данных
  ✓ Максимальная скорость обработки
  ✗ Потеря контекста на границах
  ✗ Хуже quality extraction

Overlap = 10-15% (100-200 tokens @ 1200 chunk):
  ✓ Сохранение boundary context
  ✓ Улучшенная entity extraction
  ✓ Умеренное дублирование
  ~ Стоимость: +10-15% processing

Overlap = 50%:
  ✓ Максимальное сохранение контекста
  ✗ Большое дублирование
  ✗ 2x processing cost
  ✗ Избыточность без существенного gain
```

**Рекомендация**: Overlap 8-15% оптимален для большинства случаев.

#### 3. Granularity (Гранулярность)

Размер chunk определяет **уровень гранулярности** информации:

```
┌──────────────────────────────────────────────────────────┐
│              GRANULARITY SPECTRUM                        │
└──────────────────────────────────────────────────────────┘

Very Fine (100-300 tokens)
├─ Sentence/paragraph level
├─ High precision, low coverage per chunk
├─ Many chunks (высокая стоимость)
└─ Use case: Детальный анализ, citation-heavy tasks

Fine (500-800 tokens)
├─ Multiple paragraphs
├─ Good balance for short documents
├─ Moderate chunks count
└─ Use case: Научные статьи, новости

Medium (1000-1500 tokens) ← DEFAULT GraphRAG
├─ Section-level context
├─ Optimal LLM processing window
├─ Balanced cost/quality
└─ Use case: General knowledge extraction, RAG

Coarse (2000-4000 tokens)
├─ Multi-section context
├─ High coverage, may lose precision
├─ Fewer chunks (экономия)
└─ Use case: Summarization, topic modeling

Very Coarse (5000+ tokens)
├─ Chapter/document level
├─ May exceed optimal LLM attention
├─ Lowest cost
└─ Use case: Embedding-only, no entity extraction
```

**Влияние granularity на downstream tasks**:

| Chunk Size | Entity Extraction | Relationship Detection | Embedding Quality | Cost |
|---|---|---|---|---|
| 100-300 | ★★★ (Precise) | ★☆☆ (Limited context) | ★★☆ | Very High |
| 500-800 | ★★★ (Good) | ★★☆ (Moderate context) | ★★★ | High |
| **1000-1500** | ★★★ (Optimal) | ★★★ (Good context) | ★★★ | **Medium** |
| 2000-4000 | ★★☆ (May miss details) | ★★★ (Rich context) | ★★☆ | Low |
| 5000+ | ★☆☆ (Degraded) | ★★☆ (Very rich but noisy) | ★☆☆ | Very Low |

## Стратегии Chunking в GraphRAG

GraphRAG поддерживает **две основные стратегии** chunking, каждая с различными концептуальными подходами.

### Стратегия 1: Token-based Chunking (Default)

**Концепция**: Разбиение по фиксированному количеству **токенов** с sliding window и overlap.

#### Концептуальная модель

```
Document Tokens: [T1, T2, T3, T4, ..., Tn]
                           ↓
Tokenization (tiktoken, cl100k_base encoding)
                           ↓
┌─────────────────────────────────────────────┐
│         SLIDING WINDOW WITH OVERLAP         │
└─────────────────────────────────────────────┘

Window 1: [T1...T1200]           Chunk 1
          ├────────────────────┤
          [100 overlap →]
                    Window 2: [T1101...T2300]  Chunk 2
                              ├────────────────────┤
                              [100 overlap →]
                                        Window 3: [T2201...T3400]
                                                  ├────────────────────┤
```

#### Параметры

```yaml
strategy: tokens
size: 1200              # tokens per chunk
overlap: 100            # overlap tokens
encoding_model: cl100k_base  # tiktoken encoding (GPT-4, GPT-3.5)
```

#### Алгоритм

```python
Algorithm: TokenBasedChunking

Input: document_text, chunk_size=1200, overlap=100
Output: chunks[]

1. Encode document:
   tokens = tiktoken.encode(document_text, encoding="cl100k_base")
   # Example: "Hello world" → [9906, 1917]

2. Initialize sliding window:
   start_idx = 0
   chunks = []

3. Extract chunks with overlap:
   WHILE start_idx < len(tokens):
     # Define chunk window
     end_idx = min(start_idx + chunk_size, len(tokens))
     chunk_tokens = tokens[start_idx:end_idx]

     # Decode tokens back to text
     chunk_text = tiktoken.decode(chunk_tokens)
     chunks.append(chunk_text)

     # Slide window (advance by chunk_size - overlap)
     start_idx += (chunk_size - overlap)

4. Return chunks

Complexity: O(n) where n = document length in tokens
```

#### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Semantic Preservation** | 95-98% |
| **Boundary Awareness** | Token-level (может разрывать слова) |
| **Context Continuity** | High (благодаря overlap) |
| **Determinism** | Полный (одинаковый вход → одинаковые chunks) |
| **Language Dependency** | Low (работает с любыми языками) |

#### Преимущества

✅ **Предсказуемость размера**: Chunks всегда ~1200 tokens
- Оптимизация batch processing
- Контроль стоимости LLM вызовов
- Равномерная нагрузка на embedding model

✅ **Совместимость с LLM token limits**:
- GPT-4: chunk (1200) + prompt (500) + output (500) = 2200 < 8192 limit
- Безопасный margin для вариативности

✅ **Эффективность**:
- O(n) complexity
- Быстрая tokenization через tiktoken (Rust backend)
- Низкие накладные расходы

✅ **Overlap control**:
- Точный контроль overlap в tokens
- Семантическая непрерывность

#### Недостатки

❌ **Отсутствие семантической boundary awareness**:
```
Example:
Chunk boundary может быть:
"...GraphRAG uses large language mo|dels to extract..."
                                   ↑ Разрыв внутри слова

После decode:
Chunk 1: "...GraphRAG uses large language mo"
Chunk 2: "dels to extract..."
         ↑ Нарушение word boundary (tiktoken обычно это обрабатывает корректно)
```

❌ **Может разрывать смысловые единицы**:
- Предложения
- Параграфы
- Списки

#### Когда использовать

**Рекомендуется для**:
- ✓ General-purpose RAG
- ✓ Large document collections (consistency важна)
- ✓ Многоязычные corpus
- ✓ Когда важна предсказуемость cost

**Конфигурация по use case**:

```yaml
# FAQ / Short documents
size: 800
overlap: 100

# General knowledge base (DEFAULT)
size: 1200
overlap: 100

# Long-form content (books, reports)
size: 1500
overlap: 200

# Cost optimization
size: 2000
overlap: 100

# Maximum quality (expensive)
size: 800
overlap: 150
```

#### Пример

```
Input document (simplified):
"GraphRAG is a knowledge graph construction system. It uses LLMs to extract
entities and relationships from text. The system creates hierarchical
communities through clustering."

Tokens (cl100k_base): [9906, 4760, 3891, 374, 264, 6677, 4876, 8246, 1887, ...]
Total: ~35 tokens (simplified)

With chunk_size=15, overlap=5:

Chunk 1 (tokens 0-14):
  Text: "GraphRAG is a knowledge graph construction system. It uses"

Chunk 2 (tokens 10-24):  ← overlap 5 tokens
  Text: "system. It uses LLMs to extract entities and relationships"

Chunk 3 (tokens 20-34):  ← overlap 5 tokens
  Text: "entities and relationships from text. The system creates hierarchical"

Note: "system. It uses" appears in both Chunk 1 and 2 (overlap)
```

### Стратегия 2: Sentence-based Chunking

**Концепция**: Разбиение по **семантическим границам предложений** с использованием NLP.

#### Концептуальная модель

```
Document Text
      ↓
NLP Sentence Tokenization (NLTK)
      ↓
┌─────────────────────────────────────────────┐
│         SENTENCE BOUNDARY DETECTION         │
└─────────────────────────────────────────────┘

Input: "First sentence. Second sentence! Third? Fourth."

Sentence Tokenizer:
  → Sentence 1: "First sentence."
  → Sentence 2: "Second sentence!"
  → Sentence 3: "Third?"
  → Sentence 4: "Fourth."

Each sentence = one chunk (no overlap)
```

#### Параметры

```yaml
strategy: sentence
# No size/overlap parameters - sentences are natural units
```

#### Алгоритм

```python
Algorithm: SentenceBasedChunking

Input: document_text
Output: chunks[]

1. Sentence segmentation:
   sentences = nltk.sent_tokenize(document_text)
   # Uses Punkt sentence tokenizer
   # Language-aware (trained on corpora)

2. Create chunks:
   chunks = []
   FOR each sentence in sentences:
     chunk = TextChunk(
       text=sentence,
       source_doc_idx=document_id
     )
     chunks.append(chunk)

3. Return chunks

Complexity: O(n) where n = document length
Dependency: NLTK Punkt tokenizer (pre-trained model)
```

#### Семантические характеристики

| Характеристика | Значение |
|---|---|
| **Semantic Preservation** | 99%+ (natural boundaries) |
| **Boundary Awareness** | Sentence-level (optimal) |
| **Context Continuity** | Low (no overlap between sentences) |
| **Determinism** | High (зависит от NLTK model) |
| **Language Dependency** | High (требует языковой модели) |

#### Преимущества

✅ **Естественные семантические границы**:
```
Good chunking:
Chunk 1: "GraphRAG is a knowledge graph construction system."
Chunk 2: "It uses LLMs to extract entities and relationships from text."
Chunk 3: "The system creates hierarchical communities through clustering."

Each chunk = полная законченная мысль
```

✅ **Максимальная семантическая целостность**:
- Никогда не разрывает предложения
- Сохраняет грамматическую структуру
- LLM получает полные смысловые единицы

✅ **Гибкость размера**:
- Короткие sentences → fine-grained chunks
- Длинные sentences → более крупные chunks
- Адаптация к стилю документа

#### Недостатки

❌ **Вариативность размера chunks**:
```
Chunk 1: "Hello." (1 token)
Chunk 2: "This is a very long sentence with multiple clauses, subclauses,
         and various grammatical structures that can span many tokens
         potentially exceeding optimal LLM processing windows." (45 tokens)

Проблемы:
- Непредсказуемость batch sizes
- Некоторые sentences могут превышать LLM limits
- Сложность cost estimation
```

❌ **Отсутствие overlap**:
- Потеря контекста между предложениями
- Entities на границах могут быть упущены
- Relationships across sentences могут не извлечься

❌ **Language dependency**:
- Требует NLTK модели для каждого языка
- Качество зависит от качества sentence tokenizer
- Сложнее для многоязычных corpus

❌ **Overhead NLP обработки**:
- Медленнее token-based (требует parsing)
- Зависимость от внешних моделей

#### Когда использовать

**Рекомендуется для**:
- ✓ Structured text (научные статьи, новости)
- ✓ Citation-heavy documents
- ✓ Когда важна точность attribution
- ✓ Single-language corpora

**Не рекомендуется для**:
- ✗ Неструктурированный text (chat logs, transcripts)
- ✗ Multilingual documents
- ✗ Very short or very long sentences
- ✗ High-throughput production systems

#### Пример

```
Input document:
"GraphRAG is a knowledge graph construction system. It uses LLMs to extract
entities and relationships from text. The system creates hierarchical
communities through clustering. This enables powerful graph-based retrieval."

NLTK sentence tokenization:

Chunk 1:
  Text: "GraphRAG is a knowledge graph construction system."
  Tokens: ~9

Chunk 2:
  Text: "It uses LLMs to extract entities and relationships from text."
  Tokens: ~12

Chunk 3:
  Text: "The system creates hierarchical communities through clustering."
  Tokens: ~9

Chunk 4:
  Text: "This enables powerful graph-based retrieval."
  Tokens: ~7

Total chunks: 4 (vs 2-3 with token-based @ 1200 tokens)
```

### Сравнение стратегий

| Аспект | Token-based | Sentence-based |
|---|---|---|
| **Semantic Boundaries** | Token-level | Sentence-level ✓ |
| **Chunk Size** | Fixed (~1200) ✓ | Variable (1-100+) |
| **Overlap Support** | Yes ✓ | No |
| **Context Continuity** | High (overlap) ✓ | Low (no overlap) |
| **Predictability** | High ✓ | Low |
| **Speed** | Fast ✓ | Moderate |
| **Language Support** | Universal ✓ | Language-specific |
| **Cost Estimation** | Easy ✓ | Difficult |
| **Best For** | General RAG ✓ | Structured text |

## Концептуальный процесс: От документа до chunks

### Полный жизненный цикл

```
┌────────────────────────────────────────────────────────────┐
│                   CHUNKING LIFECYCLE                       │
└────────────────────────────────────────────────────────────┘

1. INPUT PREPARATION
   ↓
   Raw Document(s)
   ├─ Format: txt, pdf, csv, json
   ├─ Size: Variable (100-100K+ tokens)
   └─ Encoding: UTF-8

2. PRE-PROCESSING
   ↓
   ├─ Text extraction (from PDF, HTML, etc.)
   ├─ Normalization (whitespace, encoding)
   ├─ Metadata attachment (doc_id, source, title)
   └─ Validation (empty documents, encoding errors)

3. STRATEGY SELECTION
   ↓
   Choose chunking strategy:
   ├─ Token-based (default) → Go to 4a
   └─ Sentence-based → Go to 4b

4a. TOKEN-BASED CHUNKING
   ↓
   ├─ Tokenize with tiktoken (cl100k_base)
   ├─ Apply sliding window (size=1200, overlap=100)
   ├─ Decode chunks back to text
   └─ Create TextChunk objects

4b. SENTENCE-BASED CHUNKING
   ↓
   ├─ Segment with NLTK sent_tokenize
   ├─ Create one chunk per sentence
   └─ Create TextChunk objects

5. CHUNK ENRICHMENT
   ↓
   For each chunk, attach metadata:
   ├─ chunk_id (unique identifier)
   ├─ source_doc_indices (origin document(s))
   ├─ n_tokens (chunk length)
   ├─ position (index in document)
   └─ text_chunk (actual text content)

6. VALIDATION & QUALITY CHECKS
   ↓
   ├─ Size validation (within limits?)
   ├─ Empty chunk detection
   ├─ Encoding verification
   └─ Overlap correctness (token strategy)

7. OUTPUT
   ↓
   List[TextChunk]
   ↓
   [Downstream: Entity Extraction, Embedding, ...]
```

### Концептуальные фазы chunking

#### Фаза 1: Tokenization (для token-based)

**Концепция**: Преобразование текста в последовательность числовых токенов.

```
Text: "GraphRAG system"
       ↓ tiktoken encoding (cl100k_base)
Tokens: [9906, 4760, 3891, 1887]
         ↑     ↑     ↑     ↑
       Graph  RAG  (space) system

Encoding properties:
- Subword tokenization (BPE - Byte Pair Encoding)
- Vocabulary: ~100K tokens
- Handles rare words through subwords
- Language-agnostic (works with any UTF-8 text)
```

**Семантическое сохранение**:
- Lossless: decode(encode(text)) == text
- Preserves ALL information
- Foundation for deterministic chunking

#### Фаза 2: Windowing

**Концепция**: Извлечение подпоследовательностей токенов фиксированного размера с overlap.

```
Token sequence: [T1, T2, T3, T4, T5, T6, T7, T8, T9, T10]

Window configuration:
- size = 4 tokens
- overlap = 1 token

Window extraction:
┌─────────────┐
│ T1 T2 T3 T4 │  Window 1 → Chunk 1
└─────────────┘
       └┬┘  ← overlap (T4)
      ┌─────────────┐
      │ T4 T5 T6 T7 │  Window 2 → Chunk 2
      └─────────────┘
             └┬┘  ← overlap (T7)
            ┌─────────────┐
            │ T7 T8 T9 T10│  Window 3 → Chunk 3
            └─────────────┘

Математика overlap:
- stride = size - overlap
- next_start = current_start + stride
- num_chunks = ⌈(total_tokens - overlap) / stride⌉
```

**Семантический эффект overlap**:

```
Without overlap:
Chunk 1: "GraphRAG uses LLMs"
Chunk 2: "to extract entities"
         ↑ Lost context: Что именно extracts entities?

With overlap:
Chunk 1: "GraphRAG uses LLMs to"
Chunk 2: "LLMs to extract entities"
         ↑ Preserved: "LLMs to extract" - полный контекст
```

#### Фаза 3: Decoding

**Концепция**: Преобразование токенов обратно в читаемый текст.

```
Chunk tokens: [9906, 4760, 3891]
       ↓ tiktoken decode
Chunk text: "GraphRAG system"

Декодирование preserves:
- Original characters
- Whitespace
- Punctuation
- Special symbols
```

**Важно**: Декодирование восстанавливает **точный** оригинальный текст (в пределах chunk boundaries).

#### Фаза 4: Metadata Attachment

**Концепция**: Обогащение chunks метаданными для traceability.

```python
TextChunk structure:
{
  "chunk_id": "doc_1_chunk_3",
  "text_chunk": "GraphRAG uses LLMs to extract entities...",
  "source_doc_indices": [1],  # From document #1
  "n_tokens": 1200,
  "chunk_index": 3,  # 3rd chunk in document
  "start_char_offset": 2400,  # Character position in original doc
  "end_char_offset": 8600,
  "overlap_with_prev": 100,  # tokens
  "overlap_with_next": 100
}
```

**Семантическая ценность metadata**:
- **Traceability**: Chunk → Source Document
- **Provenance**: Откуда взялась информация
- **Deduplication**: Обнаружение дублирующихся chunks
- **Attribution**: Ссылки на источники в ответах

## Влияние chunking на downstream процессы

### 1. Entity Extraction

**Влияние размера chunk**:

```
Small chunks (300 tokens):
  Input: "Microsoft acquired GitHub."
  Extracted Entities: [Microsoft, GitHub]
  ✓ High precision
  ✗ Может пропустить дальние relationships

Large chunks (3000 tokens):
  Input: "Microsoft, founded in 1975... [2500 tokens later] ...GitHub was acquired."
  Extracted Entities: [Microsoft, GitHub, Bill Gates, ... 50+ entities]
  ✓ Rich context
  ✗ May dilute focus, miss some entities (LLM attention limits)

Optimal chunks (1200 tokens):
  Input: "Microsoft acquired GitHub in 2018 for $7.5B. The acquisition strengthened..."
  Extracted Entities: [Microsoft, GitHub, Acquisition event]
  ✓ Balanced precision/recall
  ✓ Sufficient context for relationships
```

**Влияние overlap**:

```
Entity on boundary WITHOUT overlap:
Chunk 1: "...Microsoft is a tech company."
Chunk 2: "GitHub is a code hosting platform."
         ↑ Lost: Microsoft-GitHub relationship

Entity on boundary WITH overlap:
Chunk 1: "...Microsoft is a tech company. Microsoft acquired GitHub."
Chunk 2: "Microsoft acquired GitHub. GitHub is a code hosting platform."
         ↑ Preserved: Both chunks see "Microsoft acquired GitHub"

Result: Higher probability of extracting "Microsoft --[acquired]--> GitHub" relationship
```

### 2. Relationship Extraction

**Cross-chunk relationships**:

```
Document: "Company A partnered with Company B. Later, Company B collaborated with Company C."

Token chunking (size=15, overlap=5):
Chunk 1: "Company A partnered with Company B."
Chunk 2: "Company B. Later, Company B collaborated with Company C."

Extracted relationships:
  From Chunk 1: A --[partnered with]--> B
  From Chunk 2: B --[collaborated with]--> C

Overlap enabled: Both relationships captured despite different chunks
```

**Влияние granularity**:

| Chunk Size | Direct Relationships | Inferred Relationships | Noise |
|---|---|---|---|
| 300 tokens | ★★★ | ★☆☆ | Low |
| 1200 tokens | ★★★ | ★★☆ | Low |
| 3000 tokens | ★★☆ | ★★★ | Medium |

### 3. Embedding Quality

**Semantic coherence chunks**:

```
Good chunk (coherent topic):
"GraphRAG uses LLMs for entity extraction. The extraction process involves
prompting the LLM with text and entity types. The model returns structured
entity data."

Embedding: [0.23, -0.41, 0.56, ...]
Topic cluster: "LLM-based entity extraction"
Quality: High (coherent semantic meaning)

Bad chunk (topic shift mid-chunk):
"GraphRAG uses LLMs for entity extraction. In other news, the weather today
is sunny. Machine learning is advancing rapidly. Cats are popular pets."

Embedding: [0.05, -0.12, 0.08, ...]
Topic cluster: Unclear (mixed topics)
Quality: Low (semantic noise)
```

**Chunk size влияние на embeddings**:
- **Too small (100 tokens)**: Insufficient context, noisy embeddings
- **Optimal (500-1500 tokens)**: Rich semantic signal
- **Too large (5000+ tokens)**: Topic drift, diluted embeddings

### 4. Community Detection

**Chunk size влияет на community granularity**:

```
Fine chunks (300 tokens) → Many specific entities → Fine-grained communities
  Example communities:
  - "GPT-4 model architecture"
  - "Entity extraction techniques"
  - "Graph clustering algorithms"

Coarse chunks (2000 tokens) → Fewer, broader entities → Coarse communities
  Example communities:
  - "LLM technologies and applications"
  - "GraphRAG system components"
```

## Оптимизация chunking стратегии

### Матрица выбора стратегии

| Use Case | Strategy | Size | Overlap | Reasoning |
|---|---|---|---|---|
| **General RAG** | Tokens | 1200 | 100 | Balanced quality/cost |
| **FAQ System** | Sentence | N/A | N/A | Natural Q&A boundaries |
| **Scientific Papers** | Tokens | 800 | 150 | Preserve technical context |
| **News Articles** | Sentence | N/A | N/A | Article structure |
| **Chat Logs** | Tokens | 600 | 50 | Conversational flow |
| **Legal Documents** | Tokens | 1000 | 200 | Preserve citations |
| **Books** | Tokens | 1500 | 200 | Long-form narrative |
| **Code Documentation** | Tokens | 1000 | 100 | Code structure |
| **Social Media** | Tokens | 400 | 50 | Short-form content |

### Настройка параметров под домен

#### Domain: Scientific Papers

```yaml
chunks:
  strategy: tokens
  size: 800              # Smaller for precision
  overlap: 150           # Higher overlap (18.75%)
  encoding_model: cl100k_base

Reasoning:
- Technical terms need precise context
- Citations must be preserved
- Higher overlap ensures methodology context
- Smaller chunks → better entity precision

Expected outcomes:
- Entity extraction precision: +15%
- Relationship extraction recall: +10%
- Cost: +30% (more chunks)
```

#### Domain: News Articles

```yaml
chunks:
  strategy: sentence
  # No size/overlap for sentence strategy

Reasoning:
- News articles have clear sentence structure
- Each sentence is semantically complete
- Important for quote attribution
- Good for citation/source tracking

Expected outcomes:
- Perfect attribution accuracy
- Higher sentence-level granularity
- Variable chunk sizes (acceptable for news)
```

#### Domain: Long-form Books

```yaml
chunks:
  strategy: tokens
  size: 1500             # Larger for narrative flow
  overlap: 200           # High overlap (13.3%)
  encoding_model: cl100k_base

Reasoning:
- Narrative context spans multiple paragraphs
- Character references need long context
- Plot elements connect across pages
- Higher overlap preserves story flow

Expected outcomes:
- Better narrative entity linking
- Improved character relationship extraction
- Moderate cost (fewer large chunks)
```

### Практические рекомендации

#### 1. Выбор chunk_size

**Формула оптимального размера**:

```
optimal_size = min(
  LLM_context_window * 0.15,  # 15% of LLM capacity
  embedding_model_max * 0.9,   # 90% of embedding limit
  2000                         # Upper bound for quality
)

Примеры:
GPT-4 (8K context): min(8000*0.15, 8191*0.9, 2000) = 1200 ✓
GPT-4-turbo (128K): min(128000*0.15, 8191*0.9, 2000) = 2000
text-embedding-3-small (8191): min(..., 8191*0.9, 2000) = 2000
```

**Адаптация к длине документов**:

```python
def adaptive_chunk_size(avg_doc_length):
    if avg_doc_length < 500:
        return 300  # Short documents: small chunks
    elif avg_doc_length < 2000:
        return 800  # Medium documents
    elif avg_doc_length < 10000:
        return 1200  # Standard documents (default)
    else:
        return 1500  # Long documents
```

#### 2. Выбор overlap

**Правило большого пальца**:

```
overlap = chunk_size * 0.08 to 0.15

Examples:
chunk_size = 800  → overlap = 64-120   (recommend: 80)
chunk_size = 1200 → overlap = 96-180   (recommend: 100)
chunk_size = 1500 → overlap = 120-225  (recommend: 150)
chunk_size = 2000 → overlap = 160-300  (recommend: 200)
```

**Адаптация к типу контента**:

```python
def adaptive_overlap(chunk_size, content_type):
    base_overlap = chunk_size * 0.10

    if content_type == "technical":
        return base_overlap * 1.5  # Technical needs more context
    elif content_type == "narrative":
        return base_overlap * 1.3  # Stories need flow
    elif content_type == "structured":
        return base_overlap * 0.8  # Lists/tables need less
    else:
        return base_overlap  # Default
```

#### 3. Мониторинг качества chunking

**Метрики для отслеживания**:

```yaml
Chunk Quality Metrics:

1. Size Distribution:
   - Mean chunk size: ~1200 tokens (target)
   - Std deviation: <100 tokens (consistency)
   - Min/Max: 50-1300 tokens (acceptable range)

2. Overlap Effectiveness:
   - Chunks with shared entities: 60-80% (good overlap)
   - Boundary entity loss rate: <5% (overlap working)

3. Semantic Coherence:
   - Topic consistency score: >0.7 (within chunk)
   - Cross-chunk similarity: 0.3-0.5 (some overlap, not duplicate)

4. Downstream Impact:
   - Entity extraction recall: >85%
   - Relationship extraction F1: >0.80
   - Embedding clustering quality: >0.75 (silhouette score)
```

**Пример мониторинга**:

```python
def evaluate_chunking_quality(chunks):
    metrics = {
        "num_chunks": len(chunks),
        "avg_size": mean([c.n_tokens for c in chunks]),
        "size_std": std([c.n_tokens for c in chunks]),
        "min_size": min([c.n_tokens for c in chunks]),
        "max_size": max([c.n_tokens for c in chunks]),

        # Advanced metrics
        "chunks_with_overlap": count_overlap_entities(chunks),
        "semantic_coherence": calculate_coherence(chunks),
    }

    # Quality gates
    assert metrics["avg_size"] between (1100, 1300), "Size drift detected"
    assert metrics["size_std"] < 150, "High variance in chunk sizes"

    return metrics
```

## Специальные случаи chunking

### 1. Structured Documents (Tables, Lists)

**Проблема**: Token-based chunking может разрывать структуры.

```
Плохо:
Chunk 1:
  | Name    | Age | Cit

Chunk 2:
  y        |
  | Alice  | 30  | NYC |

Хорошо (structure-aware):
Chunk 1:
  | Name    | Age | City    |
  | Alice   | 30  | NYC     |
  | Bob     | 25  | LA      |
```

**Решение**: Preprocessing для detection структур:

```python
def structure_aware_chunking(text, chunk_size=1200, overlap=100):
    # Detect tables, lists, code blocks
    structures = detect_structures(text)

    chunks = []
    current_pos = 0

    for structure in structures:
        # Add preceding text as normal chunks
        preceding_text = text[current_pos:structure.start]
        chunks.extend(token_chunk(preceding_text, chunk_size, overlap))

        # Add entire structure as single chunk (if reasonable size)
        if structure.size < chunk_size * 1.5:
            chunks.append(structure.text)
        else:
            # Split large structures intelligently (by rows, items)
            chunks.extend(split_structure(structure, chunk_size))

        current_pos = structure.end

    return chunks
```

### 2. Multi-lingual Documents

**Проблема**: Different languages have different tokenization characteristics.

```
English: "Hello world" → 2 tokens
Chinese: "你好世界" → 4 tokens (character-based)
Russian: "Привет мир" → 4 tokens

Same semantic content, different token counts!
```

**Решение**: Language-adaptive chunking:

```python
def multilingual_chunking(text, target_semantic_size=1200):
    # Detect language
    language = detect_language(text)

    # Adjust chunk size based on language token density
    size_multipliers = {
        "en": 1.0,
        "zh": 0.6,  # Chinese is more token-dense
        "ru": 0.8,
        "ja": 0.6,
        "ar": 0.7,
    }

    chunk_size = int(target_semantic_size * size_multipliers.get(language, 1.0))

    return token_chunk(text, chunk_size, overlap=chunk_size*0.1)
```

### 3. Code + Documentation

**Проблема**: Code should not be split mid-function.

```
Плохо:
Chunk 1:
  def process_data(input):
      result = []
      for item in input:

Chunk 2:
          processed = transform(item)
          result.append(processed)
      return result

Хорошо:
Chunk 1:
  def process_data(input):
      result = []
      for item in input:
          processed = transform(item)
          result.append(processed)
      return result
```

**Решение**: AST-aware chunking:

```python
def code_aware_chunking(code, chunk_size=1200):
    # Parse code into AST
    ast_tree = parse_code(code)

    # Get function/class boundaries
    code_units = extract_code_units(ast_tree)

    chunks = []
    for unit in code_units:
        if unit.size < chunk_size:
            chunks.append(unit.text)  # Whole function
        else:
            # Large function: split by methods/sections
            chunks.extend(split_code_unit(unit, chunk_size))

    return chunks
```

## Будущие направления

### 1. Semantic-aware Chunking

**Концепция**: Используйте embedding-based similarity для определения semantic boundaries.

```
Algorithm: SemanticChunking

1. Split text into sentences
2. Compute embeddings for each sentence
3. Calculate similarity between consecutive sentences
4. Chunk boundary = where similarity drops below threshold

Example:
Sent 1: "GraphRAG is a system..."     ─┐
Sent 2: "It uses LLMs for extraction..." │ High similarity → Same chunk
Sent 3: "Entity extraction is crucial..."─┘

Sent 4: "In other news, the weather..." ─┐ Low similarity → New chunk
Sent 5: "Today's forecast shows rain..." ─┘
```

### 2. Adaptive Chunking

**Концепция**: Динамически adjust chunk size based на content complexity.

```
def adaptive_chunking(text):
    complexity = measure_complexity(text)
    # Complexity factors: vocabulary diversity, sentence length, technical terms

    if complexity > 0.8:  # Very complex
        chunk_size = 800  # Smaller chunks for precision
    elif complexity > 0.5:  # Moderate
        chunk_size = 1200  # Standard
    else:  # Simple
        chunk_size = 1800  # Larger chunks (efficiency)

    return token_chunk(text, chunk_size)
```

### 3. Hierarchical Chunking

**Концепция**: Multi-level chunks для different granularity needs.

```
Document
  ├─ Level 1 chunks (4000 tokens, 500 overlap)
  │   ├─ Level 2 chunks (1200 tokens, 100 overlap)
  │   │   └─ Level 3 chunks (300 tokens, 30 overlap)

Use cases:
- Level 1: Document-level embeddings, summarization
- Level 2: Standard entity extraction (default)
- Level 3: Fine-grained attribution, citations
```

## Заключение

Chunking — это **фундаментальное семантическое преобразование**, которое определяет качество всего GraphRAG pipeline. Правильный выбор стратегии и параметров критически важен для:

**Качества**:
- Entity extraction precision/recall
- Relationship detection
- Embedding coherence

**Эффективности**:
- LLM cost management
- Processing throughput
- Storage requirements

**Функциональности**:
- Source attribution
- Citation accuracy
- Multi-hop reasoning

### Ключевые takeaways

1. **Token-based chunking** — default choice для general-purpose RAG (predictability, consistency)

2. **Overlap 8-15%** — optimal баланс context preservation vs cost

3. **Chunk size 1000-1500 tokens** — sweet spot для LLM entity extraction

4. **Monitor metrics** — track chunk quality и adjust параметры

5. **Domain adaptation** — customize chunking для specific use cases

### Рекомендуемые конфигурации

```yaml
# Production-ready GraphRAG chunking
chunks:
  strategy: tokens
  size: 1200
  overlap: 100
  encoding_model: cl100k_base

# Rationale:
# - Token-based: Consistent, predictable, fast
# - 1200 tokens: Optimal for GPT-4 entity extraction
# - 100 overlap: 8.3% - preserves boundary context
# - cl100k_base: Standard encoding for OpenAI models
```

---

## Ссылки

- **Код**: `graphrag/index/text_splitting/text_splitting.py`
- **Конфигурация**: `graphrag/config/defaults.py` → `ChunksDefaults`
- **Стратегии**: `graphrag/index/operations/chunk_text/strategies.py`
- **Основная документация**: `spec/01-document-ingestion-chunking.md`
- **Преобразования**: `spec/transform/01-indexing-transformations.md` (T1: Text Chunking)
