# Day 35 — RAG for Financial Documents: Unique Challenges

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Financial documents are like medical records. One number wrong = wrong diagnosis. Generic chunking strategies fail. You need domain-specific approaches."*

---


### RAG for Financial Documents: Domain-Specific Strategies

Financial documents require specialized RAG approaches. Here's how to build them right.

#### 1. Financial Document Challenges

**Tables:** 10-Ks have income statements, cash flows, segment breakdowns. Chunking by word count destroys tables.

**Numbers:** $1B means nothing without context. Is it revenue? Pre-tax? Guidance?

**Cross-references:** "See Item 1A for risks." Readers need Item 1A content.

**Temporal complexity:** "Q3 2024 vs Q3 2023." Missing the reference year changes meaning.

**Regulatory sensitivity:** Some data is material non-public. Must filter by compliance rules.

#### 2. Table-Aware Chunking

```python
import pandas as pd
from pdfplumber import PDF

def extract_tables_with_context(pdf_path):
    """Extract tables and surrounding text for financial PDFs"""
    with PDF.open(pdf_path) as pdf:
        chunks = []
        for page_num, page in enumerate(pdf.pages):
            tables = page.extract_tables()
            
            for table_idx, table in enumerate(tables):
                # Convert table to markdown
                df = pd.DataFrame(table[1:], columns=table[0])
                table_md = df.to_markdown()
                
                # Extract surrounding text (context)
                text = page.extract_text()
                
                chunk = {
                    "type": "table",
                    "content": table_md,
                    "context": text[:200],  # 200 chars before table
                    "source": f"{pdf_path}#page{page_num}",
                    "table_id": f"table_{page_num}_{table_idx}"
                }
                chunks.append(chunk)
        
        return chunks

# Usage
table_chunks = extract_tables_with_context("AAPL-10K-2024.pdf")
```

#### 3. Entity Extraction for Numbers

```python
import re
from langchain.extractors import EntityExtractor

def extract_financial_entities(text):
    """Extract numbers, dates, metrics from financial text"""
    
    entities = {
        "amounts": [],
        "dates": [],
        "metrics": [],
        "references": []
    }
    
    # Amounts: $X.XB, $X.XM
    amount_pattern = r'\$[\d,.]+\s*(?:[BM](?:illion)?)?'
    entities["amounts"] = re.findall(amount_pattern, text)
    
    # Dates: Q3 2024, FY2024
    date_pattern = r'(?:Q[1-4]\s*)?(?:FY)?20\d{2}'
    entities["dates"] = re.findall(date_pattern, text)
    
    # Metrics: Revenue, Earnings, Margin, etc.
    metrics = ["Revenue", "Operating Income", "Net Income", "EPS", "Free Cash Flow", "Gross Margin"]
    for metric in metrics:
        if metric.lower() in text.lower():
            entities["metrics"].append(metric)
    
    # References: Item 1A, Note 5
    ref_pattern = r'(?:Item|Note)\s+[\dA-Z]+'
    entities["references"] = re.findall(ref_pattern, text)
    
    return entities

# Usage
text = "AAPL Q3 2024 Revenue: $93.7B, up 5% YoY. See Item 1A for risks."
entities = extract_financial_entities(text)
print(entities)
# Output: amounts=['$93.7B'], dates=['Q3 2024'], metrics=['Revenue'], references=['Item 1A']
```

#### 4. Temporal-Aware Chunking

```python
def chunk_with_temporal_context(document, filing_date, fiscal_period):
    """Add temporal metadata to all chunks"""
    
    chunks = semantic_chunk(document)
    
    for chunk in chunks:
        chunk.metadata.update({
            "filing_date": filing_date,  # e.g., 2024-08-15
            "fiscal_period": fiscal_period,  # e.g., "Q3 2024"
            "fiscal_year": extract_year(fiscal_period),  # 2024
            "document_type": "10-K",
        })
    
    return chunks

# Usage
filing_date = "2024-08-15"
fiscal_period = "Q3 2024"
chunks = chunk_with_temporal_context(aapl_10k, filing_date, fiscal_period)
```

#### 5. Financial RAG Query Handler

```python
class FinancialRAG:
    def __init__(self, vector_store, llm):
        self.vector_store = vector_store
        self.llm = llm
    
    def answer_financial_question(self, query, filters=None):
        """Answer financial questions with domain awareness"""
        
        # 1. Parse query for temporal and entity context
        parsed = self.parse_financial_query(query)
        # e.g., {"metric": "Revenue", "ticker": "AAPL", "period": "Q3 2024"}
        
        # 2. Retrieve with filters
        filters = {
            "ticker": parsed.get("ticker"),
            "fiscal_period": parsed.get("period"),
            "document_type": ["10-K", "10-Q", "8-K"]
        }
        
        docs = self.vector_store.search(query, k=10, filters=filters)
        
        # 3. Rank by recency (prefer latest filing)
        docs_sorted = sorted(docs, key=lambda d: d.metadata["filing_date"], reverse=True)
        
        # 4. Generate answer
        context = "\n".join([d.page_content for d in docs_sorted[:5]])
        prompt = f"""
        You are a financial analyst. Answer the question based on SEC filings.
        
        Question: {query}
        
        Relevant documents:
        {context}
        
        Provide specific numbers and cite the filing date.
        """
        
        answer = self.llm.invoke(prompt)
        return answer
    
    def parse_financial_query(self, query):
        """Extract financial entities from query"""
        # Q: "What was AAPL revenue in Q3 2024?"
        # Output: {ticker: AAPL, metric: Revenue, period: Q3 2024}
        
        import re
        parsed = {}
        
        # Ticker
        ticker_match = re.search(r'\b([A-Z]{1,4})\b', query)
        if ticker_match:
            parsed["ticker"] = ticker_match.group(1)
        
        # Metric
        metrics = ["Revenue", "Earnings", "EPS", "Operating Income", "Net Income"]
        for metric in metrics:
            if metric.lower() in query.lower():
                parsed["metric"] = metric
                break
        
        # Period
        period_match = re.search(r'(Q[1-4]|FY)?\s*20\d{2}', query)
        if period_match:
            parsed["period"] = period_match.group(0)
        
        return parsed
```

#### 6. Production Checklist

- [ ] Tables extracted separately and indexed
- [ ] Entity extraction (amounts, dates, metrics) implemented
- [ ] Temporal metadata on all chunks (filing date, fiscal period)
- [ ] Cross-reference expansion (Item 1A → content)
- [ ] Compliance filtering (sensitive data masking)
- [ ] Recency bias in retrieval (prefer recent filings)
- [ ] Financial-specific evaluation (RAGAS on numeric accuracy)

#### Key Terms for Day 35

| Term | What It Means |
| :--- | :--- |
| **Table-Aware Chunking** | The process of extracting financial tables intact alongside their surrounding text, preventing the destruction of structured data caused by generic word-count chunking. |
| **Entity Extraction** | Isolating specific financial variables like numbers (amounts), dates, metrics, and references from text to provide concrete meaning and context. |
| **Temporal Metadata** | Tags added to document chunks (such as filing date and fiscal period) to ensure time-sensitive comparisons (e.g., Q3 2024 vs Q3 2023) remain accurate during retrieval. |
| **Cross-Reference Expansion** | The practice of automatically retrieving and appending linked content (e.g., "See Item 1A") so the language model has the necessary context to answer. |
| **Material Non-Public Information** | Highly sensitive, regulated financial data that must be managed and masked using strict compliance filters before being processed or retrieved. |

### Summary

Unique challenges of building Retrieval-Augmented Generation (RAG) pipelines for financial documents. Because generic text chunking strategies often destroy structural tables and strip vital context from raw numbers, financial RAG requires domain-specific approaches. To solve this, we implemented table-aware chunking to preserve structured data, entity extraction to isolate amounts and metrics, and temporal-aware chunking to ensure time-series accuracy across different fiscal periods. We also built a custom Financial RAG Query Handler designed to parse user queries for tickers and dates, automatically applying a recency bias to retrieve the most up-to-date and accurate filings.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 34](./Day-34-AWS-Bedrock-KnowledgeBases.md) | [Day 36 →](./Day-36-Memory-architecture.md)
