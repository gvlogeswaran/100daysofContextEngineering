# Day 38 — Working Memory: The Agent Scratchpad Pattern

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"The scratchpad is not a log of what happened — it is the agent's private workspace for figuring out what to do next. Thinking made visible."*

---

## Explanation of Day Topic

Short-term memory (Day 37) manages the *conversation* — the back-and-forth between user and assistant across turns. Working memory is different: it is the agent's *private reasoning space* within a single task execution cycle.

When an agent receives a complex goal — "Should I rebalance my portfolio given today's market conditions?" — it cannot answer in one step. It needs to reason about the current allocation, check risk limits, look up current prices, calculate required trades, and then synthesise a recommendation. The scratchpad is where that intermediate reasoning accumulates.

This day introduces the ReAct pattern (Reasoning + Acting + Observing), which structures the scratchpad as a sequence of Thought → Action → Observation cycles. You will see why isolating the scratchpad per task is critical, how to budget its token cost against the rest of the context window, and how it integrates with the broader memory architecture.

---

### Agent Scratchpad and ReAct Pattern

#### 1. ReAct: Reasoning + Acting

```python
class ReActAgent:
    def __init__(self):
        self.scratchpad = ""
        self.tools = {}
    
    def think_act_observe(self, goal):
        # Reason
        reasoning = llm.invoke(f"Plan: {goal}")
        self.scratchpad += f"\nThought: {reasoning}"
        
        # Act
        tool_name, params = extract_action(reasoning)
        result = self.tools[tool_name](**params)
        self.scratchpad += f"\nAction: {tool_name}({params})"
        
        # Observe
        self.scratchpad += f"\nObservation: {result}"
        
        return result
```

#### 2. Extended ReAct — Financial Trading Analysis Example

The skeleton above shows the structure. Here is a realistic multi-cycle example where an agent analyses whether to rebalance a portfolio position.

```python
import boto3
import json

class FinancialReActAgent:
    """
    ReAct agent for portfolio analysis.
    Scratchpad accumulates Thought / Action / Observation cycles
    until the agent reaches a Final Answer.
    """

    def __init__(self):
        self.scratchpad = ""
        self.bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
        self.tools = {
            'get_price': self._get_price,
            'get_allocation': self._get_allocation,
            'check_risk_limits': self._check_risk_limits,
            'calculate_rebalance': self._calculate_rebalance,
        }

    # --- Stub tool implementations ---
    def _get_price(self, ticker: str) -> dict:
        prices = {'AAPL': 189.50, 'MSFT': 415.20, 'CASH': 1.00}
        return {'ticker': ticker, 'price': prices.get(ticker, 0)}

    def _get_allocation(self, user_id: str) -> dict:
        return {'AAPL': 0.45, 'MSFT': 0.40, 'CASH': 0.15}

    def _check_risk_limits(self, ticker: str, proposed_pct: float) -> dict:
        max_single = 0.40
        breached = proposed_pct > max_single
        return {'ticker': ticker, 'proposed': proposed_pct,
                'limit': max_single, 'breached': breached}

    def _calculate_rebalance(self, current: dict, target: dict) -> dict:
        trades = {}
        for t in set(list(current) + list(target)):
            delta = target.get(t, 0) - current.get(t, 0)
            if abs(delta) > 0.01:
                trades[t] = round(delta, 4)
        return {'trades': trades}

    # --- ReAct loop ---
    def _ask_llm(self, prompt: str) -> str:
        """Call Bedrock Claude with the current prompt."""
        body = json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 512,
            "messages": [{"role": "user", "content": prompt}]
        })
        resp = self.bedrock.invoke_model(
            modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
            body=body
        )
        return json.loads(resp['body'].read())['content'][0]['text']

    def _extract_action(self, thought: str):
        """
        Parse the LLM output for Action: tool_name(param=value).
        Returns (tool_name, kwargs) or (None, None) if Final Answer found.
        """
        import re
        if 'Final Answer:' in thought:
            return None, None
        match = re.search(r'Action:\s*(\w+)\((.+?)\)', thought)
        if not match:
            return None, None
        tool = match.group(1)
        raw_params = match.group(2)
        # Parse simple key=value pairs
        kwargs = {}
        for part in raw_params.split(','):
            k, _, v = part.partition('=')
            kwargs[k.strip()] = v.strip().strip('"\'')
        return tool, kwargs

    def run(self, user_id: str, goal: str, max_cycles: int = 6) -> str:
        """
        Execute the ReAct loop.

        Cycle 1 — Thought: check current allocation
        Cycle 2 — Thought: check prices for overweight position
        Cycle 3 — Thought: validate risk limits for proposed target
        Cycle 4 — Thought: calculate required trades
        Cycle 5 — Final Answer
        """
        prompt_template = (
            "You are a portfolio rebalancing agent. "
            "Use the following tools by writing 'Action: tool_name(param=value)'.\n"
            "Available tools: get_price, get_allocation, check_risk_limits, calculate_rebalance.\n\n"
            "Goal: {goal}\n\n"
            "Scratchpad so far:\n{scratchpad}\n\n"
            "Think step by step. When done, write 'Final Answer: <recommendation>'."
        )

        for cycle in range(1, max_cycles + 1):
            prompt = prompt_template.format(
                goal=goal, scratchpad=self.scratchpad
            )
            thought = self._ask_llm(prompt)
            self.scratchpad += f"\n--- Cycle {cycle} ---\nThought: {thought}"

            tool_name, kwargs = self._extract_action(thought)

            if tool_name is None:
                # Extract Final Answer
                answer_start = thought.find('Final Answer:')
                return thought[answer_start:] if answer_start != -1 else thought

            # Execute the tool
            if tool_name in self.tools:
                # For get_allocation, inject user_id
                if tool_name == 'get_allocation':
                    kwargs['user_id'] = user_id
                observation = self.tools[tool_name](**kwargs)
            else:
                observation = f"Error: unknown tool '{tool_name}'"

            self.scratchpad += (
                f"\nAction: {tool_name}({kwargs})"
                f"\nObservation: {observation}"
            )

        return "Max cycles reached — inconclusive."


# --- Usage ---
agent = FinancialReActAgent()
result = agent.run(
    user_id='trader_007',
    goal='Determine if my portfolio needs rebalancing and what trades to make.'
)
print(result)

# Scratchpad after a successful run might look like:
# --- Cycle 1 ---
# Thought: First I need to know the current allocation.
# Action: get_allocation(user_id='trader_007')
# Observation: {'AAPL': 0.45, 'MSFT': 0.40, 'CASH': 0.15}
#
# --- Cycle 2 ---
# Thought: AAPL at 45% exceeds the 40% single-stock limit. Check risk limits.
# Action: check_risk_limits(ticker='AAPL', proposed_pct=0.45)
# Observation: {'breached': True, 'limit': 0.40}
#
# --- Cycle 3 ---
# Thought: Risk breach confirmed. Target 38% AAPL, 40% MSFT, 22% CASH.
# Action: calculate_rebalance(current={...}, target={...})
# Observation: {'trades': {'AAPL': -0.07, 'CASH': 0.07}}
#
# --- Cycle 4 ---
# Thought: Trades are small and within limits.
# Final Answer: Sell 7% AAPL, move to CASH. No other changes required.
```

---

### Scratchpad Patterns

#### Scratch Space Isolation

Each task should get a **fresh scratchpad**. Never carry intermediate reasoning from a previous task into a new one — it pollutes the agent's reasoning and wastes tokens.

```python
class ScratchpadManager:
    def __init__(self):
        self._scratchpad = ""

    def clear(self):
        """Call between tasks — not between cycles."""
        self._scratchpad = ""

    def append(self, text: str):
        self._scratchpad += f"\n{text}"

    @property
    def content(self) -> str:
        return self._scratchpad

    def token_count(self) -> int:
        return len(self._scratchpad) // 4  # Approximate

    def is_over_budget(self, budget_tokens: int = 8000) -> bool:
        return self.token_count() > budget_tokens
```

#### Token Budget for the Scratchpad

The scratchpad competes with system prompt, user query, RAG context, and conversation history for the same context window. A practical budget allocation for a 128K-token model:

| Layer | Budget |
|-------|--------|
| System prompt | 1,000 tokens |
| Conversation history | 8,000 tokens |
| RAG context | 20,000 tokens |
| Scratchpad (ReAct) | 8,000 tokens |
| Response buffer | 4,000 tokens |
| **Available total** | **41,000 tokens** |

When `is_over_budget()` returns `True`, compress the oldest Thought/Action/Observation cycles before continuing.

#### Clearing the Scratchpad Between Tasks

```python
def handle_user_request(agent: FinancialReActAgent, user_id: str, query: str) -> str:
    # Always reset scratchpad at task boundaries
    agent.scratchpad = ""

    # Run the ReAct loop for this task
    answer = agent.run(user_id=user_id, goal=query)

    # Optionally persist the scratchpad for debugging
    store_debug_trace(user_id=user_id, trace=agent.scratchpad)

    return answer
```

---

#### Key Terms for Day 38

| Term | Meaning |
|------|---------|
| **ReAct** | A prompting pattern combining **Re**asoning and **Act**ing — the agent alternates between thinking (Thought), calling a tool (Action), and reading the result (Observation). |
| **Scratchpad** | The agent's private working memory within the context window — used to accumulate intermediate reasoning steps across multiple ReAct cycles. |
| **Thought** | The agent's internal reasoning step: interpreting the current state, deciding what information is still needed. |
| **Action** | The step where the agent invokes a tool (API call, database query, calculation) based on its Thought. |
| **Observation** | The tool's return value, appended to the scratchpad so the agent can reason about the result in the next cycle. |
| **Chain-of-Thought** | The broader technique of making an LLM show its reasoning step by step before producing a final answer; ReAct is Chain-of-Thought extended with tool use. |

---

## What's Next

**Day 39 — Episodic Memory: Teaching Agents to Remember Past Interactions**

The scratchpad is cleared after every task — it is purely transient. Day 39 introduces episodic memory: the persistent store of what a specific user has said and done across many sessions. You will see how to store episodes in DynamoDB, score them for relevance using embeddings, and inject the most pertinent past interactions into the agent's context to make it feel genuinely personalised.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 37](./Day-37-Short-term-memory.md) | [Day 39 →](./Day_39_Episodic-Memory.md)
