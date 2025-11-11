# Advanced RAG Strategies

This repository contains a curated collection of advanced strategies for Retrieval-Augmented Generation (RAG). It is designed to provide clear, concise explanations of powerful techniques that go beyond basic RAG implementations.

The strategies are divided into two main categories:
*   `📁 Ingestion`: Techniques focused on optimizing how documents are processed, chunked, and embedded.
*   `📁 Query`: Techniques focused on improving how information is retrieved, ranked, and utilized to generate answers.

---

## 🚀 Ingestion Strategies

| File | Description |
| :--- | :--- |
| `Context-Aware Semantic Chunking.md` | Chunks text based on semantic similarity rather than fixed sizes, keeping related sentences together to preserve context. |
| `Fine-Tuned Embeddings for Domain-Specific RAG.md` | Fine-tunes an embedding model on domain-specific data to better understand niche terminology and improve retrieval accuracy. |
| `Hierarchical RAG.md` | A "small-to-big" strategy that searches over small, specific chunks but retrieves their larger parent chunks to give the LLM broader context. |
| `Late Chunking.md` | Embeds the entire document at the token level first to capture global context, then aggregates these token embeddings into chunks. |
| `Self-Reflective RAG.md` | An iterative process where the system critically assesses retrieved documents, refines the query if needed, and verifies the final answer. |

## 💡 Query Strategies

| File | Description |
| :--- | :--- |
| `Agentic RAG.md` | Utilizes an intelligent agent to perform multi-step retrieval, such as finding a relevant snippet and then fetching the full document for complete context. Also covers Hybrid Search. |
| `knowledge_graph.md` | Explores using native graph databases like Neo4j to build knowledge graphs, enabling complex, multi-hop queries that capture explicit relationships. |
| `LPG vs RDF.md` | Compares the two primary knowledge graph models: Labeled Property Graphs (LPG, used by Neo4j) and the Resource Description Framework (RDF). |
| `Re-ranking.md` | Implements a two-stage retrieval process: a fast vector search for initial recall, followed by a powerful Cross-Encoder for high-precision re-ranking. |

---
---

# 先进的 RAG 策略

本代码库汇集了一系列用于“检索增强生成”（RAG）的先进策略。旨在为超越基础 RAG 实现的强大技术提供清晰、简洁的解释。

这些策略分为两大类：
*   `📁 Ingestion (数据处理)`: 专注于优化文档处理、分块和嵌入方式的技术。
*   `📁 Query (查询)`: 专注于改进信息检索、排序和生成答案方式的技术。

---

## 🚀 数据处理策略 (Ingestion)

| 文件 | 描述 |
| :--- | :--- |
| `Context-Aware Semantic Chunking.md` | 基于语义相似性（而非固定大小）进行文本分块，将相关句子聚合在一起以保持上下文的完整性。 |
| `Fine-Tuned Embeddings for Domain-Specific RAG.md` | 在特定领域的专业数据上微调嵌入模型，使其能更好地理解专业术语，从而提高检索准确性。 |
| `Hierarchical RAG.md` | 一种“由小到大”的策略：搜索精确的小文本块，但返回其所属的、更大的父文本块，为 LLM 提供更广阔的上下文。 |
| `Late Chunking.md` | 首先在词元（Token）级别上嵌入整个文档以捕获全局上下文，然后再将这些词元嵌入聚合成块。 |
| `Self-Reflective RAG.md` | 一种迭代的、具备自我反思能力的流程，系统会批判性地评估检索到的文档，在必要时优化查询，并对最终答案进行验证。 |

## 💡 查询策略 (Query)

| 文件 | 描述 |
| :--- | :--- |
| `Agentic RAG.md` | 利用智能体（Agent）执行多步检索任务，例如先找到相关片段，再获取完整文档以获得全面上下文。该文件也涵盖了混合搜索的概念。 |
| `knowledge_graph.md` | 探讨如何使用像 Neo4j 这样的原生图数据库来构建知识图谱，以实现能够捕捉显式关系的复杂多跳查询。 |
| `LPG vs RDF.md` | 比较了两种主流的知识图谱模型：标签属性图（LPG，被 Neo4j 使用）和资源描述框架（RDF）。 |
| `Re-ranking.md` | 实现一个两阶段检索流程：首先通过快速的向量搜索进行初步召回，然后使用强大的交叉编码器（Cross-Encoder）进行高精度的重排序。 |