# Day 20 — Negative Space in Prompts: What NOT to Include

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"The shortest clear prompt beats the longest rambling one every time. Mastery is subtraction, not addition."*

---

### Negative Space in Prompts: The Art of Omission

---

#### 1. The Problem: Context Pollution

Language models are trained on internet text. Internet text is noisy, hedging, verbose, and repetitive.

When you write a prompt like:
```
"Hey, I was wondering if you could help me check if this trade 
is maybe okay? I mean, I'm not totally sure about all the rules, 
but could you just look at it and see if there are any problems? 
Thanks!"
```

You're training the model to respond in that same tone: uncertain, verbose, hedging.

The model pattern-matches to verbose, uncertain training data. You get verbose, uncertain outputs.

---

#### 2. Types of Noise to Remove

**Type 1: Hedging Language**

Examples of hedging:
```
- "I think...", "maybe...", "possibly..."
- "I'm not sure, but...", "I could be wrong..."
- "Feel free to...", "Don't worry if..."
- "If you don't mind...", "Would you be willing to..."
```

Replace with certainty:
```
REMOVE: "I'm not 100% sure, but could you maybe check if this trade is compliant?"
KEEP:   "Check this trade for Rule B201-B305 compliance."
```

Why: Hedging makes the model uncertain. Certainty makes it crisp.

**Type 2: Explanation of Motivation**

Examples:
```
- "I'm doing this because..."
- "I need this because..."
- "This is important because..."
- "I'm working on a project where..."
```

The model doesn't care why. It only cares what.

```
REMOVE: "I'm working on a financial compliance system because we 
need to reduce risk. Can you check this trade?"

KEEP:   "Check this trade for compliance violations."
```

Why: The model's job isn't to understand your motivation. It's to follow the instruction.

**Type 3: Redundant Constraints**

Examples:
```
- Repeating the same rule three different ways
- Saying the same constraint in the system prompt AND in the user message
- Listing output format twice in different words
```

```
REMOVE:
"Return the decision in JSON format. The JSON should have the 
following structure. Here's the structure again: { decision: ..., 
violations: [...] }. Make sure to use this format. JSON only."

KEEP:
"Return JSON: { decision: APPROVED|DENIED|ESCALATE, violations: [string] }"
```

Why: Redundancy doesn't make constraints clearer. It wastes tokens and confuses priorities.

**Type 4: Task Mixing**

Examples:
```
- Asking the model to do 3 different things in one prompt
- Combining incompatible tasks (e.g., "analyze AND execute")
- Mixing trade types (FX, derivatives, commodities) without branching
```

```
REMOVE:
"Check if this trade is compliant. Also, tell me if I should 
execute it or not. Also, explain what each rule means. Also, 
suggest alternatives if it violates rules."

KEEP:
"Check if this trade is compliant against Rule B201-B305.
Respond: { decision: APPROVED|DENIED|ESCALATE, violations: [...] }"
```

Why: Each task requires different context and different reasoning. Mixing them creates confusion and errors.

**Type 5: Apologetic Language**

Examples:
```
- "Sorry for the complexity..."
- "I know this is a lot..."
- "Thank you for your patience..."
- "I apologize if this is unclear..."
```

None of these appear in high-performing prompts.

```
REMOVE: "Sorry if this is confusing, but could you please check 
if this trade is compliant? Thanks!"

KEEP:   "Check this trade for compliance."
```

Why: Apologetic language weakens authority. The model is less likely to follow uncertain requests confidently.

---

#### 3. The 80/20 Rule: What Stays, What Goes

**The Essential 20% (Always Include):**

```
1. ROLE (1-2 sentences)
   "You are a compliance analyst specializing in FX trades."

2. SCOPE (2-3 sentences)
   "Check trades against Rule B201 (settlement) and Rule B305 (position limits).
    Do NOT make execution recommendations."

3. OUTPUT FORMAT (1-2 sentences)
   "Return JSON: { decision: APPROVED|DENIED|ESCALATE, violations: [...] }"

4. EXAMPLES (optional, 2-3 only)
   "Example: EUR/USD 2M trade → APPROVED (rule check: pass)"
```

Total: ~200 tokens. But carries 80% of the signal.

**The Non-Essential 80% (Usually Remove):**

```
- Motivation (why you're doing this)
- Hedging (uncertainty language)
- Apologies (makes you sound uncertain)
- Verbosity (saying the same thing multiple ways)
- Task mixing (do one thing per prompt)
- Explanations of your thought process
- Repetition of rules (once is enough)
- Filler ("please", "thank you", "help me")
```

---

#### 4. Real Example: Before and After

**Before (Bloated, 650 words):**

```
"Hi, I'm building a compliance system for a trading desk. I'm trying to validate 
trade orders before they hit the market. I've been reading about regulations and 
there seem to be a lot of different rules depending on the trade type, the 
counterparty, the settlement date, and so on. I'm not an expert in all of them, 
so I'm hoping you can help me figure this out.

What I need is help checking whether a given trade order is compliant with 
regulations. I have a bunch of rules that trades need to follow:

Rule B201: FX trades must settle T+2
Rule B305: Position limits are $2M per counterparty per day

There might be other rules too, but I'm not entirely sure. If you can just check 
the ones I listed, that would be great.

Here's the trade I want to check: [trade data]

Please let me know if this trade violates any of these rules. If it does, tell 
me which rule it violates. If it doesn't, tell me it's compliant. If you're 
not sure, just let me know that too.

Also, if you can, please format your response in JSON so I can parse it 
programmatically later.

Thanks for your help!"
```

Problems:
- 650 words of setup
- Vague role ("help me figure this out")
- Uncertain language throughout
- Unclear output format
- No system-level constraints
- Mixes explanation with instruction

---

**After (Lean, 150 words):**

```
System Prompt:
"You are a compliance analyst. Validate trades against Rule B201 
(FX settlement T+2) and Rule B305 (position limit $2M/counterparty/day).
Return JSON only: { decision: APPROVED|DENIED|ESCALATE, violations: [string] }
Do NOT execute trades. Do NOT modify positions. Do NOT guess rules."

User Message:
"Validate this trade: [trade data]"
```

Improvements:
- 150 words total (75% reduction)
- Clear role
- Explicit scope and constraints
- Unambiguous output format
- System-level rules separated from user query
- Ready for production

Result:
- Before: Model response is 500 words, hedging, uncertain, hallucinated details
- After: Model response is tight JSON, auditable, consistent

---

#### 5. The Negative Space Principle in UI/UX

This principle comes from design. In UI, "negative space" (the empty space) is as important as the content.

A cluttered UI with too many buttons confuses users.
A clean UI with only essential buttons is intuitive.

The same principle applies to prompts:
- Cluttered prompts with too many constraints confuse the model
- Clean prompts with only essential constraints are predictable

**Example from Design:**

A busy dashboard with 20 metrics → Users don't know where to look → They miss the critical info.
A focused dashboard with 3 metrics → Users know exactly what matters → They act fast.

Apply this to prompts:

```
CLUTTERED:
"Also, you should consider the regulatory environment. And the 
market conditions. And the counterparty's history. And the 
settlement dates. And the economic indicators. And..."

FOCUSED:
"Check against Rule B201 and Rule B305 only.
Ignore other factors."
```

---

#### 6. The Edit Protocol for Prompts

Use this editing process to reduce noise:

```
Step 1: Write your prompt (let it be verbose, don't worry)

Step 2: Delete every sentence you could remove without changing output

Step 3: Replace hedging language with directiveness

Step 4: Consolidate redundant constraints into one clear statement

Step 5: Verify output quality hasn't decreased (or has improved)

Step 6: Measure tokens before/after (you should see 40-60% reduction)
```

Python implementation:

```python
def optimize_prompt(original_prompt):
    """Iteratively remove noise from a prompt."""
    
    # Split into sentences
    sentences = original_prompt.split('. ')
    
    # Try removing each sentence and measure impact
    optimized = original_prompt
    removed_count = 0
    
    for sentence in sentences:
        test_prompt = optimized.replace(sentence, '')
        
        # Test if removing this sentence changes output quality
        quality_impact = evaluate_prompt_quality(test_prompt)
        
        if quality_impact >= original_quality:  # No loss or improvement
            optimized = test_prompt
            removed_count += 1
    
    print(f"Removed {removed_count} sentences")
    print(f"Original tokens: {count_tokens(original_prompt)}")
    print(f"Optimized tokens: {count_tokens(optimized)}")
    print(f"Reduction: {(1 - count_tokens(optimized)/count_tokens(original_prompt))*100:.1f}%")
    
    return optimized
```

---

#### Key Terms for Day 20
| Term | What It Means |
|------|-----------|
| **Context Pollution** | Unnecessary, redundant, or confusing text in a prompt that reduces clarity. |
| **Hedging Language** | Uncertain language ("maybe", "I think") that makes instructions ambiguous. |
| **Negative Space** | The elements you deliberately omit to create clarity through simplicity. |
| **Signal vs. Noise** | Signal = essential information; Noise = everything else that can be removed. |
| **Prompt Compression** | Removing words while preserving (or improving) output quality. |

#### Official References
- Anthropic Prompt Optimization Guide → https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview
- "Less is More" in Prompt Engineering → https://www.anthropic.com/research

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 19](./Day-19-Protocol-Versioning.md) | [Day 21 →](./Day-21-Client-Features-Overview.md)
