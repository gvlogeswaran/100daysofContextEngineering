# Day 11 — The Context Engineering Lifecycle: From Raw Data to Model Response

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"The Context Engineering lifecycle is like water treatment. Raw water → Intake → Filtration → Storage → Pressure → Distribution → Tap. Miss one stage and the water isn't drinkable. Most teams build one or two stages and wonder why the output is contaminated."*

---

## Explanation of Day Topic

You know the 6-layer stack from Day 09. But knowing the architectural layers is different from understanding how information actually moves through them — in real time, in a production system, under operational load.

This day traces the complete journey a piece of data takes from its origin in your systems to the moment an AI model reasons over it. Seven stages. Each one with distinct responsibilities, failure modes, and engineering decisions. Each one dependent on the stages before it.

Understanding this lifecycle changes how you architect systems. Instead of thinking in isolated components — "we have a vector database" or "we have a retrieval step" — you start thinking in integrated flows: what enters Stage 1, what must be true by Stage 3, what can break at Stage 5, and how the model's response at Stage 7 feeds back to update Stage 4.

This is the systems-thinking shift that separates engineers who build demos from engineers who build production AI.

---

## Stage 1 — Source: Where Data Lives

Every context engineering system begins with raw data distributed across your organisation.

**Operational systems:**
- Relational databases (PostgreSQL, MySQL, Aurora)
- Document databases (MongoDB, DynamoDB)
- Key-value stores (Redis, ElastiCache)

**Event streams:**
- Message queues (Kafka, AWS SQS, RabbitMQ)
- Event logs (CloudTrail, application telemetry)
- Time-series feeds (market data, IoT sensors, metrics)

**External sources:**
- Third-party APIs (market data providers, news APIs, weather)
- SaaS platforms (Slack, Jira, Salesforce, ServiceNow)
- File systems (S3, Google Cloud Storage, shared drives)

**What matters at this stage:**

| Property | Engineering Question |
|----------|---------------------|
| Coverage | What data exists? What gaps remain? |
| Freshness | What is each source's update frequency? |
| Reliability | What is the SLA? What happens if a source is down? |
| Access pattern | REST API, SQL query, streaming event, file read? |

**The ceiling rule:** You cannot engineer context that doesn't exist at Layer 1. The quality of your entire AI system is bounded by what data you have and can access. No retrieval algorithm, no matter how sophisticated, retrieves data that was never ingested.

---

## Stage 2 — Ingestion: Getting Data Into Your Pipeline

Ingestion is the bridge between external sources and your processing system.

**Ingestion patterns:**

```python
# Pull pattern: your system queries the source
async def batch_ingest_positions(portfolio_id: str) -> list[dict]:
    """Periodic pull from order management system."""
    positions = await oms_client.get_positions(portfolio_id)
    return [normalize_position(p) for p in positions]

# Push pattern: the source sends data to you
@webhook_handler("/market-data/quote")
async def receive_quote_push(event: dict) -> None:
    """Real-time push from market data feed."""
    quote = validate_quote_schema(event)
    await ingestion_queue.put(quote)

# Streaming pattern: continuous event consumption
async def consume_trade_stream(stream_name: str) -> AsyncGenerator:
    """Kinesis stream consumption for real-time trades."""
    async for shard_iterator in kinesis.get_shard_iterator(stream_name):
        async for record in kinesis.get_records(shard_iterator):
            yield deserialize_trade(record)
```

**Engineering decisions at this stage:**

- **Latency:** How stale can data be? Real-time (< 1s), near-real-time (< 1min), or batch (hourly/daily)?
- **Completeness:** Are you ingesting all records or a sample? Is the subset representative?
- **Reliability:** Circuit breakers, retry logic, dead-letter queues for failed ingestion
- **Schema evolution:** What happens when the source changes its data structure?

**AWS mapping:**

| Need | AWS Service |
|------|------------|
| Streaming ingestion | Kinesis Data Streams |
| Event-triggered ingestion | Lambda + EventBridge |
| Batch ingestion | AWS Glue, AWS DMS |
| API ingestion | API Gateway + Lambda |

---

## Stage 3 — Processing: Cleaning and Structuring

Raw ingested data is almost never in a form an AI system can use directly. Stage 3 transforms it.

**Core transformations:**

```python
def process_financial_record(raw: dict) -> dict:
    """Clean, validate, and structure a raw financial record."""

    # Validation: reject structurally invalid records
    if not raw.get("symbol") or not raw.get("timestamp"):
        raise IngestionError(f"Missing required fields in record: {raw.keys()}")

    # Normalisation: standardise formats
    record = {
        "symbol": raw["symbol"].upper().strip(),          # "aapl " → "AAPL"
        "price": float(raw["price"]),                      # "185.42" → 185.42
        "timestamp": parse_iso_timestamp(raw["timestamp"]), # consistent tz handling
        "source": raw.get("source", "unknown"),
        "ingested_at": datetime.utcnow().isoformat(),
    }

    # Enrichment: add computed fields
    record["trading_session"] = classify_session(record["timestamp"])  # pre/reg/post
    record["price_formatted"] = f"${record['price']:.2f}"

    # Deduplication key: prevents re-processing
    record["dedup_key"] = f"{record['symbol']}:{record['timestamp']}"

    return record
```

**What Stage 3 failures look like downstream:**
- Inconsistent formats → Stage 5 retrieval returns mixed, unparseable results
- Missing validation → Stage 6 assembly includes null fields that confuse the model
- No timestamps → Stage 7 model cannot reason about data freshness

**AWS mapping:** Lambda (transformation logic), Glue DataBrew (visual transformation), Step Functions (orchestrating multi-step pipelines), Glue for batch ETL.

---

## Stage 4 — Storage & Indexing: Making Data Retrievable

Processed data must be stored in a form that Stage 5 retrieval can search efficiently.

**The storage landscape:**

```
Query type                   → Optimal store
─────────────────────────────────────────────
"Find record with ID X"      → DynamoDB (key-value)
"Semantic similarity to Q"   → OpenSearch or pgvector (vectors)
"Text contains keyword K"    → OpenSearch (full-text)
"Prices from 9am–10am"       → Timestream (time-series)
"Records where field = V"    → RDS/Aurora (relational)
"Entity A relates to B via…" → Neptune (graph)
```

**Embedding generation for semantic search:**

```python
async def index_document(doc: dict, bedrock_client) -> None:
    """Generate embedding and store for semantic retrieval."""
    text_for_embedding = build_embedding_text(doc)

    # Generate embedding using Bedrock Titan
    response = bedrock_client.invoke_model(
        modelId="amazon.titan-embed-text-v2:0",
        body=json.dumps({"inputText": text_for_embedding})
    )
    embedding = json.loads(response["body"].read())["embedding"]

    # Store with metadata for filtered retrieval
    await opensearch_client.index(
        index="financial-documents",
        body={
            "text": text_for_embedding,
            "embedding": embedding,
            "metadata": {
                "source": doc["source"],
                "symbol": doc.get("symbol"),
                "timestamp": doc["timestamp"],
                "doc_type": doc["type"],
                "ingested_at": datetime.utcnow().isoformat(),
            }
        }
    )
```

**Indexing decisions that matter:**
- **Chunk size:** Smaller chunks = more precise retrieval, more storage. Larger chunks = more context per result, less precise.
- **Embedding model:** Domain-specific models outperform general models on specialised vocabulary (financial terms, medical terminology, legal language).
- **Metadata schema:** Metadata fields enable Stage 5 pre-filtering before vector search — critical for filtering by date, source, type, or entity.

---

## Stage 5 — Retrieval: Finding What's Relevant

Stage 5 is the most visible part of the pipeline — and the most commonly over-optimised relative to Stages 1–4.

```python
async def retrieve_context_for_query(
    query: str,
    filters: dict,
    top_k: int = 10
) -> list[dict]:
    """Production hybrid retrieval with metadata filtering and re-ranking."""

    # Embed the query
    query_embedding = await generate_embedding(query)

    # Semantic search: conceptual similarity
    semantic_results = await opensearch_client.search(
        index="financial-documents",
        body={
            "knn": {"field": "embedding", "query_vector": query_embedding, "k": top_k * 2},
            "filter": build_opensearch_filter(filters),   # pre-filter by metadata
            "size": top_k * 2
        }
    )

    # Keyword search: exact term matching (critical for tickers, codes, proper nouns)
    keyword_results = await opensearch_client.search(
        index="financial-documents",
        body={
            "query": {"match": {"text": query}},
            "filter": build_opensearch_filter(filters),
            "size": top_k * 2
        }
    )

    # Merge via Reciprocal Rank Fusion
    merged = reciprocal_rank_fusion([
        semantic_results["hits"]["hits"],
        keyword_results["hits"]["hits"]
    ])

    # Re-rank for final precision using a cross-encoder
    reranked = cross_encoder_rerank(query, merged)

    return reranked[:top_k]
```

**What Stage 5 cannot fix:**
- Data that was never ingested (Stage 1 gap)
- Data that failed validation (Stage 3 rejection)
- Data that was chunked incorrectly (Stage 4 chunking error)

This is why optimising Stage 5 in isolation produces diminishing returns. The retrieval algorithm cannot compensate for upstream deficiencies.

---

## Stage 6 — Assembly: Formatting Context for the Model

Stage 6 takes retrieved data and assembles it into the structured context the model will reason over.

```python
def assemble_context_for_query(
    query: str,
    retrieved_docs: list[dict],
    conversation_history: list[dict],
    system_config: dict
) -> str:
    """Assemble all context sources into a structured prompt."""

    sections = []

    # Zone 1: Foundation (first — always read, frames everything)
    sections.append(f"SYSTEM INSTRUCTIONS:\n{system_config['system_prompt']}")

    # Zone 2: Retrieved data (most relevant first — critical before middle)
    if retrieved_docs:
        formatted_docs = []
        for i, doc in enumerate(retrieved_docs, 1):
            freshness = classify_freshness(doc["metadata"]["timestamp"])
            formatted_docs.append(
                f"[Source {i}] [{freshness}] {doc['metadata']['doc_type'].upper()}\n"
                f"Source: {doc['metadata']['source']}\n"
                f"As of: {doc['metadata']['timestamp']}\n"
                f"{doc['text']}\n"
            )
        sections.append("RETRIEVED CONTEXT:\n" + "\n---\n".join(formatted_docs))

    # Zone 3: Conversation history (summarised if long)
    if conversation_history:
        sections.append(
            "CONVERSATION HISTORY:\n" + format_conversation(conversation_history)
        )

    # Zone 4: Task (last — immediately before generation, maximum salience)
    sections.append(f"CURRENT TASK:\n{query}")

    assembled = "\n\n".join(sections)

    # Token budget enforcement
    if count_tokens(assembled) > system_config["max_context_tokens"]:
        assembled = compress_to_budget(assembled, system_config["max_context_tokens"])

    return assembled
```

**Assembly decisions that directly affect model output:**
- **Zone placement:** Critical information at Zone 1 (beginning) or Zone 4 (end) — never buried in the middle ("Lost in the Middle" effect)
- **Freshness labelling:** Always tag retrieved data with `[LIVE]`, `[CACHED: Xmin ago]`, `[STALE: verify]`
- **Structure over prose:** JSON and labelled fields use fewer tokens and give the model cleaner anchors than equivalent narrative text

---

## Stage 7 — Delivery: Sending to the Model and Closing the Loop

Stage 7 is where assembled context reaches the model and a response is generated.

```python
async def deliver_and_respond(
    assembled_context: str,
    model_config: dict
) -> dict:
    """Deliver context to model and handle response."""

    response = await bedrock_client.converse(
        modelId=model_config["model_id"],
        messages=[{"role": "user", "content": [{"text": assembled_context}]}],
        inferenceConfig={
            "maxTokens": model_config["max_output_tokens"],
            "temperature": model_config["temperature"],
        }
    )

    result = {
        "response": response["output"]["message"]["content"][0]["text"],
        "usage": response["usage"],
        "stop_reason": response["stopReason"],
    }

    # Close the loop: update memory with this interaction
    await update_episodic_memory(
        query=assembled_context,
        response=result["response"],
        metadata={"timestamp": datetime.utcnow().isoformat(), "usage": result["usage"]}
    )

    return result
```

**The feedback loop — where Stage 7 connects back to Stage 4:**

```
Model response triggers new context need
→ Stage 5 retrieves additional data
→ Stage 6 re-assembles with new data
→ Stage 7 delivers updated context
→ Model generates refined response

This loop is where agentic systems operate.
```

---

## The Complete Pipeline: End-to-End Trading Decision

```
Stage 1 — Source:
  Order management system (positions), market data feed (prices), risk engine (limits)

Stage 2 — Ingestion:
  Real-time push for market data (<500ms freshness), periodic pull for positions (<30s), 
  event-driven for risk breaches (<1s)

Stage 3 — Processing:
  Validate all prices (non-null, within circuit-breaker bounds), normalise timestamps 
  to UTC, compute position mark-to-market, classify trading session

Stage 4 — Storage & Indexing:
  Prices in ElastiCache (TTL: 60s), positions in DynamoDB (consistent reads), 
  historical trades in OpenSearch (embedded for semantic search), risk metrics computed on-demand

Stage 5 — Retrieval:
  Hybrid search for relevant trade history + direct lookups for current price, position, 
  risk limits for this specific symbol + portfolio

Stage 6 — Assembly:
  Zone 1: Trading rules and risk constraints
  Zone 2: [LIVE] market data → [<30s] current position → [<1min] risk metrics → 
          [SEMANTIC] relevant trade history
  Zone 4: "Should I execute 1000 shares of AAPL at market?"

Stage 7 — Delivery:
  Model receives complete structured context → generates: "Execute within limits. 
  Current price $185.42, position at 2.1% of AUM (limit 3%), recommend 800 shares 
  to stay within 2.5% threshold."
```

---

## What Breaks When Each Stage Is Weak

| Stage | Failure Mode | Downstream Impact |
|-------|-------------|-------------------|
| Stage 1 (Source) | Incomplete data | Retrieval can't find what doesn't exist |
| Stage 2 (Ingestion) | Latency or gaps | Stale or partial data reaches storage |
| Stage 3 (Processing) | Invalid or inconsistent records | Retrieval returns garbage |
| Stage 4 (Storage/Indexing) | Wrong chunking, missing embeddings | Semantic search misses relevant content |
| Stage 5 (Retrieval) | Poor ranking, wrong results | Assembly receives irrelevant context |
| Stage 6 (Assembly) | Bad structure, poor zone placement | Model reasons over noise, misses critical data |
| Stage 7 (Delivery) | No feedback loop | Memory never updates, system never improves |

---

## Key Terms

| Term | What It Means |
|------|--------------|
| **Ingestion** | Stage 2 — pulling or receiving data from sources into the pipeline |
| **Processing** | Stage 3 — cleaning, validating, normalising, and enriching raw data |
| **Storage & Indexing** | Stage 4 — storing data with embeddings and metadata for efficient retrieval |
| **Assembly** | Stage 6 — combining retrieved data, history, and task into a structured prompt |
| **Delivery** | Stage 7 — sending assembled context to the model and receiving its response |
| **Feedback Loop** | The path from model response back to memory and retrieval — where agentic systems live |

---

## What's Next

**Day 12 — Lessons from Financial Markets: When Context Failures Cost Real Money**

With the lifecycle mapped, Day 12 draws on 17+ years of pre/post-trade infrastructure experience to show what context engineering looks like under the most demanding conditions — and what those lessons mean for every AI system you build.

---

*#100DaysOfContextEngineering #ContextEngineering #DataPipeline #RAG #AIArchitecture #AWSCommunityBuilder*

[← Day 10](./Day-10-Your-First-Mental-Model.md) | [Day 12 →](./Day-12-The-MCP-Host.md)
