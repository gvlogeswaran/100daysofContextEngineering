# Day 41 — Memory Decay and Intentional Forgetting

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Infinite memory is not a feature — it is a liability. Stale preferences mislead. Forgotten consent matters. The discipline is knowing what to keep, what to compress, and what to delete."*

---

## Explanation of Day Topic

Days 39 and 40 built a memory system that accumulates: episodic interactions in DynamoDB, semantic entities in OpenSearch Serverless. Both stores grow indefinitely unless you design an expiry path from day one.

This creates three compounding problems. First, stale preferences degrade quality — a risk tolerance recorded 18 months ago may no longer reflect the user's situation, and retrieving it as current produces bad, potentially harmful recommendations. Second, storage costs compound at scale. Third, and most critically for financial and EU-regulated systems: GDPR Article 17 grants users the right to demand erasure of all their personal data, and a system with no deletion path is a compliance failure, not a missing feature.

This day covers three forgetting strategies — DynamoDB TTL for passive expiry, exponential decay scoring for soft de-prioritisation in retrieval, and a complete GDPR right-to-forget implementation spanning DynamoDB, OpenSearch, and S3.

---

### Memory Decay and Compliance

```python
from datetime import datetime, timedelta

class MemoryWithDecay:
    def decay_score(self, item):
        age_days = (datetime.now() - item['timestamp']).days
        decay = 1.0 / (1.0 + age_days * 0.1)  # Exponential decay
        return decay
    
    def cleanup_expired(self):
        # GDPR: Delete all items > 90 days
        cutoff = datetime.now() - timedelta(days=90)
        # Delete all items older than cutoff
        self.table.delete_items(cutoff)
```

---

### Why Forgetting Is a Feature

Most engineers treat memory as a pure accumulation problem: store more, retrieve better. That is wrong in at least four ways.

**1. Stale preferences degrade quality.** A user who said "I prefer growth stocks" two years ago may now be approaching retirement and want income. Retrieving that old preference as if it is current produces bad recommendations — sometimes harmful ones in a regulated financial context.

**2. Storage costs compound.** A system with 100,000 users storing 50 episodes each at 500 tokens per episode = 2.5 billion tokens of data. At embedding storage costs, that is significant. Most of it is irrelevant to any current query.

**3. GDPR and financial regulation require it.** In the EU and many financial jurisdictions, you must be able to purge all personal data for a user on request. A system with no deletion path is a compliance liability, not a feature.

**4. Retrieval precision degrades with noise.** Vector search returns the top-k nearest neighbours across *everything* you've stored. If 80% of episodes are old and irrelevant, they compete with the 20% that matter. Forgetting old episodes improves the signal-to-noise ratio of every retrieval.

---

### Three Forgetting Strategies

#### Strategy 1 — Time-Based TTL (DynamoDB Automatic Expiry)

The simplest approach: set a `ttl` attribute on every record as a Unix timestamp. DynamoDB deletes items automatically when the TTL expires — no Lambda, no cron job.

```python
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('agent-episodes')

def store_episode_with_ttl(user_id: str, query: str, response: str, ttl_days: int = 90):
    """Store an episode that auto-expires after ttl_days."""
    expiry_ts = int(datetime.now().timestamp()) + (ttl_days * 86400)
    table.put_item(Item={
        'PK': f'USER#{user_id}',
        'SK': f'EPISODE#{datetime.now().isoformat()}',
        'query': query,
        'response': response,
        'ttl': expiry_ts          # DynamoDB TTL attribute — must be a Unix epoch integer
    })

# Enable TTL on the table (run once at setup):
dynamodb_client = boto3.client('dynamodb', region_name='us-east-1')
dynamodb_client.update_time_to_live(
    TableName='agent-episodes',
    TimeToLiveSpecification={'Enabled': True, 'AttributeName': 'ttl'}
)
```

DynamoDB TTL deletion typically happens within 48 hours of expiry. For stricter compliance (e.g. real-time deletion), use Lambda triggered on expiry events via DynamoDB Streams.

#### Strategy 2 — Score-Based Decay (Exponential Decay Formula)

TTL is binary: items exist or they don't. Decay scoring is continuous: items gradually become less influential before eventual deletion. Use this when you want to *de-prioritise* old memories in retrieval rather than delete them immediately.

```python
import math
from datetime import datetime

def decay_score(stored_at: datetime, half_life_days: float = 30.0) -> float:
    """
    Exponential decay: score = e^(-λ * age_days)
    where λ = ln(2) / half_life_days

    At age 0:          score = 1.0  (fully relevant)
    At age 30 days:    score = 0.5  (half as relevant)
    At age 60 days:    score = 0.25 (quarter as relevant)
    At age 90 days:    score = 0.125 (probably worth deleting)
    """
    age_days = (datetime.now() - stored_at).days
    lam = math.log(2) / half_life_days
    return math.exp(-lam * age_days)

def combined_retrieval_score(
    semantic_similarity: float,
    stored_at: datetime,
    half_life_days: float = 30.0,
    relevance_weight: float = 0.7,
    recency_weight: float = 0.3
) -> float:
    """Blend semantic relevance with time decay for ranked retrieval."""
    recency = decay_score(stored_at, half_life_days)
    return relevance_weight * semantic_similarity + recency_weight * recency

# Example: An episode from 45 days ago with 0.85 cosine similarity
from datetime import timedelta
old_episode_date = datetime.now() - timedelta(days=45)
score = combined_retrieval_score(
    semantic_similarity=0.85,
    stored_at=old_episode_date
)
print(f"Combined score: {score:.3f}")
# → Combined score: 0.701  (still relevant but deprioritised vs a newer episode)

# Scheduled cleanup: delete episodes where decay_score < threshold
def prune_low_value_episodes(user_id: str, decay_threshold: float = 0.1):
    """Remove episodes whose decay score has fallen below the threshold."""
    from boto3.dynamodb.conditions import Key
    table = boto3.resource('dynamodb').Table('agent-episodes')

    response = table.query(
        KeyConditionExpression=Key('PK').eq(f'USER#{user_id}')
    )
    for item in response.get('Items', []):
        created = datetime.fromisoformat(item['SK'].replace('EPISODE#', ''))
        if decay_score(created) < decay_threshold:
            table.delete_item(Key={'PK': item['PK'], 'SK': item['SK']})
```

#### Strategy 3 — Right-to-Forget (GDPR Delete-by-User)

Under GDPR Article 17 and equivalent financial regulations, a user can demand erasure of all their personal data. Your memory system must support a complete, auditable delete.

```python
import boto3
from boto3.dynamodb.conditions import Key

def gdpr_delete_user(user_id: str):
    """
    Delete all memory records for a user across every storage layer.
    Call this when processing a Subject Access Request (SAR) erasure.
    """
    # 1. DynamoDB — episodic memory
    dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
    table = dynamodb.Table('agent-episodes')

    response = table.query(
        KeyConditionExpression=Key('PK').eq(f'USER#{user_id}')
    )
    with table.batch_writer() as batch:
        for item in response.get('Items', []):
            batch.delete_item(Key={'PK': item['PK'], 'SK': item['SK']})

    # 2. OpenSearch — long-term entity memory
    from opensearchpy import OpenSearch
    # (initialise client as in Day 40)
    os_client = OpenSearch(...)  # your configured client
    os_client.delete_by_query(
        index='user-entities',
        body={"query": {"term": {"user_id": user_id}}}
    )

    # 3. S3 — any stored documents or trace logs
    s3 = boto3.client('s3', region_name='us-east-1')
    paginator = s3.get_paginator('list_objects_v2')
    for page in paginator.paginate(Bucket='memory-documents', Prefix=f'users/{user_id}/'):
        for obj in page.get('Contents', []):
            s3.delete_object(Bucket='memory-documents', Key=obj['Key'])

    # 4. Audit log — record the deletion (the audit log itself is immutable)
    audit_table = dynamodb.Table('gdpr-audit-log')
    audit_table.put_item(Item={
        'PK': f'DELETION#{user_id}',
        'SK': datetime.now().isoformat(),
        'action': 'GDPR_ERASURE',
        'performed_by': 'system',
        'note': 'All episodic, semantic, and document memory deleted on user request'
    })

    print(f"[GDPR] All memory for user {user_id} deleted and audit record created.")
```

---

### When NOT to Forget

Not all memory is mutable preference data. Some records must be **immutable** — kept forever regardless of TTL, decay, or user request.

| Memory Type | Forgettable? | Reason |
|-------------|-------------|--------|
| User risk tolerance preference | Yes | Can change; stale data is harmful |
| Preferred sector / stock exclusions | Yes | Evolves with life stage |
| Trade confirmations | **No** | Regulatory record-keeping (MiFID II, SEC) |
| Audit logs of agent actions | **No** | Compliance and dispute resolution |
| GDPR deletion records | **No** | Proof of compliance |
| Chat summaries (anonymised) | Maybe | Retain for model improvement if consent given |

The rule: **preferences are mutable, events are immutable**. Build two separate storage paths with different retention policies from day one — retrofitting immutability is expensive.

---

#### Key Terms for Day 41

| Term | Meaning |
|------|---------|
| **Memory Decay** | The intentional reduction in the influence of stored memories over time, preventing stale data from polluting agent responses. |
| **TTL (Time-to-Live)** | A timestamp attribute that signals a database (e.g. DynamoDB) to automatically delete a record after a specified time. |
| **Exponential Decay** | A decay function `e^(-λt)` where influence halves every `half_life` days — produces a smooth, gradual reduction rather than a hard cutoff. |
| **Right-to-Forget** | The GDPR principle (Article 17) granting individuals the right to demand deletion of all their personal data from a system. |
| **GDPR** | General Data Protection Regulation — EU law governing personal data handling; requires erasure capability, data minimisation, and purpose limitation. |
| **Memory Pressure** | The situation where too many stored memories degrade retrieval quality — either by exceeding vector store capacity or by flooding search results with low-relevance items. |

---

## What's Next

**Day 42 — Building a Memory System on AWS**

Day 41 covered what to keep and what to delete. Day 42 puts it all together: a complete AWS memory architecture using DynamoDB for episodic storage, OpenSearch Serverless for semantic search, S3 for document storage, and Lambda to orchestrate the entire memory assembly pipeline — from query in to context out.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 40](./Day-40-long-term-memory.md) | [Day 43 →](./Day-42-Building-memory.md)
