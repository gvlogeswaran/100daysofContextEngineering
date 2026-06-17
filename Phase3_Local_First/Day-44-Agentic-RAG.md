# Day 44 — Agentic RAG: When the Agent Decides What to Retrieve
**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory**
**Topic:** Moving from passive to active retrieval. Self-query RAG. Multi-hop retrieval. Query decomposition. Bedrock Agents with Knowledge Base associations.

---

## 🎨 SLIDE 1 — The Hook (Canva)

**Headline:**
> Passive RAG is a Search Engine. Agentic RAG is a Researcher.

**Sub-headline:**
> Passive: user query → one vector search → generate. Agentic: agent plans, decomposes, runs multiple targeted searches, synthesises. The difference between a librarian who hands you one book and one who pulls five, cross-references them, and writes you a summary.

**Visual Cue:**
- **Background:** Dark mode (#0a0a0a)
- **Left panel:** "Passive RAG" — single arrow: query → vector DB → generate
- **Right panel:** "Agentic RAG" — tree: query → decompose → 3 sub-queries → 3 vector searches → synthesise → generate
- **Message:** "One retrieval vs. a retrieval strategy"

---

## 🎨 SLIDE 2 — The AHA Moment (Canva)

**Analogy:**
> Passive RAG is like Googling one query and using the first result. Agentic RAG is like a skilled analyst who reads the question, breaks it into sub-questions, searches for each independently, and then synthesises a coherent answer. The analyst doesn't just retrieve — they reason about what to retrieve.

**The Key Insight:**
> The key shift: the LLM is no longer a consumer of retrieval — it's a planner. It decides WHEN retrieval is necessary (maybe the context window already has the answer). It decides WHAT to query (not the raw user input, but a reformulated search-optimised query). It decides HOW MANY retrievals to run (one for simple, many for comparative or multi-hop questions). This is the difference that separates production agents from demos.

**Visual Cue:**
- **Decision tree:** "Is retrieval needed?" → if yes → "How many hops?" → "Simple" or "Multi-hop"
- **Comparison:** Passive RAG latency vs Agentic RAG latency (acknowledge the tradeoff)
- **Message:** "More compute. More accuracy. Production-grade."

---

## ✍️ LINKEDIN POST

📅 **Best Posting Time:** Tue/Thu 9-11am for reach

**Day 44 of 100 — Agentic RAG: When the Agent Decides What to Retrieve**

Most RAG systems are passive: user query → vector search → generate. 📚

That works for simple questions. It breaks down for anything real.

**Try this query:** "Compare AAPL's Q3 revenue growth with MSFT's and tell me which has better ESG credentials."

Passive RAG runs one search. Gets partial results. Generates a confused answer.

**Agentic RAG does this instead:**

1️⃣ **Detect complexity** — LLM sees this is a multi-entity, multi-hop question

2️⃣ **Decompose** — Breaks it into 3 sub-queries:
  - "AAPL Q3 revenue and growth rate"
  - "MSFT Q3 revenue and growth rate"
  - "ESG scores AAPL vs MSFT comparison"

3️⃣ **Run parallel retrievals** — 3 targeted vector searches

4️⃣ **Synthesise** — LLM reads all 3 result sets and generates a grounded comparative answer

**Result:** Not just more accurate. Citable. Every claim traces back to a retrieved source.

**The patterns:**
- **Self-query:** LLM reformulates the user question into search-optimised terms before retrieving
- **Multi-hop:** Answer to query 1 informs the query for step 2
- **Conditional retrieval:** Agent decides whether retrieval is even needed (saves latency on cached answers)

**Financial example — trade research agent:**
- Query: "Is TSLA a good buy for ESG portfolios today?"
- Agentic: retrieve TSLA price → retrieve ESG score → retrieve recent news → retrieve analyst ratings → synthesise all 4

Passive RAG: 1 retrieval, 1 guess.
Agentic RAG: 4 targeted retrievals, 1 grounded answer. 🎯

What retrieval pattern is your production RAG using? 👇

#100DaysOfContextEngineering #RAG #AgenticRAG #Agents #Bedrock #LLM #ContextEngineering #AWS #FinancialAI

---

## 📖 GITHUB — Day 44 Deep Dive

### Agentic RAG: When the Agent Decides What to Retrieve

Passive RAG is a fixed pipeline: user input → one retrieval → generate. Agentic RAG replaces that fixed pipeline with a reasoning step: the agent plans its retrieval, executes targeted searches, and synthesises the results. This section covers the four core agentic retrieval patterns and their AWS Bedrock implementations.

---

#### 1. Pattern 1 — Self-Query RAG (Query Reformulation)

The agent rewrites the user's natural language question into search-optimised retrieval queries before hitting the vector database.

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
MODEL_ID = 'anthropic.claude-3-5-sonnet-20241022-v2:0'


def reformulate_query(user_query: str, domain: str = "financial") -> str:
    """
    Use the LLM to convert a natural language question into
    a search-optimised retrieval query.
    """
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 200,
        "messages": [{
            "role": "user",
            "content": (
                f"Convert this user question into a concise, search-optimised query "
                f"for a {domain} knowledge base. Return ONLY the reformulated query. "
                f"No explanation.\n\n"
                f"User question: {user_query}"
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    return json.loads(response['body'].read())['content'][0]['text'].strip()


# Example
user_question = "How did Apple do in their most recent quarterly earnings?"
search_query = reformulate_query(user_question)
print(f"Original: {user_question}")
print(f"Reformulated: {search_query}")
# Reformulated: "Apple AAPL Q3 2025 quarterly earnings revenue EPS guidance"
# Much better for vector similarity search
```

---

#### 2. Pattern 2 — Query Decomposition (Multi-Retrieval)

For complex multi-part questions, the agent decomposes the query into sub-queries, runs parallel retrievals, then synthesises all results.

```python
import asyncio
from typing import Any
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import os

# ── OpenSearch Client ──────────────────────────────────────────────────────────
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


def decompose_query(complex_query: str) -> list[str]:
    """Ask the LLM to break a complex query into atomic sub-queries."""
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 400,
        "messages": [{
            "role": "user",
            "content": (
                f"Break this complex question into 2-4 atomic sub-queries "
                f"that can each be answered with a single vector search. "
                f"Return a JSON array of strings. No other text.\n\n"
                f"Complex question: {complex_query}"
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    raw = json.loads(response['body'].read())['content'][0]['text'].strip()
    # Strip markdown fences if present
    if raw.startswith("```"):
        raw = raw.split("```")[1].lstrip("json").strip()
    return json.loads(raw)


def vector_search(query: str, index: str = "financial-docs", top_k: int = 3) -> list[dict]:
    """Run a k-NN vector search against OpenSearch Serverless."""
    embedding = embed(query)
    result = opensearch.search(
        index=index,
        body={
            "size": top_k,
            "query": {
                "knn": {
                    "embedding": {"vector": embedding, "k": top_k}
                }
            },
            "_source": ["text", "source", "date", "ticker"]
        }
    )
    return [hit['_source'] for hit in result['hits']['hits']]


def multi_retrieval_rag(complex_query: str) -> dict[str, Any]:
    """
    Agentic multi-retrieval:
    1. Decompose complex query into sub-queries
    2. Run parallel vector searches
    3. Synthesise all results into one grounded answer
    """
    # Step 1: Decompose
    sub_queries = decompose_query(complex_query)
    print(f"Decomposed into {len(sub_queries)} sub-queries:")
    for i, sq in enumerate(sub_queries, 1):
        print(f"  {i}. {sq}")

    # Step 2: Parallel retrievals
    all_results = {}
    for sq in sub_queries:
        results = vector_search(sq)
        all_results[sq] = results
        print(f"Retrieved {len(results)} docs for: {sq[:50]}...")

    # Step 3: Synthesise
    context_block = ""
    for sq, docs in all_results.items():
        context_block += f"\n### Sub-query: {sq}\n"
        for doc in docs:
            context_block += f"- [{doc.get('ticker', 'N/A')} | {doc.get('date', 'N/A')}]: {doc.get('text', '')[:300]}\n"

    synthesis_body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1500,
        "system": (
            "You are a financial analyst. Answer the user's question using ONLY "
            "the provided retrieved documents. Cite your sources. "
            "If the documents don't contain enough information, say so."
        ),
        "messages": [
            {"role": "user", "content": f"Documents:\n{context_block}\n\nQuestion: {complex_query}"}
        ]
    })
    synthesis_response = bedrock.invoke_model(modelId=MODEL_ID, body=synthesis_body)
    answer = json.loads(synthesis_response['body'].read())['content'][0]['text']

    return {
        "query": complex_query,
        "sub_queries": sub_queries,
        "retrieved_docs": {sq: len(docs) for sq, docs in all_results.items()},
        "answer": answer
    }


# Example
result = multi_retrieval_rag(
    "Compare AAPL and MSFT Q3 2025 revenue growth rates, and tell me which has better ESG credentials."
)
print(f"\nAnswer: {result['answer'][:300]}...")
```

---

#### 3. Pattern 3 — Multi-Hop Retrieval

The answer to the first retrieval informs the next query. Each hop narrows the search based on what was learned.

```python
def multi_hop_retrieval(initial_query: str, max_hops: int = 3) -> dict[str, Any]:
    """
    Multi-hop retrieval: each hop uses the previous result to refine the next query.
    
    Example chain:
    Hop 1: "TSLA Q3 2025 earnings" → discovers revenue was $25.18B
    Hop 2: "TSLA revenue $25.18B analyst expectations 2025" → discovers missed by 3%
    Hop 3: "TSLA stock reaction missed earnings guidance 2025" → market reaction
    """
    hops = []
    current_query = initial_query
    accumulated_context = ""

    for hop_num in range(1, max_hops + 1):
        print(f"\n[Hop {hop_num}] Query: {current_query}")

        # Retrieve for current query
        docs = vector_search(current_query)
        docs_text = "\n".join(
            f"- {d.get('text', '')[:200]}" for d in docs
        )
        accumulated_context += f"\n[Hop {hop_num} — {current_query}]:\n{docs_text}"

        hops.append({"query": current_query, "docs": docs})

        # Ask the LLM: do we have enough, or do we need another hop?
        check_body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 300,
            "messages": [{
                "role": "user",
                "content": (
                    f"Original question: {initial_query}\n"
                    f"Retrieved so far:\n{accumulated_context}\n\n"
                    f"Can you now answer the original question with confidence? "
                    f"If YES, output 'DONE'. "
                    f"If NO, output 'NEXT: <the next search query needed>'."
                )
            }]
        })
        check_response = bedrock.invoke_model(modelId=MODEL_ID, body=check_body)
        decision = json.loads(check_response['body'].read())['content'][0]['text'].strip()

        if decision.startswith("DONE") or hop_num == max_hops:
            break
        elif decision.startswith("NEXT:"):
            current_query = decision.replace("NEXT:", "").strip()

    # Final synthesis
    final_body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1000,
        "system": "Answer using only the retrieved context. Cite hop numbers.",
        "messages": [{
            "role": "user",
            "content": f"Context:\n{accumulated_context}\n\nQuestion: {initial_query}"
        }]
    })
    final_response = bedrock.invoke_model(modelId=MODEL_ID, body=final_body)
    answer = json.loads(final_response['body'].read())['content'][0]['text']

    return {"initial_query": initial_query, "hops": len(hops), "hop_log": hops, "answer": answer}


# Example
result = multi_hop_retrieval("Did TSLA beat earnings in Q3 2025 and how did the market react?")
print(f"Completed in {result['hops']} hops")
print(f"Answer: {result['answer'][:300]}...")
```

---

#### 4. Pattern 4 — Conditional Retrieval (Skip When Unnecessary)

The most expensive retrieval is the one you didn't need to run. This pattern decides upfront whether retrieval adds value.

```python
def should_retrieve(user_query: str, context_summary: str) -> bool:
    """
    Ask the LLM: does this query require retrieval,
    or can it be answered from the current context?
    """
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 50,
        "messages": [{
            "role": "user",
            "content": (
                f"Current context summary: {context_summary}\n\n"
                f"User query: {user_query}\n\n"
                f"Does answering this query require retrieving additional documents "
                f"from the knowledge base, or can it be answered from context? "
                f"Reply with RETRIEVE or NO_RETRIEVE only."
            )
        }]
    })
    response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
    decision = json.loads(response['body'].read())['content'][0]['text'].strip()
    return "RETRIEVE" in decision


class ConditionalAgent:
    """Agent that only retrieves when the LLM determines it's necessary."""

    def __init__(self):
        self.context_summary = ""
        self.cache: dict[str, str] = {}  # Simple query cache

    def answer(self, query: str) -> dict[str, Any]:
        # Check cache first
        if query in self.cache:
            return {"answer": self.cache[query], "retrieved": False, "source": "cache"}

        # Decide whether retrieval is needed
        needs_retrieval = should_retrieve(query, self.context_summary)
        print(f"Retrieval decision for '{query[:50]}...': {'YES' % needs_retrieval}")

        retrieved_context = ""
        if needs_retrieval:
            docs = vector_search(query)
            retrieved_context = "\n".join(d.get("text", "") for d in docs)

        # Generate answer
        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 800,
            "system": "Answer concisely. Use retrieved context if provided.",
            "messages": [{
                "role": "user",
                "content": (
                    f"{'Retrieved docs:\n' + retrieved_context + chr(10) if retrieved_context else ''}"
                    f"Context so far: {self.context_summary}\n\n"
                    f"Question: {query}"
                )
            }]
        })
        response = bedrock.invoke_model(modelId=MODEL_ID, body=body)
        answer = json.loads(response['body'].read())['content'][0]['text']

        self.cache[query] = answer
        self.context_summary = f"...{answer[:200]}"  # Update context summary

        return {"answer": answer, "retrieved": needs_retrieval, "source": "generated"}
```

---

#### 5. Bedrock Agents with Knowledge Bases (Managed Agentic RAG)

AWS Bedrock Agents handle the retrieval planning automatically — you define the knowledge base and the agent decides when and what to retrieve.

```python
import boto3

bedrock_agent = boto3.client('bedrock-agent', region_name='us-east-1')
bedrock_agent_runtime = boto3.client('bedrock-agent-runtime', region_name='us-east-1')


def create_financial_agent(knowledge_base_id: str) -> str:
    """Create a Bedrock Agent with KB association — AWS manages the retrieval strategy."""
    response = bedrock_agent.create_agent(
        agentName='financial-analyst-v2',
        agentResourceRoleArn='arn:aws:iam::ACCOUNT_ID:role/BedrockAgentRole',
        foundationModel='anthropic.claude-3-5-sonnet-20241022-v2:0',
        instruction=(
            "You are a senior financial analyst. When answering questions about "
            "companies, earnings, or market data, ALWAYS retrieve from the knowledge base. "
            "For comparative questions, retrieve for EACH company separately. "
            "Cite your sources. Never guess financial data."
        )
    )
    agent_id = response['agent']['agentId']

    # Associate the knowledge base
    bedrock_agent.associate_agent_knowledge_base(
        agentId=agent_id,
        agentVersion='DRAFT',
        knowledgeBaseId=knowledge_base_id,
        description='Financial documents: 10-K filings, earnings transcripts, ESG reports',
        knowledgeBaseState='ENABLED'
    )

    # Prepare the agent (compile and validate)
    bedrock_agent.prepare_agent(agentId=agent_id)
    print(f"Created Bedrock Agent: {agent_id}")
    return agent_id


def invoke_agentic_rag(agent_id: str, alias_id: str, query: str) -> str:
    """
    Invoke the Bedrock Agent — it handles decomposition and retrieval internally.
    The agent decides how many retrievals to run and synthesises the results.
    """
    response = bedrock_agent_runtime.invoke_agent(
        agentId=agent_id,
        agentAliasId=alias_id,
        sessionId=f"session_{datetime.now().timestamp():.0f}",
        inputText=query,
        enableTrace=True  # Log retrieval decisions for debugging
    )

    # Stream the response
    full_response = ""
    trace_steps = []

    for event in response.get('completion', []):
        if 'chunk' in event:
            full_response += event['chunk']['bytes'].decode('utf-8')
        elif 'trace' in event:
            trace = event['trace'].get('trace', {})
            if 'orchestrationTrace' in trace:
                step = trace['orchestrationTrace']
                if 'rationale' in step:
                    trace_steps.append(f"[Reasoning]: {step['rationale']['text'][:100]}")
                elif 'invocationInput' in step:
                    trace_steps.append(f"[Retrieval]: {str(step['invocationInput'])[:100]}")

    print(f"\nAgent trace ({len(trace_steps)} steps):")
    for step in trace_steps:
        print(f"  {step}")

    return full_response


# Example usage (requires agent to be deployed)
# agent_id = create_financial_agent(knowledge_base_id="your-kb-id")
# answer = invoke_agentic_rag(agent_id, alias_id="TSTALIASID",
#                             query="Compare AAPL vs MSFT Q3 earnings and ESG scores")
# print(answer)
```

---

#### When to Use Each Pattern

| Pattern | Best For | Latency | Complexity |
|---------|----------|---------|------------|
| **Self-Query** | Queries with jargon or ambiguity | +100ms | Low |
| **Query Decomposition** | Multi-entity comparative questions | +500ms | Medium |
| **Multi-Hop** | Sequential reasoning ("A → then B → then C") | +1-3s | High |
| **Conditional Retrieval** | Mixed workloads (some queries don't need retrieval) | Saves latency | Low |
| **Bedrock Agents** | Production — managed, scalable, with trace | Variable | Low (managed) |

---

#### Key Terms for Day 44

| Term | What It Means |
|------|---------------|
| **Passive RAG** | A fixed pipeline: user query → one vector search → generate. No agent reasoning about retrieval. |
| **Agentic RAG** | The LLM reasons about WHEN, WHAT, and HOW to retrieve before executing retrieval. |
| **Self-Query** | The LLM reformulates the user's question into a search-optimised query before hitting the vector DB. |
| **Query Decomposition** | Breaking a complex multi-part question into atomic sub-queries, each answered by a separate retrieval. |
| **Multi-Hop Retrieval** | Each retrieval result informs the next query — chains of dependent searches for sequential reasoning. |
| **Conditional Retrieval** | The LLM decides per-query whether retrieval is needed — skips it when context window already has the answer. |
| **Knowledge Base (Bedrock)** | A managed AWS service that stores, indexes, and retrieves documents — Bedrock Agents query it automatically. |

---

## What's Next

**Day 45 — Phase 3 Complete: The Dynamic Context Playbook**

Days 26–44 have taken us from why static context fails to a full production stack: RAG pipeline, 4-type memory architecture, AWS infrastructure, and agentic retrieval. Day 45 is the recap — a decision tree for choosing your RAG architecture, a cost model, key learnings from all 20 days, and a preview of Phase 4: Model Context Protocol (MCP), where we move from building context pipelines to connecting LLMs to live data sources via a standard protocol.

---

[← Day 43](./Day-43-Context-Window.md) | [Day 45 →](./Day-45-Phase-3-Review.md)
