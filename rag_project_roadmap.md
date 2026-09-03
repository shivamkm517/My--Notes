# RAG Project Roadmap — "PaperPilot"
### A Research-Paper Q&A & Summarization Assistant, built on the 9-Layer RAG Stack

> **What we're building:** A RAG system that ingests PDFs (research papers, technical reports),
> lets users ask questions, get cited answers, compare papers, and get auto-generated summaries —
> with full observability and guardrails.
>
> Each section below maps to one layer of the reference architecture, with concrete tool choices,
> setup steps, and a "definition of done" so you can build and ship incrementally instead of
> trying to build all 9 layers at once.

---

## 0. Before You Start

**Guiding principle:** Build bottom-up in this order — get raw data flowing first (Layers 1-4),
get a dumb-but-working answer next (Layer 5), then add memory, eval, and guardrails
(Layers 6-8) once the core loop works. Layer 0 (deployment) is decided early but *executed* last.

**Suggested repo structure**
```
paperpilot/
├── infra/                # Layer 0 - deployment configs
├── ingestion/             # Layer 1 - parsing pipeline
├── embeddings/            # Layer 2
├── vectorstore/           # Layer 3
├── orchestration/         # Layer 4 - LangGraph/LlamaIndex app
├── llm/                   # Layer 5 - prompt templates, model router
├── memory/                # Layer 6 - session + long-term memory
├── evaluation/            # Layer 7 - eval harness
├── observability/         # Layer 8 - tracing, guardrails
├── api/                   # FastAPI app tying it all together
└── tests/
```

---

## Layer 0 — Deployment
**Where latency, concurrency, and cost get locked in — decide this first, build it last.**

### Steps
1. **Pick your compute tier for local dev first**: run everything in Docker Compose on your
   own machine before touching cloud. This forces you to keep the stack lightweight.
2. **Choose a cloud target for later**, based on where your other pieces live:
   - AWS → good if you'll use Textract (Layer 1) and Bedrock models
   - GCP → good if you'll use Document AI + Gemini embeddings
   - Azure → good if you're already in an enterprise Microsoft stack
3. **Separate stateless vs stateful services**:
   - Stateless: API server, ingestion workers → autoscale on ECS Fargate / Cloud Run
   - Stateful: vector DB, Postgres (memory) → managed services (RDS, Qdrant Cloud, etc.)
4. **Plan for concurrency early**: async ingestion queue (SQS/Pub-Sub) so PDF uploads don't
   block the API; a worker pool separate from your request-serving pod.
5. **Set a cost ceiling per query** (embedding calls + LLM calls + vector search) — write it
   down now, because Layer 5 decisions will blow past it if you don't.

### Definition of done
- [ ] `docker-compose.yml` runs API + vector DB + Postgres locally with one command
- [ ] A documented target cloud architecture diagram (even hand-drawn) exists before Layer 4

---

## Layer 1 — Extraction & Parsing
**Clean, structured data in. Garbage in = garbage out.**

### Steps
1. **Ingest raw PDFs** into a landing folder / S3 bucket via an upload endpoint.
2. **Choose your parser based on document complexity**:
   - Simple text PDFs → `pypdf` / `pdfplumber` (free, fast, good enough to start)
   - Papers with tables, figures, multi-column layout → **Docling** (IBM) or **Unstructured.io**
   - Scanned/handwritten → AWS Textract or Google Document AI (OCR)
3. **Normalize output into a common schema** regardless of parser used:
   ```json
   {
     "doc_id": "uuid",
     "title": "...",
     "sections": [{"heading": "Abstract", "text": "...", "page": 1}],
     "tables": [...],
     "figures": [...],
     "metadata": {"authors": [], "year": 2024, "source_file": "..."}
   }
   ```
4. **Chunk the text** — this is the single highest-leverage decision in the whole stack:
   - Start with **semantic/section-aware chunking** (split on headings, not arbitrary token counts)
   - Chunk size: 300–500 tokens with 10–15% overlap is a solid default for papers
   - Keep chunk metadata: `doc_id`, `section`, `page_number` (you'll need this for citations)
5. **Deduplicate and clean**: strip headers/footers, page numbers, reference-list noise.

### Definition of done
- [ ] Given a PDF, pipeline outputs a list of clean, metadata-tagged chunks in JSON
- [ ] Spot-check 5 papers manually — chunks should read as coherent, self-contained ideas

---

## Layer 2 — Embeddings
**Turn content into searchable vectors. Use dense + sparse for hybrid search.**

### Steps
1. **Pick a dense embedding model** — start with one, don't overthink it:
   - `text-embedding-3-large` (OpenAI) or Cohere `embed-v3` for strong general quality
   - Open-source alternative: `Qwen3-Embedding` or `bge-large` if you want to self-host
2. **Add a sparse embedding for hybrid search** (huge win for technical/keyword-heavy content
   like paper titles, equations, acronyms):
   - BM25 (via `rank_bm25` or built into your vector DB) or SPLADE
3. **Batch-embed your chunks**, storing both dense vector + sparse representation alongside
   the chunk text and metadata.
4. **Version your embeddings**: tag every vector with `embedding_model_version` — you WILL
   change embedding models later, and re-indexing without tracking this is painful.
5. **Cache embedding calls** (hash of chunk text → vector) to avoid re-paying for identical
   text during re-runs.

### Definition of done
- [ ] Every chunk has a dense vector + sparse representation stored
- [ ] A simple cosine-similarity test between a known query and known relevant chunk returns
      a high score

---

## Layer 3 — Vector Database
**Index embeddings and return the most relevant chunks at query time.**

### Steps
1. **Pick your vector DB based on scale and ops appetite**:
   - Prototyping / small scale → **pgvector** (just Postgres — simplest ops, one less service)
   - Production, purpose-built → **Qdrant** or **Pinecone** (managed, fast filtering)
   - Already using Elastic → **Elastic** with dense_vector fields
2. **Design your index schema** with metadata filters you'll actually use:
   `doc_id`, `year`, `author`, `section_type` (abstract/methods/results) — filtering by these
   before vector search dramatically improves precision.
3. **Enable hybrid search** (dense + BM25/sparse) with reciprocal rank fusion (RRF) to combine
   result lists — most modern vector DBs support this natively now.
4. **Add a reranker step** after initial retrieval: pull top 20 via vector search, rerank to
   top 5 with a cross-encoder (e.g., Cohere Rerank or `bge-reranker`) — this is often a bigger
   accuracy win than switching embedding models.
5. **Set retrieval parameters** and log them: `top_k`, similarity threshold, hybrid weighting.

### Definition of done
- [ ] Query → returns top-k chunks with scores in under 500ms locally
- [ ] Reranking measurably improves relevance on 10 hand-picked test queries

---

## Layer 4 — Orchestration Framework
**Route queries, chain steps, and build agentic workflows.**

### Steps
1. **Choose a framework** matched to how much control you want:
   - **LangGraph** — best if you want explicit control over branching/agentic logic (recommended
     for PaperPilot, since "compare papers" and "summarize" are different flows)
   - **LlamaIndex** — best if RAG-first patterns (query engines, indices) are your main need
   - Keep it simple: you can hand-roll this with plain Python + function calls too — don't
     adopt a framework just because the diagram lists one
2. **Define your query router** — classify incoming questions into flows:
   - `single_paper_qa` → retrieve from one doc, answer with citation
   - `cross_paper_comparison` → retrieve from multiple docs, synthesize
   - `summarization` → retrieve full doc sections, produce structured summary
3. **Build the RAG chain per flow**: retrieve → rerank → construct prompt → call LLM → parse
   output → attach citations.
4. **Add tool-calling for agentic behavior** (optional but powerful): a "search arXiv" tool,
   a "fetch citation graph" tool, so the agent can go beyond your local corpus when needed.
5. **Make it stateless per request**, with session state pulled from Layer 6 (Memory) —
   don't let orchestration logic hold state itself.

### Definition of done
- [ ] A LangGraph (or equivalent) graph diagram exists showing router → retrieval → generation
- [ ] All three flows (QA, comparison, summarization) can be triggered end-to-end

---

## Layer 5 — LLMs
**The reasoning engine. Powerful, but not the bottleneck.**

### Steps
1. **Pick a default model** — Claude (Sonnet-class) or GPT-4-class for quality; don't default
   to the biggest model everywhere.
2. **Use a smaller/cheaper model for cheap sub-tasks**: query classification, metadata
   extraction, and reranking don't need your flagship model — route those to a mini/haiku-class
   model to control cost (this is the "model router" pattern).
3. **Design the prompt template** to force citation discipline:
   ```
   Answer using ONLY the provided context. For every claim, cite the source
   as [doc_id, page]. If the context doesn't contain the answer, say so —
   do not use outside knowledge.

   Context:
   {retrieved_chunks}

   Question: {user_question}
   ```
4. **Structure outputs** (JSON mode / function calling) when you need citations as structured
   data rather than free text — makes it far easier to render clickable citations in the UI.
5. **Add streaming** for the chat response so users see tokens as they generate (matters a lot
   for perceived latency).

### Definition of done
- [ ] Answers reliably include correct inline citations back to `doc_id` + page
- [ ] Model correctly refuses / says "not found in context" when info isn't in the corpus

---

## Layer 6 — Memory
**What the agent remembers across turns and sessions.**

### Steps
1. **Short-term (session) memory**: store the last N turns of conversation in Redis, keyed by
   `session_id`, so follow-up questions ("what about the 2023 version?") resolve correctly.
2. **Long-term memory**: in Postgres or MongoDB, persist:
   - Which papers a user has uploaded/discussed
   - Summaries they've previously generated (avoid recomputation)
   - User preferences (e.g., "always answer in bullet points")
3. **Use a memory layer library if you want structure**: `mem0` gives you a ready-made
   abstraction for extract → store → retrieve memory instead of hand-rolling it.
4. **Decide what NOT to remember**: raw chunk text doesn't belong in conversational memory —
   only store summarized state, not full retrieval payloads (keeps context windows lean).
5. **Add a memory-retrieval step to your orchestration graph** (Layer 4): before generation,
   pull relevant session + long-term memory and merge with the retrieved chunks.

### Definition of done
- [ ] A follow-up question referencing "it" or "that paper" resolves correctly using session memory
- [ ] Returning users see previously generated summaries without recomputation

---

## Layer 7 — Evaluation
**Measure faithfulness, context precision, and answer relevancy. Measure it, don't vibe it.**

### Steps
1. **Build a small golden test set**: 20-30 question/answer pairs with known-correct citations,
   drawn from papers you understand well.
2. **Pick eval metrics that map to real failure modes**:
   - **Faithfulness** — does the answer only use retrieved context? (catches hallucination)
   - **Context precision/recall** — did retrieval actually surface the right chunks?
   - **Answer relevancy** — does the answer address the actual question asked?
3. **Choose a tool**: **Ragas** (open-source, easiest to start) for the metrics above, or
   **LangSmith**/**Arize**/**Opik** for full trace-linked evaluation dashboards as you scale.
4. **Run eval on every change** to prompts, chunking strategy, or models — treat it like a
   regression test suite, not a one-time report.
5. **Log failure cases** back into your golden set so it grows over time and catches regressions.

### Definition of done
- [ ] CI step (or manual script) runs eval suite and outputs faithfulness/precision/recall scores
- [ ] A documented baseline score exists to compare future changes against

---

## Layer 8 — Alignment & Observability
**Guardrails, tracing, and monitoring before broken output ships.**

### Steps
1. **Add input guardrails**: block prompt-injection attempts embedded in uploaded PDFs
   (papers can contain hidden instructions in text!) — sanitize extracted text before it
   reaches the LLM context.
2. **Add output guardrails**: check that citations actually exist in the retrieved context
   (catch fabricated citations) before returning the answer to the user — **NeMo Guardrails**
   or a simple custom validator both work.
3. **Instrument full tracing**: every request should log retrieval → rerank → prompt →
   LLM call → output as one linked trace (LangSmith, Arize Phoenix, or Opik all do this).
4. **Set up dashboards** for: latency per layer, cost per query, retrieval hit rate, and
   refusal rate ("not found in context" responses) — these tell you where the system is
   actually struggling.
5. **Add human-in-the-loop review** for a sample of low-confidence answers (flag anything
   below a citation-confidence threshold for manual spot-check before trusting it in prod).

### Definition of done
- [ ] Every request produces a full trace, viewable end-to-end
- [ ] A test prompt-injection PDF is correctly neutralized (doesn't hijack the system prompt)
- [ ] A dashboard shows live cost, latency, and hit-rate metrics

---

## Suggested Build Order (Milestones)

| Milestone | Layers Involved | Outcome |
|---|---|---|
| M1 — "It parses" | 0 (local only), 1 | Upload a PDF, get clean chunks out |
| M2 — "It's searchable" | 2, 3 | Query returns relevant chunks with scores |
| M3 — "It answers" | 4, 5 | End-to-end Q&A with citations on one paper |
| M4 — "It remembers" | 6 | Multi-turn conversation works correctly |
| M5 — "It's trustworthy" | 7, 8 | Eval scores tracked, guardrails block bad output |
| M6 — "It ships" | 0 (cloud) | Deployed, autoscaling, monitored in production |

---

## Tech Stack Summary (This Project's Picks)

| Layer | Choice | Why |
|---|---|---|
| 0. Deployment | Docker Compose → GCP Cloud Run | Simple local dev, easy serverless scale-up |
| 1. Extraction | Docling | Best open-source handling of academic PDF layout |
| 2. Embeddings | Cohere embed-v3 (dense) + BM25 (sparse) | Strong hybrid search out of the box |
| 3. Vector DB | Qdrant | Fast, good filtering, easy self-host or cloud |
| 4. Framework | LangGraph | Explicit control over multi-flow routing |
| 5. LLM | Claude Sonnet (main) + Haiku (routing/classification) | Quality where needed, cost control elsewhere |
| 6. Memory | Redis (session) + Postgres (long-term) | Cheap, well-understood, no new infra paradigm |
| 7. Evaluation | Ragas → LangSmith later | Free to start, upgrade when scale demands it |
| 8. Observability | Opik + custom guardrail validators | Open-source, avoids vendor lock-in early |

*Swap any row for your own preferences — the point of the layered map is that each layer is
independently replaceable as long as the interface between layers stays stable.*
