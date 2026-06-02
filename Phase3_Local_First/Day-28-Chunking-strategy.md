# Day 28 — Chunking Strategy: The Decision That Makes or Breaks RAG

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Chunking is like cutting up a novel to feed it to a speed reader. Cut every sentence, and the reader loses plot. Cut by chapter, and the reader remembers the story."*

---


### Chunking Strategy: The Lever That Multiplies RAG Quality

Chunking is where RAG succeeds or fails. This is the deep dive.

#### 1. The Chunking Problem

A 50-page earnings report is too large for a single context window. A single sentence is too small—it loses all context. Where's the sweet spot?

It depends on:
- **Document type** (PDF vs. table vs. code)
- **Query patterns** (specific facts vs. summary questions)
- **Embedding model** (some prefer longer chunks)
- **Context window** (Claude 3.5 Sonnet = 200K, older models = 4K)
- **Latency budget** (more chunks = slower retrieval)

#### 2. Fixed-Size Chunking: Simple but Dumb

```python
# Example: Fixed-size chunking with overlap
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,              # Each chunk = 512 tokens
    chunk_overlap=50,            # 50 tokens overlap between chunks
    separators=["\n\n", "\n", "."]
)

chunks = splitter.split_documents(documents)
# Result: Document split into equal pieces, overlap handled

# ❌ Problem example:
# Document has a table: "AAPL | Revenue | $93.7B | YoY | +4.2%"
# Fixed-size splitter doesn't understand tables.
# Might split "AAPL | Revenue | $93.7B" and "| YoY | +4.2%" into separate chunks.
# Each chunk alone is meaningless.
```

**Characteristics:**
- **Predictable:** You know exactly how many chunks per document
- **Fast:** O(n) where n = document size
- **Cheap:** No ML involved
- **Dumb:** No semantic awareness

**When to use:** Large homogeneous corpora (news articles, research papers) where speed > quality.

**Failure mode:** Tables, code blocks, structured data get mangled.

#### 3. Semantic Chunking: Smart but Slower

**The idea:** Instead of splitting by character count, split by semantic boundaries. Find where one idea ends and another begins.

```python
# Example: Semantic chunking
from langchain.text_splitter import SemanticChunker
from langchain.embeddings import BedrockEmbeddings
import boto3

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
embeddings = BedrockEmbeddings(
    client=bedrock,
    model_id="amazon.titan-embed-text-v2:0"
)

splitter = SemanticChunker(embeddings=embeddings)
chunks = splitter.split_documents(documents)

# How it works:
# 1. Split document into sentences
# 2. Compute embedding for each sentence
# 3. Measure semantic similarity between adjacent sentences
# 4. Where similarity drops (topic change), create a chunk boundary
# 5. Merge small chunks to maintain minimum size

# Result: Chunks that respect paragraph and section breaks
```

**Characteristics:**
- **Smart:** Respects document structure and topic transitions
- **Slower:** Requires embedding every sentence (expensive)
- **Higher quality:** Chunks are semantically coherent
- **Flexible:** Can set breakpoint threshold to control chunk sizes

**Cost trade-off:** Semantic chunking might cost 2-3x more during ingestion. But retrieval quality improves 3-5x. Worth it for production systems.

**Example with financial document:**
```
Document: AAPL Q3 2024 Earnings Report

Section 1: Revenue Highlights
Sentence A: "AAPL reported revenue of $93.7B..."
Sentence B: "YoY growth was 4.2%..."
Sentence C: "Service revenue grew to $22.1B..."
[Low semantic change = keep together]
✓ CHUNK 1: "AAPL reported revenue of $93.7B... YoY growth was 4.2%... Service revenue grew..."

Section 2: Risk Factors
Sentence D: "The company faces regulatory headwinds..."
Sentence E: "Supply chain disruptions could impact margins..."
[High semantic change from Section 1 = new chunk boundary]
✓ CHUNK 2: "The company faces regulatory headwinds..."
✓ CHUNK 3: "Supply chain disruptions could impact margins..."
```

#### 4. Hierarchical Chunking: The Flexible Powerhouse

**The idea:** Store documents at multiple levels of granularity. Let queries choose which level to retrieve.

```python
# Example: Hierarchical chunking for a 10-K filing
class HierarchicalChunker:
    def __init__(self):
        self.levels = {}
    
    def chunk(self, document):
        # Level 1: Full document summary
        level1 = self.summarize(document)
        
        # Level 2: Section summaries (Item 1, Item 1A, etc.)
        sections = document.split("\n## Item ")
        level2 = [self.summarize(section) for section in sections]
        
        # Level 3: Detailed chunks within each section
        level3 = []
        for section in sections:
            chunks = self.semantic_chunk(section, chunk_size=512)
            level3.extend(chunks)
        
        # Level 4: Granular details (tables, specific disclosures)
        level4 = self.extract_tables_and_critical_facts(document)
        
        return {
            "summary": level1,                    # Full doc summary
            "section_summaries": level2,           # ~15 sections
            "detailed_chunks": level3,             # ~200-300 chunks
            "facts": level4                        # Tables, numbers
        }

# Usage:
hierarchical = HierarchicalChunker()
result = hierarchical.chunk(aapl_10k)

# Query: "What was AAPL's total revenue?"
# Retrieve from level4 (specific fact): $93.7B (fast, precise)

# Query: "Summarize AAPL's risk factors"
# Retrieve from level2 (section summary): Management risks, supply chain risks... (fast, broader)

# Query: "Explain how revenue is distributed across segments"
# Retrieve from level3 (detailed chunks): Full breakdown by service, product...
```

**Characteristics:**
- **Flexible:** Choose retrieval granularity per query
- **Complex:** Requires document structure understanding
- **Efficient:** Small queries hit summaries, large queries hit details
- **Best for:** Structured documents (10-Ks, research reports, technical docs)

#### 5. Overlap Strategy: The Boundary Problem

Chunks at boundaries lose context. Overlap solves this.

```python
# No overlap: Context lost at boundaries
Chunk 1: "...earnings beat expectations. Guidance..."
Chunk 2: "...for next quarter remains strong."
# Chunk 2 by itself loses context of what "guidance" means

# 50% overlap:
Chunk 1: "...earnings beat. Guidance for next quarter..."
Chunk 2: "...Guidance for next quarter remains strong. Margins..."
# Chunk 2 now has context: what guidance is, what it refers to
```

**Recommended overlap:** 50-100 tokens (for 512-token chunks). Prevents information loss at boundaries.

#### 6. Chunking for Different Data Types

**Financial Documents (10-Ks, prospectuses):**
```python
# Recommendation: Semantic + hierarchical + overlap
splitter = SemanticChunker(embeddings=embeddings, breakpoint_threshold_type="percentile")
chunks = splitter.split_documents(documents, separators=[
    "\n## Item ",          # Item-level boundaries
    "\n### ",              # Subsection boundaries
    "\n\n",                # Paragraph breaks
])
```

**Tables and Structured Data:**
```python
# Problem: Fixed/semantic chunkers break tables
# Solution: Extract tables separately
import pandas as pd

tables = extract_tables(document)  # Using pdf parser or table detector
table_chunks = [
    {
        "type": "table",
        "content": table.to_string(),
        "metadata": {"table_id": i, "page": page}
    }
    for i, (table, page) in enumerate(tables)
]

# Chunks = semantic chunks from text + separate table chunks
chunks = semantic_chunks + table_chunks
```

**Code and Technical Docs:**
```python
# Problem: Code shouldn't be split mid-function
# Solution: Respect code block boundaries

splitter = RecursiveCharacterTextSplitter(
    separators=[
        "\n```",           # Code block breaks
        "\n## ",           # Section breaks
        "\n\n",            # Paragraph breaks
    ],
    chunk_size=1024,       # Larger chunks for code
    chunk_overlap=200,
)
```

#### 7. Measuring Chunking Quality

You can't optimize what you don't measure. Here's how to evaluate chunking strategies:

```python
# Evaluation metrics for chunking
class ChunkingQualityMetrics:
    @staticmethod
    def chunk_count(docs, chunks):
        """Smaller is better (fewer chunks = cheaper retrieval)"""
        return len(chunks) / len(docs)
    
    @staticmethod
    def avg_chunk_size(chunks):
        """Should be 300-800 tokens for most use cases"""
        return sum(len(c.page_content.split()) for c in chunks) / len(chunks)
    
    @staticmethod
    def boundary_quality(chunks):
        """Do chunks end mid-sentence? Mid-table? (subjective)"""
        # Manually inspect first 20 chunks
        pass
    
    @staticmethod
    def retrieval_success_rate(chunks, test_queries):
        """
        For each test query, retrieve top-5 chunks.
        Is the relevant chunk in top-5? (needs ground truth)
        """
        pass

# Example evaluation:
evaluation = ChunkingQualityMetrics()
fixed_chunks = fixed_chunker.split_documents(docs)
semantic_chunks = semantic_chunker.split_documents(docs)

print(f"Fixed-size: {evaluation.chunk_count(docs, fixed_chunks)} chunks")
# Fixed-size: 250 chunks, cost = 250 * $0.0001/embedding = $0.025

print(f"Semantic: {evaluation.chunk_count(docs, semantic_chunks)} chunks")
# Semantic: 180 chunks, cost = 180 * $0.0001/embedding = $0.018
# But quality is 3x better (measured via retrieval tests)
```

#### 8. Production Chunking Pipeline

```python
import boto3
from langchain.text_splitter import SemanticChunker
from langchain.embeddings import BedrockEmbeddings

def production_chunk(documents, doc_type="financial"):
    """
    Production-grade chunking pipeline.
    Handles different document types.
    """
    bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
    embeddings = BedrockEmbeddings(
        client=bedrock,
        model_id="amazon.titan-embed-text-v2:0"
    )
    
    if doc_type == "financial":
        # Financial docs: respect Item structure, hierarchical
        splitter = SemanticChunker(
            embeddings=embeddings,
            breakpoint_threshold_type="percentile_90"
        )
        separators = ["\n## Item ", "\n\n", "\n"]
    
    elif doc_type == "research":
        # Research papers: section-aware
        splitter = SemanticChunker(embeddings=embeddings)
        separators = ["\n# ", "\n## ", "\n\n"]
    
    else:
        # Generic: balance speed and quality
        splitter = SemanticChunker(embeddings=embeddings)
        separators = ["\n\n", "\n"]
    
    chunks = splitter.split_documents(documents, separators=separators)
    
    # Enrich metadata
    for i, chunk in enumerate(chunks):
        chunk.metadata["chunk_id"] = i
        chunk.metadata["chunk_size"] = len(chunk.page_content.split())
    
    return chunks

# Usage
financial_docs = load_docs("earnings-reports/")
chunks = production_chunk(financial_docs, doc_type="financial")
```

#### Key Terms for Day 28

| Term | What It Means |
| :--- | :--- |
| **Fixed-Size Chunking** | Splitting text by a rigid character or token count. Fast and cheap, but often destroys contextual meaning by slicing mid-sentence or mid-table. |
| **Semantic Chunking** | Splitting text based on topic changes or semantic boundaries using embeddings. More computationally expensive, but yields much higher quality retrieval. |
| **Hierarchical Chunking** | Creating nested representations of a document (e.g., full summary → section summary → paragraph detail) allowing the RAG system to retrieve at the appropriate level of granularity. |
| **Chunk Overlap** | Intentionally duplicating a small number of tokens (usually 50-100) at the boundaries of adjacent chunks to ensure no context is lost where the split occurs. |

### Summary

Chunking is the invisible foundation of RAG quality. While fixed-size chunking is fast and cheap, it routinely destroys the context of structured data, which is a massive liability for complex financial clauses or tables. Semantic and hierarchical chunking cost more in compute and time during the ingestion phase, but they pay massive dividends in retrieval accuracy and LLM comprehension. In production, always adapt your chunking strategy to your specific data type—never treat a dense regulatory filing the same way you treat a basic news article. 

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 27](./Day-27-Long-term-memory.md) | [Day 29 →](./Day-29-Embedding-Models.md)
