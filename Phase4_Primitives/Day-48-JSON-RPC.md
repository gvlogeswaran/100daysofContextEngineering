# Day 48 — JSON-RPC 2.0 — The Language MCP Speaks

**100 Days of Context Engineering | Phase 4: MCP — The Live Context Protocol** | By Logeswaran GV (AWS Community Builder)

> *"JSON-RPC 2.0 is the wire language of MCP. Master it and you can debug any tool call, any transport, any failure mode."*

---

### JSON-RPC 2.0 — The Language MCP Speaks

JSON-RPC 2.0 is a 14-year-old spec that solves one problem elegantly: How do you make a function call across a network, in any language, and handle all failure modes?

MCP adopted it. Let me show you why.

#### What is JSON-RPC?

JSON-RPC 2.0 (RFC) is a stateless, lightweight protocol for remote procedure calls. It says:

1. The client sends a JSON message with: method name, parameters, unique ID
2. The server processes it and responds with: result (or error), same ID
3. Both sides understand the contract: no ambiguity

Example (Python client → Rust server):

**Request:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "get_stock_price",
    "arguments": {"symbol": "AAPL"}
  },
  "id": 123
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "AAPL: $189.50"
      }
    ]
  },
  "id": 123
}
```

The `id` field ensures the client can match responses to requests, even if they arrive out of order.

#### The Initialize Handshake (Full Annotated Example)

When a host connects to a server, MCP first exchanges capabilities. This is a JSON-RPC request:

**Host → Server (initialize request):**
```json
{
  "jsonrpc": "2.0",
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "sampling": {}
    },
    "clientInfo": {
      "name": "claude-desktop",
      "version": "0.1.0"
    }
  },
  "id": 1
}
```

The host says: "I'm Claude Desktop. I support sampling. Here's my protocol version."

**Server → Host (initialize response):**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "tools": {},
      "resources": {},
      "prompts": {}
    },
    "serverInfo": {
      "name": "market-data-server",
      "version": "1.0.0"
    }
  },
  "id": 1
}
```

The server says: "I implement tools, resources, and prompts. I'm market-data-server version 1.0.0."

From this point, both sides know what each other supports.

#### Tool Execution Lifecycle (JSON-RPC Messages)

When the model wants to call a tool, here's the message sequence:

**1. Host tells server to execute a tool:**
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "place_order",
    "arguments": {
      "symbol": "MSFT",
      "side": "buy",
      "quantity": 100,
      "limit_price": 420.00
    }
  },
  "id": 42
}
```

**2. Server executes (this happens inside the server process, not in JSON):**
```python
# Inside the server
order_id = place_order(symbol="MSFT", side="buy", quantity=100, limit_price=420.00)
```

**3. Server responds with result:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Order placed: ID=ORD-98765, Status=PENDING, Filled=0/100"
      }
    ]
  },
  "id": 42
}
```

**4. Host gets result and sends it back to LLM context:**

The host adds this to the conversation:
```python
messages.append({
  "role": "user",
  "content": [
    {
      "type": "tool_result",
      "tool_use_id": "...",
      "content": "Order placed: ID=ORD-98765, Status=PENDING, Filled=0/100"
    }
  ]
})
```

#### Error Codes and What They Mean

JSON-RPC 2.0 defines standard error codes:

```python
# Reserved codes
-32700  # Parse error (server can't parse JSON)
-32600  # Invalid Request (missing method, wrong params type)
-32601  # Method not found (server doesn't have this tool)
-32602  # Invalid params (argument types don't match schema)
-32603  # Internal error (server crashed)
-32000 to -32099  # Server-defined errors

# MCP uses -32001 for "server error" (tool execution failed)
```

Example error response:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Insufficient balance for order: have $5000, need $42000"
  },
  "id": 42
}
```

When the host receives an error, it includes it in the conversation context so the model can reason about the failure.

#### Notifications (Fire and Forget)

JSON-RPC supports **notifications**: messages with no `id` field. The server sends these to the host *without expecting a response*.

Example - Server notifying host of market data change:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/list_changed"
}
```

No `id` field. The host receives it and knows: "The list of available resources changed. Re-fetch them if you need current data."

This enables *server-driven subscriptions* (Day 51).

#### Batching Multiple Requests

JSON-RPC allows batching. The client sends an array of requests:

```json
[
  {"jsonrpc": "2.0", "method": "tools/list", "id": 1},
  {"jsonrpc": "2.0", "method": "resources/list", "id": 2},
  {"jsonrpc": "2.0", "method": "prompts/list", "id": 3}
]
```

Server responds with array of responses (in any order):

```json
[
  {"jsonrpc": "2.0", "result": {"tools": [...]}, "id": 1},
  {"jsonrpc": "2.0", "result": {"resources": [...]}, "id": 2},
  {"jsonrpc": "2.0", "result": {"prompts": [...]}, "id": 3}
]
```

This reduces network round trips.

#### Debugging: Tracing JSON-RPC in Production

Here's why understanding JSON-RPC matters in production:

```bash
# tcpdump a stdio transport (example)
$ strace -e write python host.py 2>&1 | grep jsonrpc

write(3, "{\"jsonrpc\": \"2.0\", \"method\": \"tools/call\", \"params\": {...}, \"id\": 42}\n", 120)
write(3, "{\"jsonrpc\": \"2.0\", \"result\": {...}, \"id\": 42}\n", 85)

# You can see the exact requests/responses and diagnose failures
```

Or with HTTP+SSE transport, check your browser console or HTTP logs:

```
POST /sse
Host: server.example.com

{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {...},
  "id": 42
}

# Response arrives via Server-Sent Events
data: {"jsonrpc": "2.0", "result": {...}, "id": 42}
```

#### Python Code Example: Raw JSON-RPC Message Handling

Here's how a server processes JSON-RPC messages:

```python
import json
import asyncio
from typing import Any, Dict

class SimpleJSONRPCServer:
    def __init__(self):
        self.methods = {
            "get_price": self.get_price,
            "place_order": self.place_order,
        }
    
    async def handle_message(self, message: str) -> str:
        """Receive JSON-RPC request, return JSON-RPC response"""
        try:
            data = json.loads(message)
        except json.JSONDecodeError:
            return json.dumps({
                "jsonrpc": "2.0",
                "error": {"code": -32700, "message": "Parse error"},
                "id": None
            })
        
        method_name = data.get("method")
        params = data.get("params", {})
        request_id = data.get("id")
        
        # Validate JSON-RPC structure
        if data.get("jsonrpc") != "2.0":
            return json.dumps({
                "jsonrpc": "2.0",
                "error": {"code": -32600, "message": "Invalid Request"},
                "id": request_id
            })
        
        if method_name not in self.methods:
            return json.dumps({
                "jsonrpc": "2.0",
                "error": {"code": -32601, "message": "Method not found"},
                "id": request_id
            })
        
        try:
            # Call the actual method
            result = await self.methods[method_name](**params)
            
            # Return success response
            return json.dumps({
                "jsonrpc": "2.0",
                "result": result,
                "id": request_id
            })
        
        except TypeError as e:
            # Invalid parameters
            return json.dumps({
                "jsonrpc": "2.0",
                "error": {"code": -32602, "message": f"Invalid params: {str(e)}"},
                "id": request_id
            })
        
        except Exception as e:
            # Internal server error
            return json.dumps({
                "jsonrpc": "2.0",
                "error": {"code": -32603, "message": f"Internal error: {str(e)}"},
                "id": request_id
            })
    
    async def get_price(self, symbol: str) -> Dict[str, Any]:
        # Simulate price lookup
        return {"symbol": symbol, "price": 189.50}
    
    async def place_order(self, symbol: str, quantity: int, price: float) -> Dict[str, Any]:
        # Simulate order placement
        return {"order_id": "ORD-123", "status": "PENDING"}

# Usage
async def main():
    server = SimpleJSONRPCServer()
    
    # Simulate incoming request
    request = json.dumps({
        "jsonrpc": "2.0",
        "method": "get_price",
        "params": {"symbol": "AAPL"},
        "id": 1
    })
    
    response = await server.handle_message(request)
    print(response)
    # Output: {"jsonrpc": "2.0", "result": {"symbol": "AAPL", "price": 189.5}, "id": 1}

asyncio.run(main())
```

This is the core pattern. Every MCP server does this: receive JSON-RPC, validate, execute, return JSON-RPC.

#### Key Terms for Day 48

| Term | What It Means |
|------|--------------|
| **JSON-RPC 2.0** | A stateless remote procedure call protocol using JSON messages |
| **Request** | A JSON message from client to server with method, params, and unique id |
| **Response** | A JSON message from server to client with result (or error) and matching id |
| **Notification** | A JSON-RPC message with no id; sent one-way, no response expected |
| **Error code** | Standardized integer indicating failure type (-32700 to -32099) |
| **Batch** | Multiple JSON-RPC messages sent in a single transmission |

#### Official References
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification)
- [MCP Protocol Messages](https://modelcontextprotocol.io/docs/getting-started/intro)

---

*#100DaysOfContextEngineering #ContextEngineering #MCP #ModelContextProtocol #AWSCommunityBuilder*

[← Day 47](./Day-47-MCP-Architecture.md)
