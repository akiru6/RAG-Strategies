# Self-Reflective RAG (自我反思RAG) 流程详解

这种先进的RAG模式的核心思想是模仿人类的研究过程：**不仅仅是简单地查找信息并回答，而是对检索到的信息进行批判性评估，在结果不佳时调整策略，并在最终生成答案后进行事实核查**。它在标准的RAG流程中加入了自我评估和迭代优化的循环。

## 示例代码

```python
"""Self-Reflective RAG - Iteratively refine with self-assessment"""
from pydantic_ai import Agent
import psycopg2
from pgvector.psycopg2 import register_vector
# 假设 get_embedding, llm_grade_relevance, llm_refine,
# llm_generate, llm_verify 等函数已经定义

agent = Agent('openai:gpt-4o', system_prompt='You are a self-reflective RAG assistant.')

conn = psycopg2.connect("dbname=rag_db")
register_vector(conn)

def ingest_document(text: str):
    # 标准的文档摄入流程
    chunks = [text[i:i+500] for i in range(0, len(text), 500)]
    with conn.cursor() as cur:
        for chunk in chunks:
            embedding = get_embedding(chunk)
            cur.execute('INSERT INTO chunks (content, embedding) VALUES (%s, %s)',
                       (chunk, embedding))
    conn.commit()

@agent.tool
def search_and_grade(query: str) -> dict:
    """第一步：检索并自我评估文档相关性"""
    with conn.cursor() as cur:
        query_embedding = get_embedding(query)
        cur.execute('SELECT content FROM chunks ORDER BY embedding <=> %s LIMIT 5',
                   (query_embedding,))
        docs = [row[0] for row in cur.fetchall()]

    # 自我反思环节 1: 评估每个文档与查询的相关性
    relevant_docs = []
    for doc in docs:
        # 调用LLM来给文档打分 (0-1)
        grade = llm_grade_relevance(query, doc)
        if grade > 0.7:  # 只保留高分相关的文档
            relevant_docs.append(doc)

    # 返回过滤后的文档和一个质量分数
    return {"docs": relevant_docs, "quality": len(relevant_docs) / len(docs)}

@agent.tool
def refine_query(original_query: str, docs: list) -> str:
    """第二步(可选): 如果初次结果不佳，则优化查询"""
    # 调用LLM根据质量不高的文档来生成一个更好的查询
    return llm_refine(original_query, docs)

@agent.tool
def answer_with_verification(query: str, context: str) -> str:
    """第三步：生成答案并进行验证"""
    # 基于高质量的上下文生成答案
    answer = llm_generate(query, context)
    # 自我反思环节 2: 验证答案是否完全基于所给的上下文
    is_supported = llm_verify(answer, context)
    return answer if is_supported else "Need more context"

# Agent 会根据工具的输出来决定下一步，形成一个迭代循环
# 例如: agent.run_sync("What is quantum computing?")
```

## 核心逻辑解读

这个系统将RAG流程分解为多个可由智能代理（Agent）调度的工具，形成一个动态的、可迭代的循环。

### 1. 检索与分级 (Retrieve & Grade) - `search_and_grade`
这是整个流程的起点，也是第一个“自我反思”的环节。
*   **标准检索**：首先，它像普通的RAG一样，根据用户查询从向量数据库中检索出Top-K（这里是5）个最相似的文档块。
*   **自我评估**：**这是关键步骤**。系统不会盲目地接受所有检索到的文档。相反，它会遍历每一个文档，并调用一个LLM（`llm_grade_relevance`）来扮演“批判者”的角色，评估该文档与原始查询的**真实相关性**，并给出一个分数（例如0到1之间）。
*   **过滤**：只有分数高于某个阈值（这里是0.7）的文档才被认为是真正相关的，并被保留下来。
*   **输出质量信号**：该工具不仅返回过滤后的高质量文档，还返回一个`quality`分数（相关文档数 / 总检索文档数）。这个分数是给Agent的一个重要信号，用于决定下一步的行动。

### 2. 查询重构 (Query Refinement) - `refine_query`
这是一个可选的“自我改进”步骤，在初次检索结果不佳时触发。
*   **触发条件**：Agent在收到上一步的`quality`分数后，如果发现分数很低（例如低于0.5），就判断出初始查询可能不够好或者太模糊，导致检索效果差。
*   **重构**：此时，Agent会调用`refine_query`工具。这个工具利用LLM（`llm_refine`），结合**原始查询**和那些**被判定为不相关的文档**，来生成一个更精确、更具体的新查询。例如，将模糊的“苹果”改进为“苹果公司历史”或“苹果水果的营养价值”。

### 3. 生成与验证 (Generate & Verify) - `answer_with_verification`
这是流程的最后一步，包含第二个“自我反思”环节。
*   **生成答案**：使用经过筛选的高质量文档作为上下文，调用LLM（`llm_generate`）来生成问题的答案。
*   **最终验证**：**这是另一个关键步骤**。答案生成后，系统并不会立即将其返回给用户。而是调用另一个LLM（`llm_verify`）来执行**事实核查**。它会检查生成的答案中的每一句话是否都能在前面提供的上下文中找到依据（Groundedness Check）。
*   **决策**：只有当答案被验证为完全基于上下文、没有产生幻觉（Hallucination）时，才会将其返回给用户。否则，返回一个信息不足的提示。

## 完整的迭代循环

这个Agent的工作流程是一个动态的循环，而不是线性的：

1.  **开始**：用户提出查询 `Q1`。
2.  **检索与分级**：Agent调用 `search_and_grade(Q1)`。
3.  **决策点**：Agent检查返回的 `quality` 分数。
    *   **如果分数高**：说明检索到的文档质量很好。Agent直接跳转到第5步。
    *   **如果分数低**：说明检索效果不佳。Agent进入第4步。
4.  **查询重构与重试**：Agent调用 `refine_query(Q1, bad_docs)` 得到一个新的、更优的查询 `Q2`。然后，**循环回到第2步**，使用`Q2`重新执行检索和分级。
5.  **生成与验证**：Agent将高质量的文档作为上下文，调用 `answer_with_verification` 来生成并核查答案。
6.  **结束**：返回经过验证的、高质量的答案给用户。

---

> **核心优势**：自我反思RAG将RAG系统从一个简单的“盲目执行”的信息管道，转变为一个具备“批判性思考”能力的智能系统。它通过引入评估、迭代和验证的循环，显著提高了答案的**相关性、准确性和可靠性**，大大减少了因检索不佳或模型幻觉导致错误答案的风险。