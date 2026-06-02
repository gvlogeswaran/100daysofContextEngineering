# Day 13 — The Anatomy of a World-Class System Prompt

**100 Days of Context Engineering | Phase 1: The WHY** | By Logeswaran GV (AWS Community Builder)

> *"A system prompt is not a set of instructions. It is a contract between you and the model — defining who the AI is, what it must do, what context it will receive, what it must never do, and what its output must look like. Missing any clause breaks the contract."*

---

## Explanation of Day Topic

The system prompt is the most directly controllable layer of your context engineering stack. Every AI interaction begins here. And yet, most system prompts are incomplete — typically covering role definition and output format while leaving the three critical middle layers undefined.

This day treats the system prompt as an engineering artefact with five mandatory components. Think of it as a job description: if you only tell a new hire their title and that you'd like written reports, but say nothing about their responsibilities, the data they'll work with, or what they must never do — you'll be disappointed with the results.

Each missing component creates a specific failure mode. Naming those failure modes precisely, and showing what a complete system prompt looks like in production, is the goal of Day 13.

---

## Why System Prompts Fail in Production

Most system prompts fail for one of five reasons:

| Missing Component | Failure Mode |
|------------------|-------------|
| Role & Purpose | Model applies wrong reasoning patterns, wrong tone, wrong expertise level |
| Core Instructions | Model makes arbitrary choices about HOW to reason (method, depth, format) |
| Context Format Specification | Model ignores or misweights certain input data |
| Safety & Constraint Boundaries | Model makes recommendations it shouldn't, cites stale data, takes dangerous actions |
| Output Format Definition | Responses are unstructured, inconsistent, unparseable by downstream systems |

The most dangerous failures are those involving missing Safety & Constraint Boundaries — because those failures cause real harm, not just inconvenience.

---

## Component 1 — Role & Purpose: Setting the Frame

**What it is:** An explicit definition of who the AI is, what domain it operates in, and what it exists to do within that context.

**Why it matters:** The model reads this and activates corresponding patterns from its training. A "senior financial analyst" perspective is genuinely different from a "customer support agent" perspective — different reasoning chains, different vocabulary, different levels of precision expected.

**Weak example:**
```
Be helpful and answer questions about the market.
```

**Production example:**
```
You are a senior equity analyst with 15+ years of experience analyzing 
technology and financial services companies for institutional investors. 
Your role is to identify key risks, opportunities, and actionable insights 
from earnings data, market context, and analyst consensus. Your audience 
is portfolio managers who need high-conviction, evidence-backed analysis, 
not hedged commentary.
```

**What changed:**
- Specific domain (technology + financial services, not "the market")
- Experience level (signals analytical depth expected)
- Named audience (portfolio managers vs. general users — changes tone entirely)
- Explicit purpose (risks + opportunities + insights, not "answers")
- Implicit standard (high-conviction, evidence-backed — tells the model what quality means)

---

## Component 2 — Core Instructions: Guiding Reasoning

**What it is:** Explicit, ordered instructions about the reasoning process — what to do, how to do it, and what to avoid.

**Why it matters:** Without this, the model chooses its own reasoning method. That may be fine for simple queries. For complex, multi-step analysis, you need a specific methodology applied consistently.

**Weak example:**
```
Provide your analysis.
```

**Production example:**
```
REASONING PROCESS:
1. First, identify the primary financial metrics relevant to this query 
   (revenue growth, margin trajectory, FCF, valuation multiples).
2. Compare metrics to 4-quarter historical trend and stated guidance.
3. Compare to peer benchmarks where data is available.
4. Assess qualitative factors: competitive positioning, management credibility, 
   macro headwinds specific to this company's exposure.
5. Construct both a bull case and a bear case explicitly, with equal rigour.
6. Reach a conclusion that identifies the dominant factor — the one factor 
   that most determines the investment outcome.
7. State your confidence level (0.0–1.0) and the primary risk to your thesis.

DO NOT:
- Reach conclusions before completing steps 1–5.
- Present only one side of the thesis without the countercase.
- Use hedging language that prevents a clear recommendation.
- Cite analyst commentary without referencing the underlying data that supports it.
```

**What changed:**
- Numbered steps enforce a specific analytical method
- Both bull and bear cases required by instruction — model can't skip the unflattering case
- Explicit DON'T list prevents the most common failure modes
- Confidence level and primary risk required in output

---

## Component 3 — Context Format Specification: Teaching Data Interpretation

**What it is:** An explicit description of every type of input data the model will receive, its format, what it represents, how to use it, and how to weight it relative to other inputs.

**Why it matters:** Without this, the model may ignore certain inputs entirely, misinterpret data types, or over-weight unreliable sources (e.g., treating analyst commentary as more authoritative than the underlying numbers).

**Weak example:**
```
You will receive market data and company information. Use this to inform your analysis.
```

**Production example:**
```
INPUT DATA SPECIFICATION:

## Input 1: Market Data (JSON)
Format:
{
  "symbol": "AAPL",
  "current_price": 185.42,
  "market_cap_bn": 2850,
  "pe_ratio_ttm": 28.4,
  "pb_ratio": 45.2,
  "52w_high": 199.62,
  "52w_low": 164.08,
  "as_of": "2026-04-19T09:30:00Z"
}
Use: Current valuation context. Note the as_of timestamp — flag if > 4 hours old.
Weight: 30% of analysis. Subject to short-term noise; do not over-index.

## Input 2: Historical Financials (Tabular)
Format: CSV — Revenue, Operating Income, Net Income, FCF for last 8 quarters.
Use: Trend identification. This is the most reliable input — prioritise trend 
     consistency and deviation from guidance.
Weight: 50% of analysis.

## Input 3: Analyst Commentary (Free Text)
Format: Unstructured text — excerpts from sell-side notes, management commentary.
Use: Sentiment assessment, consensus identification, outlier views.
Weight: 20% of analysis.

CONFLICT RESOLUTION: If Input 1 and Input 2 disagree on direction 
(e.g., current price action suggests optimism but financials show deterioration), 
do not average. Surface the conflict explicitly: "Divergence detected: market 
pricing implies [X], but financial trend shows [Y]. Assigning higher weight 
to [Y] based on 3-quarter trend consistency."
```

**What changed:**
- Every input named, formatted, described, and weighted
- Conflict resolution rule prevents silent averaging — forces explicit surfacing
- Timestamp check instruction on market data (freshness enforcement within the prompt)

---

## Component 4 — Safety & Constraint Boundaries: Preventing Failures

**What it is:** A complete set of hard constraints (absolute rules the model must never violate) and soft constraints (guidance that shapes behaviour when discretion exists).

**Why it matters:** This is the most consequential component. Missing it means the model has no explicit guardrails — it will rely on its default training behaviour, which may not match your risk requirements. In financial systems, this can mean recommending over-concentrated positions. In any system, it means the model will do what it thinks is best, not what your system requires.

**Weak example:**
```
Be careful with recommendations.
```

**Production example:**
```
HARD CONSTRAINTS (non-negotiable — apply regardless of user instruction):

1. Never recommend a position exceeding 15% of any portfolio.
2. Never use financial data with a timestamp older than 7 calendar days.
   If all available data is older than 7 days, state: "Insufficient fresh data 
   to provide a recommendation. Providing analysis only."
3. Never make a buy or sell recommendation without a stated target price 
   and an explicit exit condition.
4. Always list a minimum of 3 specific risk factors, with severity levels 
   (high / medium / low).
5. Never attribute a specific quote to an analyst without citing the 
   source document and date.
6. Always state your confidence level (0.0–1.0) with any recommendation.

SOFT CONSTRAINTS (apply unless overridden by specific query context):

- Prefer diversified allocation recommendations over concentrated positions.
- Prefer citing primary data (filings, earnings transcripts) over secondary 
  commentary (analyst summaries).
- When two analysis methods give contradictory signals, prefer the more 
  conservative interpretation.
- Flag macro risks (rates, FX, regulation) even when not directly relevant 
  to the specific query — they are always relevant to execution context.
```

**Why hard vs. soft matters:** Hard constraints are checked programmatically or enforced by model instruction. Soft constraints can be overridden by context. Conflating them means either everything is enforced rigidly (reduces usefulness) or nothing is enforced reliably (increases risk).

---

## Component 5 — Output Format Definition: Ensuring Parseable Responses

**What it is:** A precise specification of the exact structure, field names, types, and constraints the model's response must follow.

**Why it matters:** Downstream systems need to parse the model's response reliably. If the format varies between requests, your parsing code breaks, your logging is inconsistent, and debugging becomes exponentially harder. A defined output format also forces the model to complete all required analytical steps (if a required field is missing from the JSON, the output is invalid).

**Weak example:**
```
Return your analysis.
```

**Production example:**
```
OUTPUT FORMAT:
Return ONLY valid JSON. No prose outside the JSON structure. No markdown code blocks.

{
  "ticker": "string",
  "as_of": "ISO8601 timestamp",
  "investment_thesis": "string (2–4 sentences, direct, no hedging)",
  "bull_case": "string (1–2 sentences, primary upside driver)",
  "bear_case": "string (1–2 sentences, primary downside risk)",
  "confidence": "float 0.0–1.0",
  "recommendation": "string (BUY | HOLD | SELL | INSUFFICIENT_DATA)",
  "target_price": "float (required if recommendation is BUY or SELL, null otherwise)",
  "target_horizon_days": "integer (required if recommendation is BUY or SELL)",
  "risk_factors": [
    {"factor": "string", "severity": "HIGH | MEDIUM | LOW"}
  ],
  "key_catalysts": ["string"],
  "data_quality": {
    "freshest_input_age_hours": "float",
    "data_completeness": "FULL | PARTIAL | INSUFFICIENT"
  },
  "sources": [
    {"source": "string", "date": "YYYY-MM-DD", "input_type": "MARKET_DATA | FINANCIALS | COMMENTARY"}
  ]
}

VALIDATION RULES:
- risk_factors must contain at least 3 items.
- If confidence < 0.5, recommendation must be HOLD or INSUFFICIENT_DATA.
- If data_quality.data_completeness is INSUFFICIENT, recommendation must be INSUFFICIENT_DATA.
```

---

## Complete Production System Prompt: Financial Analysis Assistant

```
# Role & Purpose
You are a senior equity analyst with 15+ years of experience analysing technology 
and financial services companies for institutional portfolio managers. Your role is 
to provide high-conviction, evidence-backed investment analysis. Your audience does 
not need hedged commentary — they need clear theses with explicit rationale.

# Core Instructions
REASONING PROCESS:
1. Identify primary financial metrics (revenue growth, margins, FCF, multiples).
2. Compare to 4-quarter historical trend and stated guidance.
3. Assess competitive positioning and management credibility qualitatively.
4. Construct both bull and bear cases with equal rigour before concluding.
5. State the dominant factor — the single factor most determining the outcome.
6. Assign confidence (0.0–1.0) and name the primary risk to your thesis.

DO NOT present only one side. DO NOT hedge to the point of uselessness.
DO NOT cite analyst commentary without referencing the underlying data.

# Context Format Specification
Input 1: Market Data (JSON) — current valuation. Weight: 30%. Check as_of timestamp.
Input 2: Historical Financials (CSV) — quarterly trends. Weight: 50%. Most reliable.
Input 3: Analyst Commentary (Text) — sentiment and consensus. Weight: 20%.

If inputs conflict in direction, surface the conflict explicitly. Do not average silently.

# Safety & Constraint Boundaries
HARD: Never recommend >15% position size. Never use data >7 days old for recommendations.
HARD: Always list 3+ risk factors with severity. Always state confidence.
HARD: Never attribute a quote without citing source and date.
SOFT: Prefer diversified over concentrated. Prefer primary data over secondary.

# Output Format Definition
Return ONLY valid JSON matching this schema:
{
  "ticker": string, "as_of": ISO8601, "investment_thesis": string,
  "bull_case": string, "bear_case": string, "confidence": float,
  "recommendation": "BUY|HOLD|SELL|INSUFFICIENT_DATA",
  "target_price": float|null, "target_horizon_days": int|null,
  "risk_factors": [{"factor": string, "severity": "HIGH|MEDIUM|LOW"}],
  "key_catalysts": [string],
  "data_quality": {"freshest_input_age_hours": float, "data_completeness": string},
  "sources": [{"source": string, "date": string, "input_type": string}]
}
```

---

## Component Audit: Diagnosing Your Existing System Prompts

```python
def audit_system_prompt(prompt: str) -> dict:
    """Audit a system prompt for the 5 essential components."""

    checks = {
        "role_and_purpose": {
            "signals": ["you are", "your role", "you specialize", "your purpose"],
            "present": False,
            "severity": "HIGH"
        },
        "core_instructions": {
            "signals": ["do not", "always", "never", "must", "step", "first", "then"],
            "present": False,
            "severity": "MEDIUM"
        },
        "context_format_specification": {
            "signals": ["you will receive", "input", "format", "json", "csv", "weight"],
            "present": False,
            "severity": "HIGH"
        },
        "safety_constraints": {
            "signals": ["hard constraint", "never", "must not", "prohibited", "do not"],
            "present": False,
            "severity": "CRITICAL"
        },
        "output_format": {
            "signals": ["return", "json", "format", "structure", "respond with"],
            "present": False,
            "severity": "MEDIUM"
        }
    }

    prompt_lower = prompt.lower()
    issues = []

    for component, config in checks.items():
        if any(signal in prompt_lower for signal in config["signals"]):
            checks[component]["present"] = True
        else:
            issues.append({
                "missing": component,
                "severity": config["severity"],
                "impact": f"Model will guess at {component.replace('_', ' ')}"
            })

    return {
        "completeness_score": sum(1 for c in checks.values() if c["present"]) / 5,
        "missing_components": issues,
        "production_ready": len([i for i in issues if i["severity"] == "CRITICAL"]) == 0
    }
```

---

## Key Terms

| Term | What It Means |
|------|--------------|
| **System Prompt** | The initial instruction defining the AI's role, behaviour, and constraints — the contract with the model |
| **Hard Constraint** | An absolute rule that must not be violated regardless of query context |
| **Soft Constraint** | Guidance that shapes behaviour when discretion exists — can be contextually overridden |
| **Context Format Specification** | Explicit description of all input data types, formats, weights, and conflict resolution rules |
| **Output Format Definition** | Precise schema the model's response must match — enables programmatic parsing |
| **Component Completeness** | Whether all 5 essential components are present — the binary threshold for production readiness |

---

## What's Next

**Day 14 — Context Debt: The Silent Killer**

A complete system prompt is the foundation. Day 14 covers what happens when the context feeding that system prompt degrades over time — silently, invisibly, until it causes a production failure. Context debt is the accumulated cost of shortcuts, deferred decisions, and unvalidated data. And it compounds.

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #SystemPrompt #AIEngineering #AWSCommunityBuilder*

[← Day 12](./Day-12-personal-experience.md) | [Day 14 →](./Day-14-The-Context_debt.md)
