# Context-Aware Semantic Chunking 流程详解

这种方法的核心思想是**根据语义的连贯性来切分文档，而不是固定的字符长度**。这样做可以避免将一个完整的语义单元（比如一个段落的主旨）从中间切断，从而提高检索质量。

## 示例代码

```python
"""Context-Aware Chunking - Semantic boundaries using embedding similarity"""
from pydantic_ai import Agent
import psycopg2
from pgvector.psycopg2 import register_vector
# 假设 get_embedding 和 cosine_similarity 已经定义
# from some_embedding_library import get_embedding, cosine_similarity

agent = Agent('openai:gpt-4o', system_prompt='You are a RAG assistant with semantic chunking.')

conn = psycopg2.connect("dbname=rag_db")
register_vector(conn)

def semantic_chunk(text: str, similarity_threshold=0.8) -> list[str]:
    """Chunk based on semantic similarity, not fixed size"""
    sentences = text.split('. ')  # Simple sentence split
    sentence_embeddings = [get_embedding(s) for s in sentences]

    chunks = []
    current_chunk = [sentences[0]]

    for i in range(len(sentences) - 1):
        # 计算相邻两个句子的余弦相似度
        similarity = cosine_similarity(sentence_embeddings[i], sentence_embeddings[i+1])

        if similarity > similarity_threshold:  # 语义相似，属于同一个主题
            current_chunk.append(sentences[i+1])
        else:  # 检测到主题边界，当前 chunk 结束
            chunks.append('. '.join(current_chunk))
            current_chunk = [sentences[i+1]] # 开始一个新的 chunk

    chunks.append('. '.join(current_chunk)) # 添加最后一个 chunk
    return chunks

def ingest_document(text: str):
    chunks = semantic_chunk(text)  # 使用语义分块
    with conn.cursor() as cur:
        for chunk in chunks:
            embedding = get_embedding(chunk)
            cur.execute('INSERT INTO chunks (content, embedding) VALUES (%s, %s)',
                       (chunk, embedding))
    conn.commit()

@agent.tool
def search_knowledge_base(query: str) -> str:
    with conn.cursor() as cur:
        query_embedding = get_embedding(query)
        cur.execute('SELECT content FROM chunks ORDER BY embedding <=> %s LIMIT 3',
                   (query_embedding,))
        return "\n".join([row[0] for row in cur.fetchall()])

result = agent.run_sync("What is deep learning?")
print(result.data)
```

## 核心逻辑解读

### `semantic_chunk` 函数详解
这是实现上下文感知分块的核心函数。它的工作流程如下：

1.  **切分句子**：首先，将整个文档 `text` 按照句号 (`. `) 切分成一个句子列表 `sentences`。
2.  **句子向量化**：为列表中的**每一个句子**生成一个向量嵌入（Embedding），得到 `sentence_embeddings`。
3.  **相似度比较**：
    *   从第一个句子开始，依次计算当前句子 (`i`) 和下一个句子 (`i+1`) 嵌入向量之间的**余弦相似度**。
    *   余弦相似度可以衡量两个向量在方向上的接近程度，值越接近1，代表语义越相似。
4.  **判断语义边界**：
    *   设置一个**相似度阈值** `similarity_threshold` (代码中默认为 0.8)。
    *   **如果**相邻两个句子的相似度**高于**这个阈值，说明它们在讨论同一个主题，因此将下一个句子 (`sentences[i+1]`) 添加到 `current_chunk`（当前块）中。
    *   **如果**相似度**低于**阈值，则认为这里出现了一个**语义边界**（话题发生了转变）。此时，将 `current_chunk` 中所有句子合并成一个字符串，存入最终的 `chunks` 列表，并用下一个句子开启一个新的 `current_chunk`。
5.  **生成分块**：重复上述过程，直到遍历完所有句子，最终返回一个由多个大小不一、但内部语义连贯的文本块组成的列表。

### `ingest_document` 与 `search_knowledge_base`
*   **Ingestion (摄入)**：`ingest_document` 函数调用 `semantic_chunk` 来获取语义分块。然后，它为**每一个完整的语义块**（而不是单个句子）生成一个向量嵌入，并将这个块和它的向量存入数据库。
*   **Retrieval (检索)**：`search_knowledge_base` 函数的工作方式与标准 RAG 相同。它对用户查询进行向量化，并在数据库中查找与查询向量最相似的文本块。

## 流程总结

1.  **文档预处理 (Preprocessing)**：将文档拆分为独立的句子。
2.  **逐句向量化 (Sentence Embedding)**：为每个句子创建向量表示。
3.  **边界检测 (Boundary Detection)**：通过计算相邻句子的向量相似度，识别主题的转折点。相似度低于预设阈值的地方即为边界。
4.  **分块聚合 (Chunk Aggregation)**：将语义上连续的句子合并成一个独立的、大小可变的文本块 (Chunk)。
5.  **分块向量化与存储 (Chunk Embedding & Storage)**：为每个聚合后的文本块生成一个总的向量嵌入，并将其与文本内容一同存入向量数据库。
6.  **检索 (Retrieval)**：当用户查询时，系统检索出与查询意图最相关的、语义完整的文本块。

---

> **核心优势**：与固定大小分块（Fixed-Size Chunking）相比，语义分块（Semantic Chunking）能够更好地保留上下文的完整性。它确保了每个被检索的文本块都包含了一个相对完整的思想或主题，从而为大语言模型生成更准确、更相关的答案提供了高质量的上下文信息。