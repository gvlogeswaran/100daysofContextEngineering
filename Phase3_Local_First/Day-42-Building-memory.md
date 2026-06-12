# Day 42 — Building a Memory System on AWS

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Knowing what to store is theory. Knowing how to wire DynamoDB, OpenSearch, S3, and Bedrock into a single coherent pipeline — that is engineering."*

---

## Explanation of Day Topic

Days 39 through 41 built each memory layer independently: episodic storage in DynamoDB, semantic entities in OpenSearch, decay-based forgetting. Day 42 assembles them into a single production pipeline — one Lambda function that receives a user query, fetches from all three memory stores, assembles a context block, calls Bedrock, and writes the new episode back in one orchestrated flow.

This day covers the DynamoDB single-table schema that unifies all memory types under one table (avoiding the operational overhead of multiple tables), the Lambda orchestrator that sequences the pipeline, a practical AWS cost model at realistic scale, and the architecture decisions that make the system production-safe: warm Lambda re-use, token-capped response storage, and split-function patterns for high concurrency.

Understanding this day is the payoff for the memory architecture work in Days 39–41. This is what it looks like when all the pieces run together.

---

### AWS Memory System Architecture

```python
import boto3

# DynamoDB: Episodic memory
dynamodb = boto3.resource('dynamodb')
episodes = dynamodb.create_table(
    TableName='agent-episodes',
    KeySchema=[{'AttributeName': 'user_id', 'KeyType': 'HASH'}],
    BillingMode='PAY_PER_REQUEST'
)

# OpenSearch: Semantic memory
opensearch = boto3.client('opensearchserverless')

# S3: Document store
s3 = boto3.client('s3')
s3.create_bucket(Bucket='memory-documents')

# Bedrock: LLM + embeddings
bedrock = boto3.client('bedrock-runtime')
```

**Cost estimate:**
- DynamoDB: $1/1M writes = ~$10/month
- OpenSearch Serverless: $0.24/OCU-hour = ~$175/month
- S3: $0.023/GB = ~$230 (1TB)
- Bedrock embeddings: $0.0001 per 1K = ~$100/month
- Total: ~$500-3K/month (depends on scale)

---

### DynamoDB Schema — Single-Table Design

All memory types live in one DynamoDB table using a PK/SK pattern. This keeps cross-entity queries fast and avoids managing multiple tables.

```
Table: agent-memory  (PAY_PER_REQUEST billing, TTL attribute: ttl)

PK                          SK                              Attributes
--------------------------  ------------------------------  ------------------------------------------
USER#trader_007             PROFILE#v1                      risk_tolerance, preferred_sector, horizon
USER#trader_007             EPISODE#2025-06-01T09:00:00     query, response, action, embedding, ttl
USER#trader_007             EPISODE#2025-06-01T14:32:10     query, response, action, embedding, ttl
USER#trader_007             SESSION#sess_abc123             session_start, last_active, message_count
AUDIT#trader_007            GDPR#2025-07-01T10:00:00        action=ERASURE, performed_by=system
SYSTEM#config               MEMORY_POLICY#v2                ttl_days, decay_half_life, max_episodes
```

Design rules:
- `PK = USER#<user_id>` groups all data for one user; enables single-query full erasure.
- `SK` encodes both the record type and a sortable timestamp — lets you query latest N episodes without a GSI.
- `AUDIT#` prefix is a separate partition — isolated from user data, never deleted.

```python
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('agent-memory')

def get_latest_episodes(user_id: str, limit: int = 10) -> list:
    response = table.query(
        KeyConditionExpression=(
            Key('PK').eq(f'USER#{user_id}') &
            Key('SK').begins_with('EPISODE#')
        ),
        ScanIndexForward=False,   # Newest first
        Limit=limit
    )
    return response.get('Items', [])

def get_user_profile(user_id: str) -> dict:
    response = table.get_item(
        Key={'PK': f'USER#{user_id}', 'SK': 'PROFILE#v1'}
    )
    return response.get('Item', {})
```

---

### Lambda Orchestration — Full Memory Pipeline

This Lambda function is the core of the system. It receives every user query, assembles context from all memory layers, calls Bedrock, and writes the new episode back to DynamoDB.

```python
import boto3
import json
import os
from datetime import datetime
from boto3.dynamodb.conditions import Key
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth

# --- Clients (initialised outside handler for warm re-use) ---
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
memory_table = dynamodb.Table('agent-memory')
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

credentials = boto3.Session().get_credentials()
auth = AWS4Auth(
    credentials.access_key, credentials.secret_key,
    'us-east-1', 'aoss', session_token=credentials.token
)
opensearch = OpenSearch(
    hosts=[{'host': os.environ['OPENSEARCH_ENDPOINT'], 'port': 443}],
    http_auth=auth, use_ssl=True, verify_certs=True,
    connection_class=RequestsHttpConnection
)

MODEL_ID = 'anthropic.claude-3-5-sonnet-20241022-v2:0'
EMBED_ID  = 'amazon.titan-embed-text-v2:0'


def _embed(text: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId=EMBED_ID,
        body=json.dumps({'inputText': text}),
        contentType='application/json',
        accept='application/json'
    )
    return json.loads(resp['body'].read())['embedding']


# (a) Fetch episodic memory from DynamoDB
def _fetch_episodic(user_id: str, limit: int = 5) -> list[dict]:
    resp = memory_table.query(
        KeyConditionExpression=(
            Key('PK').eq(f'USER#{user_id}') &
            Key('SK').begins_with('EPISODE#')
        ),
        ScanIndexForward=False,
        Limit=limit
    )
    return resp.get('Items', [])


# (b) Fetch semantic memory from OpenSearch
def _fetch_semantic(user_id: str, query: str, top_k: int = 5) -> list[dict]:
    embedding = _embed(query)
    result = opensearch.search(
        index='user-entities',
        body={
            "query": {
                "bool": {
                    "filter": [{"term": {"user_id": user_id}}],
                    "must": [{"knn": {"embedding": {"vector": embedding, "k": top_k}}}]
                }
            }
        }
    )
    return [hit['_source'] for hit in result['hits']['hits']]


# (c) Assemble context block for the LLM
def _assemble_context(
    user_profile: dict,
    episodes: list[dict],
    entities: list[dict],
    query: str
) -> str:
    ep_text = "\n".join(
        f"- [{ep['SK'].replace('EPISODE#', '')[:10]}] Q: {ep['query']} | A: {ep['response'][:80]}"
        for ep in episodes
    )
    ent_text = "\n".join(
        f"- {e['entity_type']}: {e['entity_value']}"
        for e in entities
    )
    profile_text = (
        f"Risk tolerance: {user_profile.get('risk_tolerance', 'unknown')}\n"
        f"Preferred sectors: {user_profile.get('preferred_sector', 'any')}\n"
        f"Investment horizon: {user_profile.get('horizon', 'unspecified')}"
    )
    return (
        f"USER PROFILE:\n{profile_text}\n\n"
        f"RECENT INTERACTIONS:\n{ep_text or 'None'}\n\n"
        f"USER PREFERENCES (semantic memory):\n{ent_text or 'None'}\n\n"
        f"CURRENT QUERY:\n{query}"
    )


# (d) Call Bedrock with assembled context
def _call_bedrock(system_context: str, query: str) -> str:
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1024,
        "system": (
            "You are a personalised financial assistant. "
            "Use the provided user profile and memory context to tailor your response.\n\n"
            + system_context
        ),
        "messages": [{"role": "user", "content": query}]
    })
    resp = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    return json.loads(resp['body'].read())['content'][0]['text']


# (e) Write new episode back to DynamoDB
def _write_episode(user_id: str, query: str, response: str):
    embedding = _embed(query)
    memory_table.put_item(Item={
        'PK': f'USER#{user_id}',
        'SK': f'EPISODE#{datetime.now().isoformat()}',
        'query': query,
        'response': response[:500],   # Cap stored response length
        'embedding': [str(v) for v in embedding],  # DynamoDB requires Decimal-safe values
        'ttl': int(datetime.now().timestamp()) + (90 * 86400)
    })


# --- Lambda Handler ---
def lambda_handler(event: dict, context) -> dict:
    user_id = event['user_id']
    query   = event['query']

    # (a) Episodic memory
    episodes = _fetch_episodic(user_id, limit=5)

    # (b) Semantic memory
    entities = _fetch_semantic(user_id, query, top_k=5)

    # (c) User profile
    profile_item = memory_table.get_item(
        Key={'PK': f'USER#{user_id}', 'SK': 'PROFILE#v1'}
    ).get('Item', {})

    # (d) Assemble and call Bedrock
    context_block = _assemble_context(profile_item, episodes, entities, query)
    answer = _call_bedrock(context_block, query)

    # (e) Persist new episode
    _write_episode(user_id, query, answer)

    return {
        'statusCode': 200,
        'body': json.dumps({'answer': answer})
    }
```

This single Lambda function implements the full memory pipeline: fetch → assemble → generate → persist. For production, split the fetch, generate, and persist steps into separate functions behind an SQS queue to handle high concurrency.

---

#### Key Terms for Day 42

| Term | Meaning |
|------|---------|
| **Lambda Orchestration** | Using AWS Lambda as the stateless glue that coordinates calls to DynamoDB, OpenSearch, and Bedrock in a defined pipeline sequence. |
| **Single-Table Design** | A DynamoDB pattern where multiple entity types share one table, differentiated by PK/SK patterns — reduces operational overhead and enables cross-entity transactions. |
| **Memory Assembly** | The process of fetching from multiple memory stores (episodic, semantic, profile) and combining them into a single context block for the LLM. |
| **Bedrock** | AWS's managed API for foundation models including Claude, Titan, and Llama — the inference layer in the memory pipeline. |
| **OpenSearch Serverless** | AWS's managed serverless OpenSearch offering with built-in k-NN (vector) search — the semantic memory store in this architecture. |
| **Context Pipeline** | The end-to-end flow from raw user query through memory retrieval, context assembly, LLM inference, and memory write-back. |

---

## What's Next

**Day 43 — Context Window Management: The Sliding Window Pattern**

The Lambda pipeline now assembles rich context from multiple sources. Day 43 addresses the next constraint: the assembled context must fit within the LLM's token limit. You will learn how to allocate the token budget across system prompt, memory, RAG context, and conversation history — and how to implement pinned context and compression triggers to handle long sessions without losing critical information.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 41](./Day-41-Memory-decay.md)
