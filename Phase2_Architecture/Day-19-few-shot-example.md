# Day 19 — Few-Shot Examples: The Most Underused Context Tool

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"Two or three well-chosen examples can shift model behavior by 40-50%. Few-shot is pattern injection at inference time."*

---

### Few-Shot Examples: Pattern Injection at Inference Time

---

#### 1. What Makes Few-Shot Work

Few-shot learning in language models is a form of **in-context learning**. The model doesn't update its weights. Instead, it uses the examples as additional context to predict the next token.

This happens because:
- Training data contains countless examples of "here's a pattern, here's an instance of that pattern"
- The model learned to recognize this meta-pattern
- When you provide examples, it pattern-matches to its training

Few-shot isn't magic. It's inference-time pattern compression.

---

#### 2. The Selection Problem: Not All Examples Are Equal

**Bad Example Selection:**
```
Trade classification task:
Example 1: EUR trade → "cross-border" [Generic]
Example 2: JPY trade → "cross-border" [Doesn't add information]
Example 3: GBP trade → "cross-border" [Redundant]

Problem: All examples are the same pattern. Model hasn't learned the contrast.
```

**Good Example Selection:**
```
Example 1: USD/EUR → "cross-border" [Currency difference is key signal]
Example 2: USD/USD domestic settlement → "domestic" [Same currency matters]
Example 3: EUR/GBP both in EU → "cross-border" [Location doesn't matter, currency does]

Problem: Now the model has learned WHERE the decision boundary lies.
```

---

#### 3. The 3-Tier Few-Shot Selection Framework

**Tier 1: Anchor Example (Unambiguous)**
The simplest, most obvious case. Establishes the basic pattern.

```json
{
  "example": "Currency: USD/EUR, Counterparty: London-based bank",
  "classification": "CROSS_BORDER",
  "reasoning": "Different currencies = cross-border by definition"
}
```

**Tier 2: Edge Case (Ambiguous but Correct)**
A case that looks like it might go either way, but has a clear right answer.

```json
{
  "example": "Currency: USD/USD, Counterparty: Toronto-based bank, Settlement: T+2",
  "classification": "DOMESTIC",
  "reasoning": "Same currency = domestic, even though counterparty is in different country"
}
```

**Tier 3: Boundary Case (Critical)**
A case that tests whether the model really learned the pattern or just memorized examples.

```json
{
  "example": "Currency: EUR/GBP, Both counterparties in EU, Settlement: T+2",
  "classification": "CROSS_BORDER",
  "reasoning": "Different currencies = cross-border, regardless of geographic proximity"
}
```

These three examples teach:
- The primary signal (currency difference)
- A counter-signal that doesn't matter (counterparty location)
- A boundary case that confirms the pattern

---

#### 4. Dynamic Few-Shot Retrieval in Production

Hardcoding examples is fine for static tasks. But in financial markets, context changes:
- Today's edge case is tomorrow's common case
- New regulatory rules create new edge cases
- Market conditions change which examples are most relevant

**Dynamic Few-Shot Pattern:**

```python
def classify_trade_with_dynamic_examples(trade_data):
    # Step 1: Determine the trade's "type" (FX, Derivative, Commodity, etc.)
    trade_type = classify_trade_type(trade_data)
    
    # Step 2: Retrieve the most relevant examples for THIS trade type
    examples = retrieve_examples_for_type(trade_type, k=3)
    
    # Step 3: Optionally, rank examples by similarity to the current trade
    # (e.g., if it's a tricky edge case, prioritize edge-case examples)
    if is_edge_case(trade_data):
        examples = prioritize_edge_cases(examples)
    
    # Step 4: Build the prompt with dynamic examples
    prompt = f"""
    You are classifying trade orders. Here are recent examples of correct classifications:
    
    {format_examples(examples)}
    
    Now classify this trade: {trade_data}
    """
    
    return call_claude(prompt)

def retrieve_examples_for_type(trade_type, k=3):
    """Retrieve from a database the k most relevant examples for this trade type."""
    # This could be a vector DB, a traditional DB, or a simple in-memory index
    # The point: examples are not hardcoded. They're selected dynamically.
    
    examples = [
        {
            "trade": "EUR/USD spot, settlement T+2",
            "classification": "CROSS_BORDER",
            "rules_checked": ["Rule B201: Multi-currency trades are cross-border"]
        },
        {
            "trade": "USD/USD with Canadian counterparty",
            "classification": "DOMESTIC",
            "rules_checked": ["Rule B201: Same currency = domestic"]
        },
        {
            "trade": "EUR/GBP both EU entities",
            "classification": "CROSS_BORDER",
            "rules_checked": ["Rule B201: Currency difference overrides geography"]
        }
    ]
    return examples[:k]
```

---

#### 5. Example Format: What Should an Example Contain?

**Minimal Format (Token-Efficient):**
```
Trade: EUR/USD | Settlement: T+2 → Classification: CROSS_BORDER
Trade: USD/USD | Settlement: T+2 → Classification: DOMESTIC
```

**Optimal Format (Maximum Information):**
```
Trade: {
  "symbol": "EUR/USD",
  "amount": "2M",
  "settlement": "T+2",
  "counterparty_location": "London"
}
Classification: CROSS_BORDER
Key Signal: Multi-currency (EUR/USD)
Rule Applied: Rule B201 (multi-currency trades are always cross-border)
```

**Production Format (Auditable):**
```
Example 1:
  Input: EUR/USD spot trade, $2M, counterparty in London
  Expected Output: CROSS_BORDER
  Rule Justification: Rule B201 § Multi-currency trades are cross-border
  Confidence: 95% (clear, unambiguous case)
  Last Updated: 2025-04-11
  Example ID: EXAMPLE_FX_001
```

---

#### 6. Few-Shot for Financial Market Compliance

**Real Scenario:** Validating trade orders for market manipulation detection.

**Task:** Classify trades as "NORMAL", "SUSPICIOUS", or "ESCALATE"

**Without Examples:**
```
Prompt: "Is this trade suspicious?"
Model: [Generates reasonable but inconsistent classifications]
Accuracy: ~60%
```

**With 3 Well-Chosen Examples:**
```
Example 1 (Anchor - Normal):
Input: 1000 shares of Apple at market price, executed during normal hours
Output: NORMAL
Reasoning: Market-price execution, normal volume, normal timing

Example 2 (Edge Case - Suspicious):
Input: 10,000 shares of Apple at 10% above market, executed at market close
Output: SUSPICIOUS  
Reasoning: Above-market price, unusual volume surge, timing at close (possible manipulation)

Example 3 (Boundary - Escalate):
Input: 5,000 shares of Apple at 2% above market, executed 5 minutes before earnings
Output: ESCALATE
Reasoning: Slightly elevated price, elevated volume, suspicious timing, requires manual review

Now classify: 3000 shares of Tesla at 3% above market, executed during power outage
```

With these examples, the model understands:
- Normal market activity
- The price/volume/timing triad
- When to escalate vs. when to flag

Result: 88% accuracy, 95% confidence, auditable decisions.

---

#### 7. Common Few-Shot Mistakes

**Mistake 1: Too Many Examples**
```
Bad: 10 examples (confuses the model, burns tokens)
Good: 2-4 examples (clear pattern)
```

**Mistake 2: Examples Without Reasoning**
```
Bad: "Trade A → CROSS_BORDER" (model doesn't learn WHY)
Good: "Trade A → CROSS_BORDER (Rule: Multi-currency trades are always cross-border)"
```

**Mistake 3: Homogeneous Examples**
```
Bad: All examples are the same type of trade (model doesn't learn the decision boundary)
Good: Examples that span different trade types and edge cases
```

**Mistake 4: Outdated Examples**
```
Bad: Using examples from 2023 when market rules changed in 2024
Good: Periodically refresh examples based on new rules and edge cases
```

---

#### 8. Measuring Few-Shot Effectiveness

```python
def measure_few_shot_impact(task_data, true_labels):
    # Baseline: Zero-shot (no examples)
    zero_shot_accuracy = evaluate_zero_shot(task_data, true_labels)
    
    # With 3 examples
    few_shot_accuracy = evaluate_few_shot(task_data, true_labels, k=3)
    
    # Impact
    improvement = few_shot_accuracy - zero_shot_accuracy
    token_cost = count_tokens_in_examples(k=3)
    
    print(f"Zero-shot: {zero_shot_accuracy:.2%}")
    print(f"Few-shot:  {few_shot_accuracy:.2%}")
    print(f"Improvement: {improvement:.2%}")
    print(f"Token cost: {token_cost}")
    
    # ROI: If token cost is low and improvement is high, few-shot wins
    if improvement > 0.10 and token_cost < 200:
        print("FEW-SHOT RECOMMENDED")
```

---

#### Key Terms for Day 19
| Term | What It Means |
|------|-----------|
| **Few-Shot Learning** | Providing 2-5 examples to demonstrate a pattern before asking the model to solve a new instance. |
| **In-Context Learning** | The model's ability to learn patterns from examples in the prompt without updating weights. |
| **Example Selection** | Choosing examples that span the decision boundary and edge cases, not just common cases. |
| **Dynamic Retrieval** | Selecting examples at inference time based on the current task context (not hardcoded). |
| **Signal vs. Noise** | In example selection, focusing on the features that matter (signal) and showing counter-examples (noise). |

#### Official References
- Brown et al. "Language Models are Few-Shot Learners" → https://arxiv.org/abs/2005.14165
- Prompt Engineering Guide (Few-Shot) → https://www.promptingguide.ai/techniques/fewshot

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 18](./Day-18-Capability-Negotiation.md) | [Day 20 →](./Day-20-MCP-Primitives-Overview.md)
