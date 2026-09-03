# PaperPilot — Ingestion, Embeddings & Vector Store

This README documents the current RAG proof-of-concept layers implemented in PaperPilot:

```text
PDF
 ↓
Docling
 ↓
Markdown / structured text
 ↓
Cleaning
 ↓
Section-aware chunking
 ↓
chunks.json
 ↓
┌───────────────────────────────┐
│                               │
├── Cohere Embeddings           │
├── BM25                        │
│                               │
└───────────────┬───────────────┘
                ↓
        Hybrid Retrieval
                ↓
               RRF
                ↓
             Qdrant
```

> **Scope:** This document covers the local PoC implementation of ingestion, embeddings/lexical retrieval, and vector storage. Cloud/deployment is intentionally excluded.

---

## 1. Architecture Overview

PaperPilot separates the RAG pipeline into independent responsibilities.

| Layer             | Responsibility                           | Technology          |
| ----------------- | ---------------------------------------- | ------------------- |
| Ingestion         | Parse, clean, and chunk source documents | Docling + Python    |
| Embeddings        | Convert text into semantic vectors       | Cohere `embed-v4.0` |
| Lexical Retrieval | Exact/keyword-oriented retrieval         | BM25                |
| Vector Store      | Store and search embeddings              | Qdrant              |
| Fusion            | Combine semantic and lexical rankings    | RRF                 |

The important design principle is:

```text
Document processing ≠ embedding ≠ storage ≠ retrieval
```

Keeping these responsibilities separate makes the system easier to test and replace.

---

# 2. Layer 1 — Ingestion

## 2.1 Purpose

The ingestion layer transforms an original document into structured, searchable chunks.

```text
PDF
 ↓
Docling
 ↓
Parsed representation
 ↓
Cleaning
 ↓
Section-aware chunking
 ↓
chunks.json
```

The output of ingestion becomes the input contract for the embedding and retrieval layers.

---

## 2.2 PDF Parsing with Docling

Docling is responsible for understanding the source PDF and extracting useful document structure.

A document can contain:

- titles
- headings
- paragraphs
- lists
- tables
- captions
- page-related information
- other structural elements

For this PoC, the parsed content is converted into Markdown.

Example:

```markdown
# Authentication

Authentication allows users to access the system.

## Login

Users enter their email and password.

## Token Management

The system creates access tokens.
```

Markdown is easier to process in Python than raw PDF data and preserves meaningful structure such as headings.

---

## 2.3 Why Structure Matters

A simple PDF-to-text extractor can produce:

```text
Authentication
Authentication allows users to access the system.
Login
Users enter their email and password.
Token Management
The system creates access tokens.
```

The text is still present, but the hierarchy is weaker.

With structured Markdown:

```text
Authentication
└── Login
└── Token Management
```

the chunking layer can associate each chunk with its section.

This becomes useful later for:

- retrieval
- filtering
- source display
- citations
- debugging
- context assembly

---

# 3. Cleaning

After parsing, the document is normalized before chunking.

Typical cleaning operations can include:

- removing unnecessary whitespace
- normalizing line breaks
- removing parser artifacts
- preserving headings
- preserving meaningful content

The rule is:

> Remove noise without destroying information required for retrieval.

For example:

```text
Authentication


Users authenticate.


Login
```

can be normalized while keeping:

```text
# Authentication

Users authenticate.

## Login
```

Headings should generally be preserved because they provide context.

---

# 4. Section-Aware Chunking

PaperPilot currently uses section-aware chunking.

Instead of blindly splitting text into fixed-size pieces, chunks retain information about the document section they belong to.

Example:

```text
# Authentication

Authentication allows users to access the system.

## Login

Users enter their email and password.

## Token Management

The system creates access tokens.
```

Possible chunks:

```text
Chunk 1
Section: Authentication
Text: Authentication allows users to access the system.
```

```text
Chunk 2
Section: Authentication > Login
Text: Users enter their email and password.
```

```text
Chunk 3
Section: Authentication > Token Management
Text: The system creates access tokens.
```

---

# 5. Why Section-Aware Chunking Is Important

Consider:

```text
Users enter their email and password.
```

Without context, this sentence could belong to:

- login
- registration
- password reset
- account recovery

With metadata:

```json
{
  "section": "Authentication > Login"
}
```

the retrieval system knows where the text came from.

This metadata can later be carried through:

```text
Chunk
 ↓
Embedding
 ↓
Qdrant payload
 ↓
Retrieved result
 ↓
Context
 ↓
Citation/source
```

---

# 6. Chunk Data Contract

The ingestion layer produces:

```text
data/processed/chunks.json
```

A chunk has the following conceptual structure:

```json
{
  "chunk_id": "unique-id",
  "text": "Users enter their email and password.",
  "metadata": {
    "doc_id": "paper-001",
    "title": "Example Document",
    "section": "Authentication > Login"
  }
}
```

### `chunk_id`

A unique identifier for tracking the chunk throughout the pipeline.

```text
chunks.json
 ↓
embedding
 ↓
Qdrant
 ↓
retrieval
 ↓
final answer
```

### `text`

The actual text that is embedded and searched.

### `metadata`

Information describing the source/context of the chunk.

---

# 7. Ingestion Output

At the end of Layer 1:

```text
data/
└── processed/
    └── chunks.json
```

The embedding layer does not need to understand the original PDF.

It only needs:

```text
chunks.json
```

This gives ingestion a clean boundary.

---

# 8. Layer 2 — Embeddings

## 8.1 Purpose

Embeddings transform text into numerical representations.

Example:

```text
"The authentication service validates credentials."
```

becomes a vector such as:

```text
[
    0.021,
   -0.183,
    0.742,
    ...
]
```

PaperPilot currently uses Cohere's `embed-v4.0` with:

```python
output_dimension=1024
```

Therefore each stored embedding has 1024 dimensions.

---

# 9. Why Embeddings Are Needed

Keyword matching only considers words.

Semantic retrieval considers meaning.

For example:

```text
Document:
"The authentication service validates user credentials."

Query:
"How does the system verify a user's identity?"
```

The wording is different, but the concepts are related.

An embedding maps both texts into vector representations so semantic similarity can be measured.

Conceptually:

```text
Document
   ↓
Embedding
   ↓
Vector

Query
   ↓
Embedding
   ↓
Vector

Vectors
   ↓
Similarity comparison
```

---

# 10. Document Embeddings

When indexing chunks, the system uses:

```python
input_type="search_document"
```

The pipeline is:

```text
chunks.json
 ↓
extract chunk text
 ↓
Cohere embed-v4.0
 ↓
document vectors
 ↓
embedded_chunks.json
```

A resulting record conceptually looks like:

```json
{
  "chunk_id": "abc123",
  "text": "Authentication validates user credentials.",
  "metadata": {
    "section": "Authentication"
  },
  "embedding": [
    0.012,
    -0.031,
    0.721
  ]
}
```

The actual embedding contains 1024 values.

---

# 11. Query Embeddings

When a user asks a question, the query is embedded differently:

```python
input_type="search_query"
```

For example:

```text
User:
"How does authentication work?"
```

becomes:

```text
User query
 ↓
Cohere
 ↓
search_query
 ↓
query vector
 ↓
Qdrant
```

The important distinction is:

```text
Documents → search_document
Queries   → search_query
```

Do not accidentally use the document mode for user queries.

---

# 12. Cohere Integration

The current client setup is conceptually:

```python
client = cohere.ClientV2(
    api_key=API_KEY
)
```

The model is:

```python
MODEL = "embed-v4.0"
```

Document embeddings use:

```python
response = client.embed(
    model=MODEL,
    texts=texts,
    input_type="search_document",
    output_dimension=1024,
    embedding_types=["float"],
)
```

Query embeddings use:

```python
response = client.embed(
    model=MODEL,
    texts=[query],
    input_type="search_query",
    output_dimension=1024,
    embedding_types=["float"],
)
```

---

# 13. `embedded_chunks.json`

The embedding script reads:

```text
data/processed/chunks.json
```

and creates:

```text
data/processed/embedded_chunks.json
```

The transformation is:

```text
chunks.json
     ↓
extract text
     ↓
Cohere
     ↓
embedding
     ↓
add embedding to chunk
     ↓
embedded_chunks.json
```

This JSON file is useful as an intermediate PoC artifact.

Later, Qdrant becomes the primary vector storage/search layer.

---

# 14. Batch Embedding

The embedding script processes multiple chunks together:

```python
texts = [
    chunk["text"]
    for chunk in chunks
]
```

Then:

```python
embeddings = generate_embeddings(texts)
```

The resulting vectors are associated back with their original chunks:

```python
for chunk, embedding in zip(chunks, embeddings):
    chunk["embedding"] = embedding
```

This preserves the relationship:

```text
chunk_id
   ↕
text
   ↕
embedding
```

---

# 15. BM25 — Lexical Retrieval

Embeddings are only one retrieval method.

PaperPilot also implements BM25.

BM25 is a lexical retrieval method that scores documents based on query terms.

Conceptually:

```text
Documents
 ↓
Tokenization
 ↓
BM25 index
```

Then:

```text
Query
 ↓
Tokenization
 ↓
BM25 scoring
 ↓
Keyword results
```

---

# 16. Why BM25 + Embeddings?

The two retrieval approaches complement each other.

### Semantic retrieval

Useful for:

- paraphrases
- conceptual similarity
- natural-language questions
- related meanings

Example:

```text
Query:
"How does the system verify users?"

Document:
"The authentication service validates credentials."
```

### BM25

Useful for:

- exact terminology
- technical identifiers
- acronyms
- names
- exact keywords

Example:

```text
Query:
"JWT refresh_token"

Document:
"The system stores the refresh_token associated with the JWT."
```

Therefore:

```text
Semantic Retrieval + Lexical Retrieval
```

is stronger than relying on only one method.

---

# 17. BM25 Tokenization

The current PoC uses basic tokenization.

Conceptually:

```python
text.lower()
```

followed by extraction of word-like tokens.

For example:

```text
"Authentication uses JWT tokens."
```

becomes approximately:

```text
[
    "authentication",
    "uses",
    "jwt",
    "tokens"
]
```

The BM25 index is then constructed from tokenized chunks.

---

# 18. Layer 3 — Vector Store

## 18.1 Purpose

The vector store is responsible for storing embeddings and efficiently retrieving vectors similar to a query vector.

PaperPilot currently uses Qdrant.

The flow is:

```text
Document chunk
 ↓
Cohere embedding
 ↓
1024-dimensional vector
 ↓
Qdrant
```

Qdrant stores:

```text
Point
├── ID
├── Vector
└── Payload
```

---

# 19. Qdrant Point Structure

A PaperPilot point conceptually looks like:

```json
{
  "id": 1,
  "vector": [
    0.012,
    -0.031,
    0.721
  ],
  "payload": {
    "chunk_id": "abc123",
    "text": "Authentication validates credentials.",
    "metadata": {
      "section": "Authentication"
    }
  }
}
```

The actual vector contains 1024 dimensions.

The payload stores the original chunk information needed after retrieval.

---

# 20. Qdrant Collection

The current collection is:

```text
paperpilot_chunks
```

It is configured with:

```text
Vector size: 1024
Distance: COSINE
```

The vector size must match the Cohere embedding dimension:

```text
Cohere
output_dimension=1024
        ↓
Qdrant
size=1024
```

A mismatch will prevent correct vector insertion.

---

# 21. Why Cosine Similarity?

Semantic retrieval needs a way to compare a query vector with document vectors.

The current Qdrant collection uses cosine distance.

Conceptually:

```text
Query Vector
      │
      │ similarity
      ▼
Document Vectors
      │
      ▼
Rank nearest vectors
```

The closer the vectors are under the selected distance metric, the more relevant they are expected to be.

---

# 22. Local Qdrant

For the PoC, Qdrant is configured with local persistent storage.

Conceptually:

```text
data/
└── qdrant/
```

This avoids introducing cloud infrastructure while the architecture is being developed.

The goal is to demonstrate the RAG pipeline, not deployment.

---

# 23. Creating the Collection

The vector store creates a collection if it does not already exist.

Conceptually:

```python
client.create_collection(
    collection_name="paperpilot_chunks",
    vectors_config=VectorParams(
        size=1024,
        distance=Distance.COSINE,
    ),
)
```

If the collection already exists, it can be reused.

---

# 24. Loading Embeddings into Qdrant

The loading process is:

```text
embedded_chunks.json
        ↓
read chunks
        ↓
extract embedding
        ↓
create Qdrant point
        ↓
attach payload
        ↓
upsert
        ↓
paperpilot_chunks
```

Each point keeps:

```text
embedding
+
chunk metadata
```

This means vector search can return the actual text and source metadata.

---

# 25. Qdrant Semantic Search

At query time:

```text
User Query
    ↓
Cohere search_query
    ↓
Query Embedding
    ↓
Qdrant
    ↓
Nearest vectors
    ↓
Top-K results
```

The returned result contains:

```text
score
+
payload
```

The payload contains the chunk:

```text
chunk_id
text
metadata
```

---

# 26. Hybrid Retrieval

The complete current retrieval design is:

```text
                         User Query
                              │
                ┌─────────────┴─────────────┐
                │                           │
                ▼                           ▼
             Cohere                       BM25
          search_query                 lexical search
                │                           │
                ▼                           ▼
          Query vector                 BM25 ranking
                │                           │
                ▼                           │
             Qdrant                        │
                │                           │
                ▼                           │
        Semantic ranking                    │
                │                           │
                └─────────────┬─────────────┘
                              ▼
                             RRF
                              │
                              ▼
                       Hybrid Top-K
```

This is the main retrieval architecture currently implemented.

---

# 27. Reciprocal Rank Fusion

RRF combines rankings rather than raw scores.

The basic formula is:

```text
RRF score = 1 / (k + rank)
```

If the same chunk appears near the top of both lists, it receives contributions from both rankings.

Example:

```text
Semantic:

1. A
2. B
3. C
```

```text
BM25:

1. B
2. A
3. D
```

A and B are strong candidates because they perform well across both retrieval systems.

The fusion produces a unified ranking.

---

# 28. Why Not Add Raw Scores?

Semantic similarity and BM25 scores have different numerical scales.

For example:

```text
Semantic score = 0.84
BM25 score     = 7.42
```

Adding them directly:

```text
0.84 + 7.42
```

would not be a fair comparison.

RRF avoids this by using rank positions.

```text
Semantic rank
      +
BM25 rank
      ↓
RRF
      ↓
combined ranking
```

---

# 29. `HybridRetriever`

The `HybridRetriever` coordinates the retrieval process.

Its responsibilities are:

```text
1. Query embedding
2. Qdrant semantic search
3. BM25 keyword search
4. RRF fusion
5. Return final Top-K chunks
```

Conceptually:

```python
class HybridRetriever:

    def semantic_search(...):
        ...

    def keyword_search(...):
        ...

    def search(...):
        ...
```

The final `search()` method coordinates both retrieval paths.

---

# 30. Current Project Structure

The relevant project structure is:

```text
paperpilot/
│
├── app/
│   │
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── parser.py
│   │   ├── cleaner.py
│   │   ├── chunker.py
│   │   └── schemas.py
│   │
│   ├── embeddings/
│   │   ├── __init__.py
│   │   ├── cohere_embeddings.py
│   │   ├── bm25.py
│   │   ├── embed_chunks.py
│   │   └── hybrid_search.py
│   │
│   └── vectorstore/
│       ├── __init__.py
│       └── qdrant_store.py
│
├── data/
│   ├── uploads/
│   │
│   ├── processed/
│   │   ├── paper.md
│   │   ├── chunks.json
│   │   └── embedded_chunks.json
│   │
│   └── qdrant/
│
├── test_chunking.py
├── test_embeddings.py
├── test_bm25.py
├── create_qdrant_collection.py
├── load_qdrant.py
└── test_hybrid.py
```

---

# 31. File Responsibilities

## `ingestion/parser.py`

Responsible for:

```text
PDF → parsed document
```

---

## `ingestion/cleaner.py`

Responsible for:

```text
parsed document → normalized document
```

---

## `ingestion/chunker.py`

Responsible for:

```text
document → section-aware chunks
```

---

## `ingestion/schemas.py`

Defines the data structure for:

```text
DocumentChunk
ChunkMetadata
```

and related ingestion data.

---

## `embeddings/cohere_embeddings.py`

Responsible for:

```text
Cohere client
document embeddings
query embeddings
```

---

## `embeddings/embed_chunks.py`

Responsible for:

```text
chunks.json
 ↓
Cohere
 ↓
embedded_chunks.json
```

---

## `embeddings/bm25.py`

Responsible for:

```text
BM25 indexing
BM25 keyword retrieval
```

---

## `embeddings/hybrid_search.py`

Coordinates:

```text
Qdrant semantic search
+
BM25 search
+
RRF
```

---

## `vectorstore/qdrant_store.py`

Responsible for:

```text
Qdrant client
collection creation
vector insertion
vector search
```

---

# 32. End-to-End Example

Suppose the PDF contains:

```text
# Authentication

The authentication service validates user credentials.

## Login

Users provide an email and password.

## Tokens

The service generates an access token.
```

### Ingestion

```text
PDF
 ↓
Docling
 ↓
Markdown
 ↓
Section-aware chunking
```

Produces:

```text
Chunk A
Authentication
```

```text
Chunk B
Authentication > Login
```

```text
Chunk C
Authentication > Tokens
```

---

### Embedding

Each chunk is passed to Cohere:

```text
Chunk A → Vector A
Chunk B → Vector B
Chunk C → Vector C
```

---

### Indexing

The vectors are stored in Qdrant:

```text
Qdrant
├── Vector A + Payload A
├── Vector B + Payload B
└── Vector C + Payload C
```

---

### User query

```text
"How does the system verify a user's identity?"
```

Cohere generates:

```text
Query Vector
```

Qdrant retrieves semantically similar chunks.

At the same time BM25 searches for lexical matches.

The two rankings are fused using RRF.

---

# 33. Testing

Each layer can be tested independently.

## Test ingestion

```bash
python test_chunking.py
```

---

## Test embeddings

```bash
python test_embeddings.py
```

---

## Generate embedded chunks

```bash
python -m app.embeddings.embed_chunks
```

---

## Test BM25

```bash
python test_bm25.py
```

---

## Create Qdrant collection

```bash
python create_qdrant_collection.py
```

---

## Load vectors

```bash
python load_qdrant.py
```

---

## Test Qdrant

```bash
python test_qdrant.py
```

---

## Test hybrid retrieval

```bash
python test_hybrid.py
```

---

# 34. Recommended Validation Order

When debugging the pipeline, test from left to right:

```text
1. PDF parsing
       ↓
2. Cleaning
       ↓
3. Chunking
       ↓
4. Cohere document embedding
       ↓
5. BM25
       ↓
6. Qdrant insertion
       ↓
7. Qdrant search
       ↓
8. Hybrid search
```

Do not debug all layers simultaneously.

If Qdrant search fails, first verify that:

```text
Cohere query embedding works
        ↓
vector dimension = 1024
        ↓
Qdrant collection dimension = 1024
        ↓
vectors were inserted
```

---

# 35. Environment Configuration

The Cohere API key should be stored in `.env`:

```text
COHERE_API_KEY=your_api_key
```

Do not hard-code the API key into Python.

The `.gitignore` should include:

```text
.env
venv/
__pycache__/
```

---

# 36. Important Data Flow Rules

### Rule 1 — Preserve chunk IDs

The same `chunk_id` should remain associated with the chunk throughout the pipeline.

```text
chunk_id
 ↓
embedding
 ↓
Qdrant
 ↓
retrieval
 ↓
RRF
```

### Rule 2 — Keep metadata

Do not throw away:

```text
document ID
section
page/source information
```

when creating embeddings.

The metadata is valuable after retrieval.

### Rule 3 — Keep retrieval components separate

```text
Cohere ≠ BM25 ≠ Qdrant
```

Cohere generates vectors.

BM25 performs lexical retrieval.

Qdrant stores/searches vectors.

RRF combines rankings.

---

# 37. Current Limitations

This is a PoC, not a production deployment.

Current limitations include:

- basic BM25 tokenization
- Cohere API dependency for embeddings
- local Qdrant storage
- no advanced query rewriting yet
- no reranking yet
- no context compression yet
- no LLM generation yet
- no cloud/deployment layer

These are intentional boundaries for the current stage.

---

# 38. What Comes Next

The current pipeline ends at:

```text
Hybrid Retrieval
       ↓
Top-K chunks
```

The next stage is typically:

```text
Hybrid Retrieval
       ↓
Reranking
       ↓
Context Assembly
       ↓
LLM Generation
       ↓
Final RAG Answer
```

The reranker will take the initial candidate chunks and reorder them based on their relevance to the query.

Then the context builder will decide which retrieved chunks should actually be sent to the LLM.

---

# 39. Mental Model

Remember the responsibilities like this:

```text
INGESTION
"What is inside the document?"
        ↓
CHUNKING
"How should the document be divided?"
        ↓
EMBEDDING
"What does each chunk mean numerically?"
        ↓
BM25
"What exact terms does each chunk contain?"
        ↓
VECTOR STORE
"Where can I efficiently search semantic vectors?"
        ↓
HYBRID RETRIEVAL
"Which chunks are relevant according to both methods?"
```

The result is a clean foundation for the next RAG stages.
