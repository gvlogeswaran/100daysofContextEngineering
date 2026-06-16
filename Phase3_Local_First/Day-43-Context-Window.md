# Day 43 — Context Window Management: The Sliding Window Pattern

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"A 128K-token context window is not a storage limit — it's a budget. Every token you spend without a plan is a token wasted on something the LLM won't use."*

---

## Explanation of Day Topic

Day 42 built the full AWS memory pipeline — DynamoDB for episodic memory, OpenSearch for semantic memory, one Lambda to orchestrate both. That pipeline produces rich context, but it introduces a new problem: the assembled context can easily exceed the LLM's token limit when a session runs long.

Day 43 addresses this with the sliding window pattern. The core idea: treat the context window as a fixed-size budget with named slots, not a pile-everything-in buffer. System prompt gets a slot. Pinned context (user profile, investment mandates, compliance rules) gets a slot. RAG results get a slot. Recent conversation turns get a slot. When the conversation slot fills up, the oldest non-pinned turns are evicted — archived to DynamoDB, not discarded. Critical messages are pinned and survive eviction indefinitely.

This day covers explicit token budgeting, the sliding window data structure with FIFO eviction, message pinning, batch compression of evicted turns (so archived summaries are cheaper to store and re-inject), and a full production integration with the Lambda memory pipeline from Day 42.

The result: an agent that can run for hours, across hundreds of turns, without ever losing the user's critical context — and without hitting a hard token ceiling.

---

### The Token Budget Framework

Define token slots explicitly before writing any buffer code.

```python
from dataclasses import dataclass

@dataclass
class TokenBudget:
    """
    Explicit token allocation for a production LLM context window.
    Model limit: 128K (Claude 3.5 Sonnet via Bedrock).
    """
    model_limit: int     = 128_000
    system_prompt: int   = 10_000  # Base instructions, persona, rules
    pinned_context: int  =  5_000  # User profile, mandates, constraints (never evicted)
    rag_context: int     = 20_000  # Retrieved documents for current query
    recent_conv: int     = 30_000  # Sliding window — recent conversation turns
    reserve: int         = 10_000  # Headroom for LLM response generation
    # 10K + 5K + 20K + 30K + 10K = 75K allocated, 53K margin

    @property
    def total_allocated(self) -> int:
        return (self.system_prompt + self.pinned_context
                + self.rag_context + self.recent_conv + self.reserve)

budget = TokenBudget()
# total_allocated = 75,000 — leaves 53K margin under the 128K limit
```

---

### Token Counting

```python
import anthropic

client = anthropic.Anthropic()

def count_tokens(messages: list[dict], system: str = "") -> int:
    """Exact count using Anthropic token counter API."""
    response = client.messages.count_tokens(
        model="claude-3-5-sonnet-20241022",
        system=system,
        messages=messages
    )
    return response.input_tokens

def count_tokens_approx(text: str) -> int:
    """Fast approximation: ~4 chars per token for English text."""
    return len(text) // 4
```

---

### The Sliding Window Buffer

```python
from collections import deque
from dataclasses import dataclass, field
from datetime import datetime
import boto3
import json

@dataclass
class Message:
    role: str
    content: str
    tokens: int = 0
    pinned: bool = False
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())
    message_id: str = field(default_factory=lambda: f"msg_{datetime.now().timestamp():.0f}")

    def to_dict(self) -> dict:
        return {"role": self.role, "content": self.content}


class SlidingWindowBuffer:
    """
    Sliding window context manager with pinning and DynamoDB archival.
    
    - pinned  → never evicted (user mandates, compliance constraints, profile)
    - window  → recent turns (FIFO, evicts oldest when budget exceeded)
    - archive → evicted turns written to DynamoDB episodic store
    """

    def __init__(self, budget: TokenBudget, window_size: int = 20,
                 user_id: str = None, table_name: str = "agent-memory"):
        self.budget = budget
        self.window_size = window_size
        self.user_id = user_id
        self.pinned: list[Message] = []
        self.window: deque[Message] = deque(maxlen=window_size)
        self.dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
        self.table = self.dynamodb.Table(table_name)

    @property
    def window_tokens(self) -> int:
        return sum(m.tokens for m in self.window)

    @property
    def pinned_tokens(self) -> int:
        return sum(m.tokens for m in self.pinned)

    def add_message(self, role: str, content: str, pinned: bool = False) -> Message:
        tokens = count_tokens_approx(content) + 4
        msg = Message(role=role, content=content, tokens=tokens, pinned=pinned)

        if pinned:
            self.pinned.append(msg)
            return msg

        # Evict oldest until the new message fits
        while (self.window_tokens + tokens > self.budget.recent_conv
               and len(self.window) > 0):
            self._evict_oldest()

        self.window.append(msg)
        return msg

    def _evict_oldest(self):
        if not self.window:
            return
        evicted = self.window.popleft()
        self._archive(evicted)

    def _archive(self, msg: Message):
        if not self.user_id:
            return
        try:
            self.table.put_item(Item={
                'PK': f'USER#{self.user_id}',
                'SK': f'EPISODE#{msg.timestamp}',
                'role': msg.role,
                'content': msg.content,
                'tokens': msg.tokens,
                'ttl': int(datetime.now().timestamp()) + (90 * 86400)
            })
        except Exception as e:
            print(f"Archive error: {e}")  # Don't crash on archive failure

    def get_context(self, rag_context: str = "") -> dict:
        """Assemble full LLM context from pinned + window + RAG."""
        pinned_block = ""
        if self.pinned:
            pinned_block = "\n\nPINNED CONTEXT:\n" + "\n".join(
                f"[{m.role.upper()}]: {m.content}" for m in self.pinned
            )
        rag_block = f"\n\nRETRIEVED CONTEXT:\n{rag_context}" if rag_context else ""

        return {
            "system_additions": pinned_block + rag_block,
            "messages": [m.to_dict() for m in self.window],
            "stats": {
                "pinned_tokens": self.pinned_tokens,
                "window_tokens": self.window_tokens,
                "window_messages": len(self.window)
            }
        }
```

---

### Context Compression

When the window fills up, summarise old turns before archiving instead of discarding them raw.

```python
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
MODEL_ID = 'anthropic.claude-3-5-sonnet-20241022-v2:0'


def compress_messages(messages: list[Message], max_summary_tokens: int = 400) -> str:
    """Summarise a batch of messages into a compact block."""
    if not messages:
        return ""

    conversation_text = "\n".join(
        f"{m.role.upper()}: {m.content}" for m in messages
    )
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": max_summary_tokens,
        "messages": [{
            "role": "user",
            "content": (
                f"Summarise this conversation in under {max_summary_tokens // 4} words. "
                f"Keep: decisions, data points, action items. Remove: pleasantries.\n\n"
                f"{conversation_text}"
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    return json.loads(response['body'].read())['content'][0]['text']


class CompressingWindowBuffer(SlidingWindowBuffer):
    """Evicts by compressing batches of old turns, not discarding them."""

    def __init__(self, *args, compress_batch: int = 5, **kwargs):
        super().__init__(*args, **kwargs)
        self.compress_batch = compress_batch

    def _evict_oldest(self):
        if len(self.window) < self.compress_batch:
            super()._evict_oldest()
            return

        # Collect oldest batch
        batch = [self.window.popleft() for _ in range(self.compress_batch)]
        original_tokens = sum(m.tokens for m in batch)

        # Compress and archive
        summary = compress_messages(batch)
        summary_tokens = count_tokens_approx(summary)
        compressed = Message(
            role="system",
            content=f"[COMPRESSED — {len(batch)} turns]: {summary}",
            tokens=summary_tokens
        )
        self._archive(compressed)
        print(f"Compressed {len(batch)} turns: {original_tokens} → {summary_tokens} tokens "
              f"(saved {original_tokens - summary_tokens})")
```

---

### Full Agent Integration

```python
import json

SYSTEM_PROMPT = """You are a personalised financial assistant.
Use pinned context and retrieved documents to ground your responses.
Never hallucinate financial data."""


class FinancialAgent:
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.buffer = CompressingWindowBuffer(
            budget=TokenBudget(),
            window_size=20,
            compress_batch=5,
            user_id=user_id
        )

    def set_mandate(self, mandate: str):
        """Pin the user's investment mandate — survives all session turns."""
        self.buffer.add_message(role="user",
                                content=f"[INVESTMENT MANDATE]: {mandate}",
                                pinned=True)

    def query(self, user_input: str, rag_context: str = "") -> str:
        self.buffer.add_message(role="user", content=user_input)

        ctx = self.buffer.get_context(rag_context=rag_context)
        full_system = SYSTEM_PROMPT + ctx["system_additions"]

        response = bedrock.invoke_model(
            modelId=MODEL_ID,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1024,
                "system": full_system,
                "messages": ctx["messages"]
            })
        )
        answer = json.loads(response['body'].read())['content'][0]['text']

        self.buffer.add_message(role="assistant", content=answer)
        print(f"[Tokens] window={ctx['stats']['window_tokens']:,} | "
              f"pinned={ctx['stats']['pinned_tokens']:,}")
        return answer


# Usage
agent = FinancialAgent(user_id="trader_007")

# This mandate is pinned — survives the entire 3-hour session
agent.set_mandate(
    "Conservative risk profile. No leverage. ESG sectors only. "
    "Max 5% per position. US equities only."
)

# 30 turns — oldest turns get compressed and archived automatically
for i in range(30):
    rag = "[AAPL: $182.50 | P/E: 29.4 | ESG Score: A]" if i % 3 == 0 else ""
    answer = agent.query(f"Turn {i+1}: Should I add to AAPL now?", rag_context=rag)
    print(f"Turn {i+1}: {answer[:60]}...")
```

---

### Key Terms for Day 43

| Term | Meaning |
|------|---------|
| **Token Budget** | An explicit allocation of token slots — system, pinned, RAG, conversation, reserve — that prevents any one category from crowding out the others. |
| **Sliding Window** | A FIFO conversation buffer where the oldest non-pinned turn is evicted when the budget is exceeded. |
| **Pinned Message** | A message immune to eviction — always present in the LLM context regardless of session length. Used for user mandates, compliance rules, and profile summaries. |
| **Context Compression** | Summarising a batch of old turns before archiving them — preserves information at a fraction of the original token cost. |
| **Episodic Archival** | Writing evicted turns to DynamoDB so they can be retrieved later if the agent needs them — the long-term backing store for the sliding window. |
| **Reserve Tokens** | Tokens withheld from the context budget to ensure the model has room to generate a complete response without being cut off. |

---

## What's Next

**Day 44 — Agentic RAG: When the Agent Decides What to Retrieve**

The sliding window manages conversation history. But the RAG context slot (20K tokens) is still filled by a single passive retrieval — one vector search per user query. Day 44 introduces agentic retrieval: the model itself plans the retrieval, decomposes multi-part queries into sub-queries, runs parallel searches, and synthesises the results. Multi-hop retrieval. Self-querying. Bedrock Agents with Knowledge Base associations. Passive RAG becomes active reasoning.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 42](./Day-42-Building-memory.md) | [Day 45 →](./Day-44-Agentic-RAG.md)
