# Day 51 — MCP Resources — Read-Only Context Windows Into Your Data

**100 Days of Context Engineering | Phase 4: MCP — The Live Context Protocol** | By Logeswaran GV (AWS Community Builder)

> *"Resources turn your data into a first-class context citizen. No polling. No webhooks. Just structured data, served on demand to the model that needs it."*

---

### MCP Resources — Read-Only Context Windows Into Your Data

Resources are the read-side of MCP. They give models safe access to data without the ability to modify it. Let me show you how to design and deploy them.

#### What is a Resource?

A resource is any data that can be:
1. Identified by a URI (Uniform Resource Identifier)
2. Read by the host
3. Optionally subscribed to for updates

Resources are **not** tools. They don't execute actions. They're purely data.

Examples:
- A file on disk (`file:///home/user/data.json`)
- A database schema (`db://postgres/users/schema`)
- An API response (`https://api.example.com/me/account`)
- A real-time data stream (`mcp://market-data/AAPL/quote`)

#### Resource Types

MCP defines three resource types:

**1. Text Resources**
Plain text data. Files, JSON, SQL schemas, logs, documentation.

```json
{
  "uri": "file:///data/schema.sql",
  "name": "Database Schema",
  "description": "PostgreSQL schema for orders, users, trades",
  "mimeType": "application/sql"
}
```

**2. Blob Resources**
Binary data. Images, PDFs, archives.

```json
{
  "uri": "blob://screenshots/trade-confirmation.png",
  "name": "Trade Confirmation",
  "description": "Screenshot of order confirmation",
  "mimeType": "image/png"
}
```

**3. URI-Addressable Resources**
Data that can be fetched by URI pattern. Market data, database records, API endpoints.

```json
{
  "uri": "mcp://market-data/AAPL/quote",
  "name": "AAPL Quote",
  "description": "Real-time price, volume, bid-ask spread",
  "mimeType": "application/json"
}
```

#### Resource Subscriptions

A key feature: the model can **subscribe** to a resource. When the resource changes, the server notifies the host.

Example: Market data resource with subscriptions

```python
@server.list_resources()
async def list_resources():
    return [
        Resource(
            uri=f"mcp://market-data/{symbol}/quote",
            name=f"{symbol} Quote",
            description=f"Real-time price for {symbol}",
            mimeType="application/json"
        )
        for symbol in ["AAPL", "MSFT", "GOOGL"]
    ]

@server.read_resource()
async def read_resource(uri: str):
    symbol = uri.split("/")[-2]
    quote = get_latest_quote(symbol)
    return quote.to_json()

@server.subscribe_resource()
async def subscribe_resource(uri: str, callback):
    symbol = uri.split("/")[-2]
    
    # Stream updates for this symbol
    async for update in stream_quotes(symbol):
        callback(update.to_json())
```

When the model subscribes to `mcp://market-data/AAPL/quote`, the server streams price updates whenever they arrive. The model's context window updates in real time.

#### Resources vs RAG

This is an important distinction:

**RAG (static knowledge):**
- Embed documents once
- Search vector database
- Retrieve relevant chunks
- Augment prompt with retrieved text
- Works great for: FAQs, docs, historical logs

**MCP Resources (live data):**
- Resources change in real time
- Model reads resource on demand
- Server pushes updates via subscriptions
- Model's context is always fresh
- Works great for: market data, sensor streams, live logs

Use RAG for unstructured historical data. Use MCP resources for structured live data.

#### Designing Resource URIs

URI design matters. Make URIs:
1. **Descriptive:** `mcp://market-data/AAPL/quote` not `mcp://data/1`
2. **Hierarchical:** Parent-child relationships in the path
3. **Addressable:** Each unique piece of data gets a unique URI
4. **Predictable:** Users can guess the URI pattern

Example hierarchy:

```
mcp://database/
├── schema/
│   ├── public.users
│   ├── public.orders
│   └── public.trades
├── data/
│   ├── latest_orders?limit=100
│   └── my_positions?account_id=ACC-123
└── stats/
    ├── daily_volume?symbol=AAPL
    └── pricing_stats?date=2024-04-11
```

#### Python Code Example: Market Data Resources

```python
from mcp.server import Server, Resource
import json
from datetime import datetime

server = Server("market-data")

# In-memory quotes (in practice, fetch from real feed)
quotes = {
    "AAPL": {"symbol": "AAPL", "price": 189.50, "bid": 189.48, "ask": 189.52},
    "MSFT": {"symbol": "MSFT", "price": 420.00, "bid": 419.98, "ask": 420.02},
}

@server.list_resources()
async def list_resources():
    """List all available market data resources"""
    return [
        Resource(
            uri="mcp://market-data/symbols",
            name="Available Symbols",
            description="List of symbols with live quotes",
            mimeType="application/json"
        ),
        *[
            Resource(
                uri=f"mcp://market-data/{symbol}/quote",
                name=f"{symbol} Quote",
                description=f"Real-time price, bid, ask for {symbol}",
                mimeType="application/json"
            )
            for symbol in quotes.keys()
        ]
    ]

@server.read_resource()
async def read_resource(uri: str) -> str:
    """Read a resource by URI"""
    if uri == "mcp://market-data/symbols":
        # Return list of symbols
        return json.dumps({
            "symbols": list(quotes.keys()),
            "timestamp": datetime.now().isoformat()
        })
    
    # Parse symbol from URI like mcp://market-data/AAPL/quote
    parts = uri.split("/")
    if len(parts) >= 4 and parts[-1] == "quote":
        symbol = parts[-2]
        if symbol in quotes:
            data = quotes[symbol].copy()
            data["timestamp"] = datetime.now().isoformat()
            return json.dumps(data)
    
    raise ValueError(f"Unknown resource: {uri}")

@server.subscribe_resource()
async def subscribe_resource(uri: str):
    """Subscribe to real-time updates for a resource"""
    parts = uri.split("/")
    if len(parts) >= 4 and parts[-1] == "quote":
        symbol = parts[-2]
        if symbol not in quotes:
            raise ValueError(f"Symbol not found: {symbol}")
        
        # Simulate streaming price updates
        import asyncio
        while True:
            # In production, fetch from market data feed
            data = quotes[symbol].copy()
            data["timestamp"] = datetime.now().isoformat()
            
            yield {
                "type": "resource_updated",
                "uri": uri,
                "data": json.dumps(data)
            }
            
            # Yield updates every second (in production, on actual price changes)
            await asyncio.sleep(1.0)
```

#### Security: Resource Access Control

For sensitive resources, implement access control:

```python
from functools import wraps

def require_permission(permission: str):
    def decorator(func):
        @wraps(func)
        async def wrapper(uri: str, user_id: str = None, *args, **kwargs):
            # Check user permission
            if not user_has_permission(user_id, permission):
                raise PermissionError(
                    f"User {user_id} does not have '{permission}' permission"
                )
            return await func(uri, *args, **kwargs)
        return wrapper
    return decorator

@server.read_resource()
@require_permission("read:data")
async def read_resource(uri: str) -> str:
    # Resource is only readable by authorized users
    ...
```

#### Key Terms for Day 51

| Term | What It Means |
|------|--------------|
| **Resource** | Read-only data accessible by URI; can be subscribed to for updates |
| **Resource URI** | Unique identifier for a resource (e.g., `mcp://market-data/AAPL/quote`) |
| **Text resource** | Plain text data (files, JSON, logs) |
| **Blob resource** | Binary data (images, PDFs) |
| **Subscription** | Server-initiated updates when a resource changes |
| **Live data** | Data that changes in real time and needs subscriptions |

#### Official References
- [MCP Resources](https://modelcontextprotocol.io/docs/concepts/resources)
- [Resource Subscriptions](https://modelcontextprotocol.io/docs/concepts/resources#subscriptions)

---

*#100DaysOfContextEngineering #ContextEngineering #MCP #ModelContextProtocol #AWSCommunityBuilder*

[← Day 50](./Day-50-MCP-Tools.md)
