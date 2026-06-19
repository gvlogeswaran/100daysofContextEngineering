# Day 47 — MCP Architecture — Host, Client, Server

**100 Days of Context Engineering | Phase 4: MCP — The Live Context Protocol** | By Logeswaran GV (AWS Community Builder)

> *"Three roles. One protocol. Host controls. Client connects. Server serves. Get the boundaries wrong and everything breaks."*

---

### MCP Architecture — Host, Client, Server

The three-layer model is the foundation of MCP's elegance and power. Let me show you why it matters.

#### The Three Roles

**Host** is the application. It's where the LLM lives. It's what the user interacts with. Examples:
- Claude Desktop (Anthropic's reference implementation)
- Your web app (Python FastAPI + Claude API)
- AWS Lambda (Bedrock Agents calling an MCP server)
- Cursor Editor (Claude in the IDE)

The host owns the conversation loop. The host decides what context to send to the model. The host handles the user experience. The host never makes direct calls to external systems.

**Client** is the protocol manager inside the host. It's a library you import. Examples:
- `mcp.client.Client` (Python SDK)
- `Client` from `@modelcontextprotocol/sdk/client` (TypeScript)

The client is stateless (per connection). It manages:
- Transport initialization (stdio, HTTP+SSE, etc.)
- Message routing (requests → servers, responses → LLM)
- Error handling and recovery
- Resource subscriptions
- Tool execution lifecycle (request → server → result)

**Server** is a process that publishes capabilities. It doesn't care about the user. It doesn't care about the LLM. It cares about what it can provide. Examples:
- Filesystem MCP server (gives access to local files)
- PostgreSQL MCP server (exposes database schema + query execution)
- Market data MCP server (streams real-time prices)
- Your custom server (anything you implement)

The server publishes a static manifest: "I have these tools, these resources, these prompts. Use them or don't."

#### Why Separation Matters: The Real-World Case

In 2022, I worked on a platform that traded FX and equities. We had:

- **Host:** A Python FastAPI app that users logged into. The app held their preferences, risk limits, audit logs.
- **Clients:** One client per active user session. Each client managed 1 connection to the execution server, 1 to the market data server, 1 to the risk server.
- **Servers:** 3 separate processes:
  - FIX server (handles order routing to 12 execution venues)
  - Market data server (Bloomberg, Reuters feeds)
  - Risk server (calculates Greeks, margin requirements, counterparty limits)

If we'd combined Host + Client, we'd have 50 user sessions each spawning 3 connections to each server. 450 concurrent connections. Chaos.

Because we separated them: 50 Host processes, 1 shared Client pool per server, 3 Server processes. The Client layer handled backpressure, connection pooling, and request ordering.

This is why the three-layer model isn't theoretical. It scales.

#### Architecture Diagram (Textual)

```
┌──────────────────────────────────────────┐
│           HOST APPLICATION               │
│  (Claude Desktop, Lambda, Web App)       │
│  - Runs the LLM                          │
│  - Manages user session/context          │
│  - Owns the conversation loop            │
└──────────────┬──────────────────────────┘
               │
        ┌──────▼──────┐
        │    CLIENT    │
        │ (Protocol    │
        │  Manager)    │
        │              │
        │ Handles:     │
        │ - Transport  │
        │ - Routing    │
        │ - Error      │
        │ - Backpressure
        └──┬───┬───┬───┘
           │   │   │
    ┌──────▼─┐│   └──────┬──────┐
    │Server 1││ Server 2 │Server N
    │        ││          │ (Any
    │Market  ││Database  │ MCP
    │Data    ││          │ Server)
    └────────┘│          │
             └┬──────────┘
```

Each server is a separate process. The client multiplexes requests. The host remains unaware of how many servers exist.

#### Example: Filesystem Server Architecture

Here's a concrete Python example showing Host → Client → Server:

```python
# HOST: Your application
import asyncio
from mcp.client import StdioClientSession

async def main():
    # Host creates a client session to a filesystem server
    async with StdioClientSession(
        StdioServerParameters(
            command="python",
            args=["-m", "mcp_servers.filesystem"]
        )
    ) as session:
        # Host uses the client to list resources from the server
        resources = await session.list_resources()
        
        # Host tells LLM about resources
        for resource in resources:
            print(f"Available: {resource.uri}")
        
        # Host reads a resource
        result = await session.read_resource("file:///Users/loki/data.txt")
        
        # Host sends this context to the LLM
        response = await client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=2048,
            system=[
                {"type": "text", "text": f"Here's the data:\n{result.contents[0].text}"}
            ],
            messages=[...]
        )

asyncio.run(main())
```

The Client (StdioClientSession) handles:
- Starting the server process (stdio transport)
- Sending initialize handshake
- Routing list_resources() → server → client → host
- Parsing JSON-RPC 2.0 responses
- Error handling if server crashes

The Server (in another process):

```python
# SERVER: MCP filesystem server
from mcp.server import Server, stdio_server

server = Server("filesystem")

@server.list_resources()
async def list_resources():
    return [
        Resource(uri="file:///Users/loki/data.txt", name="Data File"),
        Resource(uri="file:///Users/loki/config.json", name="Config"),
    ]

@server.read_resource()
async def read_resource(uri: str):
    with open(uri.replace("file://", ""), "r") as f:
        return f.read()

if __name__ == "__main__":
    stdio_server(server)
```

The server doesn't know who called it. It doesn't authenticate the user. It just does its job and returns data.

#### Comparison: REST API vs MCP Architecture

In a REST-based system:

```
Host (web app)
    → HTTP request to https://api.example.com/resources
    → (Internet, authentication, rate limits)
    → API Server
    → Database
    → Response
    → Host processes
```

Each request is stateless, independent, unauthenticated at the Server level.

In MCP:

```
Host (application)
    → (Persistent authenticated connection via Client)
    → Server (filesystem, DB, API, etc.)
    → Response
```

One persistent connection. The Host and Server trust each other via the Client.

**Key difference:** REST is request-response, unidirectional, stateless. MCP is connection-oriented, bidirectional (especially with sampling), stateful per session.

#### Financial Markets Example: Multi-Server Host

Let's say you're building a trading agent. Your Host needs three servers:

```python
async def initialize_trading_agent():
    # Market data server (real-time prices)
    market_client = await StdioClientSession(
        market_data_server_params
    ).__aenter__()
    
    # Order routing server (execute trades)
    execution_client = await StdioClientSession(
        execution_server_params
    ).__aenter__()
    
    # Risk server (margin, Greeks, limits)
    risk_client = await StdioClientSession(
        risk_server_params
    ).__aenter__()
    
    # Host routes model requests through all three
    while True:
        model_request = await get_user_input()
        
        # Model can call tools from any server
        response = await client.messages.create(
            model="claude-3-5-sonnet-20241022",
            messages=[...],
            tools=[
                *market_client.tools,  # get_price, get_orderbook
                *execution_client.tools,  # place_order, cancel_order
                *risk_client.tools,  # check_margin, calc_var
            ]
        )
        
        # Execute whatever tools the model chose
        while response.stop_reason == "tool_use":
            for tool_use in response.tool_uses:
                result = await route_tool_call(
                    tool_use,
                    market_client,
                    execution_client,
                    risk_client
                )
            # Continue conversation
            response = await client.messages.create(...)
```

The Client layer handles routing each tool call to the correct server. The Host remains unaware of server details. New servers can be added without changing Host code.

#### Security Boundaries

The three-layer model creates natural security boundaries:

- **Host ↔ Client:** Authenticated (the Host trusts its own Client library)
- **Client ↔ Server:** Authenticated per connection (API key, mTLS, etc.)
- **Server:** Never talks to users. Never sees secrets. Only executes its declared functions.

This is why MCP servers are safer to expose than direct API access. A compromised server can't access the Host's state. A compromised Host can't bypass the Client's error handling.

#### Key Terms for Day 47

| Term | What It Means |
|------|--------------|
| **Host** | The application running the LLM; owns user experience and conversation loop |
| **Client** | The connection manager inside the host; handles protocol, routing, error recovery |
| **Server** | A process publishing tools, resources, or prompts to a client |
| **Transport** | The underlying mechanism (stdio, HTTP+SSE) for Host-to-Server communication |
| **Persistent connection** | A single connection that remains open for multiple requests/responses (unlike REST) |
| **Stateless server** | A server that has no memory of previous requests; each call is independent |

#### Official References
- [MCP Architecture Overview](https://modelcontextprotocol.io/docs/concepts/architecture)
- [Client Implementation Guide](https://modelcontextprotocol.io/docs/develop/build-client)
- [Server Implementation Guide](https://modelcontextprotocol.io/docs/develop/build-server)

---

*#100DaysOfContextEngineering #ContextEngineering #MCP #ModelContextProtocol #AWSCommunityBuilder*

[← Day 46](./Day-46-MCP-Welcome-Back-Now-Youre-Ready.md) | [Day 48 →](./Day-48-JSON-RPC-The-Language-MCP-Speaks.md)
