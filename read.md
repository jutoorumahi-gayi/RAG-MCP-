# RAG & MCP: Practical Study Guide

This guide explains the core ideas behind **RAG (Retrieval-Augmented Generation)** and **MCP (Model Context Protocol)** through practical SQL Server DBA and HR policy chatbot examples.

---

## How to use this guide

1. **Understand the key terms** first.
2. **Follow the examples** to see how the pieces fit together.
3. **Answer the review questions** without referring to the notes.
4. **Build a small project** using a few documents before increasing its scope.

**Remember:**

- **RAG** retrieves trusted information and supplies it to an LLM.
- **MCP** provides a standard way for an AI application to interact with tools and data.

---

# Part 1: Retrieval-Augmented Generation

## 1. What is RAG?

**Retrieval-Augmented Generation (RAG)** is an approach where an application:

1. Retrieves relevant information from a knowledge source.
2. Supplies that information to a large language model.
3. Generates an answer based on the retrieved evidence.

For example, if an employee asks, “How many days of paternity leave can I take?”, a RAG chatbot can search company policy PDFs and provide an answer supported by the relevant passage.

| Without RAG | With RAG |
|---|---|
| The model relies mainly on its training data and prompt. | The model receives relevant internal documents. |
| It may not know recent company policies. | It can answer from current policy information. |
| It may produce unsupported claims. | It can cite retrieved evidence. |

### Why use RAG?

- Work with private or organization-specific information.
- Keep information current without retraining the model.
- Provide source references.
- Reduce unsupported answers.
- Separate knowledge updates from model updates.

### RAG limitations

RAG does not guarantee correctness:

- Retrieval may find the wrong passage.
- The LLM may misunderstand the retrieved text.
- Permissions must be enforced by the application.
- RAG is not model training or fine-tuning.

**Key idea:** RAG retrieves evidence, adds it to the prompt, and generates a grounded answer.

---

## 2. Embeddings

### What is an embedding?

An **embedding** is a numerical vector that represents text or another supported type of input.

A 384-dimensional embedding contains 384 numbers:

```text
[0.25, -0.18, 0.73, 0.42, -0.11, ...]
```

The dimensions do not have individual human-readable meanings. Meaning is distributed across all coordinates.

Text with similar meanings generally produces vectors that are relatively close to one another:

- “I love learning Python”
- “Python programming is interesting”

These are close conceptually. “I like eating biryani” is less related to programming.

### Important embedding rule

Use the **same embedding model** for:

- User questions
- Document chunks
- Stored vectors

Changing the embedding model usually requires re-embedding the documents and rebuilding or replacing the vector index.

**Key idea:** Embeddings convert text into numbers that can be compared using similarity calculations.

---

## 3. Semantic search

**Semantic search** finds relevant information by meaning rather than exact wording.

For example:

> “How can I improve database speed?”

May retrieve:

> “SQL Server performance tuning”

even though the words are not identical.

### Cosine similarity

Cosine similarity measures the angle between two vectors. It mainly considers direction rather than magnitude.

Higher cosine similarity generally means closer semantic relevance.

### Dot product

A dot product multiplies corresponding coordinates and adds the results.

```text
A = [2, 3, 4]
B = [1, 2, 3]

Dot product = (2 × 1) + (3 × 2) + (4 × 3)
             = 20
```

If two vectors are normalized to length 1, their dot product equals their cosine similarity.

### Top-k retrieval

**Top-k** means returning a fixed number of highest-ranked results.

- `k = 5` returns five candidates.
- `k = 10` returns ten candidates.

Similarity scores vary between models and metrics, so a score from one system should not be compared directly with a score from another.

**Key idea:** Embeddings create vectors, similarity search compares vectors, and clustering groups related vectors.

---

## 4. Keyword search

Keyword search ranks documents using query terms.

It works especially well for:

- Error numbers
- SQL object names
- Product codes
- Policy identifiers
- Rare technical terms
- Exact phrases

### TF-IDF

**TF-IDF** means Term Frequency–Inverse Document Frequency.

- **Term frequency:** How often a term appears in a document.
- **Inverse document frequency:** How rare or informative the term is across the collection.

### BM25

**BM25** is a more advanced keyword-ranking method based on:

- Term frequency
- Term rarity
- Document length
- Practical repetition limits

It is effective for exact and highly specific searches.

| Method | Strength | Limitation |
|---|---|---|
| TF-IDF | Basic term scoring | Less sophisticated than BM25 |
| BM25 | Exact terms and document ranking | May miss different wording |
| Embeddings | Meaning and paraphrasing | May miss rare identifiers |

**Key idea:** Use keyword search for exact terms and embeddings for semantic meaning.

---

## 5. Hybrid search and reranking

### Hybrid search

**Hybrid search** combines:

- BM25 keyword search
- Vector semantic search

It combines the strengths of both approaches.

Because keyword and vector scores use different scales, hybrid systems commonly use:

- Reciprocal Rank Fusion
- Calibrated scores
- Weighted ranking

### Reranking

A **reranker** examines the query together with each candidate and reorders the candidates by relevance.

A typical workflow is:

1. Retrieve 20–50 candidates quickly.
2. Rerank those candidates.
3. Send only the best few to the LLM.

Reranking can improve answer quality but adds latency and cost.

### Metadata filtering

Metadata describes a document chunk:

- Document name
- Page number
- Department
- Country
- Effective date
- Document version
- Access group
- Source URL

Metadata filtering can restrict retrieval before the LLM sees it.

Permissions must be enforced in application code. The LLM should not be responsible for access control.

---

## 6. Vector databases and indexes

A vector database normally stores:

- An embedding vector
- Chunk text or a reference to the original text
- A unique chunk ID
- Metadata
- Optional source and permission information

For example, 500 policy PDFs might produce around 5,000 chunks.

### Flat search

Flat search compares a query against every stored vector.

- Produces exact nearest-neighbor results
- Is simple
- Becomes expensive for large collections

### HNSW

**HNSW** is a graph-based approximate nearest-neighbor index.

It creates multiple graph layers:

- Higher layers provide fast navigation.
- Lower layers provide detailed candidate search.

### IVF

**IVF** divides vectors into clusters.

The query:

1. Identifies promising clusters.
2. Searches only those clusters.

Searching more clusters improves recall but also increases latency.

### LSH

**LSH** uses hashing to place likely similar vectors in related buckets.

It is useful in some applications but is not always the default choice.

**Key idea:** Embeddings create vectors, vector databases store and search them, and indexes make searching faster.

---

## 7. Document ingestion and chunking

### Ingestion pipeline

A normal ingestion process:

1. Collect PDF, DOCX, HTML or database sources.
2. Extract text and structure.
3. Apply OCR to scanned PDFs.
4. Clean extraction errors.
5. Split documents into chunks.
6. Attach metadata and permissions.
7. Generate embeddings.
8. Store chunks and vectors.
9. Track document versions.

### What is a chunk?

A **chunk** is a small, meaningful section of a larger document.

The goal is to create sections that can answer a question independently.

### Chunking strategies

| Strategy | Best use |
|---|---|
| Character-based | Fast prototypes |
| Sentence-based | Narrative text |
| Paragraph-based | Structured text |
| Recursive splitting | General-purpose documents |
| Structure-aware | Policies and manuals |
| Semantic | Documents with changing topics |

### Chunk size

| Chunk size | Result |
|---|---|
| Too small | Can lose context |
| Too large | Can mix unrelated topics |
| Balanced | Contains focused, meaningful information |

A chunk size of around 1,000 characters with 200 characters of overlap is a useful starting experiment, but it should be measured and tuned.

### Chunk overlap

Overlap repeats some content at chunk boundaries.

```text
Chunk 1: words 1–100
Chunk 2: words 81–180
Overlap: words 81–100
```

This helps preserve context.

When preparing chunks:

- Keep headings with their content.
- Keep tables with their headings.
- Keep rules with conditions and exceptions.
- Preserve page numbers and document versions.
- Store source URLs or filenames.

**Key idea:** A good chunk is a coherent piece of information, not merely a fixed number of characters.

---

## 8. Complete RAG workflow

### Offline indexing

1. Load documents.
2. Extract text.
3. Clean and split text.
4. Add metadata.
5. Generate embeddings.
6. Build vector and keyword indexes.
7. Track source versions.

### Online answering

1. Receive the question.
2. Verify identity and permissions.
3. Generate a query embedding.
4. Search the knowledge base.
5. Optionally rerank results.
6. Build a prompt containing the question and evidence.
7. Generate an answer.
8. Return citations and sources.

### HR chatbot example

```text
500 policy PDFs
       ↓
Extract text and create chunks
       ↓
Generate embeddings and indexes
       ↓
Employee asks: “How many days of paternity leave are allowed?”
       ↓
Retrieve relevant policy sections
       ↓
LLM creates a grounded answer
       ↓
Response includes the policy source
```

### Grounded prompt

```text
Answer using only the supplied policy context.
If the answer is not in the context, say that it was not found.
Do not invent an entitlement.
Include relevant source references.
```

A prompt guides behavior, but retrieval quality and permissions remain essential.

### Important terms

| Term | Meaning |
|---|---|
| Retriever | Finds candidate chunks |
| Top-k | Number of candidates returned |
| Context window | Prompt and response token limit |
| Reranker | Reorders candidate chunks |
| Grounding | Supporting an answer with retrieved evidence |
| Citation | Identifies evidence for a claim |
| Hallucination | Unsupported or fabricated output |
| Ingestion | Preparing documents for search |
| Re-indexing | Rebuilding or updating an index |

---

## 9. Caching

Caching avoids repeating expensive work.

| Cache | Stores | Benefit |
|---|---|---|
| LLM response | Generated answers | Avoids repeated generation |
| SQL plan | Compiled SQL plans | Avoids repeated compilation |
| Embedding | Text-to-vector results | Avoids repeated embedding |
| Query embedding | Previous query vectors | Avoids repeated query embedding |
| Retrieval | Chunk IDs or search results | Avoids repeated searches |
| Prompt prefix | Reusable provider processing | May reduce prompt processing cost |

### Cache hit vs cache miss

- **Cache hit:** A suitable result already exists.
- **Cache miss:** The result must be created.

### Exact-match and semantic caches

An **exact-match cache** only reuses results for identical or normalized inputs.

A **semantic cache** reuses results for sufficiently similar questions.

Semantic caches require careful tuning because a similar question may have a different answer.

### Cache safety

Cache keys should include:

- Document version
- Effective date
- Tenant
- User or access scope
- Retrieval configuration
- Embedding model

A cached answer should be removed or refreshed when policies change.

**Key idea:** Cache only results that are correct, current and authorized.

---

## 10. Failure handling

A RAG system may depend on:

- A vector or keyword search system
- An embedding service
- An LLM provider
- A document store

### Example fallbacks

| Failure | Fallback |
|---|---|
| Vector database unavailable | Use an independently maintained BM25 index |
| Embedding unavailable | Use cached vectors or keyword search |
| LLM unavailable | Return verified source excerpts |
| Primary LLM unavailable | Switch to a secondary provider |

Do not combine document vectors from one model with query vectors from another model unless the models are compatible.

### Reliability techniques

- **Timeout:** Stop waiting after a specified period.
- **Retry:** Retry transient failures with exponential backoff and jitter.
- **Circuit breaker:** Pause calls to a repeatedly failing dependency.
- **Bulkhead:** Prevent one failure from consuming every resource.
- **Failover:** Switch to a standby system.
- **Observability:** Collect logs, metrics and traces.

### Circuit breaker states

- **Closed:** Requests work normally.
- **Open:** Calls are blocked temporarily.
- **Half-open:** Limited requests test whether the dependency recovered.

### Graceful degradation

The application should continue providing a reduced useful service.

For example, if the LLM is unavailable:

- Display matching policy excerpts.
- State that an answer could not be generated.
- Avoid guessing.

Every fallback must still enforce permissions and freshness.

---

## 11. RAG quality and security

### Evaluate retrieval and generation separately

| Area | Example measure |
|---|---|
| Retrieval | Recall@k, Precision@k |
| Ranking | MRR or nDCG |
| Grounding | Human faithfulness review |
| Useful answer | Human ratings or task success |
| Operations | Latency, cost and error rate |

### Security checklist

- Authenticate users.
- Enforce document-level permissions.
- Treat retrieved text as untrusted.
- Protect API credentials.
- Use least-privilege database accounts.
- Require approval for destructive actions.
- Avoid unnecessary storage of sensitive content.
- Log access and actions.
- Monitor retrieval quality and drift.

---

# Part 2: Model Context Protocol

## 12. What is MCP?

**Model Context Protocol (MCP)** is an open protocol for standardizing how AI applications connect to:

- Tools
- Data sources
- Resources
- External services

MCP is not:

- An LLM
- A database
- An agent
- An automation platform

It is a communication standard that an agent or AI application can use.

### MCP versus agents

| Agent / host | MCP |
|---|---|
| Plans and interprets goals | Defines the external interface |
| Decides which tools to use | Standardizes discovery and calls |
| Controls workflow | Carries requests and responses |

An agent can use MCP, but MCP does not automatically create an agent.

---

## 13. MCP architecture

### JSON-RPC

MCP commonly uses **JSON-RPC 2.0**.

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_database_status",
    "arguments": {
      "database": "EmployeeDB"
    }
  }
}
```

A response uses the matching request ID:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "EmployeeDB is ONLINE"
      }
    ]
  }
}
```

### Host, client and server

| Component | Meaning |
|---|---|
| Host | AI application the user interacts with |
| Client | Communication component inside the host |
| Server | Program exposing tools or data |

```text
User
  ↓
MCP host
  ↓
MCP client
  ↓ JSON-RPC
MCP server
  ↓
SQL Server, files, GitHub or another system
```

A host can connect to multiple servers.

---

## 14. Tools, resources and prompts

### Tools

**Tools perform operations.**

Examples:

- Get database status
- List failed SQL Agent jobs
- Run an approved read-only query
- Create an issue

Common methods:

- `tools/list`
- `tools/call`

Tools may have side effects. They require:

- Input validation
- Authorization
- Timeouts
- Audit logging
- Explicit confirmation for sensitive actions

### Resources

**Resources expose information for reading.**

Examples:

- Database schema
- Configuration
- Runbook
- Error-log excerpt
- Policy document

Common methods:

- `resources/list`
- `resources/read`

Resources provide context but do not by themselves define permissions.

### Prompts

**Prompts provide reusable instruction templates.**

Example:

```text
Review the supplied SQL Server wait statistics.
Identify likely bottlenecks.
Explain the evidence supporting each recommendation.
```

Common methods:

- `prompts/list`
- `prompts/get`

**Key idea:**

- Tools do something.
- Resources provide information.
- Prompts provide reusable instructions.

---

## 15. Transports

### stdio

**stdio** is used for local process communication.

The host starts a local server process and communicates through standard input and output.

Typical use:

- A local filesystem server
- A database server running on the same machine
- A local development integration

### Streamable HTTP

**Streamable HTTP** is used for remote MCP communication.

It is suitable for:

- Cloud-hosted MCP servers
- Remote integrations
- Streaming responses

The HTTP+SSE transport was replaced by Streamable HTTP in MCP specification 2025-03-26. Older implementations may still use it.

---

## 16. Building a Python MCP server

A first project can expose one read-only tool:

```text
get_database_status
```

### Implementation steps

1. Create a dedicated SQL Server login.
2. Give it minimum read-only permissions.
3. Create a Python function that checks database status.
4. Add input validation and a timeout.
5. Register it with the MCP Python SDK.
6. Connect the server to an MCP host.
7. Ask the host to check a database.
8. Verify the result against SQL Server.
9. Add structured audit logging.

Start with read-only tools:

- `get_database_status`
- `get_failed_jobs`
- `get_database_size`
- `get_blocking_summary`
- `get_backup_status`

Do not start with `DROP`, `DELETE`, `ALTER` or other production-changing operations.

---

## 17. Advanced MCP features

### Roots

**Roots** are URI-based locations exposed by a client to provide workspace context.

```text
file:///C:/SQLLogs/
file:///C:/DBA/Scripts/
```

Roots are location hints, not security boundaries. The server must independently restrict file paths and permissions.

### Sampling

**Sampling** allowed an MCP server to request LLM generation through the client or host.

In the 2026-07-28 MCP specification, sampling is deprecated for new implementations.

New implementations may integrate directly with an LLM provider instead.

### Elicitation

**Elicitation** allows a server to request missing information or confirmation.

Example:

> “Delete old backup files.”

The server should ask:

- Which folder?
- Which date range?
- Should archive or permanent deletion be used?

It should obtain explicit confirmation before proceeding.

### MRTR

**Multi Round-Trip Requests (MRTR)** is a pattern where:

1. A request needs additional input.
2. The server returns an input-required result.
3. The client gathers that input.
4. The original request continues.

MRTR is useful for incomplete or confirmation-sensitive operations.

---

## 18. MCP security

- Use least privilege.
- Prefer read-only tools initially.
- Validate every argument.
- Authenticate remote clients.
- Enforce authorization for every operation.
- Never hard-code credentials.
- Use TLS for remote communication.
- Use localhost for local HTTP servers where appropriate.
- Validate HTTP `Origin` headers.
- Require confirmation for destructive actions.
- Set timeouts and rate limits.
- Limit output size.
- Log the caller, tool, timestamp and outcome.
- Keep SDKs and dependencies current.
- Treat database and file content as untrusted input.

**Key idea:** MCP standardizes communication, but it does not automatically make tools safe.

---

# Part 3: Revision

## RAG versus MCP

| RAG | MCP |
|---|---|
| A pattern for grounded answer generation | A protocol for external AI connections |
| Retrieve, augment and generate | Discover and call capabilities |
| Uses embeddings, indexes and retrievers | Uses hosts, clients and servers |
| Example: answer from SQL runbooks | Example: retrieve live database status |

They can work together:

- MCP exposes a search tool.
- The tool searches a RAG knowledge base.
- The host uses the returned evidence to answer.

---

## Essential glossary

| Term | Plain-English meaning |
|---|---|
| LLM | Model that generates and interprets text |
| Embedding | Numerical text representation |
| Dimension | One coordinate in a vector |
| Semantic search | Search based on meaning |
| Cosine similarity | Measures vector direction similarity |
| TF-IDF | Keyword weighting by frequency and rarity |
| BM25 | Keyword ranking with length and repetition controls |
| Hybrid search | Keyword plus semantic search |
| Reranker | Reorders retrieved candidates |
| Vector database | Stores and searches vectors |
| HNSW | Graph-based approximate vector index |
| IVF | Cluster-based vector index |
| Chunk | Smaller section of a document |
| Chunk overlap | Shared content between neighboring chunks |
| Metadata | Information describing a chunk |
| Top-k | Number of results returned |
| Cache hit | Reusable value was found |
| Cache miss | Reusable value was not found |
| Circuit breaker | Temporarily stops calls to a failing dependency |
| MCP | Protocol for AI tool and data connections |
| Host | AI application |
| Client | MCP communication component |
| Server | Program exposing MCP capabilities |
| Tool | Callable action |
| Resource | Readable information |
| Prompt | Reusable instruction template |
| stdio | Local process transport |
| Streamable HTTP | Remote MCP transport |
| Elicitation | Request for input or confirmation |
| Sampling | Legacy model-generation request; deprecated for new implementations |
| Roots | Workspace or location context |
| MRTR | Request additional input and continue |

---

## Interview review questions

**What is RAG?**  
A method that retrieves relevant external information and supplies it to an LLM.

**What is an embedding?**  
A numerical vector representation of text or other supported content.

**What is semantic search?**  
Finding relevant content by meaning.

**What is cosine similarity?**  
A similarity measure based on the angle between vectors.

**What is the difference between BM25 and embeddings?**  
BM25 matches keywords; embeddings match meaning.

**What is hybrid search?**  
A combination of keyword and vector search.

**What is chunking?**  
Dividing documents into focused, retrievable sections.

**What is a reranker?**  
A model that reorders retrieved candidates.

**What happens if the embedding model changes?**  
Re-embed the documents and rebuild the compatible index.

**What is graceful degradation?**  
Providing a reduced but useful service when a dependency fails.

**What is MCP?**  
A standard protocol for AI applications to interact with external capabilities.

**What are MCP tools?**  
Operations that perform actions or return live information.

**What are MCP resources?**  
Read-only information exposed by a server.

**What are MCP prompts?**  
Reusable prompt templates.

**Is Cursor an MCP server?**  
No. Cursor can act as an MCP host.

**Are Roots a security boundary?**  
No. They provide location context. The server must enforce permissions.

---

## Quick self-test

1. Which component converts text into a vector?  
   **Embedding model**

2. What does top-k=5 mean?  
   **Return five highest-ranked candidates.**

3. Which search is best for exact error codes?  
   **BM25 keyword search.**

4. What combines keyword and semantic search?  
   **Hybrid search.**

5. What is chunk overlap for?  
   **Preserving context across chunk boundaries.**

6. Which index uses a graph?  
   **HNSW.**

7. Which cache stores generated answers?  
   **LLM response cache.**

8. What should happen if the LLM is unavailable?  
   **Use a fallback or show verified source excerpts.**

9. Which MCP component interacts with the user?  
   **Host.**

10. Which MCP primitive performs an action?  
    **Tool.**

11. Which MCP primitive provides read-only context?  
    **Resource.**

12. Which transport is used for local communication?  
    **stdio.**

13. What is required before a destructive DBA action?  
    **Authorization, validation, confirmation and audit logging.**

14. Is MCP the same as an AI agent?  
    **No.**

---

## Recommended practice project

Start with a **read-only SQL Server DBA knowledge assistant**:

1. Collect 5–10 SQL Server runbooks.
2. Extract and chunk the documents.
3. Create embeddings.
4. Build a vector or hybrid search index.
5. Add grounding and source citations.
6. Test retrieval and answer quality.
7. Add caching and permission controls.
8. Build a read-only MCP SQL Server server.
9. Connect the MCP server to an AI host.
10. Test failure, fallback and security behavior.

**The best first step is small:** make the RAG flow reliable and then add MCP capabilities gradually.
