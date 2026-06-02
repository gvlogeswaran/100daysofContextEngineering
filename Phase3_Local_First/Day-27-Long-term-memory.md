# Day 27 — Long-Term Memory: Persistent Knowledge Across Sessions

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Standard RAG gives your AI an open book. Persistent memory gives your AI a diary. One provides facts; the other provides continuity."*

---

## Explanation of Day Topic

Day 26 established why static context fails and why RAG is the solution for retrieving external knowledge. But RAG alone solves only half the memory problem. RAG answers "what does the AI know about the world?" Persistent memory answers "what does the AI know about *this user*, *this conversation*, and *this context* — across all past sessions?"

A stateless LLM is a brilliant analyst with amnesia. It forgets your name, your portfolio, your risk tolerance, and every decision you've made together the moment the session ends. Long-term memory fixes that. This day covers the architecture, the patterns, and the production implementation using AWS-native services.

---

## The Amnesia Problem in Production

Standard context windows are ephemeral. The moment a session ends, everything the model learned is gone. This is not a limitation you can prompt-engineer around — it is an architectural gap.

Consider a financial advisory AI:

```
Session 1: User says "I'm conservative. Never recommend anything above 15% position size."
Session 2: User asks "Should I buy more NVDA?"
```

Without persistent memory, the AI in Session 2 has no knowledge of the constraint set in Session 1. It might recommend a 40% NVDA allocation — directly violating the user's stated preference. In a regulated environment, this is not just a bad user experience. It is a compliance failure.

**The three things persistent memory must track:**

1. **User preferences and constraints** — explicitly stated rules, risk tolerance, communication style
2. **Factual history** — past decisions, trades, positions, outcomes
3. **Inferred patterns** — what has worked, what has been rejected, emerging context that shapes future responses

---

## Memory Architecture: Four Types

Not all memory serves the same purpose. A production memory system uses all four types in combination.

| Memory Type | What It Stores | Lifespan | AWS Service |
|---|---|---|---|
| **Working Memory** | Current session context — active conversation | Session only | Lambda in-memory / ElastiCache |
| **Episodic Memory** | Specific events — "On March 4, user approved trade X" | Weeks to months | DynamoDB with TTL |
| **Semantic Memory** | General facts — user preferences, domain knowledge | Long-term | DynamoDB + OpenSearch |
| **Procedural Memory** | How to do things — learned workflows, tool sequences | Persistent | DynamoDB |

---

## Architecture: AWS-Native Persistent Memory Stack

```
User Query
    ↓
API Gateway → Lambda (Orchestrator)
    ↓
┌─────────────────────────────────────────────────────┐
│              Memory Retrieval Layer                  │
│                                                      │
│  DynamoDB          OpenSearch           ElastiCache  │
│  (Episodic +       (Semantic search     (Working     │
│  Procedural)        across memory)       memory)     │
└─────────────────────────────────────────────────────┘
    ↓
Context Assembly (merge retrieved memory + current query)
    ↓
Bedrock (LLM) → Generate Response
    ↓
Memory Extraction → Write New Memory Back to DynamoDB
    ↓
Response to User
```

---

## Implementation: Persistent Memory with DynamoDB

### Table Design

```python
# DynamoDB single-table design for memory
# Partition key: user_id
# Sort key: memory_type#timestamp

MEMORY_TABLE_SCHEMA = {
    "TableName": "context-engineering-memory",
    "KeySchema": [
        {"AttributeName": "pk", "KeyType": "HASH"},   # user_id
        {"AttributeName": "sk", "KeyType": "RANGE"}   # memory_type#timestamp
    ],
    "AttributeDefinitions": [
        {"AttributeName": "pk", "AttributeType": "S"},
        {"AttributeName": "sk", "AttributeType": "S"},
        {"AttributeName": "memory_type", "AttributeType": "S"},
    ],
    "GlobalSecondaryIndexes": [{
        "IndexName": "memory-type-index",
        "KeySchema": [
            {"AttributeName": "memory_type", "KeyType": "HASH"},
            {"AttributeName": "sk", "KeyType": "RANGE"}
        ],
        "Projection": {"ProjectionType": "ALL"}
    }],
    "BillingMode": "PAY_PER_REQUEST",
    "TimeToLiveSpecification": {
        "Enabled": True,
        "AttributeName": "ttl"
    }
}
```

### Writing Memory

```python
import boto3
import json
import time
from datetime import datetime, timedelta
from typing import Literal

dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table("context-engineering-memory")

MemoryType = Literal["preference", "episodic", "semantic", "procedural"]

def write_memory(
    user_id: str,
    memory_type: MemoryType,
    content: dict,
    ttl_days: int | None = None
) -> str:
    """
    Persist a memory record for a user.
    Returns the memory_id for reference.
    """
    timestamp = datetime.utcnow().isoformat()
    memory_id = f"{user_id}#{memory_type}#{timestamp}"

    item = {
        "pk": user_id,
        "sk": f"{memory_type}#{timestamp}",
        "memory_id": memory_id,
        "memory_type": memory_type,
        "content": content,
        "created_at": timestamp,
        "source": content.get("source", "system"),
    }

    # Apply TTL for episodic memory (auto-expire old events)
    if ttl_days:
        item["ttl"] = int((datetime.utcnow() + timedelta(days=ttl_days)).timestamp())

    table.put_item(Item=item)
    return memory_id


# Example: Store a user preference
write_memory(
    user_id="trader-loki-001",
    memory_type="preference",
    content={
        "rule": "max_position_size",
        "value": 0.15,
        "stated_at": "2026-06-02",
        "source": "user_explicit"
    }
)

# Example: Store an episodic memory (auto-expire after 90 days)
write_memory(
    user_id="trader-loki-001",
    memory_type="episodic",
    content={
        "event": "trade_approved",
        "ticker": "NVDA",
        "size": 0.08,
        "decision_rationale": "Approved after momentum signal confirmed by analyst consensus",
        "outcome": "pending"
    },
    ttl_days=90
)
```

### Reading Memory

```python
from boto3.dynamodb.conditions import Key, Attr

def retrieve_memory(
    user_id: str,
    memory_types: list[MemoryType] | None = None,
    max_items_per_type: int = 5
) -> dict:
    """
    Retrieve relevant memory records for a user.
    Returns structured memory grouped by type.
    """
    memory_types = memory_types or ["preference", "episodic", "semantic", "procedural"]
    result = {}

    for mem_type in memory_types:
        response = table.query(
            KeyConditionExpression=Key("pk").eq(user_id) & Key("sk").begins_with(mem_type),
            ScanIndexForward=False,  # most recent first
            Limit=max_items_per_type
        )
        result[mem_type] = [item["content"] for item in response["Items"]]

    return result


def assemble_memory_context(user_id: str) -> str:
    """
    Build a structured memory block to inject into the LLM context.
    Preferences first (hardest constraints), then recency-sorted episodes.
    """
    memory = retrieve_memory(user_id)

    blocks = []

    if memory.get("preference"):
        prefs = "\n".join(
            f"  - {p['rule']}: {p['value']} (stated {p.get('stated_at', 'unknown')})"
            for p in memory["preference"]
        )
        blocks.append(f"USER CONSTRAINTS (always honour):\n{prefs}")

    if memory.get("episodic"):
        episodes = "\n".join(
            f"  - [{e.get('event', 'event')}] {e.get('ticker', '')} "
            f"{e.get('decision_rationale', '')[:100]}"
            for e in memory["episodic"][:3]
        )
        blocks.append(f"RECENT HISTORY (last 3 events):\n{episodes}")

    if memory.get("semantic"):
        facts = "\n".join(
            f"  - {s}" if isinstance(s, str) else f"  - {json.dumps(s)}"
            for s in memory["semantic"][:3]
        )
        blocks.append(f"USER KNOWLEDGE BASE:\n{facts}")

    return "\n\n".join(blocks) if blocks else "No prior memory for this user."
```

---

## Memory Extraction: Identifying What to Persist

The hardest part of persistent memory is not storage — it is deciding what to remember. Not every message deserves to be stored. Storing everything leads to a cluttered, contradictory memory that degrades performance.

```python
def extract_memory_candidates(
    user_id: str,
    conversation_turn: dict
) -> list[dict]:
    """
    Use the LLM to identify memory-worthy content from a conversation turn.
    Returns list of memory candidates to persist.
    """
    extraction_prompt = f"""
Analyse this conversation turn and identify information worth storing as persistent memory.

User message: {conversation_turn["user_message"]}
AI response: {conversation_turn["ai_response"]}

Extract ONLY items matching these categories:
1. PREFERENCE: Explicit rules or constraints the user stated ("never", "always", "I prefer")
2. EPISODIC: Specific decisions or events that may be referenced later
3. SEMANTIC: Facts about the user's domain, portfolio, or context
4. PROCEDURAL: Workflow steps the user wants applied consistently

Return valid JSON array. If nothing is worth storing, return [].
Each item: {{"type": "preference|episodic|semantic|procedural", "content": {{...}}, "confidence": 0.0-1.0}}
"""

    raw = llm.generate(extraction_prompt)

    try:
        candidates = json.loads(raw)
        # Only persist high-confidence extractions
        return [c for c in candidates if c.get("confidence", 0) >= 0.80]
    except (json.JSONDecodeError, KeyError):
        return []


def post_session_memory_update(user_id: str, conversation: list[dict]) -> int:
    """
    After each session, extract and persist new memories.
    Returns count of memories written.
    """
    written = 0

    for turn in conversation:
        candidates = extract_memory_candidates(user_id, turn)
        for candidate in candidates:
            write_memory(
                user_id=user_id,
                memory_type=candidate["type"],
                content=candidate["content"],
                ttl_days=90 if candidate["type"] == "episodic" else None
            )
            written += 1

    return written
```

---

## Memory-Augmented Query: Full Flow

```python
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

def memory_augmented_query(
    user_id: str,
    user_question: str,
    rag_context: str
) -> dict:
    """
    Full query flow: memory retrieval + RAG context + generation.
    """
    # 1. Retrieve persistent memory
    memory_context = assemble_memory_context(user_id)

    # 2. Build context-rich prompt
    system_prompt = f"""
You are a financial analysis assistant with persistent memory of this user's context.

MEMORY (from prior sessions — always honour constraints here):
{memory_context}

RETRIEVED KNOWLEDGE (from live document index):
{rag_context}

INSTRUCTIONS:
- Honour all USER CONSTRAINTS unconditionally.
- Reference RECENT HISTORY when directly relevant to the question.
- If the question contradicts a USER CONSTRAINT, flag it explicitly.
- If you cannot answer without violating a constraint, say so clearly.
"""

    # 3. Generate
    response = bedrock.invoke_model(
        modelId="anthropic.claude-3-5-sonnet-20241022-v2:0",
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "system": system_prompt,
            "messages": [{"role": "user", "content": user_question}]
        })
    )

    answer = json.loads(response["body"].read())["content"][0]["text"]

    # 4. Extract and persist new memory from this interaction
    new_memories = post_session_memory_update(user_id, [{
        "user_message": user_question,
        "ai_response": answer
    }])

    return {
        "answer": answer,
        "memory_items_retrieved": len(memory_context.split("\n")),
        "new_memories_written": new_memories
    }
```

---

## Production Considerations

**Memory Conflict Resolution.** When new information contradicts stored memory, the system must not silently overwrite. The newer statement should be surfaced to the user: *"This contradicts your previous constraint of max 15% position size. Should I update your preference?"*

**Memory Privacy and Compliance.** In regulated environments, user memory is personal data. Apply:
- Encryption at rest (DynamoDB with AWS KMS)
- Access logging (CloudTrail for all DynamoDB operations)
- Right-to-forget support (delete by `user_id` partition key)
- Retention policies (TTL for episodic memory, explicit review for preferences)

**Memory Size Budget.** Context windows are finite. When assembling memory, apply a token budget:
- Preferences: always included (highest priority, usually small)
- Episodic: top 3–5 most recent, scored by recency and relevance
- Semantic: top 3 by relevance to current query

**Memory Staleness.** Stored facts go stale. A user's risk tolerance from 18 months ago may no longer reflect their current situation. Apply confidence decay: reduce the weight of older memories and prompt users to confirm them when they resurface.

---

## Key Terms

| Term | What It Means |
|------|---------------|
| **Working Memory** | In-session context — the active conversation window; lost when session ends |
| **Episodic Memory** | Records of specific events — decisions, trades, outcomes — with timestamps |
| **Semantic Memory** | General facts about a user or domain — preferences, portfolio composition, risk profile |
| **Procedural Memory** | Learned workflows or tool sequences the system applies consistently |
| **Memory Extraction** | Using an LLM to identify what content from a conversation is worth persisting |
| **Memory Augmentation** | Injecting retrieved persistent memory into the prompt before generation |
| **Confidence Decay** | Reducing the weight of older memory items to reflect the possibility of changed context |
| **TTL (Time-to-Live)** | DynamoDB attribute that auto-expires old records — critical for managing episodic memory at scale |

---

## What's Next

**Day 28 — Chunking Strategies: The Science of Splitting Documents**

Memory handles what we know about *users*. RAG handles what we know about *the world*. Day 28 opens the technical RAG deep dive with chunking — the most underrated decision in any RAG pipeline. Fixed-size, sentence-aware, semantic, and recursive chunking strategies, their trade-offs, and how the wrong chunking strategy silently destroys retrieval quality.

---

*#100DaysOfContextEngineering #ContextEngineering #LongTermMemory #RAG #DynamoDB #AWSBedrock #PersistentContext #AWSCommunityBuilder*

[← Day 26](./Day-26-Why-Context-fails.md) | [Day 28 →](./Day-28-Chunking-Strategies.md)
