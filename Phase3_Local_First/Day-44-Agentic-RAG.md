# Day 44 — Agentic RAG: When the Agent Decides What to Retrieve

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Passive RAG gives the LLM one answer to work with. Agentic RAG gives the LLM the ability to ask its own questions."*

---

## Explanation of Day Topic

Day 43 solved the context window problem: a sliding window keeps recent conversation history manageable by evicting old turns to DynamoDB and pinning critical messages so they're never lost.

Day 44 addresses the other half of the context assembly problem: the retrieval step itself. Most RAG systems are passive — they take the user's input, run one vector search, inject the top-k results, and call the LLM. This works for simple questions. It fails for anything that requires multi-entity comparison, sequential reasoning, or awareness of whether retrieval is even necessary.

Agentic RAG replaces the fixed retrieval step with a reasoning step. The LLM decides: Is retrieval needed at all? What should I actually search for? How many separate searches does this question require? Does the answer to search 1 change what I should search for in search 2?

This day covers four patterns with full Python implementations: self-query reformulation, query decomposition for parallel multi-retrieval, multi-hop chaining, and conditional retrieval that skips the vector DB when the context window already has the answer. It also covers AWS Bedrock Agents with Knowledge Base associations, where AWS manages the agentic retrieval loop.

---

### Pattern 1 — Self-Query: The LLM Reformulates Before Retrieving

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
MODEL_ID = 'anthropic.claude-3-5-sonnet-20241022-v2:0'


def reformulate_query(user_query: str, domain: str = "financial") -> str:
    """Convert a natural language question into a search-optimised retrieval query."""
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 150,
        "messages": [{
            "role": "user",
            "content": (
                f"Rewrite this question as a concise search query optimised for "
                f"vector similarity search in a {domain} document database. "
                f"Output ONLY the reformulated query — no explanation.\n\n"
                f"Question: {user_query}"
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    return json.loads(response['body'].read())['content'][0]['text'].strip()


# Before: "How did Apple do last quarter?"
# After:  "Apple AAPL Q3 FY2025 quarterly revenue earnings EPS guidance"
# The reformulated query finds semantically closer documents in the vector DB
```

---

### Pattern 2 — Query Decomposition: Parallel Multi-Retrieval

```python
import os
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth

credentials = boto3.Session().get_credentials()
auth = AWS4Auth(credentials.access_key, credentials.secret_key,
                'us-east-1', 'aoss', session_token=credentials.token)

opensearch = OpenSearch(
    hosts=[{'host': os.environ['OPENSEARCH_ENDPOINT'], 'port': 443}],
    http_auth=auth, use_ssl=True, verify_certs=True,
    connection_class=RequestsHttpConnection
)
EMBED_ID = 'amazon.titan-embed-text-v2:0'


def embed(text: str) -> list[float]:
    resp = bedrock.invoke_model(
        modelId=EMBED_ID,
        body=json.dumps({'inputText': text}),
        contentType='application/json', accept='application/json'
    )
    return json.loads(resp['body'].read())['embedding']


def vector_search(query: str, top_k: int = 3) -> list[dict]:
    embedding = embed(query)
    result = opensearch.search(
        index='financial-docs',
        body={
            "size": top_k,
            "query": {"knn": {"embedding": {"vector": embedding, "k": top_k}}},
            "_source": ["text", "source", "ticker", "date"]
        }
    )
    return [hit['_source'] for hit in result['hits']['hits']]


def decompose_query(complex_query: str) -> list[str]:
    """Ask the LLM to decompose a multi-part question into atomic sub-queries."""
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 300,
        "messages": [{
            "role": "user",
            "content": (
                f"Break this question into 2-4 atomic sub-queries, each answerable "
                f"by a single vector search. Return a JSON array of strings only.\n\n"
                f"Question: {complex_query}"
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    raw = json.loads(response['body'].read())['content'][0]['text'].strip()
    if raw.startswith("```"):
        raw = raw.split("```")[1].lstrip("json").strip()
    return json.loads(raw)


def multi_retrieval_rag(complex_query: str) -> str:
    """
    Decompose → parallel retrieve → synthesise.
    Handles: "Compare AAPL vs MSFT earnings AND ESG scores"
    """
    sub_queries = decompose_query(complex_query)
    print(f"Decomposed into {len(sub_queries)} sub-queries:")
    for i, sq in enumerate(sub_queries, 1):
        print(f"  {i}. {sq}")

    # Run all retrievals
    all_docs: dict[str, list[dict]] = {}
    for sq in sub_queries:
        docs = vector_search(sq)
        all_docs[sq] = docs
        print(f"  → {len(docs)} docs retrieved")

    # Assemble context block
    context_block = ""
    for sq, docs in all_docs.items():
        context_block += f"\n### {sq}\n"
        for doc in docs:
            context_block += f"- [{doc.get('ticker','?')} {doc.get('date','')}]: {doc.get('text','')[:250]}\n"

    # Synthesise
    synthesis = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1200,
        "system": (
            "You are a financial analyst. Answer using ONLY the retrieved documents. "
            "Cite your sources. Never hallucinate numbers."
        ),
        "messages": [{
            "role": "user",
            "content": f"Documents:\n{context_block}\n\nQuestion: {complex_query}"
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=synthesis)
    return json.loads(response['body'].read())['content'][0]['text']


# "Compare AAPL and MSFT Q3 revenue growth and ESG scores"
# → 3 sub-queries → 3 retrievals → 1 grounded comparative answer
answer = multi_retrieval_rag(
    "Compare AAPL and MSFT Q3 2025 revenue growth, and which has better ESG credentials?"
)
print(answer[:300])
```

---

### Pattern 3 — Multi-Hop: Each Retrieval Informs the Next

```python
from datetime import datetime


def multi_hop_retrieval(initial_query: str, max_hops: int = 3) -> dict:
    """
    Chain retrievals: answer to hop N refines query for hop N+1.
    
    Example:
    Hop 1: "TSLA Q3 2025 earnings" → discovers revenue missed by 3%
    Hop 2: "TSLA missed earnings Q3 2025 analyst reaction" → discovers downgrades
    Hop 3: "TSLA stock price September October 2025" → quantifies market reaction
    """
    hops = []
    current_query = initial_query
    accumulated_context = ""

    for hop in range(1, max_hops + 1):
        print(f"\n[Hop {hop}] {current_query}")
        docs = vector_search(current_query)
        docs_text = "\n".join(f"- {d.get('text','')[:200]}" for d in docs)
        accumulated_context += f"\n[Hop {hop}: {current_query}]\n{docs_text}"
        hops.append({"hop": hop, "query": current_query, "doc_count": len(docs)})

        # Should we stop or continue?
        check = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 100,
            "messages": [{
                "role": "user",
                "content": (
                    f"Original question: {initial_query}\n"
                    f"Retrieved so far:\n{accumulated_context[-800:]}\n\n"
                    f"Can you answer the original question now? "
                    f"Reply 'DONE' or 'NEXT: <next search query>'."
                )
            }]
        })
        check_response = bedrock.invoke_model(modelId=MODEL_ID, body=check)
        decision = json.loads(check_response['body'].read())['content'][0]['text'].strip()

        if decision.startswith("DONE") or hop == max_hops:
            break
        elif decision.startswith("NEXT:"):
            current_query = decision.replace("NEXT:", "").strip()

    # Final answer
    final = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 800,
        "system": "Answer from retrieved context only. Cite which hop provided which fact.",
        "messages": [{"role": "user",
                      "content": f"Context:\n{accumulated_context}\n\nQuestion: {initial_query}"}]
    })
    final_response = bedrock.invoke_model(modelId=MODEL_ID, body=final)
    answer = json.loads(final_response['body'].read())['content'][0]['text']

    return {"hops_taken": len(hops), "hop_log": hops, "answer": answer}


result = multi_hop_retrieval("Did TSLA beat Q3 2025 earnings and how did the stock react?")
print(f"Completed in {result['hops_taken']} hops")
print(result['answer'][:300])
```

---

### Pattern 4 — Conditional Retrieval: Skip When Unnecessary

```python
def should_retrieve(query: str, context_summary: str) -> bool:
    """Ask the LLM whether this query needs new information from the KB."""
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 20,
        "messages": [{
            "role": "user",
            "content": (
                f"Context available: {context_summary}\n\n"
                f"User query: {query}\n\n"
                f"Does answering require retrieving new documents? "
                f"Reply RETRIEVE or SKIP."
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    return "RETRIEVE" in json.loads(response['body'].read())['content'][0]['text']


class ConditionalAgent:
    """Only retrieves when the LLM determines retrieval adds value."""

    def __init__(self):
        self.context_summary = ""
        self.cache: dict[str, str] = {}

    def answer(self, query: str) -> dict:
        if query in self.cache:
            return {"answer": self.cache[query], "retrieved": False, "source": "cache"}

        needs_retrieval = should_retrieve(query, self.context_summary)
        retrieved_text = ""

        if needs_retrieval:
            docs = vector_search(query)
            retrieved_text = "\n".join(d.get("text", "")[:300] for d in docs)

        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 600,
            "system": "Answer concisely. Use retrieved context when provided.",
            "messages": [{
                "role": "user",
                "content": (
                    f"{'Retrieved:\n' + retrieved_text + chr(10) if retrieved_text else ''}"
                    f"Context: {self.context_summary}\n\nQuestion: {query}"
                )
            }]
        })
        response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
        answer = json.loads(response['body'].read())['content'][0]['text']

        self.cache[query] = answer
        self.context_summary = answer[:300]
        return {"answer": answer, "retrieved": needs_retrieval, "source": "generated"}
```

---

### Bedrock Agents with Knowledge Bases (Managed Agentic RAG)

```python
bedrock_agent = boto3.client('bedrock-agent', region_name='us-east-1')
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')


def create_agent_with_kb(kb_id: str) -> str:
    agent = bedrock_agent.create_agent(
        agentName='financial-analyst',
        agentResourceRoleArn='arn:aws:iam::ACCOUNT:role/BedrockAgentRole',
        foundationModel='anthropic.claude-3-5-sonnet-20241022-v2:0',
        instruction=(
            "You are a senior financial analyst. For all factual financial queries, "
            "retrieve from the knowledge base. For comparative questions, retrieve "
            "for each entity separately. Always cite your sources."
        )
    )
    agent_id = agent['agent']['agentId']

    bedrock_agent.associate_agent_knowledge_base(
        agentId=agent_id,
        agentVersion='DRAFT',
        knowledgeBaseId=kb_id,
        description='Financial documents: 10-K, earnings transcripts, ESG reports',
        knowledgeBaseState='ENABLED'
    )
    bedrock_agent.prepare_agent(agentId=agent_id)
    return agent_id


def query_agent(agent_id: str, alias_id: str, user_query: str) -> str:
    """Bedrock Agent handles decomposition, retrieval, and synthesis internally."""
    response = bedrock_agent_runtime.invoke_agent(
        agentId=agent_id,
        agentAliasId=alias_id,
        sessionId=f"s_{datetime.now().timestamp():.0f}",
        inputText=user_query,
        enableTrace=True
    )
    result = ""
    for event in response.get('completion', []):
        if 'chunk' in event:
            result += event['chunk']['bytes'].decode('utf-8')
    return result
```

---

### When to Use Each Pattern

| Pattern | Best For | Latency Cost | Complexity |
|---------|----------|-------------|------------|
| **Self-Query** | Ambiguous or jargon-heavy queries | +100ms | Low |
| **Decomposition** | Multi-entity comparative questions | +400-800ms | Medium |
| **Multi-Hop** | Sequential reasoning chains | +1-3s | High |
| **Conditional** | Mixed workloads, latency-sensitive | Saves latency | Low |
| **Bedrock Agents** | Production — managed, scalable | Variable | Low (managed) |

---

### Key Terms for Day 44

| Term | Meaning |
|------|---------|
| **Passive RAG** | Fixed pipeline: one user query → one vector search → one generation. No agent reasoning. |
| **Agentic RAG** | The LLM reasons about retrieval strategy before executing it — when, what, how many hops. |
| **Self-Query** | LLM reformulates the user's question into a search-optimised query before hitting the vector DB. |
| **Query Decomposition** | Breaking a complex question into atomic sub-queries, each answered by a separate retrieval call. |
| **Multi-Hop Retrieval** | Chained retrievals where each result informs the next query — for sequential or dependent reasoning. |
| **Conditional Retrieval** | The LLM decides per-query whether to retrieve at all — skips the vector DB when context is sufficient. |
| **Knowledge Base (Bedrock)** | AWS managed document store with k-NN vector search. Bedrock Agents query it autonomously. |

---

## What's Next

**Day 45 — Phase 3 Complete: The Dynamic Context Playbook**

Days 26–44 built the full dynamic context stack: transport layer, RAG pipeline, four-type memory architecture, AWS infrastructure, context window management, and agentic retrieval. Day 45 is the recap: a decision tree for choosing your RAG architecture, a production cost model, the eight key learnings from Phase 3, and a preview of Phase 4 — Model Context Protocol (MCP) — where LLMs connect to live data sources via a standard protocol instead of bespoke pipelines.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #AgenticRAG #AWSCommunityBuilder*

[← Day 43](./Day-43-Context-Window.md)
