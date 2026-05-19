# Day 29 — Embedding Models: Choosing the Right One

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Embeddings are like a fingerprint system for text. Same fingerprint = same meaning. Different fingerprint = different meaning. But some fingerprint systems are better than others."*

---


### Embedding Models: The Semantic Foundation of RAG

Your embedding model is *the* decision point for RAG quality. This deep dive explains why and how to choose.

#### 1. What Are Embeddings?

An embedding is a numerical representation of text. A sentence becomes a list of numbers (typically 384-3072 numbers, depending on model).

```python
from langchain.embeddings import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-large")

# Convert text to vector
text = "AAPL reported revenue of $93.7B"
vector = embeddings.embed_query(text)

print(len(vector))  # 3072 numbers
print(vector[:5])   # [0.234, -0.891, 0.123, ...]
```

**The key property:** Similar texts have similar vectors.

```python
text1 = "Apple's Q3 earnings beat analyst expectations"
text2 = "AAPL's Q3 results exceeded guidance"
text3 = "The weather was nice yesterday"

vec1 = embeddings.embed_query(text1)
vec2 = embeddings.embed_query(text2)
vec3 = embeddings.embed_query(text3)

# Cosine similarity
from sklearn.metrics.pairwise import cosine_similarity
similarity_1_2 = cosine_similarity([vec1], [vec2])[0][0]  # High (0.92)
similarity_1_3 = cosine_similarity([vec1], [vec3])[0][0]  # Low (0.15)
```

This is how RAG works: query and document embeddings are compared. High similarity = relevant.

#### 2. Embedding Model Landscape

**2.1 OpenAI text-embedding-3-large**

```python
from langchain.embeddings import OpenAIEmbeddings
import os

os.environ["OPENAI_API_KEY"] = "sk-..."

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-large",
    dimensions=3072
)

# Batch embed documents
docs = ["AAPL revenue $93.7B", "MSFT revenue $62.3B"]
vectors = embeddings.embed_documents(docs)
```

**Strengths:** Best-in-class semantic understanding, diverse training data, excellent English.
**Weaknesses:** Expensive ($0.13/1M), API dependency, rate limits.
**Best for:** Premium accuracy, non-financial domains.

**2.2 Amazon Titan Embeddings v2**

```python
import boto3
from langchain.embeddings import BedrockEmbeddings

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
embeddings = BedrockEmbeddings(
    client=bedrock,
    model_id="amazon.titan-embed-text-v2:0"
)

vectors = embeddings.embed_documents(docs)
```

**Strengths:** 150x cheaper, sub-100ms latency, AWS-native.
**Weaknesses:** Slightly lower quality, AWS-only.
**Best for:** AWS-native systems, financial applications.

**2.3 Cohere Embed v3**

```python
from langchain.embeddings import CohereEmbeddings

embeddings = CohereEmbeddings(
    model="embed-english-v3.0",
    cohere_api_key="COHERE_API_KEY"
)

vectors = embeddings.embed_documents(docs)
```

**Strengths:** Good quality, fast, multilingual support.
**Weaknesses:** Mid-tier pricing.
**Best for:** Multilingual RAG, balanced cost/quality.

**2.4 BGE (open-source)**

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("BAAI/bge-large-en-v1.5")
vectors = model.encode(docs, convert_to_tensor=True)
```

**Strengths:** Free, no vendor lock-in, good quality.
**Weaknesses:** Self-hosted (GPU required), slower.
**Best for:** High-volume, cost-sensitive, avoiding vendor lock-in.

#### 3. Cost vs Quality Matrix

| Model | Cost/1M | Quality | Latency | Best For |
|
---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 28](./Day-28-Local-Security-Model.md) | [Day 30 →](./Day-30-Your-First-MCP-Server.md)
