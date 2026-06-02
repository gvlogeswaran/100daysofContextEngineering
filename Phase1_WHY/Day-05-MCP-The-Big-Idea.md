# Day 05 — The 4 Types of Context Every LLM Uses

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"You wouldn't build a financial model using only historical data. You'd use market conditions, trading rules, past performance, and real-time prices. LLMs are no different — four distinct context types, each solving a different problem."*

---

## Explanation of Day Topic

On Day 04 we mapped the six-layer Context Engineering stack. Today we go one level deeper into the raw material of that stack: the four types of context that actually flow into any LLM at inference time.

Understanding these four types is foundational. Most engineers building AI systems in production are unknowingly using only two of them well. The third they manage poorly. The fourth — the most important one — they often get wrong entirely, and it's responsible for most production AI failures.

The four types are: **Parametric, Instructional, Conversational,** and **Retrieved**. Each has distinct properties, distinct failure modes, and distinct engineering requirements.

---

## Type 1 — Parametric Context: The Training Foundation

### What It Is

Parametric context is the knowledge baked into the model's weights during training. Every general fact the model "knows" without being told comes from here — language patterns, world knowledge, reasoning capabilities, common sense.

It's called "parametric" because it lives in the model's parameters (weights), not in the prompt.

### Properties

- **Fixed:** Cannot be updated without retraining the model
- **Scalable:** Extremely efficient — the model doesn't retrieve it, it's just there
- **Outdated:** Hard cutoff at training date — nothing after that point exists
- **Unreliable for specifics:** Hallucinations are most common when the model relies on parametric context for precise facts

### When It Works

- General reasoning ("explain how options pricing works")
- Background knowledge ("what is a central bank?")
- Language tasks that don't require current or proprietary data
- Coding patterns and standard algorithms

### When It Fails

- Any question about events after the training cutoff
- Company-specific, proprietary, or confidential information
- Real-time data (prices, availability, status, news)
- Precise numerical facts (the model will confidently hallucinate)

### Financial Markets Angle

Parametric context is your general understanding of financial markets — how futures work, what a basis point means, the mechanics of margin calls. It's necessary background knowledge but completely insufficient for trading decisions. You would never place a trade based solely on what you remember learning five years ago.

---

## Type 2 — Instructional Context: The System Prompt

### What It Is

Instructional context is the explicit guidance you provide to the model about how it should behave. This is the system prompt — the rules, role definition, constraints, output format, and guardrails you set for every interaction.

### Properties

- **Explicit:** You write it directly — full control
- **Persistent within a session:** Active for every turn in the conversation
- **Token-expensive:** A long system prompt consumes tokens that could hold other context
- **Authoritative:** Well-written instructions have significant influence on model behaviour

### When It Works

- Defining the model's role ("you are a pre-trade risk analyst")
- Setting hard constraints ("never recommend exceeding 5% position size")
- Specifying output format ("always respond in JSON with fields: recommendation, confidence, rationale")
- Encoding business rules and compliance requirements

### When It Fails

- When instructions conflict with retrieved data (ambiguity about which to trust)
- When instructions are vague or leave edge cases undefined
- When the system prompt grows so large it crowds out the context the model needs to reason over
- When instructions are too rigid for the variability of real-world inputs

### Engineering Pattern

```python
system_prompt = """
You are a pre-trade risk analyst for an institutional trading desk.

Rules:
- Always cite the data source for every claim
- Flag any recommendation that exceeds 2% of AUM as HIGH RISK
- If market data is older than 15 minutes, append [STALE DATA] to your response
- Output format: JSON with fields: action, confidence (0-100), risk_level, rationale, data_sources
"""
```

### Financial Markets Angle

Instructional context is your trading policy document. "Never take a directional position without a hedge." "All orders above $5M require two approvals." Rules are essential — but rules without data don't make trades. A well-crafted system prompt with no retrieved data still produces generic, unreliable output.

---

## Type 3 — Conversational Context: The Chat History

### What It Is

Conversational context is the running record of the current dialogue — every message exchanged between the user and the model since the conversation began. The model reads this history to understand continuity, resolve references, and build on previous reasoning.

### Properties

- **Immediate:** Reflects exactly what has been said in this session
- **Bounded:** The longer the conversation, the more tokens it consumes
- **Sequential:** Order matters — earlier context influences how later messages are interpreted
- **Ephemeral:** Does not persist across sessions without explicit memory architecture

### When It Works

- Multi-turn conversations ("do the same analysis for the tech sector")
- Referential continuity ("that stock you mentioned earlier...")
- Building progressively on a complex analysis across multiple exchanges
- Maintaining thread coherence in long analytical sessions

### When It Fails

- Very long conversations where early context falls outside the context window (truncation)
- When the model needs to "forget" a correction from earlier in the conversation
- Cross-session continuity (a new chat window has zero memory of yesterday's conversation)
- When conversation history crowds out the retrieved data the model actually needs

### Common Mistake

Teams that rely heavily on conversational context without memory architecture are surprised when their agents "forget" critical context in long sessions, or when users return the next day expecting continuity that doesn't exist.

### Financial Markets Angle

Conversational context is your order history within a single trading session. It's useful for sequential decisions — "given the position we just discussed, what's the hedge?" — but it doesn't persist across sessions. Without explicit memory architecture (Day 36–41 of this series), every new session starts from zero.

---

## Type 4 — Retrieved Context: The Live Intelligence Layer

### What It Is

Retrieved context is data you actively fetch from external sources — databases, APIs, search indexes, document stores, live feeds — and inject into the prompt at inference time. This is the core of RAG (Retrieval-Augmented Generation) and the foundation of MCP tool execution.

### Properties

- **Current:** Can reflect real-time or near-real-time data
- **Specific:** Targeted to exactly what this query needs
- **Scalable:** You don't need to fit all your data into the model — only the relevant slice
- **Engineering-intensive:** The quality of retrieval is entirely your responsibility

### When It Works

- Any task requiring current, specific, or proprietary data
- Domain-specific knowledge not in training data
- Personalised recommendations based on a user's actual history
- Data-driven decisions where accuracy matters more than approximation

### When It Fails

- **Imprecise retrieval:** The wrong data comes back (poor chunking, weak embedding, bad ranking)
- **Stale index:** Retrieved data hasn't been refreshed and no longer reflects reality
- **Context window overflow:** Too many retrieved chunks crowd out the reasoning space
- **Poor re-ranking:** Relevant data exists but is buried below irrelevant results

### Engineering Pattern

```python
def assemble_context(query: str, user_id: str) -> str:
    # Step 1: Retrieve relevant documents
    chunks = vector_store.search(query, top_k=5)
    
    # Step 2: Re-rank for precision
    ranked = reranker.rank(query, chunks)[:3]
    
    # Step 3: Fetch live data via MCP tool
    live_price = mcp_client.call_tool("get_quote", {"symbol": extract_symbol(query)})
    
    # Step 4: Assemble into context block
    retrieved_context = "\n\n".join([c.text for c in ranked])
    return f"{retrieved_context}\n\nLive data: {live_price}"
```

### Financial Markets Angle

Retrieved context is your live market data, order book, risk metrics, and position data. This is where the actual intelligence in a trading AI comes from. A trader operating without good retrieval is making decisions on memory and intuition alone — which is fine until the market moves and your mental model is wrong.

In pre/post trade systems, retrieved context must be:
- Fresh enough to be valid (staleness SLAs)
- Precise enough to be relevant (precision > recall for risk decisions)
- Complete enough to be safe (missing a critical data point is worse than having no data)

---

## How the Four Types Interact

The power of Context Engineering is in combining all four types appropriately:

| Combination | What It Enables |
|-------------|----------------|
| Parametric + Instructional | A model that knows how to reason AND follows your rules |
| Conversational + Parametric | Coherent multi-turn dialogue grounded in general knowledge |
| Retrieved + Instructional | Current data processed through principled constraints |
| All four together | Intelligent agency — trained to reason, rules to constrain, history for continuity, live data for accuracy |

### The Leverage Point

Of the four types, **Retrieved Context** is the one that Context Engineering invests the most effort in — because it's the only type you have full control over, the only type that reflects current reality, and the one most likely to determine whether your AI system is actually useful in production.

This is why Layers 2–4 of the CE stack (Retrieval, Memory, Orchestration) exist. They all exist to get the right Retrieved context to the model at the right time.

---

## Key Terms

| Term | What It Means |
|------|--------------|
| **Parametric Context** | Knowledge embedded in model weights during training — fixed, efficient, outdated |
| **Instructional Context** | Explicit guidance in the system prompt — rules, role, constraints, output format |
| **Conversational Context** | The running message history within a session — bounded, sequential, ephemeral |
| **Retrieved Context** | Data fetched from external sources at inference time — current, specific, engineered |
| **RAG** | Retrieval-Augmented Generation — the pattern of fetching external data before inference |
| **Context Quality** | How relevant, current, precise, and complete the context is for a specific task |

---

## What's Next

**Day 06 — The Context Window Is Your Most Valuable Real Estate**

Now that you know the four types of context, we tackle the hard constraint they all share: the context window. Tomorrow we cover token budgets, the "lost in the middle" research finding, priority ordering, and how to make strategic decisions about what earns a place in the context window.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #AIEngineering #PromptEngineering #LLM #AWSCommunityBuilder*

[← Day 04](./Day-04-ContextEngineering-Intro.md) | [Day 06 →](./Day-06-Context-window.md)
