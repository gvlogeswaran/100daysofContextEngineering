# Day 25 — Phase 2 Recap: The Prompt Engineering Toolkit

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"Structure beats improvisation. Patterns beat chaos. From theory to practice, from patterns to production."*

---

### Phase 2 Recap: The Prompt Engineering Toolkit

---

#### 1. The 10 Patterns: Quick Reference

| Day | Pattern | Use When | Cost | Quality |
|-----|---------|----------|------|---------|
| 16 | System Prompt Architecture | Always (foundational) | Minimal | Critical |
| 17a | Zero-Shot | Concept is well-defined | Minimal | Low-Medium |
| 17b | Few-Shot | Task has edge cases | Medium | High |
| 17c | Chain-of-Thought | Multi-step reasoning needed | Medium | Very High |
| 17d | Role Prompting | Domain expertise required | Minimal | High |
| 17e | Template Prompting | Structured output needed | Minimal | High |
| 17f | Meta-Prompting | Task type varies | High | Very High |
| 18 | CoT with Examples | Complex reasoning + audit trail | High | Critical |
| 19 | Few-Shot Selection | Pattern recognition fails | Medium | Very High |
| 20 | Negative Space | Clarity needed | Minimal | High |
| 21 | Role > Persona | Avoiding hallucinations | Minimal | Critical |
| 22 | Instruction Hierarchy | Security/compliance required | Minimal | Critical |
| 23 | Versioning | Production deployment | Medium | Critical |
| 24 | Context Compression | Large documents | Medium | High |

---

#### 2. The Production Prompt Checklist

**Tier 1: Non-Negotiable (Do Not Deploy Without)**
```
□ System prompt is architecturally sound
  - Clear role definition (function, not backstory)
  - Explicit constraints (MUST NOT / MUST)
  - Domain knowledge injected
  - Error handling defined
  - Scope boundaries stated

□ Output format specified
  - JSON schema defined
  - Fields documented
  - Examples provided

□ Testing infrastructure exists
  - 20+ golden test cases written
  - Accuracy baseline established
  - Regression detection configured

□ Version control in place
  - Git repository set up
  - Semantic versioning applied
  - Changelog maintained
```

**Tier 2: Highly Recommended**
```
□ Few-shot examples included (if task has ambiguity)
  - Anchor example (simplest case)
  - Edge case (tricky decision)
  - Boundary case (where patterns collide)

□ Chain-of-Thought structure
  - Explicit reasoning steps
  - Intermediate checkpoints
  - Audit trail visible

□ Negative space optimization
  - No hedging language
  - No redundant constraints
  - No task mixing

□ Role clarity
  - No persona fiction
  - Function-based only
  - Behavioral boundaries clear

□ Instruction hierarchy
  - System > User > Assistant order clear
  - Override prevention mechanisms
  - Fallback behaviors defined
```

**Tier 3: Nice-to-Have (Improves Robustness)**
```
□ A/B testing framework
  - Traffic splitting configured
  - Metrics comparison defined
  - Rollback automated

□ Context compression strategy
  - For large documents: summarization plan
  - For long histories: sliding window approach
  - For structured data: extraction rules

□ Monitoring dashboards
  - Accuracy per version
  - Latency per pattern
  - Error rates tracked

□ Documentation
  - "Why" this prompt design documented
  - Trade-offs explained
  - Known limitations listed
```

---

#### 3. Common Mistakes to Avoid

**Mistake 1: Mixing Pattern Types**
```
WRONG:
"You're a helpful financial advisor (PERSONA).
Zero-shot classify this trade (ZERO-SHOT).
Also provide advice on market conditions (SCOPE CREEP).
Format as JSON (TEMPLATE).
And explain your reasoning (CoT)."

RIGHT:
One prompt = one clear purpose
- Classification prompt (for classification)
- Analysis prompt (for analysis)
- Advice prompt (for advice)
```

**Mistake 2: Overstating Persona**
```
WRONG:
"You're a senior derivatives trader with 30 years on Wall Street"

RIGHT:
"You classify derivatives trades against Rule B305"
```

**Mistake 3: No Testing Before Deployment**
```
WRONG:
Change prompt in production, hope it works

RIGHT:
1. Change in branch
2. Run golden test cases
3. A/B test with 10% traffic
4. Monitor metrics
5. Promote to 100% or rollback
```

**Mistake 4: Redundant Instructions**
```
WRONG:
"Return JSON. Make sure it's valid JSON. Output should be JSON format."

RIGHT:
"Return valid JSON: { decision: string, violations: [string] }"
```

---

#### 4. Decision Tree: Which Pattern(s) Should I Use?

```
START
│
├─ Is output format flexible?
│  └─ NO → Use Template Prompting
│
├─ Is the task multi-step?
│  └─ YES → Add Chain-of-Thought
│
├─ Are there edge cases I need to show?
│  └─ YES → Add Few-Shot Examples
│
├─ Is accuracy critical?
│  └─ YES → Combine: Few-Shot + CoT + Templates
│
├─ Do I need audit trail?
│  └─ YES → Use CoT (reasoning steps are visible)
│
├─ Can my role be generic?
│  └─ NO → Use Role Prompting (domain-specific function)
│
├─ Is context large?
│  └─ YES → Add Compression Strategy
│
├─ Will prompt evolve?
│  └─ YES → Set up Versioning
│
└─ RESULT: You have your pattern stack
```

---

#### 5. From Phase 1 → Phase 2 → Phase 3

**Phase 1: The Fundamentals**
- What is context?
- Why does it matter?
- Basic techniques (system prompts, examples)

**Phase 2: The Patterns (You Are Here)**
- Which patterns exist?
- When to use each?
- How to combine them?
- How to version and test?

**Phase 3: The Implementation (Coming Soon)**
- How to retrieve live data (RAG)?
- How to select relevant context?
- How to maintain context across turns?
- How to route to optimal prompts?

**Phase 4 (Implied):**
- How to measure and optimize?
- How to monitor in production?
- How to scale across teams?

---

#### 6. The Production Workflow: Proposal → Review → Test → Deploy

```
Day 0: Someone has an idea
"We should improve trade compliance checking"

Day 1: Proposal Phase
- Draft new system prompt
- Identify which patterns needed
- Estimate impact

Day 2: Review Phase
- Senior engineer reviews design
- Checks: Are constraints clear? Is role defined? Is testing plan feasible?
- Approval or feedback

Day 3-5: Development Phase
- Write system prompt v1.0.0
- Write 20 golden test cases
- Run tests: 98% accuracy required

Day 6: Testing Phase
- Deploy to staging
- Run full test suite
- Load testing (ensure latency acceptable)

Day 7: Deployment Phase
- Deploy to 10% of production traffic
- Monitor metrics for 24 hours
- If metrics good: promote to 100%
- If metrics bad: automatic rollback to v0.9.x

Day 8+: Monitoring
- Track accuracy, latency, errors daily
- Alert if metrics degrade >2%
- Plan improvements for v1.0.1
```

---

#### 7. Real Example: Building a Compliance Validator from Scratch

**Day 0: Design**
```
Requirement: Validate FX trades against regulations

Decisions:
- Pattern 1: System Prompt (Architecture)
  → Role: "Compliance validator"
  → Constraints: "You MUST NOT execute trades"
  → Domain: "Rules B201 (settlement), B305 (position limit)"
  → Error: "If unsure, escalate"

- Pattern 2: Few-Shot Examples
  → Include 3 examples (anchor, edge, boundary)

- Pattern 3: Chain-of-Thought
  → Explicit steps: Extract → Check → Flag → Decide

- Pattern 4: Template Output
  → JSON schema: { decision, violations, rules_checked }

- Pattern 5: Versioning
  → Start with v1.0.0, semantic versioning

- Pattern 6: Testing
  → Create 50 golden test cases, target 96%+ accuracy
```

**Day 1-2: Implementation**
```
File structure:
/compliance-validator/
  /system-prompt/
    v1.0.0.md (the actual prompt)
  /test-cases/
    golden.json (50 test cases)
  /examples/
    example_1_approved.json
    example_2_denied.json
    example_3_escalate.json
  /changelog.md
  /deployment_history.json
```

**Day 3: Testing**
```
python test_runner.py v1.0.0
Results:
  - 48/50 tests passed (96% accuracy)
  - 2 failures in edge cases
  - Failure 1: "Complex portfolio hedge" → needs more context
  - Failure 2: "Settlement date T+3" → edge case in rules

Iteration:
  - Refine system prompt to handle hedges
  - Add few-shot example for T+3
  - Re-test: 50/50 passed ✓
```

**Day 4: Deployment**
```
Deploy v1.0.0 to production:
  - 10% of traffic for 24 hours
  - Metrics: 97.2% accuracy, 245ms latency
  - No degradation from baseline
  - Promote to 100% ✓
```

**Day 5-6: Monitoring**
```
Week 1:
  - Accuracy: 97.2% (stable)
  - Latency p99: 280ms (acceptable)
  - Error rate: 0.1% (low)

Observations:
  - Some Rule B305 edge cases still challenging
  - Plan v1.1.0 to add more few-shot examples for position limits
```

**Day 7-10: Improvement Cycle**
```
Design v1.1.0:
  - Add 3 position limit edge cases to few-shot
  - Clarify rule wording in system prompt
  - Target: 98%+ accuracy

Test v1.1.0:
  - 50 test cases: 99% accuracy ✓

Deploy v1.1.0:
  - v1.0.0 still handling 90% of traffic
  - v1.1.0 on 10% (new feature)
  - If good: promote within 48 hours
```

---

#### 8. Key Learnings Summary

**Technical:**
- System prompts are architecture (not casual text)
- Patterns exist for specific problems (don't improvise)
- Testing prevents catastrophe (golden test cases are mandatory)
- Versioning enables iteration (git + semantic versioning)
- Compression handles scale (large documents still analyzable)

**Strategic:**
- Structure beats improvisation (always choose the architectural approach)
- Clarity prevents bugs (explicit role, constraints, error handling)
- Accountability through versioning (know what's running in production)
- Quality through testing (96%+ accuracy before deploy)
- Iteration through monitoring (continuous improvement, not fire-and-forget)

**Financial Markets Specific:**
- Compliance requires audit trails (CoT + versioning)
- Accuracy is existential (one hallucination = regulatory fine)
- Risk is bidirectional (false positives = slow trading, false negatives = violations)
- Scale requires compression (10 years of trade data, 200K token window)

---

#### Key Terms for Day 25
| Term | What It Means |
|------|-----------|
| **Pattern** | A reusable solution to a specific problem (few-shot, CoT, templates, etc.). |
| **Toolkit** | The collection of all available patterns you can combine. |
| **Production Readiness** | Meeting all requirements before deploying to production (testing, versioning, monitoring). |
| **Iteration Cycle** | The continuous loop: measure → improve → test → deploy → monitor. |
| **Context Engineering** | The discipline of architecting context (system prompts, patterns, data) for AI systems. |

#### Official References
- Anthropic Complete Prompt Engineering Guide → https://docs.anthropic.com/en/docs/build-a-claude-app/prompt-engineering
- State-of-the-art Prompt Engineering (2025) → https://arxiv.org/search/?query=prompt+engineering&searchtype=all

---

## Phase 2 Complete

You are no longer a prompt engineer copying examples. You're an **architect** who understands the patterns, the tradeoffs, and the discipline of production AI systems.

Phase 3 begins tomorrow: **Dynamic Context** — bringing live data, user interactions, and real-world complexity into your prompts.

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 24](./Day-24-MCP-SDKs-Landscape.md) | [Day 26 →](./Day-26-Local-First-Philosophy.md)
