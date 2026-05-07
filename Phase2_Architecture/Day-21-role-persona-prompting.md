# Day 21 — Role Prompting vs Persona Prompting

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"Role is what you do. Persona is who you are. One is architecture. One is fiction. Only architecture scales."*

---

### Role Prompting vs Persona Prompting: Architecture vs. Fiction

---

#### 1. The Fundamental Distinction

**Role Prompting:** Specifying the **function** the model will perform.
```
"You are a validator."
"You check trades for compliance."
"You return JSON decisions."
```

**Persona Prompting:** Specifying the **backstory** or **expertise level** of the model.
```
"You are a derivatives expert with 20 years of experience."
"You understand market dynamics deeply."
"You have mentored senior traders."
```

One is **architecture**. The other is **fiction**. Only architecture scales.

---

#### 2. Why Persona Prompting Causes Hallucinations

Persona prompting works by the following chain of inference:

```
Step 1: "I am an expert"
Step 2: [Model learns this from training: experts make predictions about related topics]
Step 3: "I should invent expert-level predictions, even if not asked"
Step 4: [Model hallucinates detailed advice it was never asked for]
```

Example:

```
Persona: "You are a senior trading strategist with 30 years of experience 
in commodities futures."

User: "Should I buy oil futures?"

Model's reasoning chain:
- I am a trading strategist
- Trading strategists make market predictions
- The user asked about oil futures
- A good strategist would have an opinion
- Let me invent a detailed market analysis with price targets...

Output: "Yes, buy oil. Target $95/bbl by Q3 2025. Catalyst: OPEC 
production cuts. Position size: 5 contracts. Stop loss: $78/bbl."

Problem: The model invented all of this. It was never asked for predictions.
It was asked a yes/no question. But the persona gave it permission to hallucinate.
```

---

#### 3. Why Role Prompting Prevents Hallucinations

Role prompting works by **bounding the function**:

```
"You validate trades against Rule B305."

Boundaries established:
- Input: trades
- Process: check against one specific rule
- Output: APPROVED or DENIED (binary)
- Scope: Rule B305 only, nothing else
```

Now the model has nowhere to hallucinate:

```
User: "Check this trade against Rule B305."

Model's reasoning:
- My role is to check against Rule B305
- Rule B305 says position limit is $2M per counterparty
- This trade is $2.5M
- Trade violates the rule
- I should say DENIED
- I should NOT invent additional rules or recommendations
- I should NOT provide commentary beyond the rule check

Output: { "decision": "DENIED", "violation": "Position limit exceeded" }

Result: Tight, bounded, auditable. No hallucination.
```

---

#### 4. The Production Pattern: Role + Bounded Scope

Real production systems separate role from persona:

**Bad (Persona Mixed In):**
```
"You are an experienced compliance officer who has worked at JPMorgan 
for 15 years and understands the nuances of derivatives regulation. 
You have a keen eye for spotting market manipulation. You're conservative 
in your assessments."
```

This invites the model to:
- Apply "nuance" beyond what's specified
- Look for manipulation patterns (which may not be in the data)
- Apply "conservative" judgment (which is undefined)

**Good (Role Only):**
```
"You are a compliance validator. Your job: check trades against 
Rule B201 (settlement date), Rule B305 (position limit), and Rule C401 
(credit limit). Output JSON: { decision, violations }. Do NOT apply judgment 
beyond these rules. Do NOT invent additional checks."
```

This constrains the model to:
- Only check the 3 specified rules
- Return structured output
- Never add extra checks or judgments

---

#### 5. Real Financial Example: The Contrast

**Scenario:** Validating a cross-border wire transfer.

**Persona Approach (Fails):**
```
System: "You are an international banking expert with 30 years of 
experience handling complex wire transfers. You understand geopolitical 
risks, sanctions regimes, and market dynamics deeply. You have 
instincts about when something smells off."

Wire: $50M to a Russian entity, for "consulting services"

Model response:
"This wire looks suspicious to me. While technically it's between 
sanctioned and non-sanctioned entities, I have a gut feeling this 
could be sanctions evasion. I'd recommend blocking it. Also, this 
amount is unusual for consulting. Red flag. Also, the time of day 
this was submitted (3 PM) suggests trying to hide it before close..."

Problems:
- Invented "gut feeling" evidence
- Made risk inferences beyond what's specified
- Applied undocumented judgment
- Could create compliance liability if blocking a legal transfer
```

**Role Approach (Works):**
```
System: "You validate wires against Rule B501 (sanctions list), 
Rule B502 (beneficiary country), and Rule C301 (wire amount limit $10M 
per entity). Output JSON: { decision, violations }. You apply ONLY 
these 3 rules. You do NOT apply judgment about intentions, timing, or 
patterns."

Wire: $50M to a Russian entity, for "consulting services"

Model response:
{
  "decision": "DENIED",
  "violations": [
    "Rule B502: Russian entity in OFAC sanctions list",
    "Rule C301: Wire amount $50M exceeds limit $10M"
  ]
}

Result:
- Clear, rule-based decision
- Auditable (violations explicitly listed)
- No invented judgment
- Compliant with corporate policy
```

---

#### 6. When Persona Can Work (Rarely)

There are exactly two cases where persona is acceptable in production:

**Case 1: Persona Reinforces Role**
```
Role: "You validate trades for compliance."
Persona addition: "You are a compliance analyst." (reinforces the role)

This is OK because persona and role are aligned. The persona doesn't add 
new capabilities or suggest new behaviors. It just reinforces the role.

Compare to:
Role: "You validate trades for compliance."
Persona addition: "You're a senior derivatives trader." (conflicts with role)

This causes hallucination because the persona suggests trading expertise 
the role doesn't call for.
```

**Case 2: Persona Defines Behavior Within Bounded Role**
```
Role: "You explain financial concepts."
Persona addition: "You explain at a 5th-grade reading level."

The persona here defines HOW to perform the role (reading level), not 
whether to expand the role. This is acceptable.

Compare to:
Role: "You explain financial concepts."
Persona addition: "You explain while sharing your personal trading stories."

This expands the role beyond what's specified. Unacceptable.
```

---

#### 7. The Template for Production Role Prompts

**Structure:**
```
1. ROLE DEFINITION (Function, not backstory)
   "You validate trades against [specific rules]."
   "You do NOT [prohibited behaviors]."

2. INPUT SPECIFICATION (What data you receive)
   "Input: Trade object with fields [list]."

3. PROCESSING RULES (Explicit logic)
   "Check each field against Rule X. If violation found, flag it."

4. OUTPUT SPECIFICATION (Exact format)
   "Output JSON: { decision: APPROVED|DENIED|ESCALATE, violations: [...] }"

5. SCOPE BOUNDARIES (What you will NOT do)
   "You do NOT make trading recommendations."
   "You do NOT consider market conditions."
   "You do NOT interpret ambiguous rules."
```

**Example:**
```
You are a compliance validator.

Role: Check trades against Rule B201 (settlement date T+2), 
Rule B305 (position limit $2M), Rule C401 (credit limit check).

Input: Trade with fields { symbol, size, counterparty, settlement_date }

Processing:
1. If settlement_date ≠ T+2 from today, flag Rule B201 violation.
2. If size + current_position > $2M, flag Rule B305 violation.
3. If counterparty credit score < 700, flag Rule C401 violation.

Output: JSON only
{ 
  "decision": "APPROVED" | "DENIED" | "ESCALATE",
  "violations": [array of violation strings],
  "rules_checked": ["B201", "B305", "C401"]
}

Prohibitions:
- Do NOT make trading recommendations
- Do NOT interpret rules beyond their text
- Do NOT consider market conditions
- Do NOT override rules for any reason
```

---

#### 8. Measuring Role vs. Persona Effectiveness

```python
def compare_role_vs_persona(test_trades, ground_truth_decisions):
    """Compare persona prompting vs role prompting accuracy."""
    
    persona_prompt = """You are a derivatives expert with 30 years 
    on Wall Street. You understand market dynamics and nuance."""
    
    role_prompt = """You validate trades against Rule B305 
    (position limit $2M per counterparty). Return JSON: 
    { decision: APPROVED|DENIED, violation: string }"""
    
    # Test persona approach
    persona_results = []
    for trade in test_trades:
        result = call_claude(persona_prompt + f"\nValidate: {trade}")
        persona_results.append(result)
    
    # Test role approach
    role_results = []
    for trade in test_trades:
        result = call_claude(role_prompt + f"\nValidate: {trade}")
        role_results.append(result)
    
    # Compare
    persona_accuracy = measure_accuracy(persona_results, ground_truth_decisions)
    role_accuracy = measure_accuracy(role_results, ground_truth_decisions)
    
    print(f"Persona approach: {persona_accuracy:.2%} accuracy, {count_hallucinations(persona_results)} hallucinations")
    print(f"Role approach: {role_accuracy:.2%} accuracy, {count_hallucinations(role_results)} hallucinations")
    
    # Typical result:
    # Persona approach: 73% accuracy, 18 hallucinations (invented rules, recommendations)
    # Role approach: 96% accuracy, 0 hallucinations
```

---

#### Key Terms for Day 21
| Term | What It Means |
|------|-----------|
| **Role** | A functional specification of what the model will do (validate, check, extract, format). |
| **Persona** | A fictional backstory or expertise level (years of experience, credentials, judgment). |
| **Scope Boundary** | Explicit limits on what the model will and won't do. |
| **Hallucination** | Confident generation of content that wasn't in the input (caused by persona prompts). |
| **Structured Role** | A role that specifies input, process, and output formats explicitly. |

#### Official References
- Anthropic's Role-Based Prompting → https://docs.anthropic.com/en/docs/build-a-claude-app/prompt-engineering
- Research: How Role Definitions Affect Model Behavior → https://arxiv.org/abs/2302.09457

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 20](./Day-20-MCP-Primitives-Overview.md) | [Day 22 →](./Day-22-Notifications-System.md)
