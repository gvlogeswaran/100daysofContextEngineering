# Day 17 — The 6 Prompt Patterns Every AI Engineer Must Know

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"Patterns beat improvisation. Ninety-eight percent of production use cases fit into one of six core patterns."*

---

### The 6 Prompt Patterns: A Classification Framework

---

#### 1. Zero-Shot Prompting

**Definition:** Asking the model to perform a task with no examples, relying entirely on its training data.

**Mechanics:**
```
User: "What is the counterparty risk of this derivatives trade?"
Model: [Reasons from training data alone]
```

**When to use:**
- The concept is well-defined in training data (standard financial terms, common procedures)
- Speed is critical (latency matters more than 100% accuracy)
- The task has low ambiguity

**Strengths:**
- Zero token overhead
- Fastest inference
- Simplest to implement

**Weaknesses:**
- High hallucination on edge cases
- Inconsistent reasoning on ambiguous inputs
- No control over reasoning process

**Cost profile:** Minimal (1 query, no examples)

---

#### 2. Few-Shot Prompting

**Definition:** Providing 2-5 representative examples before asking the model to solve a new instance of the same task.

**Mechanics:**
```
User: "Classify these trades:
Example 1: [Trade A] -> Classification: [Correct Classification A]
Example 2: [Trade B] -> Classification: [Correct Classification B]
Example 3: [Trade C] -> Classification: [Correct Classification C]
Now classify: [New Trade]"

Model: [Applies pattern from examples to new trade]
```

**When to use:**
- The task has subtle distinctions that aren't obvious from the concept alone
- You have a small set of high-quality examples (2-5 is optimal)
- Accuracy is more important than speed
- The task involves domain-specific judgment

**Strengths:**
- Dramatically improves accuracy (often 15-40% improvement over zero-shot)
- Provides implicit "teaching" without retraining
- Examples can demonstrate edge case handling

**Weaknesses:**
- Example quality matters enormously (bad examples mislead)
- Token cost increases proportionally
- Examples must be representative of the distribution

**Cost profile:** Moderate (examples + main query)

**Python example:**
```python
def few_shot_compliance_check(new_trade):
    examples = [
        {"trade": "EUR 1M sell", "result": "T+2 settlement required"},
        {"trade": "GBP 500K buy", "result": "T+2 settlement required"},
        {"trade": "commodity future", "result": "T+1 settlement acceptable"}
    ]
    
    prompt = f"""Here are compliance examples:
{examples}

Now check this trade: {new_trade}"""
    
    return claude.messages.create(
        model="claude-opus",
        messages=[{"role": "user", "content": prompt}]
    )
```

---

#### 3. Chain-of-Thought (CoT) Prompting

**Definition:** Explicitly instructing the model to break down reasoning into sequential steps before producing the final answer.

**Mechanics:**
```
User: "Analyze this trade. First, extract the size. Then check Rule B305. 
Then determine if an override is needed. Then return the decision."

Model: [Step 1: Extract size = 2.5M]
        [Step 2: Check Rule B305 -> position limit is 2M]
        [Step 3: Size exceeds limit -> override required]
        [Step 4: Decision = ESCALATE]
```

**When to use:**
- The task involves multi-step reasoning
- Errors early in the chain cascade (e.g., wrong size leads to wrong compliance decision)
- You need to audit the reasoning trail
- The model tends to "jump to conclusions"

**Strengths:**
- Forces deliberate, step-by-step reasoning
- Makes errors transparent and auditable
- Dramatically improves complex reasoning (50-80% improvement on multi-step tasks)

**Weaknesses:**
- Token cost increases (reasoning steps are explicit)
- Model can still miss steps or reason incorrectly within steps
- Requires clear step structure in prompt

**Cost profile:** Moderate to high (explicit reasoning steps)

**Python example:**
```python
def chain_of_thought_analysis(trade_input):
    prompt = f"""Analyze this trade for compliance. Follow these steps exactly:

Step 1: Extract the trade attributes (size, symbol, counterparty, settlement_date).
Step 2: Look up the position limit for this counterparty.
Step 3: Check if size exceeds the position limit.
Step 4: If exceeded, check if a portfolio hedge exemption applies.
Step 5: Determine the final decision (APPROVED, DENIED, or ESCALATE).
Step 6: Return your answer in JSON format with all steps shown.

Trade: {trade_input}"""

    return claude.messages.create(
        model="claude-opus",
        messages=[{"role": "user", "content": prompt}]
    )
```

---

#### 4. Role Prompting

**Definition:** Anchoring the model to a specific persona or expertise level before asking the question.

**Mechanics:**
```
User: "You are a compliance officer with 20 years of experience in derivatives trading.
Analyze this trade order."

Model: [Reasons from the implicit knowledge of that role]
```

**When to use:**
- You need domain-specific judgment
- Generic reasoning would miss important context
- You want to bias the model toward conservative/defensive reasoning

**Strengths:**
- Focuses model reasoning on domain patterns
- Reduces generic/unhelpful answers
- Low token overhead

**Weaknesses:**
- Effectiveness depends on how well the role is known in training data
- Vague roles are ineffective ("helpful assistant" doesn't work)
- Model may overstate expertise

**Cost profile:** Minimal (just role definition)

**Example — Bad role:**
```
"You are a helpful assistant. Analyze this trade."
```

**Example — Good role:**
```
"You are a compliance officer specializing in post-trade settlement verification. 
You have 15 years of experience handling edge cases in FX and commodity markets. 
Your job is to identify settlement risk before trades reach the clearing house."
```

---

#### 5. Template Prompting

**Definition:** Specifying the exact output structure (schema, format, fields) before asking the model to solve the task.

**Mechanics:**
```
User: "For each trade, return ONLY this JSON structure:
{
  'symbol': string,
  'size': number,
  'decision': 'APPROVED|DENIED|ESCALATE',
  'violations': [string],
  'confidence': 0-100
}

Analyze: [Trade]"

Model: [Returns exactly that structure]
```

**When to use:**
- Downstream systems need consistent structure
- Parsing errors are expensive
- You want to prevent markdown, explanations, or chat-like responses

**Strengths:**
- Forces structured output
- Enables strict validation
- Integrates seamlessly with downstream automation

**Weaknesses:**
- Template must be flexible enough for all cases
- Model may struggle if input doesn't fit template
- Requires error handling for malformed JSON

**Cost profile:** Minimal (just structure definition)

---

#### 6. Meta-Prompting

**Definition:** Having the model generate its own prompt structure for a task, then solve the task using that structure.

**Mechanics:**
```
User: "You're about to analyze trades. First, generate the ideal prompt 
structure for analyzing [trade type]. Then analyze this trade."

Model: [Generates optimal CoT structure for trade type]
        [Then applies it to the trade]
```

**When to use:**
- Task type varies significantly (different trade types need different analyses)
- You want the model to adapt its reasoning to input characteristics
- You're working with strong models that can handle self-reflection

**Strengths:**
- Maximum flexibility
- Model adapts structure to input type
- Sophisticated reasoning on novel problems

**Weaknesses:**
- Higher token cost (meta-reasoning overhead)
- Only works well with strong models (Claude Opus, not entry-level)
- Complex to debug

**Cost profile:** High (meta-reasoning + main task)

---

#### 7. Combining Patterns in Production

Real systems rarely use a single pattern. They stack them:

**Example Stack for Financial Compliance:**
1. System Prompt (Day 16) = Role Prompting + Constraints
2. Few-Shot Examples = Domain-specific edge cases
3. Chain-of-Thought = Multi-step compliance checks
4. Template Output = Structured JSON for downstream automation

```python
def production_compliance_analysis(trade):
    system_prompt = """You are a compliance analyst specializing in pre-trade 
    regulatory validation. Never guess. Always cite the rule."""
    
    few_shot_examples = """
    Example 1: A $1M EUR trade. Rule: FX trades must be T+2. Decision: APPROVED.
    Example 2: A $2.5M trade, limit is $2M. Rule: Position limit exceeded. Decision: DENIED.
    """
    
    cot_instruction = """First extract the trade attributes. Then check each rule. 
    Then determine the decision. Then return JSON."""
    
    template = """Return only: {"decision": "...", "violations": [...]}"""
    
    prompt = f"{system_prompt}\n{few_shot_examples}\n{cot_instruction}\n{template}"
    
    return claude.messages.create(
        model="claude-opus",
        system=system_prompt,
        messages=[{"role": "user", "content": prompt + f"\nTrade: {trade}"}]
    )
```

---

#### Key Terms for Day 17
| Term | What It Means |
|------|-----------|
| **Zero-Shot** | A single request with no examples, relying on training data alone. |
| **Few-Shot** | A request with 2-5 examples demonstrating the desired pattern. |
| **Chain-of-Thought** | Explicit step-by-step reasoning before the final answer. |
| **Role Prompting** | Anchoring the model to a specific expertise level or persona. |
| **Template Prompting** | Specifying exact output structure before asking the task. |
| **Meta-Prompting** | Having the model generate its own reasoning structure before solving. |

#### Official References
- Wei et al. Chain of Thought Prompting (2022) → https://arxiv.org/abs/2201.11903
- Anthropic Guide to Prompting → https://docs.anthropic.com/en/docs/build-a-claude-app/prompt-engineering

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 16](./Day-16-JSON-RPC-2-The-Protocol-Language.md) | [Day 18 →](./Day-18-Capability-Negotiation.md)
