---
title: RAG实战（三）-检索与重排
date: 2026-03-07 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - Python
  - 学习笔记
categories:
  - 技术文章
---

本文是 RAG 实战系列第三篇，承接第二篇的向量化与索引构建。上一篇结束时，我们已经为每个文档构建好了两套索引：FAISS 向量语义索引和 BM25 关键词稀疏索引。这一篇讲的是：**用户提问时，如何从这些索引里找到最相关的 Chunk，并进一步提升检索质量**。

> 本系列基于开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) 的工程实践总结。

---

<!-- more -->

## 目录

1. [检索的整体思路：从问题到上下文](#1-检索的整体思路从问题到上下文)
2. [代码详解：VectorRetriever（向量检索）](#2-代码详解vectorretriever向量检索)
3. [代码详解：BM25Retriever（关键词检索）](#3-代码详解bm25retriever关键词检索)
4. [Parent Document Retrieval：一个重要的工程技巧](#4-parent-document-retrieval一个重要的工程技巧)
5. [LLM 重排：让 LLM 替你判断哪条结果更相关](#5-llm-重排让-llm-替你判断哪条结果更相关)
6. [HybridRetriever：向量检索 + LLM 重排的组合](#6-hybridretriever向量检索--llm-重排的组合)
7. [检索结果如何变成 LLM 的输入](#7-检索结果如何变成-llm-的输入)
8. [配置参数的权衡：top_n、sample_size 该怎么设？](#8-配置参数的权衡top_nsample_size-该怎么设)
9. [常见错误与踩坑指南](#9-常见错误与踩坑指南)
10. [完整检索流程串联](#10-完整检索流程串联)

---

## 1. 检索的整体思路：从问题到上下文

检索阶段的任务很直白：**给定用户的问题，从已建好的索引里找出最相关的若干 Chunk，拼成一段上下文（Context），交给 LLM 回答**。

但"最相关"的定义并不简单。本项目提供了三种检索策略，复杂度依次递增：

```
策略一：纯向量检索（VectorRetriever）
  查询 → Embedding → FAISS 搜索 → Top-N Chunk

策略二：纯 BM25 检索（BM25Retriever）
  查询 → 分词 → BM25 打分 → Top-N Chunk

策略三：混合检索 + LLM 重排（HybridRetriever）
  查询 → Embedding → FAISS 搜索 → 召回 K 个候选
       → LLM 逐批打分 → 加权排序 → Top-N Chunk
```

**策略一**适合大多数场景，速度快，语义理解好；  
**策略二**在关键词精确匹配场景（如具体数字、日期）表现更稳；  
**策略三**是最精准但最慢的方案，适合对质量要求极高的生产系统。

本项目在最终提交中主要使用策略三。接下来逐一拆解。

---

## 2. 代码详解：VectorRetriever（向量检索）

核心代码位于 `src/retrieval.py`。

### 2.1 初始化：预加载所有索引

```python
class VectorRetriever:
    def __init__(self, vector_db_dir: Path, documents_dir: Path):
        self.vector_db_dir = vector_db_dir
        self.documents_dir = documents_dir
        self.all_dbs = self._load_dbs()   # 启动时加载所有文档和索引
        self.llm = self._set_up_llm()
```

`_load_dbs()` 在对象初始化时就把**所有文档的 FAISS 索引和 JSON 数据一次性读入内存**：

```python
def _load_dbs(self):
    all_dbs = []
    all_documents_paths = list(self.documents_dir.glob('*.json'))
    # 构建 stem → faiss 文件路径的映射字典
    vector_db_files = {db_path.stem: db_path for db_path in self.vector_db_dir.glob('*.faiss')}
    
    for document_path in all_documents_paths:
        stem = document_path.stem  # 即 sha1_name，例如 "a1b2c3d4e5"
        if stem not in vector_db_files:
            _log.warning(f"No matching vector DB found for document {document_path.name}")
            continue
        
        # 验证文档结构完整性
        with open(document_path, 'r', encoding='utf-8') as f:
            document = json.load(f)
        if not (isinstance(document, dict) and "metainfo" in document and "content" in document):
            _log.warning(f"Skipping {document_path.name}: does not match the expected schema.")
            continue
        
        # 加载 FAISS 索引
        vector_db = faiss.read_index(str(vector_db_files[stem]))
        
        all_dbs.append({
            "name": stem,
            "vector_db": vector_db,
            "document": document
        })
    return all_dbs
```

**为什么要预加载，而不是按需加载？**

FAISS 索引从磁盘加载（`faiss.read_index`）有固定的 I/O 开销。如果每次查询都临时加载，在并发处理大量问题时，反复的磁盘读取会成为明显的性能瓶颈。预加载虽然占用更多内存，但换来了检索时的零 I/O 延迟。

### 2.2 核心检索逻辑

```python
def retrieve_by_company_name(
    self, 
    company_name: str, 
    query: str, 
    top_n: int = 3, 
    return_parent_pages: bool = False
) -> List[Dict]:
    
    # 1. 按公司名找到目标文档
    target_report = next(
        (r for r in self.all_dbs 
         if r["document"]["metainfo"]["company_name"] == company_name),
        None
    )
    if target_report is None:
        raise ValueError(f"No report found with '{company_name}' company name.")
    
    document = target_report["document"]
    vector_db = target_report["vector_db"]
    chunks = document["content"]["chunks"]
    pages = document["content"]["pages"]
    
    # 2. 将查询文本向量化
    embedding = self.llm.embeddings.create(
        input=query,
        model="text-embedding-3-large"
    )
    embedding = embedding.data[0].embedding
    embedding_array = np.array(embedding, dtype=np.float32).reshape(1, -1)
    
    # 3. FAISS 搜索 Top-N
    distances, indices = vector_db.search(x=embedding_array, k=top_n)
    
    # 4. 组装返回结果
    retrieval_results = []
    for distance, index in zip(distances[0], indices[0]):
        chunk = chunks[index]   # 通过位置索引取回 Chunk
        result = {
            "distance": round(float(distance), 4),
            "page": chunk["page"],
            "text": chunk["text"]
        }
        retrieval_results.append(result)
    
    return retrieval_results
```

整个检索过程可以浓缩为三步：

```
用户问题（文字）
    │
    ▼ text-embedding-3-large
查询向量（3072 维 float32）
    │
    ▼ faiss.search(k=top_n)
Top-N 相似向量的位置索引（indices）和相似度分数（distances）
    │
    ▼ chunks[index]
Top-N 个最相关的 Chunk 文本
```

**`distances` 返回的是什么值？**

因为使用的是 `IndexFlatIP`（内积），`distances` 返回的是**内积值**，即余弦相似度（OpenAI 向量已归一化）。值域在 `-1` 到 `1` 之间，**越大越相关**。这与欧氏距离（L2）相反——L2 距离越小越相关，注意不要混淆。

### 2.3 静态工具方法：计算任意两段文字的相似度

`VectorRetriever` 还提供了一个实用的静态方法，可以直接计算两段文字之间的语义相似度，不需要实例化对象：

```python
@staticmethod
def get_strings_cosine_similarity(str1, str2):
    llm = VectorRetriever.set_up_llm()
    embeddings = llm.embeddings.create(
        input=[str1, str2], 
        model="text-embedding-3-large"
    )
    embedding1 = embeddings.data[0].embedding
    embedding2 = embeddings.data[1].embedding
    similarity = np.dot(embedding1, embedding2) / (
        np.linalg.norm(embedding1) * np.linalg.norm(embedding2)
    )
    return round(similarity, 4)

# 使用示例
score = VectorRetriever.get_strings_cosine_similarity(
    "公司今年赚了多少钱？",
    "净利润同比增长 23%，达到 14.2 亿元"
)
print(score)  # 例如：0.7823
```

这在调试阶段非常有用，比如验证某个 Chunk 和某个查询的相似度是否符合预期。

---

## 3. 代码详解：BM25Retriever（关键词检索）

BM25 检索的代码结构和向量检索类似，但有几个关键差异。

```python
class BM25Retriever:
    def __init__(self, bm25_db_dir: Path, documents_dir: Path):
        self.bm25_db_dir = bm25_db_dir
        self.documents_dir = documents_dir
        # 注意：BM25Retriever 没有预加载，是按需加载
```

**注意：BM25Retriever 没有预加载**，每次检索时才临时加载目标文档的索引：

```python
def retrieve_by_company_name(self, company_name: str, query: str, top_n: int = 3, ...) -> List[Dict]:
    
    # 1. 遍历文档目录，找到目标公司的文档（每次都扫描磁盘）
    document_path = None
    for path in self.documents_dir.glob("*.json"):
        with open(path, 'r', encoding='utf-8') as f:
            doc = json.load(f)
            if doc["metainfo"]["company_name"] == company_name:
                document_path = path
                document = doc
                break
    
    # 2. 加载对应的 BM25 索引
    bm25_path = self.bm25_db_dir / f"{document['metainfo']['sha1_name']}.pkl"
    with open(bm25_path, 'rb') as f:
        bm25_index = pickle.load(f)
    
    # 3. 分词查询，获取所有 Chunk 的 BM25 分数
    tokenized_query = query.split()
    scores = bm25_index.get_scores(tokenized_query)
    
    # 4. 取 Top-N
    top_indices = sorted(
        range(len(scores)), 
        key=lambda i: scores[i], 
        reverse=True
    )[:top_n]
    
    # 5. 组装结果
    retrieval_results = []
    for index in top_indices:
        chunk = chunks[index]
        retrieval_results.append({
            "distance": round(float(scores[index]), 4),
            "page": chunk["page"],
            "text": chunk["text"]
        })
    
    return retrieval_results
```

**BM25 的分数含义**：BM25 打分结果是一个**非负浮点数**，没有固定上限，不同文档之间的绝对值不可直接比较，只有同一文档内的相对大小有意义。这和向量检索的余弦相似度（固定在 -1 到 1 之间）有本质区别。

---

## 4. Parent Document Retrieval：一个重要的工程技巧

### 4.1 问题背景

切块（Chunk）的目的是提高向量检索精度：小 Chunk 语义集中，更容易和查询匹配。但这带来了一个新问题：**小 Chunk 提供给 LLM 的上下文太少，LLM 可能因信息不足而回答错误**。

举个例子：

```
Chunk（300 tokens）：
"Revenue was USD 5,234 million, representing a 12.3% increase 
year-over-year compared to the prior period."

用户问题：
"公司 2023 年 Q3 的营收是多少？"
```

这个 Chunk 完全匹配，检索成功。但如果 LLM 需要知道 "prior period" 是哪个期间来核实数字的正确性，它就找不到答案了——因为那个信息在相邻的 Chunk 里，没有被一起检索进来。

### 4.2 解决方案：检索 Chunk，返回整页

**Parent Document Retrieval** 的核心思路：

- **用小 Chunk 做检索**（精度高，向量语义集中）
- **返回 Chunk 所在的整个页面给 LLM**（上下文完整）

```
检索阶段：
  查询向量 → FAISS → 命中 Chunk（300 tokens，精确）
                          │
                          ▼
              找到这个 Chunk 的 page 字段（例如 page=12）
                          │
                          ▼
              返回整个第 12 页的文本（1500 tokens，完整）
```

代码里通过 `return_parent_pages=True` 开启这个模式：

```python
def retrieve_by_company_name(self, ..., return_parent_pages: bool = False):
    ...
    seen_pages = set()  # 防止同一页被重复返回
    
    for distance, index in zip(distances[0], indices[0]):
        chunk = chunks[index]
        parent_page = next(
            page for page in pages if page["page"] == chunk["page"]
        )
        
        if return_parent_pages:
            if parent_page["page"] not in seen_pages:
                seen_pages.add(parent_page["page"])
                result = {
                    "distance": distance,
                    "page": parent_page["page"],
                    "text": parent_page["text"]   # 整页文本，而不是 Chunk 文本
                }
                retrieval_results.append(result)
        else:
            result = {
                "distance": distance,
                "page": chunk["page"],
                "text": chunk["text"]             # 仅 Chunk 文本
            }
            retrieval_results.append(result)
```

`seen_pages` 用于去重：如果两个相邻的 Chunk 恰好都在第 12 页，不会把第 12 页的内容返回两次。

### 4.3 代价与权衡

| 模式 | 单条结果 Token 数 | 优点 | 缺点 |
|------|----------------|------|------|
| Chunk 模式 | ~300 tokens | 信噪比高，LLM 聚焦 | 可能缺少上下文 |
| Parent Page 模式 | ~1500 tokens | 上下文完整，LLM 信息充足 | 消耗更多 LLM 上下文窗口 |

本项目最优配置（`max_nst_o3m_config`）中 `parent_document_retrieval=True`，这是经过实验验证对最终答案质量提升显著的设置。

---

## 5. LLM 重排：让 LLM 替你判断哪条结果更相关

### 5.1 向量检索的天花板

向量检索的精度取决于 Embedding 模型的语义理解能力。对于微妙的区分——比如"Q3 营收"和"Q3 净利润"——Embedding 向量可能相似度都很高，但 LLM 能一眼看出哪个才是真正的答案。

这就是 LLM 重排（Reranking）的价值：用 LLM 的理解能力，对向量检索的候选集做二次精排。

### 5.2 重排的整体流程

```
用户问题
    │
    ▼ 向量检索
召回 K=28 个候选 Chunk（大撒网）
    │
    ▼ LLM 重排（每批 2 个 Chunk 一起评分）
每个 Chunk 获得 0~1 的相关度分数
    │
    ▼ 加权融合（0.7 × LLM分 + 0.3 × 向量分）
按综合分排序
    │
    ▼
取 Top-N=6 个最终结果
```

### 5.3 结构化输出：用 Pydantic 约束 LLM 的打分格式

重排的评分由 LLM 输出，但 LLM 的输出如果是自由文本，解析起来很脆弱。本项目使用 **Pydantic Schema + OpenAI Structured Output** 来强制 LLM 返回结构化 JSON：

```python
# src/prompts.py
from pydantic import BaseModel, Field
from typing import List

class RetrievalRankingSingleBlock(BaseModel):
    """单块的评分结果"""
    reasoning: str = Field(description="分析文本块和查询的相关性")
    relevance_score: float = Field(description="相关度评分，0~1，越大越相关")

class RetrievalRankingMultipleBlocks(BaseModel):
    """多块的评分结果"""
    block_rankings: List[RetrievalRankingSingleBlock]
```

OpenAI 的 `beta.chat.completions.parse()` API 接受这个 Pydantic 模型作为 `response_format`，**保证 LLM 的输出 100% 符合这个 Schema**，不需要再做额外的 JSON 解析和容错处理：

```python
completion = self.llm.beta.chat.completions.parse(
    model="gpt-4o-mini-2024-07-18",
    temperature=0,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt},
    ],
    response_format=RetrievalRankingMultipleBlocks   # 直接传入 Pydantic 类
)

response = completion.choices[0].message.parsed     # 返回的直接是 Pydantic 对象
rankings = response.block_rankings                  # 类型安全，直接访问字段
```

### 5.4 重排的 Prompt 设计

Prompt 把评分标准描述得非常细致，0.1 为一档，共 11 档：

```python
system_prompt = """
你是一个 RAG 检索结果排序器。

相关度评分标准（0 到 1，以 0.1 为单位）：
  0   = 完全不相关：文本块与查询毫无关联
  0.3 = 轻微相关：涉及查询的极小方面，缺乏实质内容
  0.5 = 中等相关：部分回答了查询，但不完整
  0.7 = 相关：与查询明显相关，提供了实质性信息
  0.9 = 高度相关：几乎完整地回答了查询
  1.0 = 完全相关：直接且全面地回答了查询
"""
```

细粒度的评分标准能让 LLM 做出更有区分度的判断，而不是简单地打 0 或 1。

### 5.5 并行批处理：加速重排

每次 LLM 调用都有几百毫秒的延迟。如果对 28 个候选 Chunk 逐一调用，串行执行需要几十秒。本项目用 `ThreadPoolExecutor` 并行处理：

```python
def rerank_documents(self, query, documents, documents_batch_size=2, llm_weight=0.7):
    # 将 28 个 Chunk 分成 14 批，每批 2 个
    doc_batches = [
        documents[i:i + documents_batch_size] 
        for i in range(0, len(documents), documents_batch_size)
    ]
    
    def process_batch(batch):
        texts = [doc['text'] for doc in batch]
        # 一次 LLM 调用，给 2 个 Chunk 同时打分
        rankings = self.get_rank_for_multiple_blocks(query, texts)
        
        results = []
        for doc, rank in zip(batch, rankings['block_rankings']):
            doc_with_score = doc.copy()
            doc_with_score["relevance_score"] = rank["relevance_score"]
            # 加权融合：LLM 分（0.7）+ 向量相似度分（0.3）
            doc_with_score["combined_score"] = round(
                llm_weight * rank["relevance_score"] + 
                (1 - llm_weight) * doc['distance'],
                4
            )
            results.append(doc_with_score)
        return results
    
    # 14 批并行执行，总耗时约等于 1 次 LLM 调用的时间
    with ThreadPoolExecutor() as executor:
        batch_results = list(executor.map(process_batch, doc_batches))
    
    all_results = [doc for batch in batch_results for doc in batch]
    all_results.sort(key=lambda x: x["combined_score"], reverse=True)
    return all_results
```

**批大小（`documents_batch_size`）的选择**：每批 2 个（默认值），让 LLM 同时看到两段文本并进行相对比较，比单独评分更有区分度。批太大（>4）时 LLM 开始出现"排名遗忘"问题（前面的 Chunk 打分受后面影响），批太小（1 个）则 LLM 只能做绝对判断，区分度下降。

### 5.6 加权分数的含义

最终分数是两个维度的加权平均：

```
combined_score = llm_weight × relevance_score + (1 - llm_weight) × distance

默认：combined_score = 0.7 × LLM评分 + 0.3 × 向量相似度
```

| 分数来源 | 权重（默认） | 含义 |
|---------|------------|------|
| `relevance_score`（LLM 打分） | 0.7（70%） | LLM 对 Chunk 与查询相关性的主观判断，0~1 |
| `distance`（FAISS 内积） | 0.3（30%） | 向量空间里的语义相似度，约 0~1 |

LLM 权重更高（0.7），因为 LLM 有更强的语义理解能力。向量分数作为辅助，防止 LLM 遗漏明显相关但表述特殊的 Chunk。

---

## 6. HybridRetriever：向量检索 + LLM 重排的组合

`HybridRetriever` 是对 `VectorRetriever` 和 `LLMReranker` 的封装，对外提供统一接口：

```python
class HybridRetriever:
    def __init__(self, vector_db_dir: Path, documents_dir: Path):
        self.vector_retriever = VectorRetriever(vector_db_dir, documents_dir)
        self.reranker = LLMReranker()
    
    def retrieve_by_company_name(
        self, 
        company_name: str,
        query: str,
        llm_reranking_sample_size: int = 28,  # 向量召回候选数量
        documents_batch_size: int = 2,         # 每批 LLM 评分的 Chunk 数
        top_n: int = 6,                        # 最终返回数量
        llm_weight: float = 0.7,               # LLM 分数权重
        return_parent_pages: bool = False
    ) -> List[Dict]:
        
        # 第一步：向量检索召回 K 个候选（大网）
        vector_results = self.vector_retriever.retrieve_by_company_name(
            company_name=company_name,
            query=query,
            top_n=llm_reranking_sample_size,   # 注意：这里用 sample_size 而不是 top_n
            return_parent_pages=return_parent_pages
        )
        
        # 第二步：LLM 重排（精筛）
        reranked_results = self.reranker.rerank_documents(
            query=query,
            documents=vector_results,
            documents_batch_size=documents_batch_size,
            llm_weight=llm_weight
        )
        
        # 第三步：返回 Top-N
        return reranked_results[:top_n]
```

**`llm_reranking_sample_size` 和 `top_n` 的关系**：

```
向量检索召回 28 个候选
    │
    ▼ LLM 重排（对 28 个打分）
    │
    ▼ 取前 6 个（top_n=6）
    │
最终输入 LLM 的上下文
```

召回越多（sample_size 越大），重排后的质量越好，但 LLM 重排的 API 调用成本也越高。`28` 是本项目实验后的平衡点。

---

## 7. 检索结果如何变成 LLM 的输入

检索拿到 Chunk 之后，需要把它们格式化成一个**上下文字符串**，传给回答 LLM。格式化的方式直接影响 LLM 是否能正确理解来源和定位信息。

```python
# src/questions_processing.py
def _format_retrieval_results(self, retrieval_results) -> str:
    """把检索结果拼成 RAG Context 字符串"""
    context_parts = []
    for result in retrieval_results:
        page_number = result['page']
        text = result['text']
        context_parts.append(
            f'Text retrieved from page {page_number}: \n"""\n{text}\n"""'
        )
    
    return "\n\n---\n\n".join(context_parts)
```

格式化后的上下文长这样：

```
Text retrieved from page 12: 
"""
## Financial Highlights

Revenue in Q3 2023 reached USD 5,234 million, a 12.3% increase
year-over-year...
"""

---

Text retrieved from page 8: 
"""
## Revenue Overview

The company's primary revenue streams consist of...
"""
```

**为什么要标注页码？**

这不只是为了"显示来源"，更重要的是：**LLM 在回答时会引用页码作为参考（`relevant_pages` 字段），而这个页码会被验证是否确实出现在检索结果里**：

```python
def _validate_page_references(self, claimed_pages, retrieval_results, ...):
    """防止 LLM 幻觉出不存在的页码"""
    retrieved_pages = [result['page'] for result in retrieval_results]
    # 只保留实际出现在检索结果里的页码
    validated_pages = [page for page in claimed_pages if page in retrieved_pages]
    
    if len(validated_pages) < len(claimed_pages):
        removed = set(claimed_pages) - set(validated_pages)
        print(f"Warning: Removed {len(removed)} hallucinated page references: {removed}")
    
    return validated_pages
```

这是一个关键的**幻觉防控机制**：LLM 有时会捏造它没有看到的页码，这段验证代码会把那些不在检索结果里的页码过滤掉，确保最终引用来源的真实性。

---

## 8. 配置参数的权衡：top_n、sample_size 该怎么设？

本项目在 `pipeline.py` 里定义了多套配置，实验了不同参数组合，最终结论如下：

### 8.1 top_n_retrieval：最终返回的 Chunk 数

```python
# 本项目不同配置的 top_n 设置
base_config:      top_n_retrieval = 10   # 默认值
max_nst_o3m:      top_n_retrieval = 10   # 最优配置（实验表现最好）
big_context:      top_n_retrieval = 14   # 更大上下文窗口时可适当增大
```

- **太少（< 5）**：可能漏掉关键信息，LLM 上下文不足
- **太多（> 15）**：LLM 上下文过长，噪声增加，注意力分散，成本上升
- **建议范围：6-12**

### 8.2 llm_reranking_sample_size：LLM 重排前的向量召回数

```python
# 实验配置
max_nst_o3m:        llm_reranking_sample_size = 30  （默认）
big_context_config: llm_reranking_sample_size = 36
```

召回数 / 最终返回数 的比例，决定重排的"筛选强度"：

```
sample_size = 30, top_n = 10 → 筛选比例 3:1
sample_size = 36, top_n = 14 → 筛选比例 2.6:1
```

筛选比例越高，重排的精选效果越好，但 LLM 评分的 API 开销也越大。**3:1** 是实验中表现较好的比例。

### 8.3 documents_batch_size：每次 LLM 调用评分的 Chunk 数

```python
HybridRetriever.retrieve_by_company_name(
    documents_batch_size=2   # 默认值
)
```

- **1（单块模式）**：LLM 独立评每个 Chunk，主观判断，但无相对比较
- **2（默认）**：LLM 同时看 2 个 Chunk，做相对排序，区分度更好
- **4+（批量）**：LLM 一次评分多个，但超过 4 个时有遗忘问题

**推荐保持默认值 2**。

### 8.4 不同策略的性能对比（参考）

| 策略 | 延迟（单问题） | LLM API 调用次数 | 答案质量 |
|------|-------------|----------------|---------|
| 纯向量检索 | ~1s | 1 次（查询 Embedding）+ 1 次（回答） | 一般 |
| 向量检索 + 重排（sample=30, batch=2） | ~3-5s | 1 + 15 次（重排）+ 1（回答） | 高 |
| 全文上下文（retrieve_all） | ~10s | 0（无检索）+ 1 次（超长上下文回答） | 取决于模型上下文窗口 |

---

## 9. 常见错误与踩坑指南

### 坑 1：查询和索引用了不同的 Embedding 模型

**这是最致命的错误，会导致检索结果完全随机**。建索引时用了 `text-embedding-3-large`，检索时不小心用了 `text-embedding-ada-002`，两个模型的向量空间完全不同，计算出来的相似度毫无意义。

```python
# ❌ 错误：索引用 large，查询用 ada
index_embedding = openai.embeddings.create(input=text, model="text-embedding-3-large")
query_embedding = openai.embeddings.create(input=query, model="text-embedding-ada-002")  # 不一致！

# ✅ 正确：始终使用同一个模型
MODEL = "text-embedding-3-large"
index_embedding = openai.embeddings.create(input=text, model=MODEL)
query_embedding = openai.embeddings.create(input=query, model=MODEL)
```

建议把模型名定义为常量，两处引用同一个变量，从根本上避免这个问题。

### 坑 2：`return_parent_pages=True` 时 top_n 需要相应调整

开启 Parent Page 模式后，多个 Chunk 可能映射到同一页，导致实际返回的结果少于 `top_n`：

```
top_n=10，但 Chunk 1、3、7 都在第 12 页
→ 去重后实际只返回 8 页内容（不是 10 个）
```

如果你的下游逻辑依赖精确的 top_n 数量，需要考虑这个情况。实践建议是把 top_n 设得稍大一点（比如 12），预留去重后的余量。

### 坑 3：BM25 分数和向量分数不能直接加权

BM25 分数是无界的正数（可以是 0、3.5、12.7...），向量相似度固定在 0-1 之间。直接加权融合两者在数学上是错的：

```python
# ❌ 错误：量纲不一致，BM25 分数会主导结果
combined = 0.5 * bm25_score + 0.5 * vector_similarity

# ✅ 正确做法一：对 BM25 分数做 MinMax 归一化
max_score = max(bm25_scores)
normalized_bm25 = [s / max_score for s in bm25_scores]

# ✅ 正确做法二：分别排名，然后融合排名（Reciprocal Rank Fusion）
```

本项目的 `HybridRetriever` 只使用向量分数做加权基础（BM25 和向量是两个独立的 Retriever，不做融合），这避开了这个问题。

### 坑 4：LLM 重排的结果数量可能少于预期

LLM 在批量评分时偶尔会"少输出"一个结果（比如要求评 2 个 Chunk，只给了 1 个评分）。代码中已做保护：

```python
if len(block_rankings) < len(batch):
    print(f"Warning: Expected {len(batch)} rankings but got {len(block_rankings)}")
    # 对缺失的 Chunk 补充默认评分 0.0
    for _ in range(len(batch) - len(block_rankings)):
        block_rankings.append({
            "relevance_score": 0.0,
            "reasoning": "Default ranking due to missing LLM response"
        })
```

这个保护逻辑非常重要——如果不处理，`zip(batch, rankings)` 会默默丢掉多余的 Chunk，导致数据错位。

### 坑 5：预加载索引时内存消耗

`VectorRetriever._load_dbs()` 会把所有文档的 FAISS 索引和 JSON 数据全部加载进内存。如果有 100 个文档，每个文档有 500 个 Chunk，每个向量 3072 维 float32：

```
100 文档 × 500 Chunk × 3072 维 × 4 字节 = ~600MB
```

加上 JSON 文档数据，总内存消耗可能超过 1GB。在内存受限的环境（如小型云服务器）中需要注意，可以考虑按需加载或使用内存映射（mmap）。

---

## 10. 完整检索流程串联

将三篇串联起来，从原始 PDF 到最终答案的完整流程如下：

```
原始 PDF
    │
    ▼ [第一篇] 文档解析与切块
databases/chunked_reports/abc123.json
    {
      "metainfo": { "company_name": "Example Corp", ... },
      "content": {
        "chunks": [{"id": 0, "page": 1, "text": "..."}, ...],
        "pages": [{"page": 1, "text": "整页 Markdown 文本"}, ...]
      }
    }
    │
    ▼ [第二篇] 向量化与索引构建
databases/vector_dbs/abc123.faiss   ← FAISS 向量索引
databases/bm25_dbs/abc123.pkl       ← BM25 稀疏索引
    │
    ▼ [本篇] 检索
用户问题："Example Corp 2023 年 Q3 营收是多少？"
    │
    ├─ VectorRetriever（向量检索）
    │     查询 Embedding → FAISS.search → Top-28 候选 Chunk
    │
    ├─ LLMReranker（重排）
    │     14 批 × 并行 LLM 评分 → 加权融合分数 → 重新排序
    │
    └─ 返回 Top-6 最相关 Chunk（或整页，取决于 return_parent_pages）
    │
    ▼ 格式化为 RAG Context
"Text retrieved from page 12: \"Revenue in Q3 2023 was...\"\n\n---\n\n..."
    │
    ▼ LLM 回答
{
  "step_by_step_analysis": "...",
  "relevant_pages": [12, 8],
  "final_answer": 5234.0
}
```

### 快速上手代码（最小可运行示例）

```python
from pathlib import Path
from src.retrieval import VectorRetriever, HybridRetriever

# 初始化（预加载所有索引，耗时几秒）
retriever = VectorRetriever(
    vector_db_dir=Path("databases/vector_dbs"),
    documents_dir=Path("databases/chunked_reports")
)

# 基础向量检索
results = retriever.retrieve_by_company_name(
    company_name="Example Corp",
    query="What was the revenue in Q3 2023?",
    top_n=5
)

for r in results:
    print(f"[Page {r['page']}] 相似度: {r['distance']:.4f}")
    print(r['text'][:200])
    print("---")
```

```python
# 开启 Parent Document Retrieval（返回整页而非 Chunk）
results = retriever.retrieve_by_company_name(
    company_name="Example Corp",
    query="What was the revenue in Q3 2023?",
    top_n=5,
    return_parent_pages=True   # 检索 Chunk，返回整页
)
```

```python
# 使用带 LLM 重排的 HybridRetriever（最高精度）
from src.retrieval import HybridRetriever

hybrid = HybridRetriever(
    vector_db_dir=Path("databases/vector_dbs"),
    documents_dir=Path("databases/chunked_reports")
)

results = hybrid.retrieve_by_company_name(
    company_name="Example Corp",
    query="What was the revenue in Q3 2023?",
    llm_reranking_sample_size=28,   # 向量召回 28 个候选
    documents_batch_size=2,          # 每批 2 个 Chunk 给 LLM 评分
    top_n=6,                         # 最终返回 6 个
    llm_weight=0.7,                  # LLM 分数权重
    return_parent_pages=True
)
```

---

## 总结

检索阶段是 RAG 系统中**最直接决定答案质量**的环节。一个好的检索器能把正确的信息送到 LLM 面前；一个差的检索器，无论 LLM 多强大，也无法回答它没看到的内容。

本项目的检索策略要点回顾：

1. **向量检索是基础**：语义匹配，速度快，适合大多数查询场景
2. **查询和索引必须用同一个 Embedding 模型**，这是最不能犯的错误
3. **Parent Document Retrieval 是性价比极高的技巧**：用小 Chunk 精准定位，返回整页保证上下文完整
4. **LLM 重排可以显著提升精度**，但有成本代价，适合质量要求高的生产场景
5. **加权融合需要注意量纲**：BM25 分数和向量分数不能直接混合
6. **幻觉防控不能省**：验证 LLM 引用的页码是否确实来自检索结果

---

> 本系列下一篇：[RAG实战（四）-问答生成](#)，介绍如何设计 Prompt、如何处理不同类型的问题、如何做结构化输出。

*本文基于 RAG-Challenge-2 项目的工程实践总结，参考代码见 `src/retrieval.py`、`src/reranking.py`、`src/questions_processing.py`、`src/pipeline.py`。*
