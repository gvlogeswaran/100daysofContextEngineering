# Day 07 — The 5 Enemies of Good Context

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"Context is like food. Noise is dirt in your dish. Staleness is expired ingredients. Contradiction is a recipe that contradicts itself. Each enemy has a different cure — and each one is entirely the Context Engineer's responsibility."*

---

## Explanation of Day Topic

You have a well-allocated context window. You're retrieving from good sources. Your prompts are clear. And your AI is still giving wrong answers.

The culprit is almost always one of five context quality enemies: **noise, contradiction, staleness, over-compression, and under-specification**. These are the silent killers of production AI systems. They're invisible until something breaks, and they're entirely within the Context Engineer's control to prevent.

Most AI failures get blamed on the model. The real diagnosis is usually a context quality problem. Learn these five enemies now, before you debug them in production.

---

## Enemy 1 — Noise: The Signal-to-Noise Problem

### What it is

Irrelevant information included in the context that consumes token budget without improving the model's reasoning — and actively degrades it by diluting the signal.

### Why it happens

- Retrieval returns everything *somewhat* related instead of precisely what's needed
- Data pipelines include metadata and background the model doesn't need for this task
- No filtering or re-ranking applied before context assembly
- "More is safer" thinking — engineers include extra context just in case

### Real example

A support agent retrieves "all customer interactions from the last year" when it only needs "payment-related interactions from the last 30 days." The model reads 11 months of irrelevant exchanges before finding the two records that matter.

### How Context Engineers defend against it

```python
def retrieve_relevant_context(query: str, customer_id: str) -> list:
    # 1. Filter by relevance category upfront
    relevant_types = classify_query_intent(query)  # e.g., ["payment", "billing"]

    # 2. Apply date filter
    cutoff = datetime.now() - timedelta(days=30)

    # 3. Retrieve and rank — enforce top-k bound
    results = vector_store.search(
        query=query,
        filters={"customer_id": customer_id, "type": relevant_types, "date": {"gte": cutoff}},
        top_k=5
    )

    # 4. Re-rank for precision
    return reranker.rank(query, results)[:3]
```

**Cost if ignored:** Model spends attention on irrelevant data, diluting signal. Accuracy drops. Context window fills faster.

---

## Enemy 2 — Contradiction: The Truth Problem

### What it is

Two or more pieces of context that directly conflict, forcing the model to choose between them or produce uncertain, hedged output.

### Why it happens

- Data pulled from multiple systems with different update cycles
- A record updated in System A but not yet synced to System B
- Historical data not clearly distinguished from current data

### Real example

A customer record shows `status: gold_member` but the orders API returns zero purchases in the past 18 months. The model guesses which is authoritative — and usually guesses wrong.

### How Context Engineers defend against it

```python
def validate_context_consistency(customer_data: dict, orders_data: dict) -> dict:
    issues = []
    last_order_days_ago = (datetime.now() - orders_data["last_order_date"]).days

    if customer_data["status"] == "gold_member" and last_order_days_ago > 365:
        issues.append({
            "field": "status",
            "conflict": f"Gold status but no purchase in {last_order_days_ago} days",
            "authoritative_source": "orders_api",  # define hierarchy explicitly
            "resolution": "flag_for_review"
        })

    return {**customer_data, "data_quality_flags": issues}
```

**Source hierarchy rule:** Define upfront which source wins when sources conflict. Make this explicit in your system prompt so the model knows how to resolve conflicts it encounters.

**Cost if ignored:** In trading contexts, a contradiction between the risk limit and position record will produce paralysed or incorrect decisions.

---

## Enemy 3 — Staleness: The Recency Problem

### What it is

Data that was accurate when created but is now outdated — treated as current by the model because no freshness metadata was provided.

### Why it happens

- Caching layers that refresh infrequently (daily batch, not real-time)
- Index pipelines with lag between source update and vector store update
- No TTL (time-to-live) enforcement on retrieved data

### Real example

A trading agent retrieves a stock quote from a cache last refreshed at market open. By mid-afternoon the price has moved 3%. The agent recommends a trade based on a price four hours stale.

### How Context Engineers defend against it

```python
def retrieve_with_freshness_check(symbol: str, max_age_seconds: int = 60) -> dict:
    cached = cache.get(f"quote:{symbol}")

    if cached:
        age = (datetime.now() - cached["retrieved_at"]).total_seconds()
        if age > max_age_seconds:
            fresh = market_data_api.get_quote(symbol)
            cache.set(f"quote:{symbol}", {**fresh, "retrieved_at": datetime.now()})
            return {**fresh, "data_freshness": "live"}
        return {**cached, "data_freshness": f"cached_{int(age)}s_ago"}

    fresh = market_data_api.get_quote(symbol)
    cache.set(f"quote:{symbol}", {**fresh, "retrieved_at": datetime.now()})
    return {**fresh, "data_freshness": "live"}
```

**Always tag retrieved data with a freshness indicator** so the model can reason about reliability: `[LIVE]`, `[CACHED: 4min ago]`, `[STALE: 3hr ago — verify before acting]`.

**Cost if ignored:** In financial markets, stale data means wrong decisions. In compliance systems, stale rules mean regulatory exposure.

---

## Enemy 4 — Over-compression: The Loss of Detail

### What it is

Information compressed so aggressively that critical nuance, edge cases, and contextual detail are lost — leaving the model with an accurate-but-useless summary.

### Why it happens

- Trying to fit too much into a limited context window
- Automatic summarisation pipelines set too aggressively
- Treating all information as equally compressible

### Real example

A customer's 10-year relationship summarised as: "Long-term customer, mostly satisfied." Lost in that compression: they threatened to leave last quarter over a billing dispute, have a price-sensitive tier, and their contract renews in 6 weeks.

### How Context Engineers defend against it

```python
def compress_customer_history(interactions: list) -> dict:
    # Hierarchical compression — preserve structure, compress volume
    return {
        # Never compress — always include verbatim
        "critical_flags": [i for i in interactions if i.get("severity") == "critical"],
        "upcoming_events": [i for i in interactions if i.get("date") > datetime.now()],

        # Compress moderately — keep key data points
        "recent_interactions": [
            {"date": i["date"], "type": i["type"], "outcome": i["outcome"]}
            for i in interactions[-5:]
        ],

        # Compress aggressively — just counts and categories
        "historical_summary": {
            "total_interactions": len(interactions),
            "avg_satisfaction": sum(i["rating"] for i in interactions) / len(interactions),
            "primary_contact_reason": Counter(i["type"] for i in interactions).most_common(1)[0][0]
        }
    }
```

**The principle:** Compress proportionally to importance. Critical data stays verbatim. Important context gets structured. Background gets aggregated.

**Cost if ignored:** The model makes decisions with an oversimplified view of reality, missing the edge case that was in the compressed detail.

---

## Enemy 5 — Under-specification: The Ambiguity Problem

### What it is

Context that is technically present but so vague, ambiguous, or unstructured that the model cannot reason precisely from it.

### Why it happens

- Natural language is inherently ambiguous without domain grounding
- Missing units, scales, and definitions
- No schema or structure to anchor interpretation

### Real example

`"Sentiment: mixed"` versus `"Sentiment score: 2.8/5.0 | Positive: 38% | Negative: 42% | Neutral: 20% | Top complaint theme: delivery delays (47 mentions)"`. The first is a word. The second is actionable context.

### How Context Engineers defend against it

```python
# Use typed, defined schemas instead of free-text fields

TRADE_CONTEXT_SCHEMA = {
    "symbol": str,                                          # "AAPL" not "Apple"
    "price_usd": float,                                     # 185.42 not "about $185"
    "position_shares": int,                                 # 1000 not "roughly a thousand"
    "risk_limit_pct_aum": float,                            # 2.5 not "moderate risk"
    "time_in_force": Literal["DAY", "GTC", "IOC"],          # enumerated not free text
    "data_as_of": datetime,                                 # explicit timestamp
}

def format_trade_context(raw: dict) -> str:
    validated = TradeContext(**raw)  # Pydantic validation catches ambiguity at source
    return json.dumps(validated.dict(), indent=2, default=str)
```

**Cost if ignored:** The model applies general knowledge instead of domain-specific precision. In financial systems, "moderate risk" means different things to different firms.

---

## Integrated Defence: All Five in One Pipeline

```python
def assemble_production_context(query: str, customer_id: str) -> str:
    # Enemy 1 — Noise: filter and rank
    interactions = retrieve_relevant_context(query, customer_id)

    # Enemy 2 — Contradiction: validate consistency
    customer = validate_context_consistency(get_customer(customer_id), get_orders(customer_id))

    # Enemy 3 — Staleness: enforce freshness with TTL
    account_status = retrieve_with_freshness_check(customer_id, max_age_seconds=300)

    # Enemy 4 — Over-compression: hierarchical compression
    history_summary = compress_customer_history(customer["interactions"])

    # Enemy 5 — Under-specification: typed schema, explicit units
    return format_structured_context({
        "customer": customer,
        "account_status": account_status,
        "relevant_history": interactions,
        "history_summary": history_summary
    })
```

The result: context that is signal-rich, consistent, current, detail-preserving, and precisely specified.

---

## Key Terms

| Term | What It Means |
|------|--------------|
| **Noise** | Irrelevant information consuming context budget without improving model reasoning |
| **Contradiction** | Conflicting information from multiple sources forcing the model to guess which is authoritative |
| **Staleness** | Outdated information treated as current due to missing freshness metadata or TTL enforcement |
| **Over-compression** | Aggressive summarisation that removes critical nuance and edge cases |
| **Under-specification** | Vague or ambiguous context lacking structure, units, or definitions needed for precise reasoning |
| **Signal-to-Noise Ratio** | The proportion of useful, actionable information to irrelevant information in the context |

---

## What's Next

**Day 08 — Context Engineering vs Prompt Engineering: The Upgrade**

Now that you understand what degrades context quality, tomorrow we draw the clear line between Prompt Engineering (tactical, per-request) and Context Engineering (strategic, system-wide) — and explain why one depends on the other.

---

*#100DaysOfContextEngineering #ContextEngineering #DataQuality #RAG #AIEngineering #AWSCommunityBuilder*

[← Day 06](./Day-06-MCP-Origin-Story.md) | [Day 08 →](./Day-08-The-MCP-Ecosystem.md)
