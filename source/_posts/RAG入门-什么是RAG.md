---
title: RAG入门-什么是RAG
date: 2026-03-04 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - 学习笔记
categories:
  - 技术文章
---

这篇文章是我学习 RAG 过程中整理的入门笔记，主要参考了开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) ——一个赢得了 Enterprise RAG Challenge 竞赛全部奖项的企业级年报问答系统，以及作者 Ilya Rice 的技术复盘博客 [How I Won the Enterprise RAG Challenge](https://abdullin.com/ilya/how-to-build-best-rag/)。

本文以这个项目为主线，帮助完全没接触过 RAG 的同学快速建立概念框架。

---

<!-- more -->

## 目录

1. [RAG 是什么？先讲个故事](#1-rag-是什么先讲个故事)
2. [为什么需要 RAG？LLM 的三个硬伤](#2-为什么需要-rag-llm-的三个硬伤)
3. [RAG 能做什么？典型应用场景](#3-rag-能做什么典型应用场景)
4. [RAG 的基本流程：从文档到答案](#4-rag-的基本流程从文档到答案)
   - [阶段一：离线索引（建库）](#阶段一离线索引建库)
   - [阶段二：在线检索与生成（问答）](#阶段二在线检索与生成问答)
5. [本项目是怎么做的？结合代码看流程](#5-本项目是怎么做的结合代码看流程)
6. [进阶话题：让 RAG 更好用的几个技巧](#6-进阶话题让-rag-更好用的几个技巧)
7. [总结：一张图看懂 RAG](#7-总结一张图看懂-rag)

---

## 1. RAG 是什么？先讲个故事

想象这样一个场景：

> 你刚入职一家公司，老板扔给你一摞 500 页的内部规章制度，然后问你："我们公司差旅报销的上限是多少？"

你有两种做法：

- **做法 A（纯记忆）**：把 500 页全部背下来，然后凭记忆回答。  
- **做法 B（检索 + 理解）**：翻目录，找到"差旅报销"那几页，读完后给出答案。

**做法 B 就是 RAG 的思路。**

**RAG**，全称 **Retrieval-Augmented Generation**，中文叫**检索增强生成**。它的核心思想是：

> 先从知识库里**检索**出与问题最相关的内容，再把这些内容**塞给 LLM**，让 LLM 基于这些上下文**生成**答案。

一句话概括：**RAG = 检索相关资料 + 让 LLM 读资料回答问题**。

---

## 2. 为什么需要 RAG？LLM 的三个硬伤

大语言模型（LLM，比如 GPT-4）很强大，但在企业落地中有三个致命问题：

### 硬伤一：知识有截止日期

LLM 的训练数据有个时间截点，之后发生的事情它一无所知。如果你的业务依赖最新的数据（比如今年的财报、最新的产品文档），纯 LLM 是没法回答的。

### 硬伤二：不知道你的私有数据

你公司的内部文件、客户合同、产品手册，LLM 从来没见过。你不可能把所有私有数据都"训练进"模型——成本极高，而且数据还会不断更新。

### 硬伤三：会"幻觉"（胡说）

当 LLM 不知道某个问题的答案时，它不会说"我不知道"，而是倾向于**编造一个听起来合理的答案**。这在需要精确数字的场景（比如财务问答）是致命的。

**RAG 的出现正是为了解决这三个问题：**

| 问题 | RAG 的解法 |
|---|---|
| 知识有截止日期 | 实时更新知识库，检索最新内容 |
| 不知道私有数据 | 把私有文档建成知识库，按需检索 |
| 会幻觉 | 把检索到的原文作为依据，LLM 基于事实回答 |

---

## 3. RAG 能做什么？典型应用场景

RAG 在企业中有大量落地场景，这里列举最常见的几类：

### 企业知识库问答
员工可以直接问："我们公司的年假政策是什么？"系统自动从内部 HR 文档中检索并回答，不需要再翻几百页 PDF。

### 金融/法律文档分析
本项目就是这类场景——针对上市公司年报（PDF 格式）进行问答，例如："苹果公司 2023 年的营业利润是多少？""和去年相比增长了多少？"

### 客服机器人
接入产品手册和常见问题文档，让客服机器人能够基于实际资料回答用户问题，而不是瞎编。

### 代码库助手
把代码仓库的注释、文档、README 建成知识库，帮助开发者快速理解陌生代码。

### 医疗/科研辅助
基于最新的论文库或医疗指南回答专业问题，保证答案有文献来源。

---

## 4. RAG 的基本流程：从文档到答案

RAG 的完整流程分为两个阶段：**离线索引（建库）** 和 **在线检索与生成（问答）**。

### 阶段一：离线索引（建库）

这个阶段在用户提问之前就完成了，相当于提前"整理好图书馆"。

```
原始文档（PDF/Word/网页）
        ↓
  文档解析（提取文字）
        ↓
  文本分块（Chunking）
        ↓
  向量化（Embedding）
        ↓
  存入向量数据库
```

**Step 1：文档解析**

把 PDF、Word 等格式的文档转成纯文本。这一步看似简单，实际上很有挑战性——PDF 里可能有复杂的表格、多栏排版、图片中的文字等，需要专门的解析工具处理。

**Step 2：文本分块（Chunking）**

把长文档切成一个个小片段（Chunk）。为什么要切？因为 LLM 的上下文窗口有限，而且整篇文档丢给 LLM 处理既慢又贵。

> 类比：把一本书按章节撕开，每章单独存档，方便后续按需取用。

切块有讲究：块太小，上下文不足，意思不完整；块太大，引入太多无关内容，干扰 LLM 判断。通常一块大约 **200~500 个 Token**（大约 150~400 个汉字或英文单词）。

**Step 3：向量化（Embedding）**

这是 RAG 的核心魔法之一。

用一个 **Embedding 模型**把每个文本块转成一串数字（向量）。这串数字代表了文本的"语义"，语义相似的文本，它们的向量在空间中也会很接近。

> 类比：给每篇文章贴一个"坐标"，相似话题的文章坐标相近。

**Step 4：存入向量数据库**

把所有文本块和它们对应的向量都存到向量数据库中（比如 FAISS、Pinecone、Chroma）。向量数据库擅长做"相似度搜索"——给一个查询向量，快速找出最相似的几个文本块。

---

### 阶段二：在线检索与生成（问答）

用户提问后，实时执行以下步骤：

```
用户提问
    ↓
问题向量化
    ↓
向量数据库相似度搜索
    ↓
取出 Top-K 相关文本块
    ↓
拼接成 Prompt（提示词）
    ↓
LLM 生成答案
    ↓
返回给用户
```

**Step 1：问题向量化**

用同一个 Embedding 模型把用户的问题也转成向量。

**Step 2：相似度搜索**

在向量数据库里找出与问题向量最相似的 Top-K 个文本块。这些就是"最可能包含答案的段落"。

**Step 3：构建 Prompt**

把检索到的文本块拼在一起，构建成给 LLM 的提示词，格式大概是：

```
你是一个专业的分析师。请根据以下资料回答问题。

【参考资料】
资料1：苹果公司2023年营业利润为114亿美元...
资料2：相比2022年的98亿美元，增长了约16%...

【用户问题】
苹果公司2023年营业利润是多少？

请基于以上资料给出答案，如果资料中没有相关信息，请直接说"信息不足"。
```

**Step 4：LLM 生成答案**

LLM 读取这个 Prompt，基于提供的"参考资料"来生成答案。由于有原文依据，幻觉大大减少。

---

## 5. 本项目是怎么做的？结合代码看流程

这个项目是一个针对**上市公司年报（PDF）的问答系统**，参加了 RAG 挑战赛。下面我们对照代码来看它是如何实现上面每个步骤的。

### 项目整体结构

```
RAG-Challenge-2/
├── src/
│   ├── pipeline.py          # 主流程控制
│   ├── pdf_parsing.py       # PDF 解析
│   ├── text_splitter.py     # 文本分块
│   ├── ingestion.py         # 向量化 & 建库
│   ├── retrieval.py         # 检索
│   ├── reranking.py         # 重排序（进阶）
│   ├── questions_processing.py  # 问题处理 & 生成答案
│   └── api_requests.py      # 调用 LLM API
└── data/
    ├── pdf_reports/         # 原始年报 PDF
    └── databases/           # 向量数据库
```

### 离线建库流程

项目在 `pipeline.py` 中把建库分成了几个清晰的步骤：

```python
# pipeline.py - process_parsed_reports 方法
def process_parsed_reports(self):
    print("Step 1: Merging reports...")
    self.merge_reports()          # 解析 PDF 后整理结构

    print("Step 2: Exporting reports to markdown...")
    self.export_reports_to_markdown()  # 导出成可读格式（用于调试）

    print("Step 3: Chunking reports...")
    self.chunk_reports()          # 文本分块

    print("Step 4: Creating vector databases...")
    self.create_vector_dbs()      # 向量化 & 建库
```

**文本分块的参数设置**

在 `text_splitter.py` 中，可以看到分块的具体参数：

```python
# text_splitter.py
def _split_page(self, page, chunk_size=300, chunk_overlap=50):
    text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
        model_name="gpt-4o",
        chunk_size=300,    # 每块约 300 个 Token
        chunk_overlap=50   # 相邻块重叠 50 个 Token，避免断句
    )
```

> **为什么要有重叠（overlap）？** 因为答案可能正好跨在两块的边界处。重叠区域确保即使切块位置不理想，关键信息也不会完全丢失。

**向量化与建库**

在 `ingestion.py` 中，使用 OpenAI 的 `text-embedding-3-large` 模型进行向量化，并用 FAISS 存储：

```python
# ingestion.py - VectorDBIngestor
def _get_embeddings(self, text, model="text-embedding-3-large"):
    response = self.llm.embeddings.create(input=text, model=model)
    return [embedding.embedding for embedding in response.data]

def _create_vector_db(self, embeddings):
    embeddings_array = np.array(embeddings, dtype=np.float32)
    dimension = len(embeddings[0])
    index = faiss.IndexFlatIP(dimension)  # 用内积（余弦相似度）衡量相似性
    index.add(embeddings_array)
    return index
```

### 在线问答流程

问题进来后，在 `questions_processing.py` 中处理：

```python
# questions_processing.py - get_answer_for_company
def get_answer_for_company(self, company_name, question, schema):
    # 1. 检索相关文本块
    retrieval_results = retriever.retrieve_by_company_name(
        company_name=company_name,
        query=question,
        top_n=self.top_n_retrieval   # 默认取 Top-10 个最相关块
    )
    
    # 2. 把检索结果格式化成字符串
    rag_context = self._format_retrieval_results(retrieval_results)
    
    # 3. 调用 LLM，把问题和检索内容一起传入
    answer_dict = self.openai_processor.get_answer_from_rag_context(
        question=question,
        rag_context=rag_context,
        schema=schema,
        model=self.answering_model
    )
    return answer_dict
```

**检索结果如何格式化成 Prompt？**

```python
# questions_processing.py - _format_retrieval_results
def _format_retrieval_results(self, retrieval_results):
    context_parts = []
    for result in retrieval_results:
        page_number = result['page']
        text = result['text']
        context_parts.append(
            f'Text retrieved from page {page_number}: \n"""\n{text}\n"""'
        )
    return "\n\n---\n\n".join(context_parts)
```

检索到的段落被格式化成带页码标注的文本块，方便 LLM 在回答时引用具体来源。

---

## 6. 进阶话题：让 RAG 更好用的几个技巧

基础 RAG 搭起来之后，实际效果可能还不够好。工程师们总结了一些提升质量的技巧，本项目也用到了其中几个。

### 技巧一：混合检索（Hybrid Retrieval）

纯向量检索对"语义相似"的查询效果好，但对精确关键词（比如公司名、专有名词）可能反而不准。

**解法**：向量检索 + 关键词检索（BM25）双路并行，取两者结果的并集，再统一排序。

本项目同时实现了 `VectorRetriever`（向量检索）和 `BM25Retriever`（关键词检索）两种检索器。

```python
# retrieval.py 中存在两种检索器
class VectorRetriever:   # 向量相似度检索
    ...

class BM25Retriever:     # 关键词匹配检索（基于 BM25 算法）
    ...
```

### 技巧二：重排序（Reranking）

向量检索返回的 Top-K 结果，排序不一定完全准确。重排序的思路是：**先粗检索拿到候选集，再用更强的模型精排**。

本项目实现了两种重排器：

- **`JinaReranker`**：调用 Jina AI 的专业重排序 API，速度快
- **`LLMReranker`**：用 GPT-4o-mini 直接判断每个文本块与问题的相关性，精度更高

```python
# reranking.py - LLMReranker
def rerank_documents(self, query, documents, llm_weight=0.7):
    """
    最终得分 = 0.7 × LLM 相关性评分 + 0.3 × 向量距离
    """
    doc_with_score["combined_score"] = round(
        llm_weight * ranking["relevance_score"] +
        vector_weight * doc['distance'],
        4
    )
```

### 技巧三：父文档检索（Parent Document Retrieval）

切块越细，检索越精准；但块太小，给 LLM 的上下文可能不够完整。

**解法**：用小块做检索（提高精度），但返回小块所属的完整页面（保证上下文完整）。

```python
# 在 retrieve_by_company_name 中，通过 return_parent_pages 参数控制
if return_parent_pages:
    # 找到 chunk 对应的完整 page，返回整页内容
    parent_page = next(page for page in pages if page["page"] == chunk["page"])
    result = {"text": parent_page["text"], ...}
```

### 技巧四：防止幻觉——验证引用页码

LLM 生成答案时可能引用一些实际上不存在于检索结果中的页码（这就是幻觉）。本项目专门加了一层验证：

```python
# questions_processing.py - _validate_page_references
def _validate_page_references(self, claimed_pages, retrieval_results):
    retrieved_pages = [result['page'] for result in retrieval_results]
    # 过滤掉 LLM 捏造的页码
    validated_pages = [page for page in claimed_pages if page in retrieved_pages]
    
    if len(validated_pages) < len(claimed_pages):
        print(f"Warning: Removed hallucinated page references")
    return validated_pages
```

---

## 7. 总结：一张图看懂 RAG

```
┌─────────────────────────────────────────────────────────────────┐
│                      【离线建库阶段】                              │
│                                                                   │
│   PDF/文档  →  文本解析  →  分块(Chunking)  →  向量化(Embedding)  │
│                                               ↓                  │
│                                          向量数据库               │
└─────────────────────────────────────────────────────────────────┘
                                               ↑
                                           提前建好
                                               ↓
┌─────────────────────────────────────────────────────────────────┐
│                      【在线问答阶段】                              │
│                                                                   │
│   用户提问  →  问题向量化  →  相似度搜索  →  Top-K 文本块         │
│                                               ↓                  │
│                                         构建 Prompt              │
│                                               ↓                  │
│                                        LLM 生成答案               │
│                                               ↓                  │
│                                          返回给用户               │
└─────────────────────────────────────────────────────────────────┘
```

### 关键术语速查

| 术语 | 解释 |
|---|---|
| **RAG** | Retrieval-Augmented Generation，检索增强生成 |
| **Embedding** | 将文本转成数字向量的过程，语义相似的文本向量相近 |
| **Chunk** | 切分后的文本片段，一般 200~500 Token |
| **向量数据库** | 存储向量并支持相似度搜索的专用数据库（如 FAISS） |
| **Top-K 检索** | 找出与查询最相似的 K 个文本块 |
| **Reranking** | 对初步检索结果进行精排，提高最终质量 |
| **上下文窗口** | LLM 一次能处理的最大文本长度 |
| **幻觉（Hallucination）** | LLM 生成看似合理但实际错误信息的现象 |

### 一句话总结

> RAG 就是：**"给 LLM 配一个能实时查资料的助手，让它基于真实文档来回答问题，而不是靠背诵或瞎编。"**

---

**参考资源**

- 开源项目：[IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2)
- 作者博客：[How I Won the Enterprise RAG Challenge · Ilya Rice](https://abdullin.com/ilya/how-to-build-best-rag/)

*本文所有代码示例均来自上述开源项目，感谢作者 Ilya Rice 的无私分享。*
