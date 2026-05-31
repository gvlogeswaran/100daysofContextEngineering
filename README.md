# 🧠 100 Days of Context Engineering

> **By Logeswaran GV (Loki)** — AWS Community Builder · Financial Markets Infrastructure
>
> *"MCP is the plumbing. Context Engineering is the architecture. I started building the pipes — then realised I hadn't shown you the building."*

Every day for 100 days, I publish one production-grade insight on **Context Engineering** — the discipline of designing, structuring, retrieving, and managing the information space that AI models reason over. From prompt architecture to RAG pipelines to memory systems to MCP as the live context delivery protocol.

No theory fluff. Builder-to-builder. 17+ years of financial markets infrastructure informing every pattern.

[![GitHub Stars](https://img.shields.io/github/stars/gvlogeswaran/100daysofContextEngineering?style=flat&logo=github)](https://github.com/gvlogeswaran/100daysofContextEngineering)
[![Progress](https://img.shields.io/badge/Progress-Day33%20of%20100-brightgreen?style=flat)](#progress-tracker)

---

## 🔄 Why I Changed This from #100DaysOfMCP

I started this series as **#100DaysOfMCP**. Two days in, a respected expert stopped me:

> *"You're describing the pipes. You haven't told anyone what flows through them."*

He was right. **MCP is a protocol** — the standardised wire format for delivering context to AI models. **Context Engineering is the discipline** — the full architecture of what the model knows, when it knows it, in what form, and how much of it.

MCP is Layer 6 of a 6-layer stack. Teaching only MCP is like teaching USB-C without explaining data centres.

So I expanded. The series now covers the complete Context Engineering discipline — with MCP as its most powerful chapter, not its only one.

**The 6-Layer Context Engineering Stack:**
```
Layer 1 — Raw Data Sources        (databases, APIs, documents, streams)
Layer 2 — Chunking & Indexing     (how data becomes retrievable)
Layer 3 — Retrieval & RAG         (dynamic context injection)
Layer 4 — Memory Systems          (short-term, episodic, semantic, long-term)
Layer 5 — Orchestration           (prompt design, instruction hierarchy, agents)
Layer 6 — Protocol Delivery       (MCP — live, composable, production-grade)
```

---

## 📅 Progress Tracker

| Day | Status | Topic |
|-----|--------|-------|
| 01 | ✅ Posted | [The AI Context Problem — Why LLMs fail without engineered context](./Phase1_WHY/Day-02-How-LLMs-Actually-Work.md) |
| 02 | ✅ Posted | [How LLMs Actually Work — Tokens, context windows, statelessness](./Phase1_WHY/Day-02-How-LLMs-Actually-Work.md) |
| 03 | ✅ Posted | [Why I Changed This Series — The genuine case for Context Engineering](./Phase1_WHY/Day-03-The-Old-Way-Hardcoded-Integrations.md) |
| 04 | ✅ Posted | [What Is Context Engineering? — The full discipline defined](./Phase1_WHY/Day-04-ContextEngineering-Intro.md) |
| 05 | ✅ Posted | [The 4 Types of Context Every LLM Uses](./Phase1_WHY/Day-05-MCP-The-Big-Idea.md) |
| 06 | ✅ Posted | [The Context Window Is Your Most Valuable Real Estate](./Phase1_WHY/Day-06-Context-window.md) |
| 07 | ✅ Posted | [The 5 Enemies of Good Context](./Phase1_WHY/Day-07-Well-allocated-context.md) |
| 08 | ✅ Posted | [Context Engineering vs Prompt Engineering](./Phase1_WHY/Day-08-Context-vs-Promt-Engineering.md) |
| 09 | ✅ Posted | [The Context Engineering Stack - 6 Layers](./Phase1_WHY/Day-09-The-Context-Engineering-Stack.md) |
| 10 | ✅ Posted | [The Model Matters Less Than You Think](./Phase1_WHY/Day-10-The-Model-Matters.md) |
| 11 | ✅ Posted | [The Context Engineering Lifecycle — From Raw Data to Model Response](./Phase1_WHY/Day-11-Context-Engineering-LifeCycle.md) |
| 12 | ✅ Posted | [Lessons from Financial Markets Personal experience](./Phase1_WHY/Day-12-personal-experience.md) |
| 13 | ✅ Posted | [The Anatomy of a World-Class System Prompt](./Phase1_WHY/Day-13-The-System-Prompt.md) |
| 14 | ✅ Posted | [Context Debt: The Silent Killer of AI Systems](./Phase1_WHY/Day-14-The-Context_debt.md) |
| 15 | ✅ Posted | [The 10 Laws of Context Engineering](./Phase1_WHY/Day-15-Context-Engineering-laws.md) |
| 16 | ✅ Posted | [System Prompts Are Architecture Documents](./Phase2_Architecture/Day-16-System-Prompts.md) |
| 17 | ✅ Posted | [The 6 Prompt Patterns Every AI Engineer Must Know](./Phase2_Architecture/Day-17-6-Prompt-Patterns.md) |
| 18 | ✅ Posted | [Chain-of-Thought as Context Scaffolding](./Phase2_Architecture/Day-18-Chain-of-thoughts.md) |
| 19 | ✅ Posted | [Few-Shot Examples: The Most Underused Context Tool](./Phase2_Architecture/Day-19-few-shot-example.md) |
| 20 | ✅ Posted | [Negative Space in Prompts: What NOT to Include](./Phase2_Architecture/Day-20-Negative-space-in-prompts.md) |
| 21 | ✅ Posted | [Role Prompting vs Persona Prompting](./Phase2_Architecture/Day-21-role-persona-prompting.md) |
| 22 | ✅ Posted | [Instruction Hierarchy](./Phase2_Architecture/Day-22-Instruction-Hierarchy.md) |
| 23 | ✅ Posted | [Prompt Versioning: Treating Prompts as Production Code](./Phase2_Architecture/Day-23-Prompt-versioning.md) |
| 24 | ✅ Posted | [Context Compression Techniques](./Phase2_Architecture/Day-24-Context-Compressions.md) |
| 25 | ✅ Posted | [Phase 2 Recap: The Prompt Engineering Toolkit](./Phase2_Architecture/Day-25-Phase2-recap.md) |
| 26 | ✅ Posted | [Why Static Context Fails at Scale](./Phase3_Local_First/Day-26-Why-Context-fails.md) |
| 27 | ✅ Posted | [Long-Term Memory: Persistent Knowledge](./Phase3_Local_First/Day-27-Long-term-memory.md) |
| 28 | ✅ Posted | [Chunking Strategy](./Phase3_Local_First/Day-28-Chunking-strategy.md) |
| 29 | ✅ Posted | [Embedding Models: Choosing the Right One](./Phase3_Local_First/Day-29-Embedding-Models.md) |
| 30 | ✅ Posted | [DynamoDB for Memory State Management](./Phase3_Local_First/Day-30-Vector-database.md) |
| 31 | ✅ Posted | [Hybrid Search: Why Keyword + Semantic Beats Both Alone](./Phase3_Local_First/Day-31-Hybrid-search.md) |
| 32 | ✅ Posted | [RAG Quality: The RAGAS Framework and Evaluation](./Phase3_Local_First/Day-32-RAG-quality.md) |
| 33 | 🔥 Today | [Re-ranking: The Secret Weapon in Production RAG](./Phase3_Local_First/Day-33-ReRanking.md) |
| 34 | 🔜 Coming Next | AWS Bedrock Knowledge Bases: RAG Without the Plumbing |


---

## 🗂️ Repository Structure

```
100daysofContextEngineering/
├── README.md                  ← This master index (updated daily)
├── Phase1_WHY/                ← Days 01–15  · CE Foundations
├── Phase2_Architecture/       ← Days 16–25  · Prompt Engineering as Context Design
├── Phase3_Local_First/        ← Days 26–45  · Dynamic Context — RAG & Memory
├── Phase4_Primitives/         ← Days 46–60  · MCP — The Live Context Protocol
├── Phase5_Production/         ← Days 61–75  · MCP at Production Scale
├── Phase6_Clients/            ← Days 76–82  · Multi-Agent Clients & Hosts
├── Phase7_Advanced/           ← Days 83–90  · Advanced Multi-Agent Patterns
├── Phase8_Capstone/           ← Days 91–100 · Enterprise Capstone & Mastery
```

---

## 👤 About the Author

**Logeswaran GV**

17+ years in financial markets infrastructure — pre/post trade systems, electronic trading, market data. AWS Community Builder (4 years).

The financial markets lens is not incidental. In electronic trading, context failures don't mean bad answers — they mean wrong trades. That operational stakes mindset runs through every post in this series.

- 🔗 [LinkedIn](https://www.linkedin.com/in/logeswarangv/)
- 🐙 [GitHub](https://github.com/gvlogeswaran/100daysofContextEngineering)
- 🏷️ [#100DaysOfContextEngineering](https://www.linkedin.com/search/results/content/?keywords=%23100DaysOfContextEngineering)

---
no
## 📌 How to Use This Repo

**Following daily:** Each day's file is self-contained. Start at Day 01 and work sequentially — the series builds deliberately, each day earning the next.

**Returning visitor:** Check the Progress Tracker above. New content drops every day.

**Star ⭐ this repo** to get notified when new days are published.

---

> *"The quality of your AI agent is determined by the quality of your context infrastructure — not the quality of your model."*

---

![Progress](https://img.shields.io/badge/Day33%20of%20100-In%20Progress-orange?style=for-the-badge)
*Series started April 2026 · Updated daily*
