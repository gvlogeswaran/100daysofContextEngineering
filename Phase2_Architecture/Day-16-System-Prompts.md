# Day 16 — System Prompts Are Architecture Documents

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"Your system prompt is your infrastructure layer. One tight system prompt ensures consistent decisions and audit trail clarity."*

---

### System Prompts Are Architecture Documents

---

#### 1. What Makes a System Prompt an Architecture Document?

A **system prompt** is not just text that guides behavior. It's the formal specification of how your AI system will behave under all conditions.

Like an architecture document in traditional software:
- It defines interfaces (what inputs are accepted)
- It specifies constraints (what outputs are valid)
- It draws boundaries (what the system will and won't do)
- It handles error states (what happens when rules are violated)

The difference: traditional docs are read by developers. System prompts are read by the model during every single inference.

---

#### 2. Component #1: Role Definition

**What it does:** Anchors the model to a specific persona and expertise level.

**Why it matters:** Models are pattern-matching machines. Without a clear role, they revert to their training distribution (generic, helpful, often hallucinating).

**Example — Bad:**
```
You are a helpful assistant.
```

**Example — Good:**
```
You are a financial risk analyst specializing in pre-trade compliance. 
You have 15 years of experience in commodities and FX markets. 
Your job is to validate trade orders against regulatory constraints 
before they reach execution venues.
```

The second version constrains the model's reasoning to domain-specific patterns it learned during training.

---

#### 3. Component #2: Output Constraints

**What it does:** Defines the exact format and structure of all outputs.

**Why it matters:** Ambiguous output formats force downstream parsing logic to be defensive (slow). Explicit formats enable strict validation (fast).

**Example — Bad:**
```
Provide your analysis.
```

**Example — Good:**
```
You MUST respond with valid JSON only. No markdown, no code blocks, no explanations.
Schema: { "decision": "APPROVED"|"DENIED", "rules_checked": [string], "violations": [object] }
If input violates any rule, set decision to "DENIED" and list the violation.
```

This forces the model to structure its reasoning into the exact shape your downstream system expects.

---

#### 4. Component #3: Domain Knowledge

**What it does:** Embeds critical facts, rules, and edge cases directly into the system prompt.

**Why it matters:** Models are only as good as their training data. For specialized domains (financial compliance, medical protocols), you must explicitly state the rules.

**Example:**
```
Compliance Rules (Non-Negotiable):
- Rule A124: Position size cannot exceed $2M per customer per day.
- Rule B305: FX trades must settle T+2, not T+1.
- Rule C401: If counterparty has credit limit < $500K, flag for manual review.

Edge Cases:
- Portfolio hedges are exempt from Rule A124.
- Intra-day position netting is allowed under Rule B305.
```

This is your "training data injection" without retraining.

---

#### 5. Component #4: Error Handling

**What it does:** Specifies how the model should handle ambiguous, contradictory, or invalid inputs.

**Why it matters:** In production, incomplete data is common. Graceful degradation (refuse cleanly) beats confident hallucinations (guess incorrectly).

**Example:**
```
Error Handling Rules:
- If a required field is missing, return: { "decision": "REJECTED", "error": "Missing field: [name]" }
- If you cannot determine a rule's applicability, return: { "decision": "ESCALATE", "reason": "Ambiguous case" }
- NEVER guess the missing data. NEVER assume default values.
```

---

#### 6. Component #5: Scope Limits

**What it does:** Explicitly states what the system CAN do and what it CANNOT do.

**Why it matters:** Models often overstep boundaries when not explicitly constrained. A trader system that starts making position adjustments (instead of just analyzing) is dangerous.

**Example:**
```
ALLOWED Actions:
- Analyze trade orders against compliance rules
- Flag violations
- Suggest corrective modifications (you do NOT execute them)

FORBIDDEN Actions:
- Execute trades
- Modify positions
- Contact counterparties
- Override compliance flags
```

---

#### 7. Before/After Production Example

**Before (No Architecture):**
```
User: Analyze this trade order.
Model: This trade looks fine. Go ahead and execute it.
Downstream System: [Executes the trade, which violates a compliance rule]
Result: Regulatory violation. Cost: $500K fine.
```

**After (System Prompt as Architecture):**
```
[System Prompt constrains model to output structured JSON with compliance checks]

User: Analyze this trade order.
Model: { "decision": "DENIED", "violation": "Rule C401: Counterparty credit limit exceeded" }
Downstream System: [Blocks execution, escalates to compliance team]
Result: Risk prevented. Audit trail clear.
```

---

#### Key Terms for Day 16
| Term | What It Means |
|------|-----------|
| **System Prompt** | The foundational instruction that shapes all model behavior before user input. |
| **Architecture Document** | A formal specification of system constraints, interfaces, and behavior. |
| **Role Grounding** | Anchoring the model to a specific expertise to reduce hallucination. |
| **Output Schema** | The exact structure (format, types, fields) of all model outputs. |
| **Domain Injection** | Embedding critical domain rules directly into the prompt to avoid training requirements. |
| **Scope Boundary** | Explicit statements of what the system will and will not do. |

#### Official References
- Anthropic's System Prompt Best Practices → https://docs.anthropic.com/en/docs/build-a-Claude-app/system-prompts-best-practices
- Claude Documentation on Prompt Engineering → https://docs.anthropic.com/en/docs/introduction

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 15](./Day-15-Context-Engineering-laws.md) | [Day 17 →](./Day-17-6-Prompt-Patterns.md)
