# Day 36 — Memory Architecture: The 4 Types an AI Agent Needs

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Memory types are like human cognition. Working: what you're thinking now. Episodic: past events. Semantic: facts you know. Procedural: how to do things. Agents need all 4."*

---


### Memory Architecture for AI Agents

Agents require integrated memory systems. Here's how to build them.

#### 1. The Four Memory Types

```
Working Memory (Context Window)
├── Current conversation
├── Recent task state
└── Max size: 128K tokens

Episodic Memory (Conversation History)
├── Past user interactions
├── Order history
└── Stored in: DynamoDB, PostgreSQL

Semantic Memory (Vector Store)
├── Facts, entities, relationships
├── Market data, company info
└── Stored in: Vector DB (Pinecone, OpenSearch)

Procedural Memory (Tools)
├── APIs, functions, skills
├── How to execute trades, query data
└── Stored in: Tool definitions (JSON)
```

#### 2. Working Memory: Context Window Management

```python
from langchain.memory import ConversationBufferWindowMemory

# Working memory: keep last 5 interactions
working_memory = ConversationBufferWindowMemory(
    k=5,  # Keep 5 exchanges
    memory_key="history",
    return_messages=True
)

# As conversation grows beyond context limit,
# old exchanges are pushed to episodic memory
```

#### 3. Episodic Memory: Conversation History

```python
import boto3
from datetime import datetime

class EpisodicMemory:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table('agent-episodes')
    
    def store_episode(self, user_id, interaction):
        """Store conversation turn"""
        self.table.put_item(Item={
            'user_id': user_id,
            'timestamp': datetime.now().isoformat(),
            'query': interaction['query'],
            'response': interaction['response'],
            'action_taken': interaction.get('action'),
            'ttl': int(datetime.now().timestamp()) + (90 * 86400)  # 90 days
        })
    
    def retrieve_episodes(self, user_id, limit=10):
        """Get past interactions"""
        response = self.table.query(
            KeyConditionExpression='user_id = :uid',
            ExpressionAttributeValues={':uid': user_id},
            ScanIndexForward=False,  # Newest first
            Limit=limit
        )
        return response['Items']

# Usage
episodic = EpisodicMemory()
episodic.store_episode('user123', {
    'query': 'Buy 100 AAPL',
    'response': 'Executed',
    'action': 'trade_executed'
})

past_episodes = episodic.retrieve_episodes('user123')
```

#### 4. Semantic Memory: Vector Store of Facts

```python
from langchain.embeddings import BedrockEmbeddings
import boto3

class SemanticMemory:
    def __init__(self):
        self.bedrock = boto3.client('bedrock-runtime')
        self.embeddings = BedrockEmbeddings(client=self.bedrock)
        self.vector_store = None  # Pinecone, OpenSearch, etc.
    
    def store_fact(self, fact, entity_type, metadata):
        """Store semantic facts"""
        embedding = self.embeddings.embed_query(fact)
        self.vector_store.upsert({
            'values': embedding,
            'metadata': {
                'fact': fact,
                'entity_type': entity_type,  # 'company', 'metric', 'regulation'
                **metadata
            }
        })
    
    def recall_facts(self, query, entity_type=None, top_k=5):
        """Retrieve relevant facts"""
        query_embedding = self.embeddings.embed_query(query)
        
        results = self.vector_store.query(
            vector=query_embedding,
            top_k=top_k,
            filter={'entity_type': entity_type} if entity_type else None
        )
        
        return [r['metadata'] for r in results]

# Usage
semantic = SemanticMemory()

# Store company facts
semantic.store_fact(
    "Apple Inc. trades on NASDAQ under ticker AAPL",
    entity_type='company',
    metadata={'ticker': 'AAPL', 'exchange': 'NASDAQ'}
)

# Recall when needed
facts = semantic.recall_facts("What exchange does AAPL trade on?")
```

#### 5. Procedural Memory: Tools and Skills

```python
class ProceduralMemory:
    """Registry of available tools/skills"""
    
    tools = {
        'get_stock_price': {
            'description': 'Get current stock price',
            'params': {'ticker': str},
            'function': lambda ticker: get_price_api(ticker)
        },
        'execute_trade': {
            'description': 'Execute a trade order',
            'params': {'ticker': str, 'quantity': int, 'side': str},
            'function': lambda ticker, qty, side: trading_api.execute(ticker, qty, side)
        },
        'query_market_data': {
            'description': 'Query market data for analysis',
            'params': {'query': str},
            'function': lambda query: semantic_memory.recall_facts(query, entity_type='market')
        }
    }
    
    @classmethod
    def get_available_tools(cls):
        """Return tool descriptions for agent"""
        return {
            name: tool['description']
            for name, tool in cls.tools.items()
        }
    
    @classmethod
    def execute_tool(cls, tool_name, params):
        """Execute a tool"""
        tool = cls.tools[tool_name]
        return tool['function'](**params)

# Usage: Agent calls tools
price = ProceduralMemory.execute_tool('get_stock_price', {'ticker': 'AAPL'})
```

#### 6. Integrated Agent with All 4 Memory Types

```python
class Agent:
    def __init__(self):
        self.working = ConversationBufferWindowMemory(k=5)
        self.episodic = EpisodicMemory()
        self.semantic = SemanticMemory()
        self.procedural = ProceduralMemory()
    
    async def process_query(self, user_id, query):
        """Process user query using all memory types"""
        
        # 1. Check working memory (recent context)
        recent = self.working.load_memory_variables({})
        
        # 2. Retrieve episodic (past interactions)
        past = self.episodic.retrieve_episodes(user_id, limit=5)
        
        # 3. Retrieve semantic (relevant facts)
        facts = self.semantic.recall_facts(query, top_k=5)
        
        # 4. List available tools (procedural)
        tools = self.procedural.get_available_tools()
        
        # 5. Build augmented prompt
        prompt = f"""
        User history: {[e['query'] for e in past]}
        Current conversation: {recent}
        Relevant facts: {facts}
        Available tools: {tools}
        
        User query: {query}
        
        Respond thoughtfully, using relevant memories and available tools.
        """
        
        # 6. Generate response
        response = llm.invoke(prompt)
        
        # 7. Store interaction in episodic memory
        self.episodic.store_episode(user_id, {
            'query': query,
            'response': response,
        })
        
        return response

# Usage
agent = Agent()
response = await agent.process_query('user123', 'What was my last trade?')
```

#### Key Terms for Day 36

| Term | What It Means |
|------|---------------|
| **Working Memory** | The agent's active context window — what it's currently "thinking about." Bounded by token limits (typically 8K–200K tokens). |
| **Episodic Memory** | A record of past interactions with a specific user. Stored in DynamoDB or PostgreSQL; retrieved by user ID and recency. |
| **Semantic Memory** | Long-term factual knowledge — company info, market data, regulations — stored as vector embeddings in a vector store. |
| **Procedural Memory** | The agent's toolkit: API definitions, tool schemas, and function signatures that tell it *how* to act in the world. |
| **Context Window** | The total token budget available to the LLM at inference time — the hard ceiling for all four memory types combined. |
| **Memory Eviction** | The process of removing older items from working memory when the context window is full; evicted items may be compressed or moved to episodic storage. |
| **Vector Store** | A database optimised for storing and searching high-dimensional embedding vectors; the backbone of semantic memory. |
| **Embedding** | A numerical vector representation of text that captures semantic meaning, enabling similarity-based retrieval across the semantic memory store. |

---

## What's Next

**Day 37 — Short-Term Memory: Conversation Buffer Strategies**

Now that you understand the four memory types at an architectural level, Day 37 goes deep on the layer agents work with most: short-term, in-session memory. You will compare Full History, Sliding Window, Summary Buffer, and Token-Compressed strategies — and learn which to choose based on conversation length, latency budget, and how much context fidelity the task demands.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 35](./Day-35-RAG-Financial.md) | [Day 37 →](./Day-37-Short-term-memory.md)
