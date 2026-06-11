# Day 40 — Long-Term Memory: Vector Store + Entity Extraction

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Episodic memory gives the agent a diary. Long-term semantic memory gives it a dossier — a structured profile of who each user is, extracted and refined from every conversation."*

---

## Explanation of Day Topic

Episodic memory (Day 39) stores *what happened* — events, queries, and decisions timestamped and decaying over time. Long-term semantic memory stores *who the user is* — stable preference facts and constraints that should persist indefinitely and inform every future interaction.

The distinction matters for retrieval strategy. Episodes are retrieved by recency and query similarity. Semantic entities are retrieved by type: when a user asks about portfolio allocation, you always want their risk tolerance — not just the most recent time they mentioned it.

This day covers LLM-based entity extraction from free-form conversation using Bedrock Claude, embedding and indexing extracted entities in OpenSearch Serverless, and assembling a structured user profile for system prompt injection. The result: an agent that knows a user's risk tolerance, preferred sectors, and excluded stocks without being told again each session.

---

### Long-Term Memory via Vector Store

```python
from langchain.embeddings import BedrockEmbeddings
import boto3

class LongTermMemory:
    def __init__(self):
        self.embeddings = BedrockEmbeddings(
            client=boto3.client('bedrock-runtime'),
            model_id='amazon.titan-embed-text-v2:0'
        )
        self.vector_store = None  # Pinecone/OpenSearch
    
    def extract_and_store_entities(self, user_id, interaction):
        # Extract: "User is risk-averse tech investor"
        entities = self.extract_entities(interaction)
        
        for entity in entities:
            embedding = self.embeddings.embed_query(entity)
            self.vector_store.upsert({
                'id': f"{user_id}_{entity}",
                'values': embedding,
                'metadata': {'user_id': user_id, 'entity': entity}
            })
    
    def retrieve_user_profile(self, user_id):
        # Get all user entities
        results = self.vector_store.query(
            vector=self.embeddings.embed_query(user_id),
            filter={'user_id': user_id},
            top_k=20
        )
        
        profile = [r['metadata']['entity'] for r in results]
        return profile
```

---

### Entity Extraction in Practice

The stub above calls `self.extract_entities(interaction)` without showing what that means. Here is the full pipeline: extract, store, and retrieve user preference entities from a real financial conversation.

```python
import boto3
import json
import numpy as np
from datetime import datetime

class LongTermMemoryFull:
    """
    Full entity extraction + vector store pipeline using Bedrock + OpenSearch Serverless.
    """

    def __init__(self, opensearch_endpoint: str, index_name: str = 'user-entities'):
        from opensearchpy import OpenSearch, RequestsHttpConnection
        from requests_aws4auth import AWS4Auth
        import boto3

        self.bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
        self.index_name = index_name

        credentials = boto3.Session().get_credentials()
        auth = AWS4Auth(
            credentials.access_key,
            credentials.secret_key,
            'us-east-1',
            'aoss',
            session_token=credentials.token
        )
        self.opensearch = OpenSearch(
            hosts=[{'host': opensearch_endpoint, 'port': 443}],
            http_auth=auth,
            use_ssl=True,
            verify_certs=True,
            connection_class=RequestsHttpConnection
        )

    # --- Step 1: Extract entities from a conversation turn ---
    def extract_entities(self, conversation_text: str) -> list[dict]:
        """
        Ask Claude to extract structured preference entities.
        Returns a list of {entity_type, entity_value, confidence} dicts.
        """
        prompt = f"""Extract user preference entities from this financial conversation.
Return a JSON array. Each item must have:
  - entity_type: one of [risk_tolerance, preferred_sector, investment_horizon, excluded_stock, currency_preference]
  - entity_value: the extracted value as a short phrase
  - confidence: 0.0–1.0

Conversation:
{conversation_text}

JSON array only, no explanation."""

        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "messages": [{"role": "user", "content": prompt}]
        })
        resp = self.bedrock.invoke_model(
            modelId='anthropic.claude-3-5-sonnet-20241022-v2:0', body=body
        )
        raw = json.loads(resp['body'].read())['content'][0]['text']
        try:
            return json.loads(raw)
        except json.JSONDecodeError:
            return []

    # --- Step 2: Embed an entity value ---
    def _embed(self, text: str) -> list[float]:
        body = json.dumps({'inputText': text})
        resp = self.bedrock.invoke_model(
            modelId='amazon.titan-embed-text-v2:0',
            body=body,
            contentType='application/json',
            accept='application/json'
        )
        return json.loads(resp['body'].read())['embedding']

    # --- Step 3: Store entities in OpenSearch ---
    def store_entities(self, user_id: str, conversation_text: str):
        entities = self.extract_entities(conversation_text)
        for entity in entities:
            if entity.get('confidence', 0) < 0.6:
                continue   # Skip low-confidence extractions

            embedding = self._embed(entity['entity_value'])
            doc = {
                'user_id': user_id,
                'entity_type': entity['entity_type'],
                'entity_value': entity['entity_value'],
                'confidence': entity['confidence'],
                'embedding': embedding,
                'updated_at': datetime.now().isoformat()
            }
            doc_id = f"{user_id}_{entity['entity_type']}"
            self.opensearch.index(index=self.index_name, id=doc_id, body=doc)

    # --- Step 4: Retrieve user profile for personalisation ---
    def retrieve_user_profile(self, user_id: str, current_query: str) -> dict:
        """
        Return the user's stored preference entities, ranked by relevance
        to the current query via semantic search.
        """
        query_embedding = self._embed(current_query)
        search_body = {
            "query": {
                "bool": {
                    "filter": [{"term": {"user_id": user_id}}],
                    "must": [{
                        "knn": {
                            "embedding": {
                                "vector": query_embedding,
                                "k": 10
                            }
                        }
                    }]
                }
            }
        }
        result = self.opensearch.search(index=self.index_name, body=search_body)
        hits = result['hits']['hits']

        profile = {}
        for hit in hits:
            etype = hit['_source']['entity_type']
            profile[etype] = hit['_source']['entity_value']

        return profile


# --- End-to-end example ---
ltm = LongTermMemoryFull(opensearch_endpoint='your-collection.us-east-1.aoss.amazonaws.com')

# After a conversation turn:
conversation = (
    "User: I'm quite risk-averse — no crypto or meme stocks. "
    "I prefer tech and healthcare, and I have a 10-year horizon. "
    "Please exclude TSLA from any recommendations.\n"
    "Agent: Noted. I'll focus on blue-chip tech and large-cap healthcare."
)

ltm.store_entities(user_id='trader_007', conversation_text=conversation)
# Stored entities:
# { entity_type: 'risk_tolerance',    entity_value: 'risk-averse' }
# { entity_type: 'preferred_sector',  entity_value: 'tech and healthcare' }
# { entity_type: 'investment_horizon', entity_value: '10-year horizon' }
# { entity_type: 'excluded_stock',    entity_value: 'TSLA' }

# Later session — personalise a recommendation:
profile = ltm.retrieve_user_profile(
    user_id='trader_007',
    current_query='Suggest some ETFs for my portfolio'
)
# profile = {
#   'risk_tolerance': 'risk-averse',
#   'preferred_sector': 'tech and healthcare',
#   'excluded_stock': 'TSLA',
#   'investment_horizon': '10-year horizon'
# }

system_prompt = f"""You are a financial advisor.
User profile:
- Risk tolerance: {profile.get('risk_tolerance', 'unknown')}
- Preferred sectors: {profile.get('preferred_sector', 'any')}
- Excluded stocks: {profile.get('excluded_stock', 'none')}
- Horizon: {profile.get('investment_horizon', 'unspecified')}

Tailor all recommendations to this profile."""
```

The key insight: entities extracted from *past* conversations are injected into the *current* system prompt — giving the LLM persistent knowledge of who it is talking to without re-asking every session.

---

#### Key Terms for Day 40

| Term | Meaning |
|------|---------|
| **Long-Term Memory** | Durable, cross-session storage of facts about a user or domain — persists indefinitely until explicitly updated or deleted. |
| **Entity Extraction** | The process of identifying and labelling structured facts (entities) in free-form text, e.g. extracting `risk_tolerance: risk-averse` from a conversation. |
| **Vector Store** | A database that stores embedding vectors and supports approximate nearest-neighbour search; the engine behind semantic retrieval from long-term memory. |
| **Embedding** | A high-dimensional numerical vector representing the semantic content of a piece of text; similar meanings produce similar vectors. |
| **Semantic Search** | Retrieval based on meaning similarity (cosine distance between embeddings) rather than exact keyword matching. |
| **User Profile** | The aggregated set of preference entities extracted and stored for a specific user — used to personalise agent responses across sessions. |
| **Knowledge Graph** | An alternative long-term memory representation where entities are nodes and their relationships are edges; richer than flat vector stores for complex domains. |

---

## What's Next

**Day 41 — Memory Decay and Intentional Forgetting**

Long-term memory is powerful — but it accumulates forever unless you actively manage it. Day 41 covers intentional forgetting: TTL-based expiry, exponential decay scoring, and GDPR right-to-forget implementations. You will learn why stale memories are worse than no memories, and how to design a system that keeps only what is still relevant.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 39](./Day_39_Episodic-Memory.md) | [Day 41 →](./Day-Day-41-Memory-decay.md)
