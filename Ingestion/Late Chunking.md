# Late Chunking (后期分块) 流程详解

这种方法的核心思想是：**先将整个文档输入到 Transformer 模型中以获得带有全局上下文的词元（Token）级嵌入，然后再将这些词元嵌入进行分块和聚合**。这与传统方法（先切块，再独立嵌入每个块）截然相反。

## 示例代码

```python
"""Late Chunking - Embed full document first, then chunk token embeddings"""
from pydantic_ai import Agent
import psycopg2
from pgvector.psycopg2 import register_vector
# 假设 transformer_embed 和 mean_pool 已经定义
# from some_embedding_library import transformer_embed, mean_pool

agent = Agent('openai:gpt-4o', system_prompt='You are a RAG assistant with late chunking.')

conn = psycopg2.connect("dbname=rag_db")
register_vector(conn)

def late_chunk(text: str, chunk_size=512) -> list[tuple[str, list]]:
    """Process full document through transformer BEFORE chunking"""
    # 步骤 1: 嵌入整个文档，获取每个词元（Token）的嵌入向量
    # 这个函数返回的是一个包含文档中所有 token 嵌入的列表/矩阵
    full_doc_token_embeddings = transformer_embed(text)

    # 步骤 2: 定义分块边界
    tokens = text.split()  # 简化的分词方法
    chunk_boundaries = range(0, len(tokens), chunk_size)

    # 步骤 3: 对每个块内的词元嵌入进行池化（Pooling）
    chunks_with_embeddings = []
    for i, start in enumerate(chunk_boundaries):
        end = start + chunk_size
        chunk_text = ' '.join(tokens[start:end])

        # 对属于该块的 token embeddings 进行平均池化
        # 这样得到的 chunk_embedding 保留了整个文档的上下文信息
        chunk_embedding = mean_pool(full_doc_token_embeddings[start:end])
        chunks_with_embeddings.append((chunk_text, chunk_embedding))

    return chunks_with_embeddings

def ingest_document(text: str):
    chunks = late_chunk(text)
    with conn.cursor() as cur:
        for chunk_text, embedding in chunks:
            cur.execute('INSERT INTO chunks (content, embedding) VALUES (%s, %s)',
                       (chunk_text, embedding))
    conn.commit()

@agent.tool
def search_knowledge_base(query: str) -> str:
    with conn.cursor() as cur:
        query_embedding = get_embedding(query) # 这里的 get_embedding 应该与 transformer_embed 兼容
        cur.execute('SELECT content FROM chunks ORDER BY embedding <=> %s LIMIT 3',
                   (query_embedding,))
        return "\n".join([row[0] for row in cur.fetchall()])

result = agent.run_sync("Explain transformers")
print(result.data)

```

## 核心逻辑解读

### `late_chunk` 函数详解

这是实现“后期分块”的核心，其流程可以分解为三个关键步骤：

1.  **全局嵌入 (Global Embedding)**：
    *   `full_doc_token_embeddings = transformer_embed(text)`
    *   首先，将**整个文档**（在模型允许的上下文长度内，如8192个token）一次性送入一个 Transformer 嵌入模型。
    *   模型返回的不是单个代表整个文档的向量，而是文档中**每一个词元（Token）的嵌入向量**。
    *   **关键点**：由于 Transformer 的自注意力机制，每个词元的嵌入都包含了来自文档**全局的上下文信息**。例如，第五段中一个代词的词元嵌入会“知道”它在第一段中指代的名词。

2.  **定义边界 (Boundary Definition)**：
    *   `chunk_boundaries = range(0, len(tokens), chunk_size)`
    *   在获得了所有词元的嵌入之后，再来定义分块的边界。这里的实现方式是基于固定大小（`chunk_size=512` 个词元）来划分。

3.  **分块池化 (Chunk Pooling)**：
    *   `chunk_embedding = mean_pool(full_doc_token_embeddings[start:end])`
    *   遍历第二步定义的边界，对于每个块（例如，从第0个到第511个词元），找到其对应的词元嵌入向量。
    *   然后，通过一个**池化操作**（代码中使用了平均池化 `mean_pool`），将这个块内所有词元的嵌入聚合成**一个单独的向量**，作为这个块的最终嵌入。
    *   这个最终的块嵌入，因为它是由包含了全局上下文的词元嵌入聚合而成的，所以它本身也间接保留了整个文档的上下文信息。

### `ingest_document` 与 `search_knowledge_base`
*   **Ingestion (摄入)**：`ingest_document` 函数调用 `late_chunk` 来获取带有全局上下文的文本块及其对应的聚合嵌入，并将它们存入向量数据库。
*   **Retrieval (检索)**：`search_knowledge_base` 函数进行标准的向量检索，用用户的查询向量来匹配数据库中存储的、经过池化的块嵌入。

## 流程总结

1.  **全文处理 (Full-Document Processing)**：将整个文档作为一个单一输入，送入 Transformer 模型。
2.  **获取词元嵌入 (Token-Level Embedding Extraction)**：模型输出文档中每一个词元的、富含全局上下文信息的嵌入向量。
3.  **逻辑分块 (Logical Chunking)**：在词元级别上定义块的边界（例如，每 512 个词元）。
4.  **聚合与池化 (Aggregation & Pooling)**：将每个逻辑块内的所有词元嵌入聚合（如通过平均池化）成一个单一的向量，代表该文本块。
5.  **存储 (Storage)**：将文本块及其对应的聚合向量存入向量数据库。
6.  **检索 (Retrieval)**：使用用户的查询向量在数据库中进行相似度搜索。

---

> **核心优势**：后期分块（Late Chunking）技术使每个块的嵌入向量都能够“感知”到其在整个文档中的位置和上下文。这解决了传统分块方法中“上下文丢失”的问题，尤其对于理解跨越多个块的复杂关系、代词指代和长距离依赖的查询非常有帮助，从而显著提升了检索的准确性和相关性。