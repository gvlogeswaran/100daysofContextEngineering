# Day 09 — The Context Engineering Stack: 6 Layers

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"A Context Engineering stack is like a fine restaurant. Suppliers (data sources), prep kitchen (chunking), sous chefs (retrieval), head chef (memory), plating (orchestration), dining experience (protocol delivery). Skip any station, and what reaches the table is broken."*

---

## Explanation of Day Topic

You've heard about RAG. You've implemented retrieval. But your system still feels brittle — queries that should work don't, the agent forgets things it shouldn't, and adding new tools creates unexpected failures elsewhere.

The reason is almost always the same: you're operating on one or two layers of a six-layer stack while treating the rest as someone else's problem.

The Context Engineering Stack is the complete architecture between your raw data and your model's response. Every layer is necessary. Every layer has distinct failure modes. And every layer that's missing or weak degrades everything built on top of it.

---

## Layer 1 — Data Sources: The Raw Material

### What it is

Every piece of data your AI system has potential access to: relational databases, document stores, APIs, event streams, file systems, message queues, time-series data.

### Why it matters

You cannot engineer context that doesn't exist. The quality ceiling for your entire AI system is set here. If your data sources are incomplete, unreliable, or inaccessible, every layer above is constrained by that limitation.

### Key engineering decisions

- **Coverage:** What data do you have? What gaps exist?
- **Freshness:** What is the update frequency? Real-time or daily batch?
- **Reliability:** What is the SLA? What happens when a source is unavailable?
- **Access patterns:** REST, SQL, GraphQL, streaming? This shapes Layer 2 requirements.

### For financial professionals

Your market data feeds, order management system, position tracker, risk engine, and compliance database are your Layer 1. The quality and coverage of these systems directly determines the intelligence ceiling of any AI built on top of them.

---

## Layer 2 — Chunking & Indexing: Making Data Retrievable

### What it is

The process of transforming raw data into a form that retrieval systems can search efficiently — breaking documents into chunks, creating embeddings, building indexes, attaching metadata.

### Why it matters

Raw data is not retrievable. A 500-page annual report cannot be passed to an LLM. Layer 2 is the pre-processing layer that makes Layer 3 possible.

### Key engineering decisions

```python
def chunk_financial_document(doc: str, doc_type: str) -> list[dict]:
    """Chunking strategy varies by document type."""
    if doc_type == "earnings_report":
        # Semantic chunking by section — preserve financial structure
        return chunk_by_section(doc, sections=["revenue", "guidance", "risks", "outlook"])
    elif doc_type == "analyst_note":
        # Fixed-size with overlap — dense text, preserve continuity
        return chunk_fixed_size(doc, chunk_tokens=400, overlap_tokens=50)
    elif doc_type == "regulatory_filing":
        # Hierarchical — keep parent context with child chunks
        return chunk_hierarchical(doc, parent_size=1500, child_size=400)
```

Decisions that matter: chunk size, overlap strategy, embedding model choice (domain-specific vs general), and metadata attachment (`{source, date, section, type}` per chunk for Layer 3 filtering).

---

## Layer 3 — Retrieval: Finding What Matters

### What it is

The layer that finds the most relevant chunks for a given query — using vector search, keyword search, hybrid approaches, and re-ranking.

### Why it matters

This is the signal-separation layer. Bad retrieval corrupts the entire context window regardless of how good the model is.

### Production retrieval pattern

```python
async def hybrid_retrieve(query: str, index: SearchIndex, top_k: int = 10) -> list[Chunk]:
    """Hybrid retrieval: semantic + keyword, then re-rank."""
    # Semantic: find conceptually similar chunks
    semantic_results = await index.vector_search(embed(query), top_k=top_k * 2)

    # Keyword: find exact term matches (critical for financial queries like ticker symbols)
    keyword_results = await index.keyword_search(query, top_k=top_k * 2)

    # Merge using Reciprocal Rank Fusion
    merged = reciprocal_rank_fusion([semantic_results, keyword_results])

    # Re-rank for precision
    return cross_encoder_rerank(query, merged)[:top_k]
```

Pure semantic search misses exact financial terms. Pure keyword search misses semantic relationships. Combined + re-ranked outperforms either alone.

---

## Layer 4 — Memory & State: Maintaining Context Across Time

### What it is

The layer that preserves information across multiple interactions — conversation history, learned preferences, past decisions and their outcomes, domain knowledge accumulated over time.

### Why it matters

LLMs are stateless by design. Without an explicit memory layer, every session starts from zero. Agents can't learn, adapt, or provide continuity — which is immediately obvious to users after their first multi-session interaction.

### The four memory types in production

```python
class AgentMemorySystem:
    def __init__(self, agent_id: str, user_id: str):
        self.working_memory = ConversationBuffer(max_tokens=20000)      # current session
        self.episodic_store = DynamoDBEpisodicStore(agent_id, user_id)  # past sessions
        self.semantic_store = OpenSearchSemanticStore(agent_id)          # accumulated knowledge
        self.procedural_store = ProceduralPatternStore(agent_id)         # learned tool patterns

    def retrieve_relevant_memory(self, query: str) -> dict:
        return {
            "recent_context": self.working_memory.get_recent(turns=10),
            "relevant_episodes": self.episodic_store.search(query, top_k=3),
            "relevant_knowledge": self.semantic_store.search(query, top_k=5),
            "applicable_patterns": self.procedural_store.match(query)
        }
```

### For financial professionals

A trading agent that remembers: this portfolio manager prefers momentum strategies, has a maximum drawdown tolerance of 8%, last discussed hedging AAPL exposure three sessions ago, and reviews positions on Friday afternoons.

---

## Layer 5 — Orchestration & Reasoning: Multi-Step Intelligence

### What it is

The coordination layer — where agents make decisions, call tools, handle errors, loop through multi-step workflows, and delegate to other agents.

### Why it matters

Complex decisions require multiple steps and feedback loops. A single retrieval-and-respond pass is insufficient for non-trivial tasks. Orchestration is where the intelligence of the system actually emerges.

### Production orchestration pattern

```python
async def execute_pre_trade_analysis(symbol: str, quantity: int, portfolio_id: str):
    context = ContextAccumulator()

    # Step 1: Parallel data fetches
    market_data, news = await asyncio.gather(
        get_live_quote(symbol),
        get_recent_news(symbol, hours=4)
    )
    context.add("market", market_data)
    context.add("news", news)

    # Step 2: Risk check (depends on Step 1)
    risk = await assess_risk(symbol, quantity, market_data, portfolio_id)
    context.add("risk", risk)

    if risk["breach_detected"]:
        return {"action": "block", "reason": risk["breach_reason"]}

    # Step 3: Compliance (depends on Steps 1 and 2)
    compliance = await check_compliance(symbol, quantity, market_data, risk)
    context.add("compliance", compliance)

    # Step 4: Recommendation (reasons over full accumulated context)
    return await generate_recommendation(context.assemble())
```

Each step's output becomes context for the next step. Miss orchestration and these steps become independent, uncoordinated calls.

---

## Layer 6 — Protocol Delivery: Standardised Context at Scale

### What it is

The standardised interface layer that exposes the five layers below to any AI model, any application, any client — without requiring custom integration code for each combination.

### Why it matters

Without standardisation, every application that wants to access your context system must reimplement connectivity to Layers 1–5. With a standard protocol, you build the delivery layer once and every compliant client benefits.

This is where **Model Context Protocol (MCP)** lives. MCP is the delivery mechanism — not the foundation. An MCP server that exposes a poorly-chunked index with no memory and no orchestration is just a well-plumbed system delivering bad context.

```
MCP Server = exposes Layers 1–5 as standardised Tools, Resources, and Prompts
MCP Client = consumes these primitives without knowing Layer 1–5 implementation details
```

---

## How the Layers Work Together

Data flows upward through the stack:
```
Layer 1 → raw data exists in databases and feeds
Layer 2 → data is chunked, embedded, indexed, tagged
Layer 3 → relevant chunks are retrieved for each query
Layer 4 → retrieved data is combined with persistent memory
Layer 5 → multi-step reasoning orchestrates across tools and agents
Layer 6 → the complete context package is delivered to the model via protocol
```

And inference flows back down — model decisions trigger memory updates, new retrievals, and data source writes. It is a bidirectional system.

---

## What Breaks When Each Layer Is Skipped

| Skipped Layer | Consequence |
|---------------|-------------|
| Layer 1 (Data Sources) | Everything downstream is limited by incomplete or unreliable data |
| Layer 2 (Chunking) | Retrieval cannot find specific information in large documents |
| Layer 3 (Retrieval) | Model only knows what's hardcoded into the prompt |
| Layer 4 (Memory) | Agents can't learn, adapt, or maintain state across sessions |
| Layer 5 (Orchestration) | Cannot build multi-step workflows or multi-agent systems |
| Layer 6 (Protocol) | Cannot scale to multiple models and applications |

Most teams operate with layers 1–2, partial layer 3, and nothing else. That is why they fail.

---

## Key Terms

| Term | What It Means |
|------|--------------|
| **Data Sources** | Layer 1 — all raw data the system can access |
| **Chunking** | Layer 2 — breaking raw data into retrievable, embeddable pieces |
| **Retrieval** | Layer 3 — finding relevant chunks via vector search, keyword search, and re-ranking |
| **Memory** | Layer 4 — maintaining state and learning across interactions and sessions |
| **Orchestration** | Layer 5 — coordinating multi-step workflows, tool calls, and agent delegation |
| **Protocol Delivery** | Layer 6 — standardised interface (MCP) exposing the stack to any client or model |

---

## What's Next

**Day 10 — Context Quality > Model Quality: The Biggest Mindset Shift**

Tomorrow we put a number on it: the empirical case for why investing in context quality yields 5–10x more return than investing in model quality — and why this single insight is the reason this 100-day series exists.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #AIArchitecture #SystemsDesign #AWSCommunityBuilder*

[← Day 08](./Day-08-Context-vs-Promt-Engineering.md) | [Day 10 →](./Day-10-The-Model-Matters.md)
