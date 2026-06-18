# Day 45 — Phase 3 Complete: The Dynamic Context Playbook

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"The hardest part of production RAG isn't the LLM call — it's everything around it: chunking, memory, budgeting, compliance, cost, and retrieval strategy. You've now built all of it."*

---

## Explanation of Day Topic

Days 26 through 44 built a complete dynamic context system from the ground up. Day 26 diagnosed why static context fails. Days 27–35 covered the transport layer and local-first MCP foundations. Days 36–42 built the four-type memory architecture on AWS (working, episodic, semantic, procedural) with a full Lambda pipeline. Day 43 solved context window overflow with the sliding window pattern. Day 44 upgraded passive retrieval to agentic RAG.

Day 45 is the synthesis. It provides a decision tree for choosing the right RAG architecture for any domain, the complete AWS cost model at realistic scale, eight key learnings that each represent a hard lesson most teams learn the wrong way, and a preview of Phase 4: Model Context Protocol — the live data connection layer that sits on top of everything Phase 3 built.

After Day 45, you have everything you need to build a production RAG + memory system. What comes next is connecting it to data that changes in real time.

---

### Decision Tree: Choose Your RAG Architecture

```
START: "I need a RAG system"
  │
  ├─→ Q1: Document type?
  │     ├─ Structured (10-K, tables, financial reports)
  │     │     → Semantic chunking + table-aware parser
  │     └─ Unstructured (articles, transcripts, general PDFs)
  │           → Fixed-size chunks, 512 tokens, 20% overlap
  │
  ├─→ Q2: Query complexity?
  │     ├─ Simple factual ("What is AAPL's revenue?")
  │     │     → Passive RAG — single vector search
  │     ├─ Comparative ("AAPL vs MSFT earnings")
  │     │     → Agentic RAG — query decomposition
  │     └─ Sequential ("What caused the drop AND how did analysts react?")
  │           → Agentic RAG — multi-hop retrieval
  │
  ├─→ Q3: Scale?
  │     ├─ < 1M documents → Pinecone or pgvector
  │     └─ > 1M documents → OpenSearch Serverless or Bedrock Knowledge Bases
  │
  ├─→ Q4: Precision requirements?
  │     ├─ General Q&A → No re-ranking (top-5 vector is fine)
  │     └─ Financial / legal / medical → Cross-encoder re-ranking (Cohere or local)
  │
  ├─→ Q5: User memory needed?
  │     ├─ No (stateless) → Passive RAG only
  │     └─ Yes → Episodic (DynamoDB) + semantic (OpenSearch) memory
  │
  └─→ Q6: Long sessions (> 30 turns)?
        ├─ No → Standard conversation buffer
        └─ Yes → Sliding window with pinning + compression

RESULT: Your production RAG architecture
```

---

### Phase 3 Summary: The Complete Technical Stack

| Component | Purpose | Technology |
|-----------|---------|-----------|
| **Chunking** | Split documents into retrieval-sized pieces | Semantic splitter + 20% overlap |
| **Embeddings** | Convert text to vectors | Amazon Titan Embed v2 (1536-dim) |
| **Vector DB** | Index and search by similarity | OpenSearch Serverless (k-NN) |
| **BM25 Index** | Keyword-based complementary search | OpenSearch BM25 |
| **Hybrid Search** | Semantic + keyword fusion | Reciprocal Rank Fusion (RRF) |
| **Re-ranking** | Precision improvement on top-k results | Cohere Rerank or cross-encoder |
| **Generation** | Grounded answer from retrieved context | Claude 3.5 Sonnet via Bedrock |
| **Episodic Memory** | Per-user conversation history | DynamoDB (PK/SK, TTL=90 days) |
| **Semantic Memory** | User profile facts and entity graph | OpenSearch Serverless (k-NN) |
| **Context Window** | Active conversation buffer | Sliding window (Python deque) |
| **Orchestration** | Pipeline coordination | AWS Lambda |
| **Evaluation** | Measure retrieval and generation quality | RAGAS framework |

---

### Cost Model: Production RAG + Memory

**Scenario:** 10M documents, 100K monthly queries, 5-year retention

| Component | Monthly Cost | Notes |
|-----------|-------------|-------|
| OpenSearch Serverless | $175 | Vector + BM25, 2 OCU minimum |
| S3 Document Store | $230 | ~10GB for 10M avg-1KB docs |
| Bedrock Embeddings (Titan v2) | $100 | 100K queries × 500 tokens avg |
| DynamoDB Episodic Memory | $50 | 1M writes; TTL handles deletion |
| Bedrock Generation (Claude Sonnet) | $500–2,000 | Scales with context length |
| Lambda Orchestration | $15 | 100K × 500ms, PAY_PER_REQUEST |
| **Total** | **$1,070–2,570/month** | Scales linearly with query volume |

**Top cost optimisations:**
- Cache query embeddings in ElastiCache → saves $100/month at 100K queries
- Use Haiku for summarisation tasks → ~10× cheaper than Sonnet
- Aggressive DynamoDB TTL (30 days for low-value sessions) → saves storage
- Bedrock Provisioned Throughput → 40–60% savings at high sustained volume

---

### Eight Key Learnings from Phase 3

**1. Chunking is foundational.** Wrong chunk size breaks retrieval before the LLM sees a single token. For financial documents: semantic chunking with table-aware parsing. For general text: 512 tokens with 20% overlap. Test with real retrieval queries before indexing at scale.

**2. Hybrid search wins.** BM25 catches exact ticker symbols and proper nouns that vector search misses. Vector search catches semantic similarity that BM25 misses. Reciprocal Rank Fusion of both consistently outperforms either alone. Never skip BM25.

**3. Re-ranking is worth the latency for precision workloads.** For general Q&A: top-5 vector results are good enough. For financial and legal applications: cross-encoder re-ranking gives 20–40% precision improvement. The 200ms latency cost is worth it when wrong answers have consequences.

**4. Memory is not optional.** A stateless agent that forgets every session is a search engine with extra steps. Episodic memory (DynamoDB) and semantic memory (vector store) transform a one-shot tool into a personalised assistant that improves over time.

**5. Token budget management is the invisible ceiling.** Define your token budget (system, pinned, RAG, conversation, reserve) before you build. Most teams hit this wall at production and discover the fix is architectural, not incremental.

**6. Agentic retrieval pays off for complex queries.** Passive RAG is faster and cheaper for simple factual questions. For comparative, multi-entity, or sequential reasoning queries, decomposition and multi-hop retrieval produce materially better answers. Know which workload you have before choosing a pattern.

**7. Compliance is first-class from day one.** GDPR erasure and CCPA opt-out cannot be bolted on after launch. The single-table DynamoDB design (PK=USER#id) enables single-query full erasure. Audit logging needs a separate partition from day one. Design for compliance before your first user.

**8. Evaluation is not optional.** RAGAS (faithfulness, answer relevancy, context precision, context recall) gives you four numbers that tell you whether your pipeline actually works. Without evaluation, architectural changes are guesses. Run RAGAS on every significant change.

---

### Phase 3 Cheat Sheet: The Complete Stack in 25 Lines

```python
# Chunk
chunks = SemanticChunker(titan_embeddings).split_documents(raw_docs)

# Embed + Index
for chunk in chunks:
    vec = titan_embed(chunk.page_content)
    opensearch.index(index='docs', body={'text': chunk.page_content, 'embedding': vec})

# Hybrid Retrieve
semantic_hits = knn_search(embed(query), top_k=10)
keyword_hits  = bm25_search(query, top_k=10)
combined      = reciprocal_rank_fusion(semantic_hits, keyword_hits)

# Re-rank (financial/legal only)
reranked = cohere_rerank(query=query, documents=combined, top_n=5)

# Generate
answer = claude_bedrock(
    system=build_system(pinned_context, user_profile),
    context=reranked,
    messages=sliding_window.get_messages()
)

# Memory — write back
episodic.store(user_id, query, answer)         # DynamoDB
semantic.upsert(user_id, extract_entities(answer))  # OpenSearch
sliding_window.add("user", query)
sliding_window.add("assistant", answer)
```

---

## Phase 4 Preview: Model Context Protocol (Days 46–65)

Phase 3 built the infrastructure for grounding LLMs in static or periodically updated documents. Phase 4 solves the next constraint: data that changes faster than you can re-ingest it.

Stock prices change by the millisecond. Order books update thousands of times per second. News breaks continuously. Account balances change with every transaction. No RAG pipeline can keep up with this velocity via batch ingestion.

MCP (Model Context Protocol) is Anthropic's open standard for connecting LLMs to live data sources. Instead of ingesting data into a vector store, an MCP server exposes data as real-time tools and resources that the LLM calls on-demand, within a structured protocol, with proper capability negotiation and authentication.

**Phase 4 topics:**
- Days 46–48: MCP concepts, Host/Client/Server architecture, JSON-RPC foundations
- Days 49–52: MCP primitives — Tools, Resources, Prompts, Sampling
- Days 53–55: Building MCP servers in Python and TypeScript
- Days 56–58: Security, production hardening, AWS Lambda deployment
- Days 59–60: Financial market data MCP server + Phase 4 complete review

The shift: Phase 3 = LLM retrieves from a static store. Phase 4 = LLM calls live APIs via standard protocol. Same grounding principle. Real-time data freshness.

---

## Final Thoughts on Phase 3

Twenty days ago, you understood why static prompts fail.

Today, you can architect a production system that retrieves the right documents, remembers every user, manages the context window across a 3-hour session, and runs agentic retrieval for complex multi-entity questions — all on AWS-native infrastructure.

That is not a demo. That is production context engineering.

See you in Phase 4. 🚀

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #Phase3Complete #AWSCommunityBuilder*

[← Day 44](./Day-44-Agentic-RAG.md) | [Phase 4 Begins: Day 46 →](../Phase4_Primitives/Day_46_MCP_Welcome_Back.md)
