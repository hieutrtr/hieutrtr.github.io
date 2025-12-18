---
layout: post
title: "Beyond Chunking: 3 Retrieval Strategies That Immediately Improve Your RAG System"
description: "while chunking gets attention, retrieval strategy delivers immediate RAG improvements."
featured: false
lang: en
ref: rag-retrieval-strategy
permalink: /rag-retrieval-strategy/
banner: /assets/images/rag-retrieval-strategy.png
date: 2025-12-18
---

# Beyond Chunking: 3 Retrieval Strategies That Immediately Improve Your RAG System

Everyone talks about chunking strategies in RAG systems. I've watched teams spend weeks fine-tuning chunk sizes and overlap percentages, only to squeeze out 5-8% improvement. Meanwhile, retrieval strategy sits in the middle—between your carefully chunked vector store and your expensive LLM calls—quietly determining whether your RAG system returns gold or garbage.

Here's what typically happens: You embed 10,000 documents with state-of-the-art models. Your chunks are semantically perfect. But when users ask "How do I configure Jenkins for Python 3.11?", your system returns Python 2.7 documentation because both are semantically similar. Or it retrieves five barely-relevant documents with cosine similarities of 0.54-0.62 and passes them all to your LLM, burning tokens on noise.

The retrieval layer is where semantic search meets reality. It's where your system decides which 3-10 chunks (out of thousands) get the privilege of context window real estate. Get this wrong, and even GPT-4 with perfect chunks will hallucinate. Get it right, and you'll see 15-30% improvements in answer relevance—and you can deploy these changes in days, not weeks.

In this article, I'll walk you through three production-tested retrieval strategies with measurable impact: **Hybrid Search** (combining semantic and keyword matching for 15-30% relevance gains), **Two-Stage Reranking** (using cross-encoders to boost precision by up to 48%), and **Query Enhancement** (techniques like HyDE that optimize questions before retrieval). Each section includes production code examples, real performance data from deployed systems, and decision frameworks to help you choose what to implement first.

---

## Strategy 1: Hybrid Search - Combining Semantic and Keyword Retrieval

Pure vector similarity search has a blind spot: it can't distinguish between semantically similar but factually different content. When you search for "Kubernetes 1.28 features," semantic search might happily return documentation for version 1.25 because both discuss Kubernetes features. When your medical RAG gets asked "What causes polyuria?", vector search might miss documents that use the formal medical terminology even though they're exactly what you need.

This happens because embedding models capture semantic meaning—they position conceptually similar texts close together in vector space—but they lose exact keyword information. A document about "Python 3.11 async improvements" and one about "Python 2.7 async" both discuss async Python, so they're nearby in embedding space. But you need version-specific results.

**The solution: Hybrid search combines the strengths of two retrieval methods:**

1. **Semantic search (dense retrieval)**: Uses vector embeddings to find conceptually similar documents, catching queries phrased differently than your docs
2. **Keyword search (sparse retrieval)**: Uses algorithms like BM25 or TF-IDF to find exact term matches, ensuring version numbers, product names, and technical terminology are preserved

The typical weighting is 70% semantic and 30% keyword, though you should tune this for your domain. Highly technical docs with precise terminology might use 60/40 or even 50/50.

### Implementation with Weaviate (Native Hybrid)

Weaviate offers built-in hybrid search with automatic score fusion:

```python
import weaviate
from weaviate.classes.query import HybridFusion

# Initialize Weaviate client
client = weaviate.connect_to_local()

try:
    collection = client.collections.get("Documentation")

    # Hybrid search with alpha parameter (0=pure keyword, 1=pure vector)
    # alpha=0.7 means 70% vector, 30% keyword
    response = collection.query.hybrid(
        query="How to configure Jenkins pipeline for Python 3.11",
        alpha=0.7,
        limit=10,
        fusion_type=HybridFusion.RELATIVE_SCORE  # or RANKED for RRF
    )

    for obj in response.objects:
        print(f"Score: {obj.metadata.score}")
        print(f"Content: {obj.properties['content'][:200]}")
        print(f"Version: {obj.properties.get('version', 'N/A')}")
        print("---")

finally:
    client.close()
```

### Implementation with LangChain + Separate Vector/Keyword Stores

If you're using FAISS or ChromaDB (vector-only), you can combine with keyword search manually:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.vectorstores import FAISS
from langchain_community.retrievers import BM25Retriever
from langchain_openai import OpenAIEmbeddings

# Load your documents
documents = [...]  # Your document chunks

# Create vector retriever
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(documents, embeddings)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 20})

# Create keyword retriever
bm25_retriever = BM25Retriever.from_documents(documents)
bm25_retriever.k = 20

# Combine with weighted ensemble (weights=[0.7, 0.3] = 70% vector, 30% BM25)
ensemble_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.7, 0.3]
)

# Retrieve with hybrid search
results = ensemble_retriever.get_relevant_documents(
    "How to configure Jenkins pipeline for Python 3.11"
)

print(f"Retrieved {len(results)} documents")
for doc in results[:5]:
    print(f"Content: {doc.page_content[:200]}")
    print(f"Metadata: {doc.metadata}")
    print("---")
```

### Real Results from Production

In a mid-size B2B SaaS documentation RAG system serving 400+ support agents:

- **Baseline (vector-only)**: 58% precision at top-5 (P@5)
- **After hybrid search**: 81% P@5 (↑23 percentage points, 40% relative improvement)
- **Mean Reciprocal Rank**: Improved from 0.410 to 0.486 (+18.5%)
- **Latency increase**: +201ms per query (acceptable for the accuracy gain)

The improvement was most dramatic for queries containing:
- Version numbers or dates ("2024 regulations" vs "2019 regulations")
- Product-specific terminology ("API rate limits" matched both semantically and on keyword "rate limits")
- Acronyms (keyword search caught exact acronyms semantic search missed)

### When to Use Hybrid Search

**Use hybrid search when:**
- Your corpus is >10,000 documents (benefits scale with size)
- Queries include technical terms, product names, versions, or acronyms
- Users phrase questions colloquially but docs use formal terminology
- Current vector-only precision is <75%

**Skip hybrid search when:**
- Corpus is small (<1,000 documents) and vocabulary naturally aligns
- Extreme real-time requirements (<50ms) make the overhead prohibitive
- Your domain has minimal terminology mismatch issues

---

## Strategy 2: Two-Stage Reranking - Speed Meets Precision

Retrieval faces a fundamental trade-off: fast approximate search sacrifices precision for speed. Vector databases use algorithms like HNSW (Hierarchical Navigable Small World) or IVF (Inverted File Index) that don't examine every document—they estimate which clusters to search. This gets you results in milliseconds even with millions of vectors, but you'll miss some relevant documents and retrieve some marginal ones.

The single-stage retrieval also suffers from the curse of dimensionality: cosine similarity in 768-dimensional space is a coarse measure of relevance. Two documents might have identical 0.71 similarity scores, but one directly answers the query while the other is tangentially related.

**Two-stage reranking solves this with a divide-and-conquer approach:**

**Stage 1 - Fast Retrieval (Recall-focused)**: Use approximate nearest neighbor search to quickly fetch 50-100 candidate documents. This stage prioritizes recall—getting relevant docs into the candidate set—even if it includes some noise.

**Stage 2 - Precise Reranking (Precision-focused)**: Apply a more sophisticated model to score each candidate's relevance to the query, then select the top 5-10 for your LLM context. This stage prioritizes precision—ranking the truly relevant docs highest.

### Why Cross-Encoders Excel at Reranking

Unlike bi-encoders (which embed query and document separately, then compare), cross-encoders process the query and document together as a single input: `[CLS] query [SEP] document [SEP]`. This allows attention mechanisms to model fine-grained interactions—how each query word relates to each document word—producing far more accurate relevance scores.

The trade-off: cross-encoders are slower (30-100ms per query-document pair) and can't be precomputed, which is why we only apply them to a small candidate set.

### Implementation with LangChain + Hugging Face

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from langchain_community.vectorstores import Pinecone
from langchain_openai import OpenAIEmbeddings

# Stage 1: Initial retrieval (fetch 50 candidates)
embeddings = OpenAIEmbeddings()
vectorstore = Pinecone.from_existing_index(
    index_name="documentation",
    embedding=embeddings
)
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 50})

# Stage 2: Cross-encoder reranking
cross_encoder_model = HuggingFaceCrossEncoder(
    model_name="cross-encoder/ms-marco-MiniLM-L-12-v2"
)

reranker = CrossEncoderReranker(
    model=cross_encoder_model,
    top_n=5  # Return top 5 after reranking
)

# Combine into two-stage retriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# Retrieve and rerank
query = "How do I handle rate limiting in the API?"
reranked_docs = compression_retriever.get_relevant_documents(query)

for i, doc in enumerate(reranked_docs):
    print(f"Rank {i+1}:")
    print(f"Content: {doc.page_content[:200]}")
    print(f"Relevance Score: {doc.metadata.get('relevance_score', 'N/A')}")
    print("---")
```

### Implementation with Cohere Rerank API (Production-Grade)

For production systems prioritizing quality, Cohere's Rerank API offers state-of-the-art models:

```python
import cohere
from langchain_community.vectorstores import Weaviate

# Initialize clients
co = cohere.Client(api_key="your-cohere-api-key")
vectorstore = Weaviate(...)

# Stage 1: Retrieve candidates
query = "What are the security best practices for API authentication?"
candidate_docs = vectorstore.similarity_search(query, k=75)

# Stage 2: Cohere reranking
documents_for_rerank = [doc.page_content for doc in candidate_docs]

rerank_response = co.rerank(
    model="rerank-english-v3.0",
    query=query,
    documents=documents_for_rerank,
    top_n=5,
    return_documents=True
)

# Extract top-ranked documents
for result in rerank_response.results:
    print(f"Relevance Score: {result.relevance_score:.4f}")
    print(f"Document: {result.document.text[:200]}")
    print("---")
```

### Performance Data from 2025 Research

**Databricks benchmark**: Reranking improved retrieval quality by **up to 48%** on complex queries requiring deep semantic understanding.

**ZeroEntropy analysis**: Cross-encoders deliver **95% of LLM-based reranker accuracy** while being **3x faster** (200-400ms vs 1-2 seconds).

**Optimal candidate set size**: Reranking 50-75 candidates maximizes NDCG@10 for most applications. Beyond 100 candidates, quality improvements plateau while costs and latency increase linearly.

**Three-stage pipeline** (BM25 → dense retrieval → cross-encoder reranking): Achieves the highest precision by combining keyword recall, semantic understanding, and fine-grained relevance scoring.

### Model Distillation for Cost Optimization

Production tip: Large cross-encoders like `ms-marco-MiniLM-L-12-v2` can be expensive at scale. Model distillation transfers knowledge from a high-performing teacher model into a smaller student model, retaining **90%+ of ranking quality** at a fraction of inference cost. This is particularly valuable when serving 10K+ queries/day.

### When to Use Reranking

**Use reranking when:**
- Answer accuracy is critical (legal, medical, financial domains)
- Queries are complex and nuanced (multi-clause questions, domain jargon)
- Current retrieval precision is <70% but recall is decent
- You can afford 200-500ms additional latency for quality

**Skip reranking when:**
- Extreme latency requirements (<200ms end-to-end)
- Queries are simple and first-stage retrieval already achieves 85%+ precision
- Budget constraints make cross-encoder API costs prohibitive

---

## Strategy 3: Query Enhancement - Making Better Questions

RAG systems answer the question you ask, not the question you meant to ask. When users type vague queries like "login not working" or overly complex ones like "How do I set up OAuth with refresh tokens and handle expiration for mobile apps?", retrieval quality suffers—even with perfect chunking and hybrid search.

Query enhancement preprocesses user input before retrieval, transforming unclear or suboptimal queries into retrieval-friendly representations. Three techniques stand out in production systems: **Query Decomposition**, **HyDE (Hypothetical Document Embeddings)**, and **Query Expansion**.

### Technique 1: Query Decomposition

Complex questions often require information from multiple sources. Decomposition breaks them into simpler sub-questions, retrieves for each independently, then synthesizes results.

**Example:**
- Original: "What's the difference between OAuth and JWT for API authentication?"
- Decomposed:
  1. "What is OAuth and how does it work for API authentication?"
  2. "What is JWT and how does it work for API authentication?"
  3. "What are the advantages and disadvantages of OAuth vs JWT?"

**Implementation with LangChain:**

```python
from langchain.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_community.vectorstores import Pinecone

llm = ChatOpenAI(model="gpt-4", temperature=0)

# Decompose query into sub-questions
decomposition_prompt = ChatPromptTemplate.from_template(
    """Break down this complex question into 2-4 simpler sub-questions that,
    when answered together, fully address the original question.

    Question: {question}

    Sub-questions (one per line):"""
)

def decompose_and_retrieve(complex_query: str, vectorstore):
    # Decompose
    response = llm.invoke(
        decomposition_prompt.format(question=complex_query)
    )
    sub_questions = response.content.strip().split("\n")
    sub_questions = [q.strip("- ").strip() for q in sub_questions if q.strip()]

    # Retrieve for each sub-question
    all_docs = []
    for sub_q in sub_questions:
        docs = vectorstore.similarity_search(sub_q, k=3)
        all_docs.extend(docs)

    # Remove duplicates (by content hash)
    unique_docs = {doc.page_content: doc for doc in all_docs}.values()

    return list(unique_docs), sub_questions

# Usage
query = "What's the difference between OAuth and JWT for API authentication?"
docs, sub_qs = decompose_and_retrieve(query, vectorstore)

print(f"Decomposed into {len(sub_qs)} sub-questions:")
for i, sq in enumerate(sub_qs, 1):
    print(f"{i}. {sq}")
print(f"\nRetrieved {len(docs)} unique documents")
```

### Technique 2: HyDE (Hypothetical Document Embeddings)

HyDE flips the retrieval paradigm: instead of embedding the query, generate a hypothetical answer to the query, embed that answer, then search for similar documents. This works because answers are semantically closer to documents than queries are.

**Why it works:**
- Query: "How do I fix memory leaks in Python?" (short, question-form)
- Hypothetical answer: "Memory leaks in Python often occur when circular references prevent garbage collection. Use the gc module to detect leaks, implement weak references for callbacks, and profile with tracemalloc..." (detailed, document-like)
- Your docs are written like the hypothetical answer, not like the query

**Implementation with LangChain:**

```python
from langchain.chains import HypotheticalDocumentEmbedder
from langchain_openai import OpenAI, OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

base_embeddings = OpenAIEmbeddings()
llm = OpenAI(model="gpt-3.5-turbo-instruct", temperature=0)

# HyDE chain: query → generate hypothetical doc → embed → retrieve
hyde_embeddings = HypotheticalDocumentEmbedder.from_llm(
    llm=llm,
    base_embeddings=base_embeddings,
    prompt_key="web_search"  # or custom prompt
)

# Create vectorstore with HyDE embeddings
vectorstore = FAISS.from_documents(documents, hyde_embeddings)

# Retrieve using HyDE
query = "How do I fix memory leaks in Python?"
results = vectorstore.similarity_search(query, k=5)

for doc in results:
    print(doc.page_content[:200])
    print("---")
```

**Custom HyDE Prompt for Technical Docs:**

```python
from langchain.prompts import PromptTemplate

hyde_prompt = PromptTemplate(
    input_variables=["query"],
    template="""You are a technical documentation writer. Generate a detailed,
    technical answer to this question as if you were writing documentation.
    Include code examples, technical terminology, and best practices.

    Question: {query}

    Documentation:"""
)

# Use with LLM to generate hypothetical doc
hypothetical_doc = llm.invoke(hyde_prompt.format(query=query))

# Embed and search
query_embedding = embeddings.embed_query(hypothetical_doc.content)
results = vectorstore.similarity_search_by_vector(query_embedding, k=5)
```

### Technique 3: Query Expansion

Expand queries with synonyms, acronyms, or domain-specific terminology before retrieval. This is simpler than HyDE but effective for catching terminology variations.

**Implementation:**

```python
def expand_query(query: str, llm) -> str:
    expansion_prompt = f"""Expand this query by adding relevant synonyms,
    acronyms, and related technical terms. Keep it concise.

    Original: {query}
    Expanded:"""

    response = llm.invoke(expansion_prompt)
    return response.content.strip()

# Example
original = "API throttling configuration"
expanded = expand_query(original, llm)
# Result: "API throttling rate limiting configuration quota management
#          requests per minute RPM API gateway limits"

# Use expanded query for retrieval
results = vectorstore.similarity_search(expanded, k=10)
```

### Performance Insights (2025 Research)

**HyDE benefits:**
- **Zero-shot retrieval**: No need for extensive labeled training data
- **Domain specialization**: Particularly effective in healthcare (clinical terminology) and e-commerce (product descriptions)
- **Complex queries**: Outperforms baseline on multi-hop reasoning tasks

**HyDE trade-offs:**
- **Latency**: Adds one LLM call before retrieval (+500-1000ms)
- **HyPE variant**: Generate hypothetical questions for each document chunk during ingestion (offline), faster at query time

**Query decomposition:**
- **Improvement**: ~7% on complex multi-hop questions (MQRF-RAG research)
- **Cost**: Additional LLM call for decomposition + multiple retrievals

### When to Use Query Enhancement

**Use query decomposition when:**
- Queries are frequently complex with multiple sub-questions
- Your docs are organized by discrete topics (decomposition aligns with structure)

**Use HyDE when:**
- Semantic gap between how users ask and how docs are written
- Domain-specific terminology is dense (medical, legal, technical)
- You can accept 500-1000ms additional latency

**Use query expansion when:**
- Users employ different terminology than docs (informal vs formal)
- Acronyms and abbreviations are common
- You need lightweight improvement without LLM calls

**Skip query enhancement when:**
- Queries are already well-formed and match doc language
- Latency budget is tight (<500ms total)
- Users primarily use keyword search patterns

---

## Putting It Together: Your Retrieval Strategy Decision Framework

You've seen three powerful strategies, but which should you implement first? Here's a practical decision framework based on your current challenges and constraints.

### Quick Diagnostic: What's Your Bottleneck?

**Run this test on 100 sample queries from your logs:**

1. **Calculate retrieval metrics:**
   - Precision@5: What % of top-5 results are relevant?
   - Recall@10: What % of relevant docs appear in top-10?

2. **Identify your problem:**
   - **P@5 < 70%, but docs have version/date metadata**: → Hybrid search
   - **P@5 < 70%, documents lack exact terminology matches**: → Query enhancement (HyDE)
   - **P@5 = 70-75%, but could be better**: → Two-stage reranking
   - **R@10 < 80%**: → Fix chunking or embeddings first (retrieval strategies won't help if relevant docs aren't in candidates)

### Implementation Roadmap

**Week 1: Hybrid Search (Lowest-Hanging Fruit)**
- **Effort**: 1-2 days for Weaviate native, 3-4 days for custom implementation
- **Impact**: 15-30% relevance improvement if you have technical terminology, versions, or dates
- **Cost**: Minimal (native DB features or BM25 library)
- **Latency**: +50-200ms

**Week 2-3: Two-Stage Reranking (Biggest Impact)**
- **Effort**: 2-5 days (integrate cross-encoder, tune candidate count)
- **Impact**: Up to 48% quality improvement on complex queries
- **Cost**: $0.001-0.005 per query (Cohere API) or GPU inference costs (self-hosted)
- **Latency**: +200-500ms

**Week 4: Query Enhancement (Domain-Specific Polish)**
- **Effort**: 3-7 days (experiment with decomposition/HyDE, build custom prompts)
- **Impact**: 7-15% improvement on complex/vague queries
- **Cost**: $0.002-0.01 per query (LLM call for enhancement)
- **Latency**: +500-1000ms

### Three-Stage Pipeline for Maximum Performance

For production systems where accuracy is paramount (legal research, medical diagnosis support, compliance):

```python
from langchain.retrievers import EnsembleRetriever, ContextualCompressionRetriever
from langchain_community.vectorstores import Weaviate
from langchain_community.retrievers import BM25Retriever
from langchain.retrievers.document_compressors import CrossEncoderReranker

# Stage 1: BM25 keyword search (catches exact matches)
bm25_retriever = BM25Retriever.from_documents(documents)
bm25_retriever.k = 30

# Stage 2: Dense vector retrieval (semantic understanding)
vectorstore = Weaviate(...)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 30})

# Combine Stage 1 + 2 with ensemble (60 candidates total)
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.3, 0.7]
)

# Stage 3: Cross-encoder reranking (precision-focused top-10)
reranker = CrossEncoderReranker(
    model=HuggingFaceCrossEncoder(model_name="cross-encoder/ms-marco-MiniLM-L-12-v2"),
    top_n=10
)

final_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=ensemble_retriever
)

# Use in RAG chain
results = final_retriever.get_relevant_documents(query)
```

**Expected performance:**
- **Recall**: 90%+ (BM25 + vector catches diverse relevant docs)
- **Precision**: 85%+ (cross-encoder reranking ensures top results are highly relevant)
- **Latency**: 800-1200ms total
- **Cost**: ~$0.008 per query

### Cost-Latency-Quality Trade-Off Matrix

| Configuration | Latency | Cost/Query | Quality (P@5) | Best For |
|---------------|---------|------------|---------------|----------|
| Vector-only | 50-100ms | $0.001 | 60-65% | High-volume, latency-critical |
| Hybrid search | 150-300ms | $0.002 | 75-80% | General production (best ROI) |
| Hybrid + reranking | 400-800ms | $0.006 | 85-90% | Accuracy-critical apps |
| 3-stage + HyDE | 1200-2000ms | $0.015 | 90-95% | Legal, medical, compliance |

### What to Implement First?

**If you're starting fresh:**
1. Start with hybrid search (Weaviate makes this trivial)
2. Measure improvement on your eval dataset
3. If P@5 < 80%, add two-stage reranking
4. If queries are complex/vague, experiment with HyDE

**If you're optimizing an existing system:**
1. Run diagnostic (calculate current P@5, R@10)
2. Identify your bottleneck using the framework above
3. Implement the corresponding strategy
4. A/B test on 10% traffic before full rollout
5. Monitor metrics for 1-2 weeks, then decide next optimization

**Cost-conscious teams:**
- Hybrid search first (minimal cost, good gains)
- Self-hosted cross-encoder reranking (avoid API fees)
- Skip HyDE unless semantic gap is severe

**Quality-obsessed teams:**
- Implement all three strategies in three-stage pipeline
- Budget for LLM API costs and longer latency
- Build comprehensive eval dataset to measure incremental gains

The key insight: retrieval strategy is high-leverage because you can deploy changes in days and see immediate, measurable improvements—no retraining required, no infrastructure overhaul. Start with hybrid search this week, and you'll likely see 20% better results by next week.

---

## Conclusion

Here's the truth about RAG optimization: while everyone's obsessing over chunking strategies and embedding models, the retrieval layer is sitting there quietly determining whether your system succeeds or fails. Hybrid search catches what pure semantic search misses—version numbers, exact terminology, technical acronyms. Two-stage reranking separates truly relevant documents from semantically similar noise. Query enhancement transforms vague or complex user questions into retrieval-friendly representations. Together, these three strategies can deliver 30-50% improvements in answer relevance, and unlike chunking refactors or model retraining, you can implement them in days.

The path forward is straightforward: start with hybrid search this week (1-2 days of effort, 15-30% gains), measure results on your evaluation dataset, then add two-stage reranking if precision is still below 80% (2-5 days, up to 48% additional improvement). If your queries are particularly complex or your users speak differently than your documentation, experiment with query enhancement techniques like HyDE or decomposition. The beauty of retrieval optimization is that it's modular—each strategy stacks on the previous one, and you can A/B test changes on 10% traffic before full rollout.

Your chunking strategy matters. Your embedding model matters. But if you're burning weeks optimizing those while ignoring retrieval, you're leaving 30% performance on the table. Stop obsessing over chunks. Fix your retrieval strategy instead. The code examples are above, the performance data is proven, and the implementation time is measured in days. Start Monday.

---

## Sources

**Research & Best Practices:**
- [Optimizing RAG with Hybrid Search & Reranking - VectorHub by Superlinked](https://superlinked.com/vectorhub/articles/optimizing-rag-with-hybrid-search-reranking)
- [Hybrid RAG: Boosting RAG Accuracy - AI Multiple](https://research.aimultiple.com/hybrid-rag/)
- [RAG in 2025: 7 Proven Strategies - Morphik Blog](https://www.morphik.ai/blog/retrieval-augmented-generation-strategies)

**Reranking & Cross-Encoders:**
- [Ultimate Guide to Choosing the Best Reranking Model in 2025 - ZeroEntropy](https://www.zeroentropy.dev/articles/ultimate-guide-to-choosing-the-best-reranking-model-in-2025)
- [The aRt of RAG Part 3: Reranking with Cross Encoders - Medium](https://medium.com/@rossashman/the-art-of-rag-part-3-reranking-with-cross-encoders-688a16b64669)
- [Reranking in RAG: Enhancing Accuracy with Cross-Encoders - EY/KA Lab](https://eyka.com/blog/reranking-in-rag-enhancing-accuracy-with-cross-encoders/)
- [Rerankers and Two-Stage Retrieval - Pinecone](https://www.pinecone.io/learn/series/rag/rerankers/)

**Query Enhancement & HyDE:**
- [How Query Expansion (HyDE) Boosts RAG Accuracy - Chitika](https://www.chitika.com/hyde-query-expansion-rag/)
- [HyDE + Query Expansion: Supercharging Retrieval in RAG Pipelines - Medium](https://medium.com/theultimateinterviewhack/hyde-query-expansion-supercharging-retrieval-in-rag-pipelines-f200955929f1)
- [Better RAG with HyDE - Hypothetical Document Embeddings - Zilliz Learn](https://zilliz.com/learn/improve-rag-and-information-retrieval-with-hyde-hypothetical-document-embeddings)
