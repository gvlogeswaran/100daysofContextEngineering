# Day 34 — AWS Bedrock Knowledge Bases: RAG Without the Plumbing

**100 Days of Context Engineering | Phase 3: Dynamic Context — RAG & Memory** | By Logeswaran GV (AWS Community Builder)

> *"Building RAG from scratch is like building a car. Bedrock KB is like renting one. You just drive."*

---


### AWS Bedrock Knowledge Bases: Fully Managed RAG

AWS Bedrock Knowledge Bases abstracts away RAG complexity. Here's how to use it in production.

#### 1. Architecture Overview

```
Data Sources → Bedrock KB → OpenSearch (managed) → Bedrock API → Your App
   (S3, Web)   (Ingestion)   (Vector Index)      (Retrieve+Gen)
```

#### 2. Creating a Knowledge Base

```python
import boto3

bedrock = boto3.client('bedrock', region_name='us-east-1')

# Create knowledge base
response = bedrock.create_knowledge_base(
    name='financial-documents-kb',
    description='RAG for earnings reports and 10-Ks',
    roleArn='arn:aws:iam::ACCOUNT:role/BedrockKBRole',
    knowledgeBaseConfiguration={
        'type': 'VECTOR',
        'vectorKnowledgeBaseConfiguration': {
            'embeddingConfiguration': {
                'embeddingModel': 'cohere.embed-english-v3'  # or amazon.titan-embed-text-v2:0
            }
        }
    },
    storageConfiguration={
        'type': 'OPENSEARCH_SERVERLESS',
        'opensearchServerlessConfiguration': {
            'collectionArn': 'arn:aws:aoss:us-east-1:ACCOUNT:collection/financial-docs'
        }
    }
)

kb_id = response['knowledgeBaseId']
print(f"Knowledge Base created: {kb_id}")
```

#### 3. Uploading Data Sources

```python
# Method 1: S3
response = bedrock.create_data_source(
    knowledgeBaseId=kb_id,
    dataSourceConfiguration={
        'type': 'S3',
        's3Configuration': {
            'bucketArn': 'arn:aws:s3:::my-financial-docs',
            'inclusionPatterns': ['**/*.pdf', '**/*.txt'],
            'exclusionPatterns': ['**/temp/**'],
        }
    },
    name='earnings-reports-s3',
)

# Method 2: Web URLs
response = bedrock.create_data_source(
    knowledgeBaseId=kb_id,
    dataSourceConfiguration={
        'type': 'WEB',
        'webConfiguration': {
            'sourceUrls': [
                'https://investor.apple.com/sec-filings/',
                'https://ir.microsoft.com/'
            ],
            'crawlDepth': 2,
            'crawlFilter': {
                'pattern': '*.pdf'  # Only PDFs
            }
        }
    },
    name='investor-relations-web',
)
```

#### 4. Configuring Chunking

```python
# Bedrock KB supports chunking configuration
response = bedrock.update_data_source(
    knowledgeBaseId=kb_id,
    dataSourceId=data_source_id,
    dataSourceConfiguration={
        'type': 'S3',
        's3Configuration': {
            'bucketArn': 'arn:aws:s3:::my-financial-docs',
            'chunkingConfiguration': {
                'chunkingStrategy': 'SEMANTIC',  # or FIXED_SIZE
                'semanticChunkingConfiguration': {
                    'maxTokens': 512,
                    'overlapPercentage': 20,
                    'breakpointPercentileThreshold': 95
                }
            }
        }
    }
)
```

#### 5. Querying the Knowledge Base

```python
bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

# Method 1: Retrieve documents only
response = bedrock_runtime.retrieve(
    knowledgeBaseId=kb_id,
    retrievalConfiguration={
        'vectorSearchConfiguration': {
            'numberOfResults': 10,
            'overrideSearchType': 'HYBRID'  # or 'SEMANTIC_SEARCH'
        }
    },
    text='What was AAPL revenue in Q3 2024?'
)

# Results
for result in response['retrievalResults']:
    print(f"Score: {result['score']}")
    print(f"Content: {result['content']['text']}")
    print(f"Source: {result['sourceAttributeValues']}\n")

# Method 2: Retrieve AND generate (RAG end-to-end)
response = bedrock_runtime.retrieve_and_generate(
    input={
        'text': 'What was AAPL revenue in Q3 2024?'
    },
    retrieveAndGenerateConfiguration={
        'type': 'KNOWLEDGE_BASE',
        'knowledgeBaseConfiguration': {
            'knowledgeBaseId': kb_id,
            'modelArn': 'arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-20241022-v2:0',
            'retrievalConfiguration': {
                'vectorSearchConfiguration': {
                    'numberOfResults': 5,
                    'overrideSearchType': 'HYBRID'
                }
            },
            'generationConfiguration': {
                'inferenceConfig': {
                    'textInferenceConfig': {
                        'temperature': 0.5,
                        'maxTokens': 1000,
                        'topP': 0.9
                    }
                },
                'promptTemplate': {
                    'textPromptTemplate': 'Answer the question based on documents provided:\n\n${CONTEXT}\n\nQuestion: ${INPUT}\n\nAnswer:'
                }
            }
        }
    }
)

print(f"Answer: {response['output']['text']}")
print(f"Retrieved docs: {response['retrievalResults']}")
```

#### 6. Cost Estimation

```python
# Bedrock KB pricing (us-east-1):
# - OCU: $0.24/hour = ~$1,729/month
# - Storage: ~$0.003/GB/month
# - API calls: ~$0.0001 per retrieve, $0.001 per generate

def estimate_kb_cost(num_documents, avg_doc_size_mb, monthly_queries):
    """Estimate Bedrock KB monthly cost"""
    
    # Fixed cost
    ocus = 1  # Minimum
    fixed_cost = ocus * 0.24 * 730  # 730 hours/month
    
    # Storage cost
    total_gb = (num_documents * avg_doc_size_mb) / 1024
    storage_cost = total_gb * 0.003
    
    # API cost (estimate)
    retrieve_cost = monthly_queries * 0.0001
    generate_cost = monthly_queries * 0.001
    api_cost = retrieve_cost + generate_cost
    
    total = fixed_cost + storage_cost + api_cost
    return {
        'fixed_ocus': fixed_cost,
        'storage': storage_cost,
        'api': api_cost,
        'total_monthly': total
    }

costs = estimate_kb_cost(
    num_documents=1000,
    avg_doc_size_mb=2,
    monthly_queries=100_000
)

print(f"Estimated monthly cost: ${costs['total_monthly']:.2f}")
```

#### 7. When KB vs Custom RAG

| Aspect | Bedrock Knowledge Base (KB) | Custom RAG Pipeline |
| :--- | :--- | :--- |
| **Setup Time** | Minutes to Hours | Days to Weeks |
| **Infrastructure Management** | Fully managed (OpenSearch Serverless abstracted away) | Self-managed (Vector DB provisioning, scaling, ingestion pipelines) |
| **Data Parsing & Chunking** | Standardized (Fixed-size, basic Semantic chunking natively supported) | Infinite flexibility (Custom OCR, structural chunking, complex table extraction) |
| **Model Choice** | Limited to Bedrock-supported embedding and generation models | Unlimited (Any open-source, HuggingFace, OpenAI, or local models) |
| **Baseline Cost** | High baseline (Requires minimum OCUs for OpenSearch, ~$1.7k/mo) | Can be very low (e.g., utilizing existing RDS with pgvector, or free-tier SaaS Vector DBs) |
| **Best For** | Enterprises prioritizing speed-to-market, low maintenance, and staying entirely within the AWS ecosystem | Specialized use cases requiring advanced retrieval tuning, complex document parsing, or strict budget constraints |

### Summary

AWS Bedrock Knowledge Bases act as a massive accelerator for standard RAG workloads, drastically reducing the boilerplate code needed to connect data sources to vector databases and generation APIs. You trade granular control and low baseline costs for immediate production readiness and zero infrastructure maintenance. If your documents are standard text/PDFs and you have the enterprise budget for the OpenSearch Serverless baseline, it is the most efficient path to production.

---

*#100DaysOfContextEngineering #ContextEngineering #RAG #MemoryArchitecture #AWSCommunityBuilder*

[← Day 33](./Day-33-ReRanking.md) | [Day 35 →](./Day-35-RAG-Financial.md)
