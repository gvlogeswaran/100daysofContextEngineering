# Day 39 — Episodic Memory: Teaching Agents to Remember Past Interactions
**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory**
**Topic:** User interaction history. Retrieval from past episodes. Relevance scoring with embeddings. Recency decay. DynamoDB as episodic store.

---

## 🎨 SLIDE 1 — The Hook (Canva)

**Headline:** Agents Without Memory Repeat Mistakes

**Sub-headline:** Store past interactions in DynamoDB. Retrieve relevant episodes on new queries. Your agent learns user preferences, past decisions, and patterns — without being told every session.

**Visual Cue:**
- **Background:** Dark mode (#0a0a0a)
- **Left:** Stateless agent — user states preferences again on every session
- **Right:** Episodic agent — past interactions retrieved, preferences already known
- **Arrow:** DynamoDB → Lambda → Context Assembly → Bedrock
- **Message:** "Remember the conversation. Not just this one."

---

## 🎨 SLIDE 2 — The AHA Moment (Canva)

**Analogy:**
> Episodic memory is your agent's diary. Every session, it records what the user asked, what decisions were made, and what actions were taken. Next session, it opens the diary to the relevant pages. The agent doesn't ask "what's your risk tolerance?" — it already knows. It checked the diary.

**The Key Insight:**
> Not all past interactions are equally relevant. Blend recency (how recent?) and semantic similarity (how related to the current query?) to retrieve only the most pertinent episodes. A 45-day-old episode about AAPL is less useful than a 3-day-old episode on the same topic — but more useful than a recent episode about unrelated sectors.

**Visual Cue:**
- **Timeline:** Episodes stored in DynamoDB with timestamps
- **Scoring diagram:** Current query vs past episodes — cosine similarity × recency decay
- **Top-3 retrieved:** Injected into context before Bedrock call
- **Message:** "The agent remembers what matters — not everything"

---

## ✍️ LINKEDIN POST

**Day 39 of 100 — Episodic Memory: Agents That Remember**

Your agent forgets every conversation. User repeats themselves every session. Frustrating. Unprofessional. 📝

**Episodic memory:** Store past interactions externally. Retrieve on demand.

"User bought AAPL at $150 in March. Stated max position: 15%. Approved conservative rebalancing last week."

Agent retrieves this on the next query. No need to re-explain. No need to re-state preferences.

**The retrieval trick:** Don't retrieve by date alone. Blend:
- Recency score (newer = more relevant)
- Semantic similarity (closer in meaning = more relevant)
- Combined: `0.7 × similarity + 0.3 × recency`

Cost: ~$1/1M interactions on DynamoDB. The cheapest personalisation layer you'll build.

#100DaysOfContextEngineering #Agents #Memory #AWS #Bedrock #ContextEngineering

---

## 📖 GITHUB — Day 39 Deep Dive

### Episodic Memory: Learning from Past Interactions

#### 1. Basic Episode Store

```python
import boto3
from datetime import datetime

class EpisodicMemory:
    def __init__(self):
        self.table = boto3.resource('dynamodb').Table('agent-episodes')
    
    def store_episode(self, user_id, interaction):
        self.table.put_item(Item={
            'user_id': user_id,
            'timestamp': datetime.now().isoformat(),
            'query': interaction['query'],
            'response': interaction['response'],
            'action': interaction.get('action'),
            'ttl': int(datetime.now().timestamp()) + (90 * 86400)
        })
    
    def retrieve_episodes(self, user_id, query, limit=5):
        response = self.table.query(
            KeyConditionExpression='user_id = :uid',
            ExpressionAttributeValues={':uid': user_id},
            ScanIndexForward=False,
            Limit=limit
        )
        
        episodes = response['Items']
        scored = [(ep, self.relevance_score(query, ep['query'])) for ep in episodes]
        return sorted(scored, key=lambda x: x[1], reverse=True)
    
    def relevance_score(self, q1, q2):
        # Simple word overlap; in production use embeddings (see below)
        words1 = set(q1.lower().split())
        words2 = set(q2.lower().split())
        intersection = words1 & words2
        return len(intersection) / (len(words1) + len(words2))
```

---

### Production Patterns

#### Semantic Relevance Scoring with Bedrock Embeddings

The word-overlap stub above is a starting point. In production, replace it with real embedding similarity so that "Tell me about my AAPL position" correctly matches an old episode titled "I bought Apple stock last quarter" — even though the words don't overlap.

```python
import boto3
import json
import numpy as np
from datetime import datetime

class EpisodicMemoryPro:
    def __init__(self):
        self.table = boto3.resource('dynamodb').Table('agent-episodes')
        self.bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

    def _embed(self, text: str) -> list[float]:
        body = json.dumps({'inputText': text})
        response = self.bedrock.invoke_model(
            modelId='amazon.titan-embed-text-v2:0',
            body=body,
            contentType='application/json',
            accept='application/json'
        )
        return json.loads(response['body'].read())['embedding']

    def _cosine_similarity(self, a: list[float], b: list[float]) -> float:
        va, vb = np.array(a), np.array(b)
        norm = np.linalg.norm(va) * np.linalg.norm(vb)
        return float(np.dot(va, vb) / norm) if norm > 0 else 0.0

    def store_episode(self, user_id: str, interaction: dict):
        embedding = self._embed(interaction['query'])
        self.table.put_item(Item={
            'PK': f'USER#{user_id}',
            'SK': f'EPISODE#{datetime.now().isoformat()}',
            'query': interaction['query'],
            'response': interaction['response'],
            'action': interaction.get('action', ''),
            'embedding': embedding,
            'ttl': int(datetime.now().timestamp()) + (90 * 86400)
        })

    def retrieve_episodes(
        self,
        user_id: str,
        current_query: str,
        limit: int = 20,
        top_k: int = 5,
        recency_weight: float = 0.3,
        relevance_weight: float = 0.7
    ) -> list[dict]:
        """
        Fetch the last `limit` episodes, score each with a weighted combination
        of recency and semantic similarity, return the top_k.
        """
        from boto3.dynamodb.conditions import Key

        response = self.table.query(
            KeyConditionExpression=Key('PK').eq(f'USER#{user_id}'),
            ScanIndexForward=False,
            Limit=limit
        )
        episodes = response.get('Items', [])
        if not episodes:
            return []

        query_embedding = self._embed(current_query)
        now_ts = datetime.now().timestamp()

        scored = []
        for ep in episodes:
            # Recency: 1.0 for brand new, approaches 0 for old episodes
            ep_ts = datetime.fromisoformat(ep['SK'].replace('EPISODE#', '')).timestamp()
            age_days = max((now_ts - ep_ts) / 86400, 0)
            recency = 1.0 / (1.0 + age_days / 30)   # Half-life ≈ 30 days

            # Semantic similarity
            ep_embedding = [float(v) for v in ep['embedding']]
            similarity = self._cosine_similarity(query_embedding, ep_embedding)

            combined = recency_weight * recency + relevance_weight * similarity
            scored.append((ep, combined))

        scored.sort(key=lambda x: x[1], reverse=True)
        return [ep for ep, _ in scored[:top_k]]
```

#### Episode Compression for Long-Running Users

```python
class EpisodicMemoryWithCompression(EpisodicMemoryPro):
    """
    When an episode is older than compress_after_days, summarise it
    with Claude and replace the stored text to reduce token cost.
    """

    def __init__(self, compress_after_days: int = 30):
        super().__init__()
        self.compress_after_days = compress_after_days

    def _summarise(self, text: str) -> str:
        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 200,
            "messages": [{
                "role": "user",
                "content": f"Summarise this in one sentence, preserving key facts:\n{text}"
            }]
        })
        resp = self.bedrock.invoke_model(
            modelId='anthropic.claude-3-5-sonnet-20241022-v2:0', body=body
        )
        return json.loads(resp['body'].read())['content'][0]['text']

    def compress_old_episodes(self, user_id: str):
        from boto3.dynamodb.conditions import Key
        cutoff = datetime.now().timestamp() - self.compress_after_days * 86400

        response = self.table.query(
            KeyConditionExpression=Key('PK').eq(f'USER#{user_id}'),
        )
        for ep in response.get('Items', []):
            ep_ts = datetime.fromisoformat(ep['SK'].replace('EPISODE#', '')).timestamp()
            if ep_ts < cutoff and not ep.get('compressed'):
                summary = self._summarise(ep['query'] + ' → ' + ep['response'])
                self.table.update_item(
                    Key={'PK': ep['PK'], 'SK': ep['SK']},
                    UpdateExpression='SET query = :s, compressed = :t',
                    ExpressionAttributeValues={':s': summary, ':t': True}
                )
```

---

#### Key Terms for Day 39

| Term | What It Means |
|------|---------------|
| **Episodic Memory** | A persistent record of specific past interactions between the agent and a user, stored externally (e.g. DynamoDB) and retrieved on demand. |
| **Episode** | A single stored interaction: the user's query, the agent's response, any action taken, and metadata such as timestamp and TTL. |
| **Recency Bias** | The tendency to weight recent episodes more heavily than old ones; implemented as a decay function over the episode's age. |
| **Relevance Score** | A similarity measure between the current query and a stored episode's query — in production, computed as cosine similarity between their embeddings. |
| **TTL (Time-to-Live)** | A DynamoDB attribute that automatically expires and deletes records after a specified Unix timestamp, enabling passive memory decay. |
| **Semantic Similarity** | The degree of meaning overlap between two texts, computed via cosine similarity of their embedding vectors — more reliable than word overlap for noisy financial language. |

---

## What's Next

**Day 40 — Long-Term Memory: Vector Store and Entity Extraction**

Episodic memory is per-user and event-based. Day 40 introduces long-term semantic memory: a vector store that accumulates durable facts *about* each user (risk tolerance, preferred sectors, trading history) as named entities. You will see how to extract these entities with an LLM, embed and store them in OpenSearch Serverless, and retrieve them to personalise every recommendation.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 38](./Day-38-Working-Memory.md) | [Day 40 →](./Day-40-long-term-memory.md)
