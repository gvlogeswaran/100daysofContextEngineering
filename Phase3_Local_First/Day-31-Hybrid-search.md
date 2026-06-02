# Day 31 — Hybrid Search: Why Keyword + Semantic Beats Both Alone

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Semantic search is like finding books by meaning. Keyword search is like using an index. One without the other fails. Together, you get the perfect book."*

---


### Hybrid Search: Combining BM25 and Vector Similarity

Financial search requires both semantic understanding and keyword precision. Here's how to build it.

#### 1. The Hybrid Search Problem

**Scenario:** Query "AAPL Q3 guidance vs actual Q3 results"

**Pure semantic search:**
```
Top result: Earnings discussion (semantically similar, but vague)
Problem: Might miss the specific guidance figure
```

**Pure keyword search:**
```
Top result: "AAPL", "Q3", "guidance", "actual" all present
Problem: No understanding of context or relevance
```

**Hybrid:**
```
BM25 finds documents with exact terms
Vector search finds semantically similar documents
Results ranked by both metrics
```

#### 2. BM25: The Keyword Search Algorithm

BM25 is a ranking function used by Elasticsearch/OpenSearch. It scores documents based on term frequency and inverse document frequency (TF-IDF).

```python
# BM25 scoring (implemented in Elasticsearch/OpenSearch)
# Score = sum of term scores
# Term score = (frequency in doc) / (rarity of term across corpus)

# Example:
# "AAPL" appears 5 times in doc, 100 times in corpus = moderate score
# "guidance" appears 2 times in doc, 10 times in corpus = high score
# "the" appears 50 times in doc, 1M times in corpus = near-zero score

# BM25 naturally handles:
# - Repeated terms (boosts score)
# - Common words (reduces score)
# - Exact phrase matching
```

**Advantages:**
- Fast (inverted index)
- Handles exact phrase matching
- Good for specific terms (ticker symbols, regulation names)

**Disadvantages:**
- No semantic understanding
- "Revenue" and "earnings" are treated as completely different terms

#### 3. Vector Search: Semantic Similarity

Vector search uses embeddings to find semantically similar documents.

```python
# Vector search
# Embed query: "AAPL revenue guidance"
# Embed all documents
# Find documents with highest cosine similarity

# Example:
# Query embedding for "AAPL revenue guidance"
# Doc 1 embedding: "Apple earnings projection" (high similarity = 0.92)
# Doc 2 embedding: "Stock market prediction" (low similarity = 0.35)
# Doc 3 embedding: "AAPL guidance vs results" (high similarity = 0.88)
```

**Advantages:**
- Semantic understanding (synonyms, paraphrases)
- Context-aware
- Handles variations in wording

**Disadvantages:**
- Can miss specific terms (especially domain terms like ticker symbols)
- Slower than keyword search

#### 4. Reciprocal Rank Fusion (RRF)

RRF combines multiple ranked lists into a single ranked list.

```python
# RRF formula: score = sum(1 / (k + rank_i))
# k = typically 60, rank_i = rank in list i

# Example:
# Query: "AAPL Q3 guidance"

# BM25 results:
# Rank 1: "AAPL Q3 guidance exceeded" (score: 1/(60+1) = 0.0164)
# Rank 2: "Q3 results vs guidance" (score: 1/(60+2) = 0.0159)
# Rank 10: "Earnings analysis" (score: 1/(60+10) = 0.0139)

# Vector search results:
# Rank 1: "Apple Q3 earnings discussion" (score: 1/(60+1) = 0.0164)
# Rank 5: "AAPL Q3 guidance exceeded" (score: 1/(60+5) = 0.0152)
# Rank 20: "Quarterly updates" (score: 1/(60+20) = 0.0128)

# Combined RRF:
# Doc "AAPL Q3 guidance exceeded": 0.0164 + 0.0152 = 0.0316 (highest)
# Doc "Apple Q3 earnings discussion": 0.0164 (only in vector)
# Doc "Q3 results vs guidance": 0.0159 (only in BM25)
```

#### 5. Implementing Hybrid Search in AWS OpenSearch

```python
import boto3
import json
from opensearchpy import OpenSearch

client = OpenSearch(
    hosts=[{'host': 'your-opensearch-domain.us-east-1.aoss.amazonaws.com', 'port': 443}],
    http_auth=(username, password),
    use_ssl=True,
    verify_certs=True
)

# Hybrid query: BM25 + Vector search
hybrid_query = {
    "size": 10,
    "query": {
        "hybrid": {
            "queries": [
                # BM25 (keyword search)
                {
                    "match": {
                        "text": {
                            "query": "AAPL Q3 guidance vs actual",
                            "operator": "AND"
                        }
                    }
                },
                # Vector search (semantic)
                {
                    "knn": {
                        "embedding": {
                            "vector": query_embedding,
                            "k": 10
                        }
                    }
                }
            ]
        }
    },
    "ext": {
        "fusion": {
            "algorithm": "rrf",  # Reciprocal Rank Fusion
            "weight": [0.5, 0.5]  # Equal weight to BM25 and vector
        }
    }
}

results = client.search(index="financial-docs", body=hybrid_query)
```

#### 6. Implementing Hybrid Search Manually

```python
import boto3
from langchain.embeddings import BedrockEmbeddings
from opensearchpy import OpenSearch

class HybridSearchRag:
    def __init__(self):
        self.os_client = OpenSearch(...)  # OpenSearch client
        self.embeddings = BedrockEmbeddings(...)
    
    def hybrid_search(self, query, top_k=10):
        # 1. BM25 search
        bm25_query = {
            "size": top_k * 2,  # Get more results for fusion
            "query": {
                "multi_match": {
                    "query": query,
                    "fields": ["text", "ticker", "entity^2"],  # Boost entity matches
                    "operator": "OR"
                }
            }
        }
        bm25_results = self.os_client.search(index="docs", body=bm25_query)
        bm25_docs = {hit["_id"]: (idx + 1, hit) for idx, hit in enumerate(bm25_results["hits"]["hits"])}
        
        # 2. Vector search
        query_embedding = self.embeddings.embed_query(query)
        vector_query = {
            "size": top_k * 2,
            "query": {
                "knn": {
                    "embedding": {"vector": query_embedding, "k": top_k * 2}
                }
            }
        }
        vector_results = self.os_client.search(index="docs", body=vector_query)
        vector_docs = {hit["_id"]: (idx + 1, hit) for idx, hit in enumerate(vector_results["hits"]["hits"])}
        
        # 3. RRF fusion
        all_doc_ids = set(bm25_docs.keys()) | set(vector_docs.keys())
        rrf_scores = {}
        
        for doc_id in all_doc_ids:
            bm25_rank = bm25_docs.get(doc_id, (top_k * 2 + 1,))[0]
            vector_rank = vector_docs.get(doc_id, (top_k * 2 + 1,))[0]
            
            rrf_scores[doc_id] = (
                1.0 / (60 + bm25_rank) +
                1.0 / (60 + vector_rank)
            )
        
        # 4. Rank by RRF score
        ranked = sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
        
        # 5. Return top K with source documents
        results = []
        for doc_id, score in ranked[:top_k]:
            if doc_id in bm25_docs:
                doc = bm25_docs[doc_id][1]
            else:
                doc = vector_docs[doc_id][1]
            
            results.append({
                "id": doc_id,
                "score": score,
                "text": doc["_source"]["text"],
                "source": doc["_source"]["source"]
            })
        
        return results

# Usage
rag = HybridSearchRag()
results = rag.hybrid_search("What was AAPL's Q3 guidance vs actual results?")

for result in results:
    print(f"Score: {result['score']:.4f}, Text: {result['text'][:100]}...")
```

#### 7. Tuning Hybrid Search for Financial Queries

```python
# Financial queries need boosting on:
# - Ticker symbols (AAPL, MSFT, SPY)
# - Financial terms (earnings, guidance, revenue)
# - Time periods (Q3, 2024, FY2024)

financial_hybrid_query = {
    "size": 10,
    "query": {
        "bool": {
            "should": [
                # BM25 with field boosts
                {
                    "multi_match": {
                        "query": "AAPL Q3 guidance",
                        "fields": [
                            "ticker^5",           # Boost ticker matches 5x
                            "text^2",             # Boost main text 2x
                            "section"             # Include section heading
                        ],
                        "operator": "AND"
                    }
                },
                # Vector search
                {
                    "knn": {
                        "embedding": {"vector": query_embedding, "k": 10}
                    }
                }
            ]
        }
    }
}
```

#### Key Terms for Day 31

| Term | What It Means |
| :--- | :--- |
| **BM25** | Best Matching 25. A sparse-vector ranking function based on TF-IDF, perfect for exact keyword and phrase matches. |
| **Dense Vector Search** | Search based on semantic embeddings (e.g., Amazon Titan, Cohere), capturing underlying context rather than exact vocabulary. |
| **Reciprocal Rank Fusion (RRF)** | A mathematical algorithm that combines multiple ranked lists into a single, cohesive ranking without needing to normalize the underlying raw scores. |
| **TF-IDF** | Term Frequency-Inverse Document Frequency. The foundation of keyword search; it heavily weights words that appear frequently in a specific document but rarely across the broader corpus. |

### Summary

In complex or highly regulated architectural environments, pure semantic search is rarely enough. If a user searches for exact terminology—whether it's a specific market data product from LSEG, a rigid Basel III compliance metric like the LCR, or a unique API parameter—they need the pinpoint precision that BM25 provides. Hybrid search bridges this gap beautifully. By running keyword and semantic searches in parallel and fusing the results with RRF, you build a retrieval engine that understands both the *meaning* of the question and the *exact vocabulary* required to answer it.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 30](./Day-30-Vector-database.md) | [Day 32 →](./Day-32-RAG-quality.md)
