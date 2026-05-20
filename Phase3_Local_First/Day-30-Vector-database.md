# Day 30 — Vector Databases: The Index Behind RAG

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"A vector database without an index is like a library with no card catalog. A vector database with an index is like a perfectly organized library where you find the right book in seconds."*

---


### Vector Databases: Indexing the Semantic Web

Vector search without indexing is just linear search. Here's how to use vector databases in production.

#### 1. The Nearest Neighbor Problem

Given 1M vectors and a query vector, find the 10 most similar. Naive approach:

```python
# Naive: O(n) per query. For 1M vectors = 1M distance calculations
import numpy as np
from sklearn.metrics.pairwise import cosine_distances

all_vectors = np.random.rand(1_000_000, 1536)  # 1M vectors, 1536 dims
query_vector = np.random.rand(1, 1536)

# This is SLOW
distances = cosine_distances(query_vector, all_vectors)
top_10_indices = np.argsort(distances[0])[:10]
```

**Problem:** Linear scan doesn't scale. At 1000 QPS (queries per second), you're doing 1B distance calculations per second. That's CPU hell.

**Solution:** Index the vectors.

#### 2. HNSW: Hierarchical Navigable Small World

Most vector databases use HNSW indexing. The idea: build a graph where similar vectors are neighbors.

```python
# HNSW indexing (approximate nearest neighbor search)
from hnswlib import Index

# Create index
index = Index(space='cosine', dim=1536)
index.init_index(max_elements=1_000_000, ef_construction=200, M=16)

# Add vectors
for i, vector in enumerate(all_vectors):
    index.add_items(vector, i)

# Query: now O(log n) instead of O(n)
query_vector = np.random.rand(1, 1536)
labels, distances = index.knn_query(query_vector, k=10)

# 10 nearest neighbors found in ~100 distance calculations (not 1M)
```

**Result:** 10,000x faster for 1M vectors.

#### 3. Using Pinecone

```python
import pinecone

pinecone.init(api_key="YOUR_API_KEY", environment="us-east-1")

# Create index
pinecone.create_index(
    name="financial-docs",
    dimension=1536,
    metric="cosine",  # Similarity metric
    spec={"serverless": {"cloud": "aws", "region": "us-east-1"}}
)

index = pinecone.Index("financial-docs")

# Upsert vectors (vectors + metadata)
vectors_to_upsert = [
    {
        "id": "doc_1",
        "values": embedding_1,  # 1536-dimensional vector
        "metadata": {
            "source": "AAPL-10Q.pdf",
            "text": "AAPL reported revenue of $93.7B",
            "page": 5,
        }
    },
    {
        "id": "doc_2",
        "values": embedding_2,
        "metadata": {"source": "MSFT-10Q.pdf", "text": "MSFT revenue $62.3B", "page": 3}
    },
]

index.upsert(vectors=vectors_to_upsert)

# Query
query_embedding = embedding_model.embed_query("What was AAPL's revenue?")
results = index.query(
    vector=query_embedding,
    top_k=5,
    include_metadata=True
)

# Returns top 5 nearest vectors + metadata
for match in results['matches']:
    print(f"Similarity: {match['score']:.3f}, Text: {match['metadata']['text']}")
```

#### 4. Using AWS OpenSearch Serverless

```python
import boto3
from opensearchpy import OpenSearch, RequestsHttpConnection
from opensearchpy.connection import Connection
from requests_auth_aws4auth import AWS4Auth

# Create OpenSearch client
service = 'aoss'
credentials = boto3.Session().get_credentials()
auth = AWS4Auth(
    credentials.access_key,
    credentials.secret_key,
    'us-east-1',
    service,
    session_token=credentials.token
)

client = OpenSearch(
    hosts=[{'host': 'your-collection.us-east-1.aoss.amazonaws.com', 'port': 443}],
    http_auth=auth,
    use_ssl=True,
    verify_certs=True,
    connection_class=RequestsHttpConnection,
)

# Create index with vector field
index_body = {
    "settings": {
        "index": {
            "knn": True,
            "knn.algo_param.ef_search": 512,
        }
    },
    "mappings": {
        "properties": {
            "text": {"type": "text"},
            "embedding": {
                "type": "knn_vector",
                "dimension": 1536,
                "method": {"name": "hnsw", "space_type": "cosinesimil"},
            }
        }
    }
}

client.indices.create(index="financial-docs", body=index_body)

# Index a document
doc = {
    "text": "AAPL reported revenue of $93.7B in Q3 2024",
    "embedding": embedding_vector,
}
client.index(index="financial-docs", body=doc)

# Query
query_body = {
    "size": 5,
    "query": {
        "knn": {
            "embedding": {
                "vector": query_embedding,
                "k": 5
            }
        }
    }
}

results = client.search(index="financial-docs", body=query_body)
```

#### 5. Using pgvector (PostgreSQL)

```python
import psycopg2
from pgvector.psycopg2 import register_vector

conn = psycopg2.connect("postgresql://user:password@localhost/financial_db")
register_vector(conn)

# Create table with vector column
cur = conn.cursor()
cur.execute("""
    CREATE TABLE IF NOT EXISTS documents (
        id SERIAL PRIMARY KEY,
        text TEXT,
        embedding vector(1536),
    );
""")

# Create index (HNSW)
cur.execute("CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);")

# Insert
cur.execute(
    "INSERT INTO documents (text, embedding) VALUES (%s, %s)",
    ("AAPL revenue $93.7B", embedding_vector)
)

# Query (cosine similarity)
cur.execute("""
    SELECT id, text, embedding <=> %s AS distance
    FROM documents
    ORDER BY distance
    LIMIT 5;
""", (query_embedding,))

results = cur.fetchall()
conn.commit()
```

#### 6. Hybrid Search: Combining Semantic + Keyword

Financial queries often need both semantic AND keyword search:

```
Query: "AAPL earnings beat vs guidance"
- Semantic search: finds earnings documents
- Keyword search: finds "AAPL", "earnings", "beat", "guidance"
- Combined: documents matching BOTH semantic + keywords
```

AWS OpenSearch supports this:

```python
hybrid_query = {
    "size": 5,
    "query": {
        "bool": {
            "must": [
                {
                    "knn": {
                        "embedding": {
                            "vector": query_embedding,
                            "k": 10  # Get top 10 semantic
                        }
                    }
                },
                {
                    "match": {
                        "text": {
                            "query": "AAPL earnings beat",
                            "operator": "AND"
                        }
                    }
                }
            ]
        }
    }
}

results = client.search(index="financial-docs", body=hybrid_query)
```

#### 7. Production Vector Database Checklist

- [ ] Index created with appropriate similarity metric (cosine for embeddings)
- [ ] Vectors stored with source metadata (source, date, page)
- [ ] Batch upsert used (faster than individual inserts)
- [ ] Similarity threshold tuned (typically 0.7+ for relevant)
- [ ] Monitoring in place (query latency, recall metrics)
- [ ] Backup/replication configured
- [ ] Cost estimates validated at scale

#### Key Terms for Day 30

| Term | What It Means |
|
---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 29](./Day-29-Subprocess-Management.md) | [Day 31 →](./Day-31-Debugging-MCP-Servers.md)
