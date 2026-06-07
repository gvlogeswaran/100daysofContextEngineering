# Day 37 — Short-Term Memory: Managing the Conversation Buffer

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Managing conversation context is like managing a bank account. Full history = saving everything. Sliding window = keeping only recent receipts. Summary buffer = summarizing old statements, keeping recent ones."*

---


### Conversation Buffer Strategies

#### 1. Full History Buffer

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory()

# Just keep appending
for message in conversation:
    memory.save_context({"input": message.user}, {"output": message.bot})

# Eventually: tokens exceed limit
total_tokens = count_tokens(memory.buffer)  # Grows unbounded
```

#### 2. Sliding Window Buffer

```python
from langchain.memory import ConversationBufferWindowMemory

memory = ConversationBufferWindowMemory(k=10)  # Keep last 10 messages

# Old messages automatically discarded
for message in conversation:
    memory.save_context({"input": message.user}, {"output": message.bot})
    # After 11 messages: message 1 is deleted

# Token usage: stable, predictable
```

#### 3. Summary Buffer

```python
from langchain.memory import ConversationSummaryBufferMemory
from langchain.llms import Bedrock

bedrock_llm = Bedrock(model_id="anthropic.claude-3-5-sonnet-20241022")

memory = ConversationSummaryBufferMemory(
    llm=bedrock_llm,
    max_token_limit=2000,  # Summarize when exceed 2K tokens
    memory_key="history"
)

for message in conversation:
    memory.save_context({"input": message.user}, {"output": message.bot})
    # When exceeding 2K tokens: old messages auto-summarized
    # New messages kept verbatim

# Token usage: stable, context preserved via summaries
```

#### 4. Comparison Table

| Strategy | Max Tokens | Context Loss | Quality |
|----------|------------|--------------|---------|
| **Full History** | Unbounded — grows indefinitely | None until limit hit, then hard failure | Highest — every word preserved |
| **Sliding Window** | Fixed and predictable | Abrupt — old messages vanish entirely | Good for recent turns; blind to early context |
| **Summary Buffer** | Stable — summaries replace old messages | Graceful — meaning preserved, wording lost | High — semantic continuity maintained |
| **Token-Compressed** | Aggressively bounded | Minimal — LLM rewrites to preserve key facts | Very high — best for long sessions with cost constraints |

#### 5. Token-Compressed Buffer (Advanced)

```python
import boto3
import json

class TokenCompressedBuffer:
    """Compress old messages with an LLM call when buffer gets full."""

    def __init__(self, max_tokens: int = 4000, compression_threshold: float = 0.8):
        self.max_tokens = max_tokens
        self.threshold = compression_threshold
        self.messages = []
        self.bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

    def _count_tokens(self, text: str) -> int:
        # Approximate: 1 token ≈ 4 characters
        return len(text) // 4

    def _total_tokens(self) -> int:
        return sum(self._count_tokens(m['content']) for m in self.messages)

    def _compress_oldest(self):
        """Ask Claude to compress the oldest half of the conversation."""
        half = len(self.messages) // 2
        old_messages = self.messages[:half]
        self.messages = self.messages[half:]

        conversation_text = "\n".join(
            f"{m['role'].upper()}: {m['content']}" for m in old_messages
        )

        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "messages": [{
                "role": "user",
                "content": (
                    f"Compress this conversation to 3-5 bullet points, "
                    f"preserving key facts and decisions:\n\n{conversation_text}"
                )
            }]
        })

        response = self.bedrock.invoke_model(
            modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
            body=body
        )
        summary = json.loads(response['body'].read())['content'][0]['text']

        # Prepend the compressed summary as a system note
        self.messages.insert(0, {
            'role': 'assistant',
            'content': f"[Earlier conversation summary]: {summary}"
        })

    def add_message(self, role: str, content: str):
        self.messages.append({'role': role, 'content': content})

        # Compress when approaching the threshold
        if self._total_tokens() > self.max_tokens * self.threshold:
            self._compress_oldest()

    def get_messages(self) -> list:
        return self.messages


# Usage in a financial advisory session
buffer = TokenCompressedBuffer(max_tokens=4000)
buffer.add_message('user', 'I want to discuss my portfolio rebalancing.')
buffer.add_message('assistant', 'Of course — what is your current allocation?')
buffer.add_message('user', 'Sixty percent equities, thirty bonds, ten cash.')
# ... many more turns ...
# When 80% full, oldest messages are compressed automatically
context = buffer.get_messages()
```

---

### Best Practices for Short-Term Memory

1. **Never let the buffer grow unbounded.** Full History works for demos; in production a 200-turn financial conversation will blow a 128K context limit and raise latency dramatically. Set an explicit `max_tokens` ceiling.

2. **Match strategy to task type.** Transactional queries (single trade execution) suit a Sliding Window of 5–10 turns. Advisory conversations that span hours need a Summary Buffer or Token-Compressed approach to preserve the user's stated goals.

3. **Compress before evicting, not after.** Trigger summarisation when you hit 75–80% of capacity, not at 100%. This gives the compressor enough tokens to write a quality summary without being squeezed itself.

4. **Always pin system instructions and user constraints.** Risk tolerance, jurisdiction, and account type should survive any compression event. Mark them as pinned messages that are never evicted (see Day 43).

5. **Track compression cost.** Each compression event costs one LLM call (~200–500 input tokens + ~100 output tokens). For a high-volume system, budget this explicitly alongside your primary inference cost.

6. **Log what was evicted.** Move compressed/evicted messages to episodic memory (DynamoDB) so nothing is permanently lost — only demoted from hot to warm storage.

---

#### Key Terms for Day 37

| Term | What It Means |
|------|---------------|
| **Conversation Buffer** | The in-memory structure that holds recent message turns for the current session. |
| **Sliding Window** | A buffer strategy that keeps only the N most recent messages; older ones are discarded entirely. |
| **Summary Buffer** | A strategy that compresses messages exceeding a token limit into a running summary, preserving meaning without storing verbatim text. |
| **Token-Compressed Buffer** | An advanced strategy that calls an LLM to rewrite and shrink older portions of the conversation on demand. |
| **Compression Threshold** | The token occupancy percentage (e.g. 80%) at which a buffer triggers compression rather than waiting for hard overflow. |
| **Pinned Message** | A message (e.g. system prompt, user risk profile) that is excluded from compression and eviction — always present in context. |

---

## What's Next

**Day 38 — Working Memory: The Agent Scratchpad and ReAct Pattern**

Short-term memory handles *conversation history*, but agents also need a private workspace for multi-step reasoning. Day 38 introduces the scratchpad — the agent's internal notepad — and shows how the ReAct (Reasoning + Acting) pattern uses it to chain Thought → Action → Observation cycles for complex tasks like multi-leg financial analysis.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 36](./Day-36-Memory-architecture.md) | [Day 38 →](./Day-38-Working-Memory.md)