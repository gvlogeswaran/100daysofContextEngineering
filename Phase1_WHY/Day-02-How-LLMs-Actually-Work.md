# Day 02 — How LLMs Actually Work

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"LLMs don't 'know' anything — they process tokens through a narrow window, and everything outside that window doesn't exist."*

---

## Explanation of Day Topic

Large Language Models are fundamentally different from how most people imagine them. They aren't databases that "look up" information. They aren't reasoning engines that store knowledge between conversations. Instead, LLMs are stateless language processors that work within strict constraints: a token limit, a fixed context window, and no persistent memory between requests.

Understanding how LLMs actually work is foundational to understanding why Model Context Protocol exists in the first place. Without grasping the limitations of statelessness, token budgets, and context windows, you'll build systems that either fail silently or become prohibitively expensive. MCP exists precisely because LLMs can't be treated as general-purpose knowledge stores — they need a smarter bridge between their constrained capabilities and your unlimited data.

This isn't abstract theory. If you've ever built a chatbot that "forgets" context after a few messages, or spent thousands training an AI system that performs poorly on retrieval tasks, you've hit these constraints head-on.

---

## Problem Area

### The Stateless Architect

Every time an LLM processes a request, it starts from zero. There is no persistent state, no long-term memory, no "profile" of the user across sessions. The model generates its response token-by-token based solely on what you provide in that single request. Once the response is complete, the model has no record of what was said. If you send the same request again tomorrow, the model has no idea you've spoken before.

This creates a fundamental mismatch between how people expect AI to work and how it actually works. In traditional software systems, you build stateful services with databases, session stores, and audit logs. You expect consistency, memory, and continuity. LLMs offer none of that by design.

### The Token Constraint

LLMs don't process text the way humans do. They break everything into tokens — chunks of approximately 0.75 words or 4 characters each. Each model has a hard token limit. If you exceed that limit, the model simply cannot process your request. You can't "overflow" gracefully — you hit a wall.

This matters because everything costs tokens: your prompt, the conversation history, the data you provide, even the response itself. A 500-page manual becomes 200,000 tokens. If your model supports a 128k token limit, you've already exceeded it by trying to include the entire manual in a single request. And that's before accounting for the actual question you're asking.

### The "Lost in the Middle" Problem

Even when you stay within token limits, LLMs exhibit a well-documented phenomenon: they perform worse on information in the middle of a large context window. They excel at the beginning and end but often miss critical details buried in the middle of massive blocks of text. For anyone trying to use an LLM as a retrieval system over large documents, this is a serious limitation.

---

## Solution — With Real-World Examples

The solution is not to build bigger windows or more powerful models. The solution is to be *selective* about what enters the window in the first place. Instead of trying to cram your entire knowledge base into every request, you retrieve only the relevant pieces of information, pass them through the window, let the model reason about them, and then discard them.

This is where dynamic context retrieval comes in. Rather than sending a static blob of data with every request, you build a system that:
1. Receives a query from the user
2. Searches for the relevant information needed to answer that query
3. Passes *only that relevant information* through the context window
4. Gets the model's response
5. Moves to the next query with a fresh window

This approach keeps latency low, costs reasonable, and the model's reasoning sharp.

**Real-World Example 1: Market Data Systems in Financial Services**

Imagine you're building a trading assistant for a major financial institution. The organization has millions of market data points: historical prices, company fundamentals, economic indicators, analyst reports, and internal research notes. You could theoretically include all of this in the LLM's context, but you'd run into three immediate failures:

First, cost. If you send 2 million tokens of historical data with every query about a single stock, you're spending $50 per request just on context. That scales to thousands of dollars per day.

Second, latency. Sending 2 million tokens through the model takes 30+ seconds per request. Your traders need sub-second responses to identify opportunities.

Third, accuracy. The model will struggle with relevance. A query like "Is this stock oversold?" gets drowned in noise from unrelated economic data.

Instead, you build a retrieval system. The trader asks a question. The system searches the market database for the last 12 months of price data for that ticker, the most recent earnings report, and the current analyst consensus. That's maybe 15,000 tokens. The model processes that focused dataset in under 2 seconds, gives a confident answer, and the trader acts. That's the difference between a system that works and one that's abandoned.

**Real-World Example 2: Enterprise IT Documentation and Support**

A large enterprise has accumulated decades of internal documentation: system design docs, troubleshooting guides, runbooks, architectural decisions, and historical incident reports. New engineers struggle to navigate this chaos. You want to build a search assistant that can answer questions like "Why did we choose Kafka over RabbitMQ?" or "What's the recovery procedure for the billing database?"

If you tried to embed all 50,000 pages into an LLM's context for every query, you'd face the same problems: extreme latency, massive cost, and poor accuracy from too much noise. Instead, you build a vector database of your documentation. When an engineer asks a question, the system performs a semantic search, retrieves the top 5-10 most relevant documents (maybe 8,000 tokens total), and passes those to the model. The model then reasons about your actual practices and gives a precise, grounded answer. Now your assistant is useful — it answers questions in 3 seconds instead of 30, and it costs cents instead of dollars.

---

## Key Topics

| Term | What It Means |
|------|---------------|
| **Stateless** | An AI model that does not retain information between requests. Each conversation starts fresh. |
| **Token** | The basic unit of text an LLM processes. Roughly 0.75 words or 4 characters. |
| **Context Window** | The fixed-size active memory buffer available to an LLM during a single request, measured in tokens. |
| **Token Limit** | The maximum number of tokens a model can process in a single request. Exceeding this causes failure. |
| **Truncation** | The automatic removal of old messages to make room for new ones when a context window fills up. |
| **Dynamic Context Retrieval** | The practice of retrieving only relevant information needed to answer a specific question, rather than sending all data. |
| **Lost in the Middle** | The phenomenon where LLMs perform worse on information in the middle of a large context block. |
| **Latency** | The time it takes for an LLM to process a request and return a response. Longer context windows increase latency. |
| **Cost per Token** | The pricing model for LLMs, where you pay for the number of tokens processed. Larger requests cost more. |
| **Vector Database** | A specialized database that stores data as embeddings, enabling semantic search for relevant information. |

---

## What's Next

**Day 03 — The Old Way: Hardcoded Integrations**

Before MCP standardized how systems connect to LLMs, teams solved the context retrieval problem the hard way: by building custom integrations, glue code, and ad-hoc solutions for each new data source. Day 03 explores the pain, expense, and fragility of that approach — and why it motivated the creation of MCP itself.

---

*Published: April 8, 2026 | #100DaysOfContextEngineering #ContextEngineering #AIEngineering #AgenticAI #AWSCommunityBuilder*

[← Day 01](./Day-01-The-AI-Context-Problem.md) | [Day 03 →](./Day-03-The-Old-Way-Hardcoded-Integrations.md)
