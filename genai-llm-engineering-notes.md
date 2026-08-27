# GenAI / LLM Engineering — In-Depth Notes with Real-World Analogies

---

## 1. Agentic Workflows

**What it is:** Instead of a single LLM call producing a final answer, an "agent" runs a loop: **observe → think → act → observe result → repeat** until the goal is met. The LLM decides *which tool to call, when to call it, and when it's done* — not a human writing the exact steps in advance.

**Core patterns:**
- **ReAct (Reason + Act):** LLM alternates between reasoning ("I need to check the database") and acting (calling a tool), using the tool's output to decide the next reasoning step.
- **Plan-and-Execute:** LLM first drafts a full plan (list of subtasks), then a controller executes each step, optionally replanning if a step fails.
- **Reflexion / Self-critique loops:** Agent generates an answer, critiques its own output, and revises before returning it.

**Real-world analogy:** Think of a **new employee handling a customer complaint** vs. a **script-reading call-center bot**. The bot follows a fixed flowchart (traditional pipeline). The new employee (agent) *investigates*: checks the order history (tool call), notices a shipping delay (observation), decides to issue a refund (action), then double-checks company policy before confirming (reflection) — adapting the process based on what they discover, rather than following one predetermined script.

**AWS mapping:** Bedrock Agents (with Action Groups + Lambda), Step Functions for durable multi-step orchestration, EventBridge for triggering agent runs, Lambda for tool execution, DynamoDB for agent state/scratchpad.

---

## 2. MCP (Model Context Protocol) Servers

**What it is:** MCP is a standardized protocol (open-sourced by Anthropic) that lets an LLM application discover and call external tools/data sources through a common interface, instead of every app writing bespoke integration code for every tool.

**Why it matters:** Before MCP, connecting an LLM to Slack, a database, and a ticketing system meant three different custom integrations. MCP defines one interface — the LLM app is the **client**, each tool/data source exposes an **MCP server** — so any MCP-compatible client can talk to any MCP-compatible server without custom glue code.

**Real-world analogy:** MCP is like a **USB-C port** for AI tools. Before USB-C, every device (camera, phone, hard drive) needed its own proprietary cable and adapter. USB-C standardized the connector so any compliant device plugs into any compliant port. MCP does the same for LLM-to-tool connections — build one MCP server for your internal CRM, and *any* MCP-aware agent (Claude, another LLM app) can use it without custom wiring.

**AWS mapping:** Host MCP servers as Lambda functions behind API Gateway, or as long-running services on ECS/Fargate; use IAM for auth between the agent runtime and MCP servers; Bedrock AgentCore Gateway can expose existing APIs/Lambdas as MCP-compatible tools.

---

## 3. Memory Management in AI Agents

**Types of memory:**
| Type | Description | Analogy |
|---|---|---|
| **Short-term / working memory** | The current conversation's context window — what's actively "in mind" | A person's **working memory** while solving a math problem on a whiteboard — limited space, gets erased after the session |
| **Long-term episodic memory** | Facts/events from past sessions, retrieved when relevant | A **diary or photo album** you flip back through to recall what happened last month |
| **Long-term semantic memory** | Distilled facts/preferences about the user or domain, not tied to one event | Knowing your friend's **birthday and coffee order** — you don't remember the exact conversation where you learned it, you just *know* it now |
| **Procedural memory** | Learned patterns of *how* to do a task | **Muscle memory** — you don't consciously recall each driving lesson, you just drive |

**Implementation approaches:**
- Sliding window (keep last N turns) — simplest, loses old context.
- Summarization buffer — periodically compress old turns into a running summary.
- Vector-store memory — embed past interactions, retrieve top-k relevant memories via similarity search (this is essentially RAG applied to conversation history).
- Structured memory (key-value facts, knowledge graph) — explicit fact extraction and storage, avoids "lossy" summarization.

**Real-world analogy for the overall system:** A good **executive assistant** doesn't try to remember every word you've ever said (that's the sliding window failing at scale). Instead they keep a **notebook of key facts** (structured memory: "boss prefers morning meetings"), a **filed archive** they can search when needed (vector store), and only the **current day's agenda** actively in mind (working memory/context window).

**AWS mapping:** DynamoDB or ElastiCache for short-term session state; OpenSearch/Bedrock Knowledge Bases (vector store) for long-term semantic memory; S3 for archived conversation logs; Bedrock AgentCore Memory for managed short/long-term memory out of the box.

---

## 4. Session Context Management for Agents

**The problem:** A single user may have multiple conversations, across multiple devices, possibly with multiple agents collaborating — you need to track *whose turn it is, what's been decided, and what context is still valid* without re-sending the entire history every time.

**Techniques:**
- **Session IDs** mapped to a context store (Redis/DynamoDB) holding conversation state, tool call results, and user metadata.
- **Context windowing + summarization**: as the conversation grows, older turns get compressed so the active context stays within the token budget.
- **Scoped context injection**: only inject the memory/tool results relevant to the *current* subtask, not the entire history (reduces noise and token cost).
- **Stateless agent + stateful store pattern**: the agent itself is stateless (easy to scale horizontally); all state lives in an external store keyed by session ID, fetched fresh each turn.

**Real-world analogy:** Think of a **hospital shift handover**. Each nurse doesn't personally remember every patient's full history — instead there's a **shared patient chart** (external context store) that any nurse on duty can pull up. The incoming nurse reads the *relevant recent notes* (scoped context), not the patient's entire 10-year medical history, unless it's specifically needed.

**AWS mapping:** DynamoDB (session/state store) with session_id as partition key, TTL for expiry; ElastiCache for low-latency active sessions; Step Functions execution history as an audit trail of a session's tool calls.

---

## 5. Evaluation Framework Integration (LangSmith / Langfuse)

**Why evaluation matters:** Unlike traditional software, LLM output is non-deterministic and quality is fuzzy ("is this answer good?"). You need continuous observability + evaluation, not just unit tests.

**LangSmith (by LangChain):**
- Tracing: full visibility into every LLM call, tool call, and intermediate step in a chain/agent run.
- Datasets & evals: run your agent against a fixed test set, score outputs (exact match, LLM-as-judge, custom functions), track regressions across prompt/model versions.
- Feedback capture: log human thumbs-up/down or annotations tied to specific traces.

**Langfuse (open-source alternative):**
- Similar tracing/observability, but self-hostable — important for teams with data residency requirements (e.g., can be run inside a VPC on AWS).
- Prompt management (versioning prompts outside code), cost/latency tracking per trace, dataset-based evals, session/user-level analytics.

**Real-world analogy:** These tools are the **flight data recorder + quality inspector** for your AI system. A black box (tracing) records exactly what happened on every "flight" (LLM call) — which tools fired, what the model said, how long it took, what it cost — so when something goes wrong, you can replay the exact sequence instead of guessing. The quality inspector (eval framework) then runs a checklist against those recordings to certify: is this answer accurate, is this agent behaving safely, did this new prompt version fly better than the old one?

**AWS mapping:** Both integrate via SDK regardless of hosting; Langfuse can be self-hosted on ECS/Fargate + RDS Postgres + S3 within your VPC for compliance; CloudWatch can complement with infra-level metrics (latency, throughput) while LangSmith/Langfuse handle semantic/quality metrics.

---

## 6. How Do You Evaluate LLM Responses? / LLM-as-Judge

**Evaluation dimensions:**
- **Correctness/faithfulness** — is the answer factually accurate and grounded in the provided context (no hallucination)?
- **Relevance** — does it actually address the question asked?
- **Completeness** — does it cover what was needed, not partial?
- **Coherence/fluency** — is it well-structured and readable?
- **Safety/tone** — no harmful, biased, or off-brand content?
- **Task-specific correctness** — e.g., for code generation: does it compile/pass tests; for structured extraction: does it match schema.

**LLM-as-Judge:** Use a strong LLM (often a larger/more capable model, or the same model with a carefully designed rubric prompt) to score another model's output — e.g., "Rate this answer 1-5 for faithfulness to the provided context; cite which sentence is unsupported if any." This scales far better than human review, though it has known biases (favoring longer answers, verbose style, or its own output style — mitigated by pairwise comparison and rubric anchoring).

**Approaches:**
- **Reference-based**: compare against a "gold" answer (BLEU/ROUGE for text similarity — usually too rigid for open-ended generation; better: LLM-judge comparing semantic equivalence).
- **Reference-free**: judge model scores based on a rubric alone, no gold answer needed (e.g., "does this response hallucinate beyond the given context?").
- **Pairwise comparison**: show the judge two candidate answers (e.g., new prompt vs. old prompt) and ask which is better — more reliable than absolute scoring.

**Real-world analogy:** LLM-as-judge is like using an **experienced senior editor to review a junior writer's article**, rather than mechanically checking if the article contains the exact same words as a "model" article (that's reference-based scoring — too rigid, since two good articles rarely use identical phrasing). The editor reads it fresh and asks: is this accurate, on-topic, complete, well-written? — a rubric-based, holistic judgment call, much closer to how a human editor actually evaluates work.

**AWS mapping:** Bedrock Model Evaluation (built-in automatic + human eval jobs), or custom eval pipelines using Step Functions to orchestrate judge-model calls at scale, storing results in S3/Athena for analysis.

---

## 7. Metrics to Measure GenAI Application Success

**Model/response-quality metrics:**
- Faithfulness / groundedness score (no hallucination)
- Answer relevance, context precision & recall (for RAG specifically — see below)
- Task success rate (did the agent actually complete the user's goal, end to end?)
- Human/LLM-judge preference win-rate vs. baseline

**System/ops metrics:**
- Latency (p50/p95/p99), time-to-first-token
- Cost per request / per resolved task
- Token usage (input/output split — useful for cost optimization)
- Tool-call success rate, retry rate, error rate

**Business metrics:**
- User satisfaction (CSAT/thumbs-up rate), deflection rate (for support bots — % resolved without human escalation)
- Task completion rate, retention/repeat usage
- Escalation/override rate (how often a human had to step in — inverse proxy for trust)

**Real-world analogy:** Measuring a GenAI app only by "does it generate fluent text" is like judging a **customer-service employee only on how polite they sound**, while ignoring whether they actually resolved the customer's problem, how long the call took, and whether the customer had to call back. Real evaluation needs the **full scorecard**: quality of the response, efficiency of delivery, and the actual business outcome.

---

## 8. Retrieval Accuracy Techniques (RAG)

**Core techniques to improve retrieval:**

1. **Chunking strategy** — semantic chunking (split at natural topic boundaries via embeddings) instead of fixed character-count chunks; overlapping windows to avoid cutting context mid-thought.
2. **Hybrid search** — combine dense vector similarity (semantic meaning) with sparse keyword search (BM25) so you catch both "meaning matches" and exact term matches (e.g., product SKUs, error codes that embeddings alone may under-weight).
3. **Re-ranking** — retrieve a broad top-k (e.g., 50) with a cheap/fast method, then re-rank with a more expensive cross-encoder model to surface the true top 3-5 most relevant chunks.
4. **Multi-query retriever** — have the LLM generate several *rephrasings* of the user's question, retrieve for each, then merge/de-duplicate results. This compensates for the fact that a single query phrasing may not match how the answer is worded in the source documents.
5. **Metadata filtering** — pre-filter by document type, date, department, etc., before vector search, narrowing the search space and reducing noise.
6. **Query transformation / HyDE (Hypothetical Document Embeddings)** — have the LLM first write a *hypothetical answer*, embed that instead of the raw question, and search with it (hypothetical answers often resemble the source document's phrasing more than a terse user question does).
7. **Parent-child chunk retrieval** — retrieve small precise chunks for matching, but return the larger parent section for context (small chunks match better, large chunks answer better).

**Real-world analogy for multi-query retriever:** Imagine asking a **librarian** to find books about "how plants make food." A single search on that exact phrase might miss the biology textbook that only uses the word "photosynthesis." A good librarian instinctively rephrases your question several ways ("photosynthesis," "how plants make food," "chlorophyll energy conversion") and searches each — then hands you the combined best results. That's exactly what a multi-query retriever does with an LLM generating those rephrasings automatically.

**Real-world analogy for hybrid search:** It's like searching for a person using **both a photo and their exact name** — the photo (semantic/vector search) finds people who *look* like the description even if you don't know the exact spelling, while the name (keyword/BM25 search) nails an exact match instantly. Relying on only one method misses cases the other would catch.

**AWS mapping:** Bedrock Knowledge Bases (managed chunking + hybrid search + re-ranking with OpenSearch Serverless or Aurora pgvector as the vector store), Kendra for enterprise hybrid search, Lambda for custom query-rewriting steps before retrieval.

---

## 9. Hybrid RAG

**Definition:** "Hybrid RAG" typically refers to combining **multiple retrieval strategies** in one pipeline — most commonly (a) dense vector search + sparse keyword search (hybrid search, above), and/or (b) vector retrieval + structured retrieval from a knowledge graph (combining unstructured document chunks with structured entity-relationship facts).

**Why combine them:** Pure vector RAG is good at "fuzzy meaning" questions ("what's our refund policy tone like?") but weak at precise multi-hop relational questions ("which vendors did we work with in 2023 that later had a contract dispute?"). A knowledge graph is the opposite — great at precise relationships, poor at "meaning/similarity" questions. Hybrid RAG routes or combines both so each type of question gets the retrieval method suited to it.

**Real-world analogy:** It's like a **detective's investigation** using both a **witness interview** (vector search — fuzzy, meaning-based, "what do people recall about that night?") and a **case file with a suspect relationship chart** (knowledge graph — precise, structured, "who is connected to whom, and how"). Neither alone solves the case; combining fuzzy recollection with precise structured facts gets the full picture.

**AWS mapping:** Bedrock Knowledge Bases for the vector side + Amazon Neptune for the graph side, with a router (Lambda or an agent's own reasoning) deciding which source(s) to query per question; Neptune can also feed GraphRAG patterns (traversal-based retrieval for multi-hop questions).

---

## 10. Knowledge Graphs (in GenAI context)

**What it is:** A network of entities (nodes) and their relationships (edges) — e.g., `(Company A) —[acquired]→ (Company B)`. Instead of retrieving loosely-related text chunks, you can traverse exact relationships, which is powerful for multi-hop reasoning ("who are the suppliers of the suppliers of Company A?") that plain vector RAG struggles with.

**GraphRAG pattern:** LLM extracts entities/relationships from source documents to *build* the graph, then at query time, retrieval traverses the graph (plus optionally still uses vector search over node/edge descriptions) to gather precise, connected context before generation.

**Real-world analogy:** A knowledge graph is like an **organizational chart / family tree** versus a **pile of biography books** (unstructured documents). If you want to know "who is this person's great-grandmother's second cousin," flipping through biography books (vector search over text) is slow and error-prone — but tracing the family tree (graph traversal) gets you the exact answer in a few precise hops.

**AWS mapping:** Amazon Neptune (graph database, supports Gremlin/openCypher), Neptune Analytics for GraphRAG-style vector + graph hybrid queries, or Neptune ML for graph-based embeddings.

---

## 11. Multi-Agent Systems — Design Approach

**When to use multi-agent vs. single agent:** Use multiple specialized agents when a task naturally decomposes into distinct sub-domains requiring different tools/expertise/context, and a single agent's context window or tool set would get overloaded or the prompt would become unwieldy trying to handle "everything."

**Common architectures:**
- **Orchestrator-worker (supervisor pattern):** One "manager" agent breaks the task into subtasks and delegates to specialized "worker" agents (e.g., a Researcher agent, a Coder agent, a Reviewer agent), then synthesizes their outputs.
- **Sequential pipeline:** Agents pass work to each other in a fixed order (e.g., Research → Draft → Edit → Publish), each one specialized for its stage.
- **Debate/collaborative:** Multiple agents propose independent answers and critique each other, converging on a better final answer (useful for reducing individual-agent blind spots).
- **Hierarchical teams:** Sub-supervisors manage their own worker agents, reporting up to a top-level orchestrator — used for very complex, large-scale tasks.

**Design considerations:**
- Clear **role boundaries** (each agent has a narrow, well-defined responsibility — avoids overlap/conflicting actions).
- **Shared vs. isolated context** — decide what each agent sees; too much shared context wastes tokens, too little causes miscoordination.
- **Communication protocol** — structured messages (not free text) between agents reduce ambiguity.
- **Failure handling** — what happens if a worker agent fails or times out; the orchestrator needs a retry/fallback strategy.

**Real-world analogy:** A multi-agent system is like a **hospital care team** treating one patient: the **general physician** (orchestrator) doesn't personally perform surgery or read every lab result in detail — they diagnose the overall situation and route the patient to a **cardiologist** for the heart issue, a **radiologist** to read the scan, and a **pharmacist** to check drug interactions. Each specialist (worker agent) has deep, narrow expertise and their own tools; the physician synthesizes all their inputs into one coherent treatment plan, rather than trying to be an expert in everything themselves.

**AWS mapping:** Bedrock multi-agent collaboration (native supervisor + sub-agent support), Step Functions to orchestrate agent hand-offs with durable state, SQS/EventBridge for async agent-to-agent messaging, separate Lambda functions per specialized agent for isolation and independent scaling.

---

## 12. Structured Output

**What it is:** Forcing/guiding the LLM to return output in a strict, machine-parseable format (JSON matching a schema, XML tags, function-call arguments) rather than free-form prose — essential when the output feeds directly into downstream code (a database write, an API call, a UI component).

**Techniques:**
- **Function/tool calling** — define a schema (JSON Schema) as a "tool," and the model returns arguments matching that schema instead of prose.
- **Constrained decoding / grammar-based generation** — the decoding process itself is restricted so only tokens matching the target schema can be produced (guarantees valid output, not just "usually valid").
- **Prompt-based JSON mode with validation + retry** — ask for JSON directly, validate against a schema (e.g., Pydantic), and automatically retry with the validation error fed back to the model if it fails.

**Real-world analogy:** Structured output is like handing someone a **fill-in-the-blank form** instead of asking them to "write a paragraph about yourself" and then trying to manually extract their age and address from the prose afterward. The form (schema) guarantees you get exactly the fields you need, in the exact format your downstream system (e.g., a database) expects — no fragile parsing of free text required.

**AWS mapping:** Bedrock Converse API's native tool-use/structured output support; validate downstream with Lambda + Pydantic/JSON Schema; store validated structured results directly in DynamoDB.

---

## 13. Handling Hallucinations in LLM Applications

**Root causes:** The model generates plausible-sounding but ungrounded text because it's predicting likely tokens, not verifying facts — worse when the answer requires knowledge outside its training data or context, or when the prompt encourages confident-sounding completion regardless of certainty.

**Mitigation techniques:**
1. **Grounding via RAG** — force the model to answer *only* from retrieved, verifiable context rather than parametric memory; explicitly instruct "if the answer isn't in the provided context, say you don't know."
2. **Citation requirements** — require the model to cite the specific source/passage for each claim; makes ungrounded claims visible and checkable (and discourages fabrication since the model must "point to" a real source).
3. **Faithfulness checking / self-verification** — a second pass (or a separate judge model) checks whether each claim in the draft answer is actually supported by the retrieved context, flags/removes unsupported claims.
4. **Lower temperature / constrained decoding** for factual tasks — reduces creative deviation.
5. **Structured output + validation** — for factual extraction tasks, constrain the answer format so there's less room for free-form fabrication.
6. **Human-in-the-loop for high-stakes outputs** — flag low-confidence or unverifiable answers for human review rather than auto-publishing.
7. **Fine-tuning/RLHF on "I don't know" behavior** — explicitly reward the model for abstaining when uncertain, rather than always producing an answer.

**Real-world analogy:** An ungrounded LLM is like a **student who didn't study but is a confident public speaker** — when asked a question they don't know the answer to, they'll construct something that *sounds* plausible rather than admitting "I don't know." Grounding via RAG is like giving that student an **open-book exam**: they must find and cite the actual page in the textbook to support their answer, which makes it much harder to bluff — and if they can't find it in the book, the rules require them to say so instead of guessing.

---

## 14. 128K Token Limit — Handling Long Context

**Challenges with large context windows:**
- **Cost** — you pay for every input token even if only a fraction is relevant.
- **"Lost in the middle" effect** — models tend to attend better to information at the start/end of a long context than content buried in the middle, so simply "stuffing everything in" degrades quality even when it technically fits.
- **Latency** — larger context = higher time-to-first-token.

**Techniques to manage it:**
- **Retrieve, don't dump** — even with a 128K window, use RAG to select only the most relevant chunks rather than pasting an entire document set, preserving both quality (avoiding "lost in the middle") and cost.
- **Hierarchical summarization** — for very long documents, summarize sections first, then summarize the summaries, progressively compressing while preserving key facts.
- **Sliding window with overlap** — process long documents in overlapping windows so no cross-boundary context is lost, merging results afterward.
- **Context compaction / rolling summarization for conversations** — periodically compress older conversation turns into a summary, keeping only recent turns verbatim.
- **Prioritized placement** — since models attend better to the start and end, place the most critical instructions/context there rather than in the middle.

**Real-world analogy:** A 128K context window is like giving someone a **massive stack of case files** to review before a meeting, versus giving them a **well-organized 2-page briefing** with only the relevant facts highlighted. Technically the person *could* read the entire stack (it fits in their work day, just like it fits the token limit), but they'll retain and use the middle-of-the-stack details far worse than a concise, well-curated summary — so a good assistant pre-filters and summarizes rather than just forwarding everything.

---

## 15. AWS Cloud Infrastructure for GenAI Systems (Reference Architecture)

| Layer | AWS Service | Role |
|---|---|---|
| Model inference | **Amazon Bedrock** | Managed access to foundation models (Claude, etc.), Converse API, Guardrails |
| Agent orchestration | **Bedrock Agents / AgentCore**, **Step Functions** | Multi-step reasoning, tool calls, durable multi-agent workflows |
| Tool hosting (MCP) | **Lambda**, **API Gateway**, **ECS/Fargate** | Expose internal APIs/tools to agents |
| Vector store (RAG) | **OpenSearch Serverless**, **Aurora pgvector**, **Bedrock Knowledge Bases** | Semantic search over document embeddings |
| Knowledge graph | **Amazon Neptune** | Structured entity-relationship retrieval, GraphRAG |
| Short-term/session memory | **DynamoDB**, **ElastiCache (Redis)** | Fast session state, conversation buffers |
| Long-term memory/logs | **S3**, **Bedrock AgentCore Memory** | Archived transcripts, persistent user facts |
| Evaluation/observability | **LangSmith / Langfuse (self-hosted on ECS+RDS)**, **CloudWatch**, **Bedrock Model Evaluation** | Tracing, quality evals, cost/latency dashboards |
| Event-driven triggers | **EventBridge**, **SQS** | Async agent triggers, inter-agent messaging |
| Security/governance | **IAM**, **Bedrock Guardrails**, **VPC** | Access control, content filtering, data isolation |

**Real-world analogy for the whole stack:** This is like running a **modern hospital**: Bedrock is the **medical staff** (the actual expertise/decision-making), Step Functions/Agents are the **case management system** routing patients through the right departments in order, DynamoDB/ElastiCache are the **bedside chart** (immediate, fast-access info), S3/Neptune/OpenSearch are the **medical records archive and specialist reference library** (deep, searchable history), and LangSmith/CloudWatch are the **hospital's quality-assurance and audit department**, reviewing every case afterward to make sure standards were met and catching patterns of error before they become systemic.

---

## Quick-Reference Answers to the Interview-Style Questions

- **"How do you improve retrieval accuracy?"** → Better chunking, hybrid (dense+sparse) search, re-ranking, multi-query retrieval, metadata filtering, HyDE.
- **"How do you handle hallucinations?"** → Ground answers in retrieved context, require citations, add a faithfulness-checking pass, allow/encourage "I don't know," human review for high-stakes outputs.
- **"How would you design a multi-agent system?"** → Identify natural task decomposition, pick an architecture (orchestrator-worker is the most common default), define narrow agent roles and a structured communication protocol, plan failure/retry handling.
- **"How would you implement memory in an agent?"** → Layer it: short-term (context window/session store) + long-term semantic (vector store or structured facts) + summarization to compress old context, retrieved selectively per turn.
- **"How do you manage session context?"** → External stateful store (DynamoDB/Redis) keyed by session ID, stateless compute layer, scoped/summarized context injection rather than full history replay.
- **"How do you evaluate LLM responses?"** → Multi-dimensional (faithfulness, relevance, completeness, safety) using LLM-as-judge with rubrics and pairwise comparison, backed by a dataset of test cases run continuously (LangSmith/Langfuse).
- **"What metrics measure GenAI success?"** → Quality metrics (faithfulness, relevance, task success rate) + ops metrics (latency, cost/token usage) + business metrics (CSAT, deflection/resolution rate).
