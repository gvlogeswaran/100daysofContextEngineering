# 🧠 100 Days of Context Engineering

> **By Logeswaran GV (Loki)** — AWS Community Builder · Financial Markets Infrastructure
>
> *"MCP is the plumbing. Context Engineering is the architecture. I started building the pipes — then realised I hadn't shown you the building."*

Every day for 100 days, I publish one production-grade insight on **Context Engineering** — the discipline of designing, structuring, retrieving, and managing the information space that AI models reason over. From prompt architecture to RAG pipelines to memory systems to MCP as the live context delivery protocol.

No theory fluff. Builder-to-builder. 17+ years of financial markets infrastructure informing every pattern.

[![GitHub Stars](https://img.shields.io/github/stars/gvlogeswaran/100daysofContextEngineering?style=flat&logo=github)](https://github.com/gvlogeswaran/100daysofContextEngineering)
[![Progress](https://img.shields.io/badge/Progress-Day26%20of%20100-brightgreen?style=flat)](#progress-tracker)

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
| 03 | ✅ Posted | Why I Changed This Series — The genuine case for Context Engineering |
| 04 | ✅ Posted | What Is Context Engineering? — The full discipline defined |
| 05 | ✅ Posted | The 4 Types of Context Every LLM Uses
| 06 | ✅ Posted | The Context Window Is Your Most Valuable Real Estate
| 07 | ✅ Posted | The 5 Enemies of Good Context
| 08 | ✅ Posted | Context Engineering vs Prompt Engineering
| 09 | ✅ Posted | The Context Engineering Stack - 6 Layers
| 10 | ✅ Posted | The Model Matters Less Than You Think
| 11 | ✅ Posted | The Context Engineering Lifecycle — From Raw Data to Model Response
| 12 | ✅ Posted | Lessons from Financial Markets Personal experience
| 13 | ✅ Posted | The Anatomy of a World-Class System Prompt
| 14 | ✅ Posted | Context Debt: The Silent Killer of AI Systems
| 15 | ✅ Posted | The 10 Laws of Context Engineering
| 16 | ✅ Posted | System Prompts Are Architecture Documents
| 17 | ✅ Posted | The 6 Prompt Patterns Every AI Engineer Must Know
| 18 | ✅ Posted | Chain-of-Thought as Context Scaffolding
| 19 | ✅ Posted | Few-Shot Examples: The Most Underused Context Tool
| 20 | ✅ Posted | Negative Space in Prompts: What NOT to Include
| 21 | ✅ Posted | Role Prompting vs Persona Prompting
| 22 | ✅ Posted | Instruction Hierarchy
| 23 | ✅ Posted | Prompt Versioning: Treating Prompts as Production Code
| 24 | ✅ Posted | Context Compression Techniques
| 25 | ✅ Posted | Phase 2 Recap: The Prompt Engineering Toolkit
| 26 | 🔥 Today | Why Static Context Fails at Scale
| 27 | 🔜 Coming Next | Why Static Context Fails at Scale


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

### ✅ Day 3 — Why I Changed This Series *(Today)*
**Phase 1: The WHY**

The honest story behind moving from **#100DaysOfMCP** to **#100DaysOfContextEngineering**. MCP is a protocol. Context Engineering is the discipline that gives that protocol meaning. This day explains the 6-layer CE stack, where MCP lives within it, and why the full picture matters — especially in high-stakes environments like financial markets where wrong context means wrong decisions.

**Key insight:** You can't engineer a context protocol without first understanding what context engineering is.

[→ Read Day 3](./Phase1_WHY/Day-03-The-Old-Way-Hardcoded-Integrations.md)

---

### ✅ Day 4 — What Is Context Engineering?
**What Is Context Engineering? — The Full Discipline Defined**

Context Engineering is the complete practice of designing, building, maintaining, and optimising the flow of information into AI systems. It covers everything from how many tokens you have and what they cost, to how you retrieve relevant data, maintain agent memory, craft instructions, orchestrate multi-step workflows, and deliver context through standardised protocols.

[→ Read Day 4](./Phase1_WHY/Day-04-ContextEngineering-Intro.md)
---

### ✅ Day 5 — Four Types of Context Every LLM Uses

The four types are: Parametric, Instructional, Conversational, and Retrieved. Each has distinct properties, distinct failure modes, and distinct engineering requirements.


[→ Read Day 5](./Phase1_WHY/Day-05-MCP-The-Big-Idea.md)
---

### ✅ Day 6 — The Context Window Is Your Most Valuable Real Estate

The context window is a constrained strategic resource. Every token is an investment. Every token in the wrong position reduces the quality of the model's output. And because of a well-documented research finding called "Lost in the Middle," where you place information matters as much as what information you include.

[→ Read Day 6](./Phase1_WHY/Day-06-Context-window.md)
---

### ✅ Day 7 — The 5 Enemies of Good Context

The culprit is almost always one of five context quality enemies: noise, contradiction, staleness, over-compression, and under-specification. These are the silent killers of production AI systems. They're invisible until something breaks, and they're entirely within the Context Engineer's control to prevent.

[→ Read Day 7](./Phase1_WHY/Day-07-Well-allocated-context.md)
---

### ✅ Day 8 — Context Engineering vs Prompt Engineering

The most common misconception in production AI development is conflating Prompt Engineering with Context Engineering. They are related but fundamentally different disciplines — and understanding the relationship between them determines whether your AI systems scale or plateau.

[→ Read Day 8](./Phase1_WHY/Day-08-Context-vs-Promt-Engineering.md)
---

### ✅ Day 9 — The Context Engineering Stack — 6 Layers

You've heard about RAG. You've implemented retrieval. But your system still feels brittle — queries that should work don't, the agent forgets things it shouldn't, and adding new tools creates unexpected failures elsewhere.

[→ Read Day 9](./Phase1_WHY/Day-09-The-Context-Engineering-Stack.md)
---

### ✅ Day 10 — The Model Matters Less Than You Think

For years — including through my own production deployments in financial markets — the instinct was to blame the model when something went wrong. The model hallucinated. The model was inconsistent. We need a better model.

[→ Read Day 10](./Phase1_WHY/Day-10-The-Model-Matters.md)
---

### ✅  Day 11 — The Context Engineering Lifecycle — From Raw Data to Model Response

This day traces the complete journey a piece of data takes from its origin in your systems to the moment an AI model reasons over it. Seven stages. Each one with distinct responsibilities, failure modes, and engineering decisions. Each one dependent on the stages before it.

[→ Read Day 11](./Phase1_WHY/Day-11-Context-Engineering-LifeCycle.md)
---

### ✅ Day 12 — Personal experience

The reason financial markets produce unusually rigorous engineers is that the cost of context failures is immediate, precise, and unambiguous. You don't get "the output quality degraded slightly." You get a number: this trade lost £X, this regulatory breach cost €Y, this latency spike caused Z milliseconds of wrong decisions.

[→ Read Day 12](./Phase1_WHY/Day-12-personal-experience.md)
---

### ✅ Day 13 — The Anatomy of a World-Class System Prompt

A system prompt is not a set of instructions. It is a contract between you and the model — defining who the AI is, what it must do, what context it will receive, what it must never do, and what its output must look like. Missing any clause breaks the contract.

[→ Read Day 13](./Phase1_WHY/Day-13-The-System-Prompt.md)
---

### ✅ Day 14 — The Anatomy of a World-Class System Prompt

Context debt is like ignoring oil changes. Fine for the first 1,000 miles. Damaging at 50,000. Catastrophic at 100,000. And the engine looked completely normal the entire time.

[→ Read Day 14](./Phase1_WHY/Day-14-The-Context_debt.md)
---

### ✅ Day 15 — The 10 Laws of Context Engineering

These 10 laws are like the periodic table of context engineering. Not everything is on the table, but everything important is. Master these and you can build any context system.

[→ Read Day 15](./Phase1_WHY/Day-15-Context-Engineering-laws.md)
---

### ✅ Day 16 — System Prompts Are Architecture Documents

Every AI system has a foundation. It's not the model. It's not the data. It's the **System Prompt**. 📋🧠 This 200-word block of text defines every decision the model will make. Get it wrong, and you have hallucinations. Get it right, and you have production-grade reasoning

[→ Read Day 16](./Phase2_Architecture/Day-16-System-Prompts.md)
---

### ✅ Day 17 — The 6 Prompt Patterns Every AI Engineer Must Know

Zero-shot, few-shot, chain-of-thought, role, template, and meta-prompts. When to use each.

[→ Read Day 17](./Phase2_Architecture/Day-17-6-Prompt-Patterns.md)
---

### ✅ Day 18 — Chain-of-Thought as Context Scaffolding

CoT doesn't change the model. It changes the **context structure** that guides the model's output generation.

[→ Read Day 18](./Phase2_Architecture/Day-18-Chain-of-thoughts.md)
---

### ✅ Day 19 — Few-Shot Examples: The Most Underused Context Tool

Few-shot learning in language models is a form of **in-context learning**. The model doesn't update its weights. Instead, it uses the examples as additional context to predict the next token.

[→ Read Day 19](./Phase2_Architecture/Day-19-few-shot-example.md)
---

### ✅ Day 20 — Negative Space in Prompts: What NOT to Include

The shortest clear prompt beats the longest rambling one every time. Mastery is subtraction, not addition.

[→ Read Day 20](./Phase2_Architecture/Day-20-Negative-space-in-prompts.md)
---

### ✅ Day 21 — Role Prompting vs Persona Prompting

Role is what you do. Persona is who you are. One is architecture. One is fiction. Only architecture scales.

[→ Read Day 21](./Phase2_Architecture/Day-21-role-persona-prompting.md)
---

### ✅ Day 22 — Instruction Hierarchy: System > User > Assistant

System is law. User is input. Assistant is reasoning. System always wins. Understanding this hierarchy prevents chaos and injection attacks.

[→ Read Day 22](./Phase2_Architecture/Day-22-Instruction-Hierarchy.md)
---


### ✅ Day 23 — Prompt Versioning: Treating Prompts as Production Code

We will be covering more detail about If you can't rollback a prompt change, you shouldn't deploy it. Version control prevents disasters & Why Prompts Need Versioning.

[→ Read Day 23](./Phase2_Architecture/Day-23-Prompt-versioning.md)
---


### ✅ Day 24 — Context Compression Techniques

You have a 500-page document and a 200K token window. Compression is the art of keeping signal while cutting noise.

[→ Read Day 24](./Phase2_Architecture/Day-24-Context-Compressions.md)
---


### ✅ Day 25 — Recap of Phase 2

Recap of Phase 2 - Structure beats improvisation. Patterns beat chaos. From theory to practice, from patterns to production

[→ Read Day 25](./Phase2_Architecture/Day-25-Phase2-recap.md)
---

### 🔥 Day 26 — Why Static Context Fails at Scale

Static context is like reading yesterday's newspaper to make today's trades. Dynamic context is having a Bloomberg terminal connected to your brain

[→ Read Day 26](./Phase3_Local_First/Day-26-Why-Context-fails.md)
---

### 🔜 Day 27 — Why Static Context Fails at Scale

We will be covering more detail about Long-Term Memory: Persistent Knowledge

*Follow on [LinkedIn](https://www.linkedin.com/in/logeswarangv/) or ⭐ star this repo to be notified.*


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

![Progress](https://img.shields.io/badge/Day26%20of%20100-In%20Progress-orange?style=for-the-badge)
*Series started April 2026 · Updated daily*
