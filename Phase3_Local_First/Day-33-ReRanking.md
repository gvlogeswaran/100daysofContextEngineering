# Day 33 — Re-ranking: The Secret Weapon in Production RAG

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Vector search is fast but approximate. Re-ranking is slow but accurate. Do the fast search first, then use re-ranking to polish the final results. Best of both worlds."*

---


### Re-ranking: Improving Retrieval Accuracy with Cross-Encoders

Re-ranking is the difference between "works" and "production-ready" RAG. Here's how to implement it.

#### 1. Bi-Encoders vs Cross-Encoders

**Bi-encoders (retrieval):**
- Embed query and documents separately
- Compare embeddings (fast)
- Used for initial retrieval (top 50-100)

**Cross-encoders (re-ranking):**
- Take query + document as input
- Score relevance directly
- More accurate but slower
- Used for final ranking (top 5-10)

```python
from sentence_transformers import SentenceTransformer, CrossEncoder
import numpy as np
from sklearn.metrics.pairwise import cosine_similarity

# Bi-encoder: Fast but approximate
bi_encoder = SentenceTransformer('all-MiniLM-L6-v2')
query_embedding = bi_encoder.encode("AAPL Q3 revenue")
doc_embeddings = bi_encoder.encode(documents)

similarity_scores = cosine_similarity([query_embedding], doc_embeddings)[0]
top_50_indices = np.argsort(similarity_scores)[-50:][::-1]

# Cross-encoder: Slow but accurate
cross_encoder = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')
pairs = [[query, documents[i]] for i in top_50_indices]
scores = cross_encoder.predict(pairs)

reranked_indices = top_50_indices[np.argsort(scores)[::-1]]
```

#### 2. Using Cohere Rerank

```python
import cohere

client = cohere.ClientV2(api_key="YOUR_COHERE_API_KEY")

retrieved_docs = [
    "AAPL Q3 revenue was $93.7B",
    "Apple earnings beat expectations",
    "AAPL operating expenses...",
]

response = client.rerank(
    query="AAPL Q3 revenue guidance vs actual",
    documents=retrieved_docs,
    model="rerank-english-v3.0",
    top_n=5
)

for result in response.results:
    print(f"Rank: {result.index}, Score: {result.relevance_score:.3f}")
```

#### 3. AWS Bedrock Re-ranking

```python
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")

response = bedrock.retrieve_and_generate(
    input={"text": "AAPL Q3 revenue guidance vs actual"},
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "your-kb-id",
            "retrievalConfiguration": {
                "vectorSearchConfiguration": {
                    "numberOfResults": 50,
                }
            }
        }
    }
)
```

#### 4. Custom Re-ranking Pipeline

```python
from sentence_transformers import CrossEncoder
from langchain.embeddings import BedrockEmbeddings
import boto3

class RAGWithReranking:
    def __init__(self):
        self.bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
        self.embeddings = BedrockEmbeddings(
            client=self.bedrock,
            model_id="amazon.titan-embed-text-v2:0"
        )
        self.reranker = CrossEncoder('cross-encoder/mmarco-mMiniLMv2-L12-H384-v1')
        self.vector_store = None  # Your vector DB
    
    def retrieve_and_rerank(self, query, k=5):
        query_embedding = self.embeddings.embed_query(query)
        candidates = self.vector_store.similarity_search_with_score(query_embedding, k=50)
        
        candidate_docs = [doc for doc, _ in candidates]
        candidate_texts = [doc.page_content for doc in candidate_docs]
        
        rerank_pairs = [[query, text] for text in candidate_texts]
        rerank_scores = self.reranker.predict(rerank_pairs)
        
        ranked_indices = np.argsort(rerank_scores)[::-1]
        final_docs = [candidate_docs[i] for i in ranked_indices[:k]]
        
        return final_docs
```

#### 5. Re-ranking Benchmarks

| Stage | Latency | Accuracy | Best Use |
|
---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 32](./Day-32-MCP-Inspector-Tool.md) | [Day 34 →](./Day-34-Server-Side-Logging.md)
