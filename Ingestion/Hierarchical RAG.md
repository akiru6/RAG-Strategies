# Hierarchical RAG 流程详解

## 示例代码

```python
"""Hierarchical RAG - Search small chunks, return big parents with metadata"""
from pydantic_ai import Agent
import psycopg2
from pgvector.psycopg2 import register_vector
import json

agent = Agent('openai:gpt-4o', system_prompt='You are a RAG assistant with hierarchical retrieval.')

conn = psycopg2.connect("dbname=rag_db")
register_vector(conn)

def ingest_document(text: str, doc_title: str):
    # Create parent chunks (large sections)
    parent_chunks = [text[i:i+2000] for i in range(0, len(text), 2000)]

    with conn.cursor() as cur:
        for parent_id, parent in enumerate(parent_chunks):
            # Store parent with simple metadata
            metadata = json.dumps({"heading": f"{doc_title} - Section {parent_id}", "type": "detail"})
            cur.execute('INSERT INTO parent_chunks (id, content, metadata) VALUES (%s, %s, %s)',
                       (parent_id, parent, metadata))

            # Create child chunks from parent
            child_chunks = [parent[j:j+500] for j in range(0, len(parent), 500)]
            for child in child_chunks:
                embedding = get_embedding(child)
                cur.execute(
                    'INSERT INTO child_chunks (content, embedding, parent_id) VALUES (%s, %s, %s)',
                    (child, embedding, parent_id)
                )
    conn.commit()

@agent.tool
def search_knowledge_base(query: str) -> str:
    """Search children, return parents with heading context"""
    with conn.cursor() as cur:
        query_embedding = get_embedding(query)

        # Find matching children and join with parent metadata
        cur.execute(
            '''SELECT p.content, p.metadata
               FROM child_chunks c
               JOIN parent_chunks p ON c.parent_id = p.id
               ORDER BY c.embedding <=> %s LIMIT 3''',
            (query_embedding,)
        )

        # Return parents with heading context
        results = []
        for content, metadata_json in cur.fetchall():
            metadata = json.loads(metadata_json)
            results.append(f"[{metadata['heading']}]\n{content}")

        return "\n\n".join(results)

result = agent.run_sync("What is backpropagation?")
print(result.data)

```

## 核心流程总结

### 1. 文档切分 (Ingestion)
当一个新文档被 ingest（摄入）时，系统首先将其切成较大的**父文档块**（2000字符），并为每个父块关联上包含标题和类型的元数据。然后，每个父块再被切成更小的**子文档块**（500字符）。

### 2. 向量化 (Embedding)
系统只对这些更小的**子文档块**进行向量化（`get_embedding(child)`），并将这些向量与对应的父块ID（`parent_id`）一起存入 `child_chunks` 表。

### 3. 检索 (Retrieval)
当用户提出问题时（例如 "What is backpropagation?"），系统会对用户的**问题**进行向量化。然后，它在 `child_chunks` 表中进行向量相似度搜索，找到与问题最相关的**子文档块**。

### 4. 返回结果 (Return)
系统并不会直接返回找到的子文档块。相反，它会利用子文档块中的 `parent_id`，去 `parent_chunks` 表中检索出这些子块所属的、完整的**父文档块**及其元数据。最终，将带有元数据标题（如 `[文档标题 - Section 0]`）的整个父文档块内容返回给大语言模型（LLM），从而为模型提供更完整、更丰富的上下文信息来生成答案。

---

> 这种“搜索小的，返回大的”策略，既保证了搜索的精确性（小的子块语义更集中），又保证了生成答案时上下文的完整性（大的父块信息更全面）。