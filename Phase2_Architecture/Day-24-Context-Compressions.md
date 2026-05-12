# Day 24 — Context Compression Techniques

**100 Days of Context Engineering | Phase 2: Prompt Engineering as Context Design** | By Logeswaran GV (AWS Community Builder)

> *"You have a 500-page document and a 200K token window. Compression is the art of keeping signal while cutting noise."*

---

### Context Compression Techniques

---

#### 1. The Compression Problem

Large documents exceed model context windows. Solutions:

| Problem | Solution | Cost | Quality |
|---------|----------|------|---------|
| Chunking (analyze separately) | High (many calls) | Low (loses context) |
| Aggressive summarization | Low (one call) | Low (loses details) |
| Intelligent compression | Medium (structured) | High (signal preserved) |

You want high quality + medium cost. That's compression.

---

#### 2. Technique 1: Hierarchical Summarization Chain

**The Pattern:**
```
Raw Document (100K tokens)
  ↓ [Summarize to key points]
Tier-1 Summary (10K tokens)
  ↓ [Summarize further]
Tier-2 Summary (2K tokens)
  ↓ [Extract final insights]
Tier-3 Summary (200 tokens)
```

**Implementation:**
```python
def hierarchical_summarization(document):
    """Compress document through multiple summarization layers."""
    
    # Layer 1: Section summaries
    sections = split_into_sections(document)
    section_summaries = []
    
    for section in sections:
        summary = call_claude(
            system="Summarize this section in 1-2 sentences, preserving key facts.",
            user=section
        )
        section_summaries.append(summary)
    
    tier_1 = "\n".join(section_summaries)  # ~10% of original size
    
    # Layer 2: Aggregate summary
    tier_2 = call_claude(
        system="Summarize these summaries into key facts only.",
        user=tier_1
    )  # ~5% of original
    
    # Layer 3: Final extraction
    tier_3 = call_claude(
        system="""Extract ONLY:
        - Critical violations
        - Policy changes
        - Actionable items
        
        Format: Bullet list""",
        user=tier_2
    )  # ~1% of original
    
    return {
        "original_tokens": count_tokens(document),
        "final_tokens": count_tokens(tier_3),
        "compression_ratio": count_tokens(document) / count_tokens(tier_3),
        "summary": tier_3
    }

# Usage
result = hierarchical_summarization(compliance_filing)
print(f"Compressed to {result['compression_ratio']:.0f}x smaller")
```

**Quality Tradeoff:**
- Layer 1 → Layer 2: Lose details, keep structure (90% quality)
- Layer 2 → Layer 3: Lose context, keep facts (70% quality)
- Best for: Factual extraction, not nuanced analysis

---

#### 3. Technique 2: Sliding Window (Streaming)

**The Pattern:**
Process a long document as a stream. Never exceed window size.

```
Document: 2M tokens (too big for one call)

Step 1: Load first 50K tokens
        Call model: "Summarize these facts"
        Keep summary (5K)

Step 2: Load next 50K tokens
        + previous summary (5K)
        Call model: "What are NEW facts in this section?"
        Add new facts to summary

Step 3: Repeat for all 2M tokens

Final summary: Contains facts from entire 2M token document, 
but never exceeded 60K token window
```

**Implementation:**
```python
def sliding_window_compression(large_document, window_size=50000):
    """Process large document through sliding window."""
    
    chunks = split_into_chunks(large_document, size=window_size)
    accumulated_summary = ""
    
    for i, chunk in enumerate(chunks):
        if i == 0:
            # First chunk: create initial summary
            accumulated_summary = call_claude(
                system="Extract key facts from this section.",
                user=chunk
            )
        else:
            # Subsequent chunks: identify NEW facts only
            prompt = f"""
            Previous facts summary: {accumulated_summary}
            
            New chunk: {chunk}
            
            What NEW facts are in this chunk that weren't in the summary?
            Add them to the summary.
            """
            
            accumulated_summary = call_claude(
                system="Identify and add new facts to the summary.",
                user=prompt
            )
        
        print(f"Processed chunk {i+1}/{len(chunks)}, summary size: {count_tokens(accumulated_summary)}")
    
    return accumulated_summary

# Usage
result = sliding_window_compression(compliance_log_2m_tokens)
```

**Quality Tradeoff:**
- No loss of content (entire document processed)
- Risk: Later sections might duplicate earlier facts (redundancy)
- Best for: Sequential documents (logs, timelines, historical data)

---

#### 4. Technique 3: Key Extraction (Signal Isolation)

**The Pattern:**
Instead of summarizing: extract only signals you care about.

```
Raw Document:
[20K tokens of background]
[5K tokens describing a policy change]
[50K tokens of examples]
[3K tokens describing a violation]
[10K tokens of historical context]

After extraction:
- Policy change: [extracted fact]
- Violation: [extracted fact]
- Total: 2K tokens
```

**Implementation:**
```python
def signal_extraction(document):
    """Extract only signals relevant for analysis."""
    
    signals = {
        "regulatory_changes": [],
        "compliance_violations": [],
        "policy_updates": [],
        "data_anomalies": [],
        "deadlines": []
    }
    
    extraction_prompt = """
    Extract ONLY the following from this document:
    
    1. Regulatory changes (new rules, enforcement actions)
    2. Compliance violations (explicit violations found)
    3. Policy updates (internal policy changes)
    4. Data anomalies (unusual patterns)
    5. Deadlines (dates by which action must be taken)
    
    Format: JSON with keys matching the signal types above.
    Ignore everything else.
    """
    
    result = call_claude(
        system=extraction_prompt,
        user=document
    )
    
    signals = parse_json(result)
    
    # Combine all signals
    combined = "\n".join([
        f"Regulatory changes: {signals['regulatory_changes']}",
        f"Violations: {signals['compliance_violations']}",
        f"Policy updates: {signals['policy_updates']}",
        f"Anomalies: {signals['data_anomalies']}",
        f"Deadlines: {signals['deadlines']}"
    ])
    
    return {
        "original_tokens": count_tokens(document),
        "extracted_tokens": count_tokens(combined),
        "compression_ratio": count_tokens(document) / count_tokens(combined),
        "extracted_signals": signals
    }
```

**Quality Tradeoff:**
- Extremely high compression (100:1 is common)
- Zero context loss (only extracting what matters)
- Risk: You predefined what "matters"; if you miss a category, it's gone
- Best for: Structured extraction (violations, dates, changes)

---

#### 5. Technique 4: Map-Reduce Over Documents

**The Pattern:**
Break large task into parallel sub-tasks, then combine results.

```
Raw Document: 2M tokens

MAP phase (parallel):
  Task 1: Analyze chunk 1 (20K)
  Task 2: Analyze chunk 2 (20K)
  Task 3: Analyze chunk 3 (20K)
  ... 100 parallel tasks

REDUCE phase (sequential):
  Combine all 100 results into final analysis
```

**Implementation:**
```python
import concurrent.futures

def map_reduce_analysis(large_document, chunk_size=20000):
    """Analyze large document using map-reduce pattern."""
    
    chunks = split_into_chunks(large_document, size=chunk_size)
    
    # MAP phase: Analyze each chunk in parallel
    def analyze_chunk(chunk):
        return call_claude(
            system="Analyze this section for violations, changes, and key facts.",
            user=chunk
        )
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=10) as executor:
        map_results = list(executor.map(analyze_chunk, chunks))
    
    # REDUCE phase: Combine all results
    combined_analysis = "\n---\n".join(map_results)
    
    final_analysis = call_claude(
        system="Synthesize these chunk analyses into a unified report.",
        user=combined_analysis
    )
    
    return final_analysis

# Usage
result = map_reduce_analysis(compliance_document_2m)
```

**Quality Tradeoff:**
- Fast (parallel processing)
- Good quality (full document analyzed)
- Cost: Slightly higher (multiple calls) but parallelizable
- Best for: Time-sensitive analysis (can't afford sequential processing)

---

#### 6. Real Financial Market Example: Analyzing 10 Years of Trade Data

**Scenario:** Analyze 10 years of FX trade history (5M tokens) for market manipulation patterns.

**Approach: Hybrid Compression**

```python
def analyze_trade_history_hybrid(years_of_trades):
    """Analyze large trade history using hybrid compression."""
    
    # Step 1: Key extraction (MAP)
    # Extract only: violations, unusual patterns, high-value trades
    violations = []
    patterns = []
    
    for year in years_of_trades:
        extracted = signal_extraction(year)
        violations.extend(extracted['violations'])
        patterns.extend(extracted['anomalies'])
    
    # Step 2: Hierarchical summarization (REDUCE)
    # Tier 1: Summarize violations by category
    violations_summary = hierarchical_summarization(violations)
    
    # Tier 2: Identify trends
    trends = call_claude(
        system="Identify manipulation patterns from these violations.",
        user=violations_summary
    )
    
    # Tier 3: Final report
    report = call_claude(
        system="""Create a compliance report with:
        1. Summary of violations
        2. Identified manipulation patterns
        3. Recommended actions
        """,
        user=trends
    )
    
    return report

# Cost analysis
# Naive approach: Load all 5M tokens = $150 (5M * $0.03/1K)
# Compressed approach: Extract (1M) + Summarize (200K) = $35
# Savings: 77% cost reduction
```

---

#### 7. AWS Context Compression Patterns

**AWS Bedrock Prompt Caching:**
```
# Cache frequently accessed documents
aws bedrock cache-prompt \
  --document "2025 Compliance Regulations.pdf" \
  --cache-ttl 86400 \
  --id "compliance-2025"

# Reference cached document in prompt
# (Bedrock handles deduplication, tokens charged only once)
prompt = """
Using cached document 'compliance-2025', 
analyze this trade for violations.

Trade: [data]
"""
```

**AWS Lambda for Streaming Processing:**
```python
def lambda_handler(event, context):
    """AWS Lambda for streaming compression."""
    
    document_s3_uri = event['document_uri']
    
    # Step 1: Download in chunks (S3 streaming)
    for chunk in stream_from_s3(document_s3_uri, chunk_size=50000):
        # Step 2: Compress each chunk
        compressed = call_claude(
            system="Extract facts from this chunk.",
            user=chunk
        )
        
        # Step 3: Store compressed result
        store_in_dynamodb(compressed)
    
    # Step 4: Combine all compressed chunks
    final = reduce_all_compressions()
    
    return final
```

---

#### 8. Compression Metrics

```python
def measure_compression_quality(original, compressed):
    """Measure compression quality."""
    
    # Metric 1: Compression ratio
    ratio = len(original) / len(compressed)
    
    # Metric 2: Information retention
    # (How much of the original meaning is preserved?)
    retention = evaluate_semantic_similarity(original, compressed)
    
    # Metric 3: Task accuracy
    # (Can downstream analysis still work with compressed version?)
    task_accuracy = test_on_golden_cases(compressed)
    
    print(f"Compression ratio: {ratio:.1f}x")
    print(f"Information retention: {retention:.1%}")
    print(f"Task accuracy: {task_accuracy:.1%}")
    
    # Best compression: High ratio + high retention + high accuracy
    score = ratio * retention * task_accuracy
    return score
```

---

#### Key Terms for Day 24
| Term | What It Means |
|------|-----------|
| **Context Compression** | Reducing document size while preserving signal and meaning. |
| **Hierarchical Summarization** | Multiple layers of progressively more aggressive summarization. |
| **Sliding Window** | Processing long documents as streams, never exceeding context window. |
| **Signal Extraction** | Identifying and extracting only the relevant facts from noise. |
| **Map-Reduce** | Parallel processing of document chunks followed by result synthesis. |

#### Official References
- Anthropic Long Context Best Practices → https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- AWS Bedrock Documentation → https://docs.aws.amazon.com/bedrock/

---

*#100DaysOfContextEngineering #ContextEngineering #PromptEngineering #AWSCommunityBuilder*

[← Day 23](./Day-23-Prompt-versioning.md) | [Day 25 →](./Day-25-Architecture-Review-Mental-Map.md)
