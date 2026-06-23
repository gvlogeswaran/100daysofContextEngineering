# Day 49 — MCP Transports — Stdio vs HTTP+SSE. Pick the Right One.

**100 Days of Context Engineering | Phase 4: MCP — The Live Context Protocol** | By Logeswaran GV (AWS Community Builder)

> *"Stdio for local. HTTP+SSE for cloud. Wrong transport choice = wrong architecture. Right choice = the difference between a demo and a deployment."*

---

### MCP Transports — Stdio vs HTTP+SSE. Pick the Right One.

MCP doesn't define how Host and Server communicate. It only defines *what* they communicate (JSON-RPC 2.0). The *how* is the transport. There are two: stdio and HTTP+SSE.

#### Stdio Transport: Local, Zero-Latency

Stdio (standard input/output) is the simplest transport. The host spawns a subprocess. Both communicate via pipes.

**Architecture:**
```
Host Process
├── stdin (receives from Server)
├── stdout (sends to Server)
└── Server Process
    ├── stdin (receives from Host)
    └── stdout (sends to Host)
```

**How it works:**

1. Host runs: `subprocess.Popen(["python", "-m", "mcp_server"])`
2. Server starts and waits on stdin
3. Host writes JSON-RPC request to Server's stdin
4. Server reads, processes, writes response to stdout
5. Host reads response from Server's stdout

**Advantages:**
- **Zero network latency:** Everything is on the same machine. Microsecond round trips.
- **Simple setup:** No TLS, no authentication, no load balancing. Just spawn a process.
- **Tight coupling:** Host can monitor server's exit code, restart on failure easily.
- **Claude Desktop compatibility:** This is how Claude Desktop runs local servers.

**Disadvantages:**
- **Can't scale horizontally:** One host, one server process. To serve 100 clients, spin up 100 servers.
- **No cloud compatibility:** Can't run on Lambda, Fargate, or other serverless. No stdin/stdout.
- **No authentication:** Host and server trust each other implicitly (they're on the same machine).
- **OS dependency:** Subprocess handling varies on Windows vs. Unix.

**Use case: Financial market data server (local development)**

```python
# host.py: Claude Desktop running locally
import subprocess
import json

# Spawn a local market data server
proc = subprocess.Popen(
    ["python", "market_data_server.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    text=True,
)

# Send initialize request
request = json.dumps({
    "jsonrpc": "2.0",
    "method": "initialize",
    "params": {...},
    "id": 1
})
proc.stdin.write(request + "\n")
proc.stdin.flush()

# Read initialize response
response_line = proc.stdout.readline()
response = json.loads(response_line)
print(f"Server initialized: {response}")
```

#### HTTP + SSE Transport: Cloud-Ready, Scalable

HTTP + Server-Sent Events is the cloud-native transport. The host makes an HTTP request. The server streams responses via SSE.

**Architecture:**
```
Host (AWS Lambda)
    ↓ HTTP POST (with JSON-RPC batch)
Cloud Network
    ↓
Server (AWS ECS)
    ↓ HTTP 200 OK (with SSE stream)
    ↓ data: {...JSON-RPC response...}
    ↓ data: {...JSON-RPC response...}
Host receives
```

**How it works:**

1. Host makes HTTP POST to `https://server.example.com/sse`
2. Server accepts the connection
3. Host writes JSON-RPC request to request body
4. Server streams responses as SSE events
5. Each event is: `data: {JSON-RPC response}\n\n`
6. Host reads events and processes them
7. Connection stays open for multiple round trips

**Advantages:**
- **Cloud-native:** Works on Lambda, ECS, Fargate, Google Cloud Run, anywhere HTTP works
- **Horizontal scaling:** Load balancer in front of multiple server instances. One IP, many servers.
- **Multiple clients:** 1 server can serve 100 clients (bounded by open connections).
- **Firewall-friendly:** HTTP is standard. Works through proxies, load balancers, CDNs.
- **Authentication:** Can use OAuth 2.0, API keys, mTLS, AWS IAM

**Disadvantages:**
- **Network latency:** RTT adds 10-100ms (or more if distant region)
- **More setup:** TLS certificate, authentication, load balancer configuration
- **Stateless:** Each HTTP connection is independent. Server can't rely on connection state.
- **SSE complexity:** Requires proper async handling, connection management, backpressure

**Use case: Trading agent on AWS Lambda calling ECS market data server**

```python
# host.py: AWS Lambda running trading agent
import httpx
import json

async def call_market_data_server(symbol: str):
    request = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
            "name": "get_price",
            "arguments": {"symbol": symbol}
        },
        "id": 1
    }
    
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "https://market-data.internal.example.com/sse",
            json=[request],  # Batch of 1
            headers={
                "Authorization": f"Bearer {AWS_SIGNATURE}",  # IAM auth
            },
            timeout=5.0
        )
        
        # Parse SSE responses
        for line in response.text.strip().split('\n'):
            if line.startswith('data: '):
                event = json.loads(line[6:])
                print(f"Price: {event['result']}")
```

#### Comparison: Stdio vs HTTP+SSE

| Aspect | Stdio | HTTP+SSE |
|--------|-------|----------|
| **Latency** | Microseconds | 10-100ms |
| **Setup** | Subprocess.Popen | HTTP server + TLS |
| **Scaling** | 1:1 host:server | 1:N host:server |
| **Cloud** | No | Yes |
| **Auth** | Implicit (same machine) | OAuth, API key, mTLS |
| **Debugging** | strace, process logs | tcpdump, browser console |
| **Connection model** | One pipe, bidirectional | HTTP request, SSE response stream |

#### Decision Framework

**Use Stdio if:**
- Developing locally (Claude Desktop)
- Embedding in your application
- Sub-millisecond latency is critical
- Single host, single server (no scaling needed)

**Use HTTP+SSE if:**
- Deploying to cloud (Lambda, ECS, GCP, Azure)
- Multiple hosts need to reach one server
- Authentication/authorization is required
- Horizontal scaling matters

#### Production Case: Stdio → HTTP+SSE Migration

In 2023, I deployed a market data MCP server:

**Phase 1: Stdio (local development)**
- Claude Desktop on my laptop
- Server runs as subprocess
- Zero latency, perfect for prototyping

**Phase 2: HTTP+SSE (production)**
- Server deployed to AWS ECS with NLB load balancer
- 3 replicas (high availability)
- 50 trading agents (Lambda) connecting simultaneously
- IAM-authenticated (no API keys in code)
- ~30ms average latency (acceptable for pre-trade checks)

The server code stayed the same. Only the transport changed.

#### Key Terms for Day 49

| Term | What It Means |
|------|--------------|
| **Stdio** | Local inter-process communication via stdin/stdout pipes |
| **HTTP+SSE** | Remote communication via HTTP POST (request) and Server-Sent Events (response stream) |
| **Transport** | The mechanism for delivering JSON-RPC messages (stdio or HTTP+SSE) |
| **Server-Sent Events (SSE)** | HTTP response that streams multiple events to the client |
| **Stateless server** | Each HTTP connection is independent; server doesn't retain state between requests |
| **Backpressure** | The client's ability to slow down the server when overwhelmed |

#### Official References
- [MCP Transports](https://modelcontextprotocol.io/docs/concepts/transports)
- [Stdio Transport Spec](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
- [HTTP+SSE Transport Spec](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)

---

*#100DaysOfContextEngineering #ContextEngineering #MCP #ModelContextProtocol #AWSCommunityBuilder*

[← Day 48](./Day-48-JSON-RPC.md)
