# 🧠 100 Days of Context Engineering

> **By Logeswaran GV (Loki)** — AWS Community Builder · Financial Markets Infrastructure
>
> *"MCP is the plumbing. Context Engineering is the architecture. I started building the pipes — then realised I hadn't shown you the building."*

Every day for 100 days, I publish one production-grade insight on **Context Engineering** — the discipline of designing, structuring, retrieving, and managing the information space that AI models reason over. From prompt architecture to RAG pipelines to memory systems to MCP as the live context delivery protocol.

No theory fluff. Builder-to-builder. 17+ years of financial markets infrastructure informing every pattern.

[![LinkedIn](https://img.shields.io/badge/Follow%20on-LinkedIn-0077B5?style=flat&logo=linkedin)](https://www.linkedin.com/in/logeswarangv/)
[![GitHub Stars](https://img.shields.io/github/stars/gvlogeswaran/100daysofContextEngineering?style=flat&logo=github)](https://github.com/gvlogeswaran/100daysofContextEngineering)
[![Progress](https://img.shields.io/badge/Progress-Day%203%20of%20100-brightgreen?style=flat)](#progress-tracker)

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
| 01 | ✅ Posted | The AI Context Problem — Why LLMs fail without engineered context |
| 02 | ✅ Posted | How LLMs Actually Work — Tokens, context windows, statelessness |
| 03 | 🔥 Today | Why I Changed This Series — The genuine case for Context Engineering |
| 04 | 🔜 Coming Next | What Is Context Engineering? — The full discipline defined |

---

## 🗺️ Series Curriculum

### ✅ Day 1 — The AI Context Problem
**Phase 1: The WHY**

Every LLM you've used has the same blind spot — it can't see your database, your files, or what happened in your business today. This day defines the **Context Gap**: the root cause of most AI agent failures in production.

**Key insight:** The problem isn't the model. The problem is the architecture.

[→ Read Day 1](./Phase1_WHY/Day-01-The-AI-Context-Problem.md)

---

### ✅ Day 2 — How LLMs Actually Work
**Phase 1: The WHY**

Before building context systems, you need to understand what you're feeding. This day covers tokens, context windows, statelessness, and the critical difference between **parametric knowledge** (baked into weights) and **contextual knowledge** (what you provide at runtime).

**Key insight:** Every inference starts from zero. Context is the only bridge between the model and your world.

[→ Read Day 2](./Phase1_WHY/Day-02-How-LLMs-Actually-Work.md)

---

### 🔥 Day 3 — Why I Changed This Series *(Today)*
**Phase 1: The WHY**

The honest story behind moving from **#100DaysOfMCP** to **#100DaysOfContextEngineering**. MCP is a protocol. Context Engineering is the discipline that gives that protocol meaning. This day explains the 6-layer CE stack, where MCP lives within it, and why the full picture matters — especially in high-stakes environments like financial markets where wrong context means wrong decisions.

**Key insight:** You can't engineer a context protocol without first understanding what context engineering is.

[→ Read Day 3](./Phase1_WHY/Day-03-The-Old-Way-Hardcoded-Integrations.md)

---

### 🔜 Day 4 — COMING NEXT
**What Is Context Engineering? — The Full Discipline Defined**

Tomorrow we step back and map the entire discipline: what separates Context Engineering from prompt engineering, why it's the architectural layer that determines AI agent quality, and the hierarchy that connects RAG, memory, prompts, and MCP into one coherent system.

*Follow on [LinkedIn](https://www.linkedin.com/in/logeswarangv/) or ⭐ star this repo to be notified.*

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
├── Github/                    ← Deep-dive articles (published days only)
├── Instagram/                 ← Instagram caption files
├── Carousel/                  ← LinkedIn carousel PDFs
└── _template/                 ← Reusable daily post template
```

---

## 👤 About the Author

**Logeswaran GV (Loki)**

17+ years in financial markets infrastructure — pre/post trade systems, electronic trading, market data. AWS Community Builder (4 years).

The financial markets lens is not incidental. In electronic trading, context failures don't mean bad answers — they mean wrong trades. That operational stakes mindset runs through every post in this series.

- 🔗 [LinkedIn](https://www.linkedin.com/in/logeswarangv/)
- 🐙 [GitHub](https://github.com/gvlogeswaran/100daysofContextEngineering)
- 🏷️ [#100DaysOfContextEngineering](https://www.linkedin.com/search/results/content/?keywords=%23100DaysOfContextEngineering)

---

## 📌 How to Use This Repo

**Following daily:** Each day's file is self-contained. Start at Day 01 and work sequentially — the series builds deliberately, each day earning the next.

**Returning visitor:** Check the Progress Tracker above. New content drops every day.

**Star ⭐ this repo** to get notified when new days are published.

---

> *"The quality of your AI agent is determined by the quality of your context infrastructure — not the quality of your model."*

---

![Progress](https://img.shields.io/badge/Day%203%20of%20100-In%20Progress-orange?style=for-the-badge)
*Series started April 2026 · Updated daily*
