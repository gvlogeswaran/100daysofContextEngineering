# Day 50 — MCP Tools — Giving Your Model the Ability to Act

**100 Days of Context Engineering | Phase 4: MCP — The Live Context Protocol** | By Logeswaran GV (AWS Community Builder)

> *"A tool is a contract. The model commits to calling it correctly. You commit to executing it reliably. Write the schema like a senior engineer, not an afterthought."*

---

### MCP Tools — Giving Your Model the Ability to Act

Tools are how models execute actions in the real world. A tool is a function the server exposes, and the model can call. Here's how to design them correctly.

#### What is a Tool?

An MCP tool is:
1. A **function** that does something (read from database, execute trade, send email)
2. A **JSON Schema** describing the parameters
3. A **description** telling the model when to use it

The server publishes the schema. The model sees the description and decides whether to call it.

#### Tool Schema (JSON Schema)

Tools use JSON Schema for parameter validation. Example:

```json
{
  "name": "place_order",
  "description": "Place a market or limit order on the exchange. Requires explicit user confirmation. Returns order ID and status.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "symbol": {
        "type": "string",
        "description": "Stock ticker symbol (e.g., AAPL, MSFT)"
      },
      "side": {
        "type": "string",
        "enum": ["buy", "sell"],
        "description": "Buy or sell"
      },
      "quantity": {
        "type": "integer",
        "minimum": 1,
        "maximum": 1000000,
        "description": "Number of shares"
      },
      "order_type": {
        "type": "string",
        "enum": ["market", "limit"],
        "description": "Order type"
      },
      "limit_price": {
        "type": "number",
        "minimum": 0.01,
        "description": "Limit price (required if order_type is 'limit')"
      }
    },
    "required": ["symbol", "side", "quantity", "order_type"]
  }
}
```

The schema is **not** just documentation. The client uses it to:
- Validate the model's arguments before sending to server
- Display UI hints to the user
- Help the model understand parameter constraints

#### Tool Execution Lifecycle

When the model calls a tool:

```
1. Host (Claude running in app)
   → Model generates tool_use block:
      {
        "type": "tool_use",
        "name": "place_order",
        "input": {
          "symbol": "AAPL",
          "side": "buy",
          "quantity": 100,
          "order_type": "market"
        }
      }

2. Host validates input against schema
   → symbol: "AAPL" ✓ (is string)
   → side: "buy" ✓ (in enum)
   → quantity: 100 ✓ (integer, >= 1)
   → order_type: "market" ✓ (in enum)

3. Host asks for approval (optional human-in-the-loop)
   → "Model wants to: Place order for 100 AAPL @ market. OK? (yes/no)"

4. Host sends tool call to server via JSON-RPC:
   {
     "jsonrpc": "2.0",
     "method": "tools/call",
     "params": {
       "name": "place_order",
       "arguments": {...}
     },
     "id": 42
   }

5. Server executes the function
   → place_order(symbol="AAPL", side="buy", ...)
   → Returns: {"order_id": "ORD-12345", "status": "FILLED"}

6. Server sends result back via JSON-RPC:
   {
     "jsonrpc": "2.0",
     "result": {
       "content": [
         {
           "type": "text",
           "text": "Order ORD-12345 filled: 100 AAPL @ $189.50"
         }
       ]
     },
     "id": 42
   }

7. Host adds result to conversation context
   → Model sees: "Tool result: Order ORD-12345 filled..."
   → Model continues reasoning with new information
```

#### Tool Description Best Practices

The description is the model's only clue about when to use the tool. Make it count.

**Bad description:**
```
"Execute order"
```
The model doesn't know:
- Should it call this proactively or only when user asks?
- What are the preconditions?
- What could go wrong?

**Good description:**
```
"Place a market or limit order on the exchange. Use ONLY when:
1. User explicitly requests to trade (e.g., 'buy 100 AAPL')
2. You've confirmed the symbol, quantity, and price with the user
3. The account has sufficient balance (check with check_balance tool first)

Returns: order_id and status. May return error if insufficient balance 
or symbol is invalid. Never call without explicit user confirmation."
```

The good description tells the model:
- Preconditions (user must ask)
- Validation steps (confirm, check balance)
- What to do on error
- When NOT to call it

#### Python Code Example: Defining Tools

```python
from mcp.server import Server
from typing import Any

server = Server("market-data")

# Define a tool
@server.call_tool()
async def place_order(
    symbol: str,
    side: str,  # "buy" or "sell"
    quantity: int,
    order_type: str,  # "market" or "limit"
    limit_price: float = None
) -> str:
    """
    Place an order on the exchange.
    
    Args:
        symbol: Stock ticker (e.g., AAPL)
        side: Buy or sell
        quantity: Number of shares (1-100000)
        order_type: Market or limit
        limit_price: Required if order_type is 'limit'
    
    Returns:
        Order ID
    """
    
    if order_type == "limit" and limit_price is None:
        raise ValueError("limit_price required for limit orders")
    
    if quantity < 1 or quantity > 100000:
        raise ValueError("quantity must be 1-100000")
    
    # Execute order (pseudocode)
    order_id = execute_order(
        symbol=symbol,
        side=side,
        quantity=quantity,
        price=limit_price if order_type == "limit" else "market"
    )
    
    return f"Order {order_id} placed"


# Register the tool with MCP
server.register_tool(
    name="place_order",
    description="""Place a market or limit order on the exchange.

Use ONLY when:
1. User explicitly requests to trade (e.g., "buy 100 AAPL")
2. You've confirmed the symbol, quantity, and price with the user
3. The account has sufficient balance (use check_balance first)

Required parameters:
- symbol: Stock ticker
- side: "buy" or "sell"
- quantity: 1-100000 shares
- order_type: "market" or "limit"
- limit_price: Required if order_type is "limit"

Returns: Order ID and status. May fail if insufficient balance or symbol invalid.
Never call without explicit user confirmation.""",
    input_schema={
        "type": "object",
        "properties": {
            "symbol": {"type": "string"},
            "side": {"type": "string", "enum": ["buy", "sell"]},
            "quantity": {"type": "integer", "minimum": 1, "maximum": 100000},
            "order_type": {"type": "string", "enum": ["market", "limit"]},
            "limit_price": {"type": "number", "minimum": 0.01}
        },
        "required": ["symbol", "side", "quantity", "order_type"]
    },
    handler=place_order
)
```

#### Security: Human-in-the-Loop Approval

For high-risk tools (like order execution), implement approval workflows:

```python
@server.call_tool()
async def place_order(...) -> str:
    # Log the request
    log_audit_event(
        action="place_order",
        symbol=symbol,
        quantity=quantity,
        user_id=get_current_user()
    )
    
    # Check policy
    if not can_user_trade(get_current_user()):
        raise PermissionError("User not authorized to trade")
    
    # Execute
    order_id = execute_order(...)
    
    # Log result
    log_audit_event(
        action="order_placed",
        order_id=order_id,
        status="success"
    )
    
    return order_id
```

#### Key Terms for Day 50

| Term | What It Means |
|------|--------------|
| **Tool** | A function the model can call to perform actions in the real world |
| **Tool name** | The identifier for the tool (e.g., "place_order") |
| **Input schema** | JSON Schema describing the tool's parameters |
| **Tool description** | Natural language text telling the model when to use the tool |
| **Tool execution** | The process of the model calling the tool and receiving a result |
| **Tool approval** | Human-in-the-loop step before executing high-risk tools |

#### Official References
- [MCP Tools](https://modelcontextprotocol.io/docs/concepts/tools)
- [JSON Schema](https://json-schema.org/)

---

*#100DaysOfContextEngineering #ContextEngineering #MCP #ModelContextProtocol #AWSCommunityBuilder*

[← Day 49](./Day-49-MCP-Transports.md)| [Day 50 →](./Day-50-MCP-Tools.md)
