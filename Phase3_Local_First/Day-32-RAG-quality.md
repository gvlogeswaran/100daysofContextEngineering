# Day 32 — RAG Quality: The RAGAS Framework and Evaluation

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"RAG evaluation is like a medical test. You don't just say "the patient looks healthy." You run tests: blood work, heart rate, oxygen. Same with RAG: run metrics."*

---


### RAGAS: Evaluating RAG System Quality

RAGAS provides quantitative metrics for RAG evaluation. Here's how to implement and use it.

#### 1. RAGAS Framework Overview

RAGAS evaluates 4 dimensions:

```
Query → Retrieval → Generation → Answer
         ↓                         ↓
    Context metrics         Output metrics
    - Precision             - Faithfulness
    - Recall               - Relevance
    - Coverage
```

#### 2. Faithfulness: Is the Answer Grounded?

Faithfulness measures whether the generated answer is supported by the retrieved documents.

```python
from ragas.metrics import faithfulness
from ragas.llm import LangchainLLMWrapper
from langchain.llms import ChatAnthropic

# Set up RAGAS evaluation LLM
eval_llm = LangchainLLMWrapper(ChatAnthropic(model="claude-3-5-sonnet-20241022"))

# Score a single answer
score = faithfulness.score(
    row={
        "question": "What was AAPL's Q3 2024 revenue?",
        "answer": "AAPL reported revenue of $93.7B",
        "contexts": ["AAPL Q3 2024 earnings: Revenue $93.7B, EPS $2.07"]
    },
    llm=eval_llm
)

print(f"Faithfulness: {score:.2f}")  # Range: 0-1

# How it works internally:
# 1. Extract claims from answer: ["AAPL revenue was $93.7B"]
# 2. For each claim, ask LLM: "Is this claim supported by the context?"
# 3. Calculate % of claims supported
```

**Interpretation:**
- 1.0 = All statements supported by documents
- 0.8 = 80% of statements grounded, 20% inferred/hallucinated
- 0.5 = Half the answer is hallucinated
- 0.0 = Completely fabricated

#### 3. Answer Relevance: Does It Address the Query?

Answer relevance measures whether the answer actually addresses the question.

```python
from ragas.metrics import answer_relevance

score = answer_relevance.score(
    row={
        "question": "What was AAPL's Q3 2024 revenue?",
        "answer": "AAPL reported revenue of $93.7B",
    },
    llm=eval_llm
)

print(f"Answer Relevance: {score:.2f}")

# How it works:
# 1. Use LLM to generate 5 alternative questions the answer would address
# 2. Compare those questions to the original query
# 3. If they're similar, the answer was relevant
# 4. If different, the answer went off-topic
```

**Examples:**
- Query: "AAPL revenue?" Answer: "AAPL revenue was $93.7B" = 1.0 (perfect)
- Query: "AAPL revenue?" Answer: "Apple was founded in 1976" = 0.0 (irrelevant)
- Query: "AAPL revenue?" Answer: "AAPL revenue was $93.7B, founded in 1976" = 0.7 (mostly relevant)

#### 4. Context Precision: Quality of Retrieved Docs

Context precision measures what fraction of retrieved documents are relevant to the query.

```python
from ragas.metrics import context_precision

score = context_precision.score(
    row={
        "question": "What was AAPL's Q3 2024 revenue?",
        "contexts": [
            "AAPL Q3 2024: Revenue $93.7B",          # Relevant
            "MSFT Q3 2024: Revenue $62.3B",          # Not relevant
            "AAPL operating expenses: $40.2B",       # Somewhat relevant
            "Weather forecast for tomorrow: Sunny",  # Not relevant
        ],
        "ground_truth": "AAPL Q3 revenue"
    },
    llm=eval_llm
)

print(f"Context Precision: {score:.2f}")

# Calculation:
# Precision = (relevant docs) / (total retrieved docs)
# = 2 / 4 = 0.5
```

**Interpretation:**
- 1.0 = All retrieved docs are relevant
- 0.75 = 3 out of 4 are relevant
- 0.5 = Half are garbage
- 0.0 = No relevant docs retrieved

#### 5. Context Recall: Coverage of All Relevant Docs

Context recall measures what fraction of all relevant documents were actually retrieved.

```python
from ragas.metrics import context_recall

score = context_recall.score(
    row={
        "question": "What were AAPL's Q3 2024 financial results?",
        "contexts": [
            "AAPL Q3 2024: Revenue $93.7B",
            "AAPL Q3 2024: Operating income $29.2B",
        ],
        "ground_truth": """
        AAPL Q3 2024: Revenue $93.7B, Operating income $29.2B, EPS $2.07,
        Services revenue $22.1B, installed base grew...
        """
    },
    llm=eval_llm
)

print(f"Context Recall: {score:.2f}")

# How it works:
# 1. Use LLM to extract entities from ground_truth
# 2. Check if each entity is mentioned in retrieved contexts
# 3. Recall = (entities found) / (total entities)
```

**Interpretation:**
- 1.0 = Retrieved all important facts
- 0.8 = Got 80% of the facts, missed 20%
- 0.5 = Retrieved only half the relevant information

#### 6. Production RAGAS Implementation

```python
import pandas as pd
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevance,
    context_precision,
    context_recall
)
from datasets import Dataset

def evaluate_rag_system(queries, expected_answers, retrieved_docs_list, generated_answers):
    """
    Evaluate a RAG system using RAGAS.
    
    Args:
        queries: List of test queries
        expected_answers: Ground truth answers
        retrieved_docs_list: Retrieved documents for each query
        generated_answers: LLM-generated answers
    """
    
    # Create RAGAS dataset
    evaluation_data = {
        "question": queries,
        "answer": generated_answers,
        "contexts": [[doc.page_content for doc in docs] for docs in retrieved_docs_list],
        "ground_truth": expected_answers
    }
    
    dataset = Dataset.from_dict(evaluation_data)
    
    # Run evaluation
    results = evaluate(
        dataset=dataset,
        metrics=[
            faithfulness,
            answer_relevance,
            context_precision,
            context_recall
        ]
    )
    
    return results

# Example usage
test_queries = [
    "What was AAPL's Q3 2024 revenue?",
    "How did MSFT's earnings compare to guidance?",
    "What are the main risks in the 10-K?"
]

expected_answers = [
    "AAPL reported revenue of $93.7B",
    "MSFT earnings beat guidance by 2%",
    "Regulatory risks, supply chain disruptions"
]

# Assume you run RAG and get results
results = evaluate_rag_system(
    queries=test_queries,
    expected_answers=expected_answers,
    retrieved_docs_list=rag_retrieved_docs,
    generated_answers=rag_generated_answers
)

# Print results
print(f"Faithfulness: {results['faithfulness']:.2%}")
print(f"Answer Relevance: {results['answer_relevance']:.2%}")
print(f"Context Precision: {results['context_precision']:.2%}")
print(f"Context Recall: {results['context_recall']:.2%}")

# Overall score
overall = (
    results['faithfulness'] * 0.35 +
    results['answer_relevance'] * 0.25 +
    results['context_precision'] * 0.20 +
    results['context_recall'] * 0.20
)
print(f"\nOverall RAG Score: {overall:.2%}")
```

#### 7. Benchmarks for Financial RAG

| Metric | Poor | Acceptable | Good | Excellent |
| :--- | :--- | :--- | :--- | :--- |
| **Faithfulness** | < 0.80 | 0.80 - 0.90 | 0.90 - 0.95 | > 0.95 |
| **Answer Relevance**| < 0.70 | 0.70 - 0.85 | 0.85 - 0.90 | > 0.90 |
| **Context Precision**| < 0.50 | 0.50 - 0.70 | 0.70 - 0.85 | > 0.85 |
| **Context Recall** | < 0.60 | 0.60 - 0.80 | 0.80 - 0.90 | > 0.90 |

### Summary

In financial and enterprise contexts, **Faithfulness is non-negotiable**. An incomplete but highly grounded answer is always better than a perfectly relevant hallucination. 

By integrating RAGAS directly into your CI/CD pipeline, you stop treating your RAG implementation as a mystical black box and start treating it like a standard, testable software system. Going back to our medical analogy: you can't diagnose or cure what you refuse to measure. Run the tests.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 31](./Day-31-Hybrid-search.md) | [Day 33 →](./Day-33-ReRanking.md)
