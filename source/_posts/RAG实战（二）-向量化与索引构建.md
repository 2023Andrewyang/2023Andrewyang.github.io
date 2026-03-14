---
title: RAG实战（二）-向量化与索引构建
date: 2026-03-16 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - Python
  - FAISS
  - 学习笔记
categories:
  - 技术文章
---

本文是 RAG 实战系列第二篇，承接第一篇的文档解析与切块。上一篇结束时，我们已经得到了一批带元信息的 Chunk。这一篇讲的是：**如何把这批 Chunk 变成可以被快速检索的索引**。

> 本系列基于开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) 的工程实践总结。

---

<!-- more -->

## 目录

1. [为什么需要向量化？直觉理解 Embedding](#1-为什么需要向量化直觉理解-embedding)
2. [向量数据库：FAISS 的工作方式](#2-向量数据库faiss-的工作方式)
3. [稀疏检索：BM25 是什么，为什么还需要它？](#3-稀疏检索bm25-是什么为什么还需要它)
4. [双路索引：项目的实际设计](#4-双路索引项目的实际设计)
5. [代码详解：VectorDBIngestor](#5-代码详解vectordbingestor)
6. [代码详解：BM25Ingestor](#6-代码详解bm25ingestor)
7. [整个 Ingestion 流程串联](#7-整个-ingestion-流程串联)
8. [常见错误与踩坑指南](#8-常见错误与踩坑指南)
9. [完整流水线示意图](#9-完整流水线示意图)

---

## 1. 为什么需要向量化？直觉理解 Embedding

### 1.1 从"关键词匹配"到"语义匹配"

传统搜索引擎（包括 Elasticsearch 的默认模式）依赖**关键词匹配**：你搜 "revenue growth"，它就找包含这两个词的文档。

这有一个根本性的问题：**自然语言中同一个意思可以用很多不同的词表达**。

```
用户的问题：  "公司今年赚了多少钱？"
文档里的内容："净利润同比增长 23%，达到 14.2 亿元"
```

"赚了多少钱"和"净利润"没有一个相同的关键词，但它们在语义上是高度相关的。传统关键词搜索会直接错过这条结果。

**Embedding（向量化）** 解决的正是这个问题。它将一段文字转换成一个高维数值向量，这个向量在数学上**捕捉了文字的语义含义**。语义相近的文字，其向量在高维空间中的距离也更近。

### 1.2 Embedding 向量是什么样的？

OpenAI 的 `text-embedding-3-large` 模型会将一段文字输出为一个 **3072 维的浮点数数组**：

```python
# 一段文字对应的向量（维度极长，这里只展示前几个数字）
[0.0123, -0.0456, 0.0789, -0.0234, 0.0567, ...]  # 共 3072 个数字
```

这 3072 个数字共同定义了这段文字在"语义空间"里的位置。

两段文字的语义相似度，用它们向量的**内积（dot product）**或**余弦相似度（cosine similarity）**来衡量：

```python
import numpy as np

# 向量已经做过 L2 归一化，直接用内积等价于余弦相似度
similarity = np.dot(vector_a, vector_b)
# 结果在 -1 到 1 之间，越接近 1 越相似
```

### 1.3 用内积还是余弦相似度？

这是一个经常让新手困惑的问题。

**当向量经过 L2 归一化**（即 `||v|| = 1`）时，内积 = 余弦相似度，两者等价。OpenAI 的 Embedding 模型输出的向量**已经做过 L2 归一化**，所以直接用内积即可，不需要额外除以向量模长。

```python
# 验证 OpenAI 向量已经归一化
import numpy as np
norm = np.linalg.norm(embedding_vector)
print(norm)  # 输出：1.0000（非常接近 1）
```

这也是为什么 FAISS 在这个项目中使用 `IndexFlatIP`（内积索引）而不是 `IndexFlatL2`（欧氏距离索引）。

---

## 2. 向量数据库：FAISS 的工作方式

### 2.1 FAISS 是什么？

**FAISS**（Facebook AI Similarity Search）是 Meta 开源的高性能向量检索库。它能在数百万乃至数十亿条向量中，极快地找出与查询向量最相似的 Top-K 个向量。

在这个项目里，**每个文档都有一个独立的 FAISS 索引**，存储该文档所有 Chunk 的向量。

### 2.2 为什么每个文档单独建索引，而不是所有文档共用一个大索引？

这是一个重要的架构决策，原因如下：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 全局大索引 | 只需查询一次 | 无法按公司/文档过滤；更新一个文档需要重建整个索引 |
| 每文档单独索引（本项目采用）| 天然支持"按公司检索"；更新某个文档只需重建那一个索引 | 需要先找到目标文档，再查询其索引 |

本项目的使用场景是"查询某公司的年报"，用户问题里通常会指定公司名称，因此**每文档一个索引**更合适。

### 2.3 IndexFlatIP：最简单但不总是最优的索引类型

项目使用的是 `faiss.IndexFlatIP`，这是一个**精确搜索索引**：

```python
dimension = 3072  # text-embedding-3-large 的维度
index = faiss.IndexFlatIP(dimension)
index.add(embeddings_array)  # 把所有向量加进去
```

`Flat` 的含义是：查询时它会和索引中**每一个向量**都计算一次内积，保证找到绝对正确的 Top-K 结果，但时间复杂度是 O(N)。

对于单文档级别的索引（几百到几千个 Chunk），这完全够用，速度很快。如果你的场景是全局搜索数百万 Chunk，则需要考虑 `IndexIVFFlat`（近似搜索）等更高效的索引类型。

### 2.4 向量如何持久化存储？

FAISS 索引可以直接序列化为文件：

```python
# 保存
faiss.write_index(index, "abc123.faiss")

# 加载
index = faiss.read_index("abc123.faiss")
```

本项目以**文档的 SHA1 哈希值作为文件名**（如 `a1b2c3d4e5.faiss`），避免文件名冲突，并与 JSON 文档文件一一对应。

目录结构：
```
databases/
├── vector_dbs/
│   ├── a1b2c3d4e5.faiss    ← 文档 A 的向量索引
│   ├── f6e7d8c9b0.faiss    ← 文档 B 的向量索引
│   └── ...
├── bm25_dbs/
│   ├── a1b2c3d4e5.pkl      ← 文档 A 的 BM25 索引
│   └── ...
└── chunked_reports/
    ├── a1b2c3d4e5.json     ← 文档 A 的 Chunk 数据（含元信息）
    └── ...
```

---

## 3. 稀疏检索：BM25 是什么，为什么还需要它？

### 3.1 向量检索的盲区

向量检索擅长语义相似，但有一个致命盲区：**对精确的关键词、数字、专有名词不敏感**。

考虑这个例子：

```
用户问：  "2023年第三季度营收是多少？"
文档内容："Q3 2023 revenue was USD 5,234 million"
```

向量化模型在处理 "5,234" 这个具体数字时，其向量表达和其他数字（如 "5,100"、"6,000"）可能非常接近，因为它们在"数字"这个语义维度上是相似的。这就导致向量检索可能无法精确区分不同的数字值。

对于财务文档来说，**精确数字、日期、专有名词（公司名、人名）** 的检索是核心需求，向量检索在这里表现不稳定。

### 3.2 BM25：经典但有效的关键词检索

**BM25**（Best Matching 25）是一种基于词频统计的经典检索算法，是传统搜索引擎的标配。它的核心思路：

1. 查询词在文档中出现频率越高（TF，词频），分数越高
2. 查询词在**所有文档中**越稀少（IDF，逆文档频率），匹配到它价值越高
3. 对文档长度做归一化，避免长文档因词数多而占优

对于精确关键词和数字，BM25 非常擅长：

```
查询：    "Q3 2023 5,234"
文档A：  "Q3 2023 revenue was USD 5,234 million"  → 高分（完全匹配）
文档B：  "revenue was USD 5,100 million"           → 低分（关键词不匹配）
```

### 3.3 向量检索 vs BM25 优劣对比

| 维度 | 向量检索（Dense） | BM25（Sparse） |
|------|-----------------|----------------|
| 语义理解 | ✅ 强（同义词、近义词） | ❌ 弱（只看词形） |
| 精确关键词 | ⚠️ 不稳定 | ✅ 极强 |
| 数字/日期精确匹配 | ❌ 弱 | ✅ 强 |
| 专有名词 | ⚠️ 一般 | ✅ 强 |
| 速度 | 快（FAISS 优化） | 极快（纯数学计算） |
| 需要外部 API | ✅ 需要（Embedding 模型） | ❌ 不需要 |

**两者互补**，这就是为什么本项目同时构建了两套索引。

---

## 4. 双路索引：项目的实际设计

本项目采用**双路索引 + 可选 LLM 重排**的架构：

```
Chunk 列表
    │
    ├─── [路径1] 向量化 → FAISS 索引 (.faiss)
    │              ↑ text-embedding-3-large
    │
    └─── [路径2] 分词 → BM25 索引 (.pkl)

检索时：
    查询 → 同时走两条路 → 合并结果 → （可选）LLM 重排
```

两路索引都以文档的 SHA1 哈希值命名，通过文件名与 JSON Chunk 数据一一对应，检索时才从 JSON 里捞具体的 Chunk 文本内容。

---

## 5. 代码详解：VectorDBIngestor

核心代码位于 `src/ingestion.py`。我们逐步拆解。

### 5.1 初始化 OpenAI 客户端

```python
# src/ingestion.py
class VectorDBIngestor:
    def __init__(self):
        self.llm = self._set_up_llm()

    def _set_up_llm(self):
        load_dotenv()
        llm = OpenAI(
            api_key=os.getenv("OPENAI_API_KEY"),
            timeout=None,
            max_retries=2
        )
        return llm
```

`timeout=None` 是关键设置：Embedding API 对大批量文本的处理可能需要几十秒，不设 `None` 容易触发超时错误。

### 5.2 批量获取 Embedding

```python
@retry(wait=wait_fixed(20), stop=stop_after_attempt(2))
def _get_embeddings(self, text: Union[str, List[str]], model: str = "text-embedding-3-large") -> List[float]:
    if isinstance(text, list):
        # OpenAI API 单次最多处理 2048 条，这里每批 1024 条
        text_chunks = [text[i:i + 1024] for i in range(0, len(text), 1024)]
    else:
        text_chunks = [text]

    embeddings = []
    for chunk in text_chunks:
        response = self.llm.embeddings.create(input=chunk, model=model)
        embeddings.extend([embedding.embedding for embedding in response.data])
    
    return embeddings
```

**关键细节：**

1. **批量调用**：一次 API 调用可以传入一个文字列表，比逐条调用效率高很多（减少网络往返）
2. **分批处理**：OpenAI API 对单次请求的 Token 总量有限制，这里以 1024 条为一批，避免超出限制
3. **重试机制**：`@retry(wait=wait_fixed(20), stop=stop_after_attempt(2))` 在 API 失败时等待 20 秒后重试，最多重试 2 次。对于 Rate Limit 错误（429），这是必要的保护

### 5.3 创建 FAISS 索引

```python
def _create_vector_db(self, embeddings: List[float]):
    embeddings_array = np.array(embeddings, dtype=np.float32)
    dimension = len(embeddings[0])
    index = faiss.IndexFlatIP(dimension)   # IP = Inner Product（内积）
    index.add(embeddings_array)
    return index
```

几个注意点：

- `dtype=np.float32`：FAISS 只接受 float32 格式，不能用 float64。这是新手常见报错之一
- `IndexFlatIP`：使用内积作为相似度度量，适配 OpenAI 已归一化的向量
- `index.add()` 后，向量的位置（索引 0, 1, 2...）和 Chunk 列表的顺序一一对应，这是检索时能通过索引位置找回 Chunk 文本的关键

### 5.4 批量处理所有文档

```python
def process_reports(self, all_reports_dir: Path, output_dir: Path):
    all_report_paths = list(all_reports_dir.glob("*.json"))
    output_dir.mkdir(parents=True, exist_ok=True)

    for report_path in tqdm(all_report_paths, desc="Processing reports"):
        with open(report_path, 'r', encoding='utf-8') as file:
            report_data = json.load(file)
        
        # 从 JSON 中提取所有 Chunk 的文本
        index = self._process_report(report_data)
        
        # 以 sha1_name 为文件名保存
        sha1_name = report_data["metainfo"]["sha1_name"]
        faiss_file_path = output_dir / f"{sha1_name}.faiss"
        faiss.write_index(index, str(faiss_file_path))

def _process_report(self, report: dict):
    text_chunks = [chunk['text'] for chunk in report['content']['chunks']]
    embeddings = self._get_embeddings(text_chunks)
    index = self._create_vector_db(embeddings)
    return index
```

数据流：JSON 里的每个 Chunk → 提取 `text` 字段 → 一次性送给 OpenAI 做 Embedding → 得到向量列表 → 存入 FAISS → 写到磁盘。

**向量和 Chunk 的对应关系如何维护？**

这里没有额外存储映射表，而是依赖一个**隐式约定**：FAISS 索引中向量的位置 `i`，对应 JSON 文件 `content.chunks[i]`。两者的顺序必须完全一致。检索时通过 FAISS 返回的下标，直接从 chunks 数组里取对应的 Chunk：

```python
distances, indices = vector_db.search(embedding_array, k=top_n)
for index in indices[0]:
    chunk = chunks[index]   # index 就是 FAISS 返回的位置，直接对应 chunks 数组
```

---

## 6. 代码详解：BM25Ingestor

BM25 的索引构建比向量化简单很多，不需要任何外部 API。

### 6.1 创建 BM25 索引

```python
class BM25Ingestor:
    def create_bm25_index(self, chunks: List[str]) -> BM25Okapi:
        # BM25 需要预先分词，这里用最简单的空格分词
        tokenized_chunks = [chunk.split() for chunk in chunks]
        return BM25Okapi(tokenized_chunks)
```

`BM25Okapi` 是 `rank_bm25` 库实现的 Okapi BM25 变体（目前最常用的 BM25 变种）。

**分词是 BM25 的质量关键**：这里用 `split()` 按空格分词，对英文文档基本够用，但**对中文文档这是错的**。中文没有空格分隔，直接 `split()` 整段文字就是一个 token。如果你的文档包含中文，需要用 `jieba` 或 `pkuseg` 等中文分词工具：

```python
# 处理中文的正确方式
import jieba

tokenized_chunks = [list(jieba.cut(chunk)) for chunk in chunks]
```

### 6.2 序列化保存

```python
def process_reports(self, all_reports_dir: Path, output_dir: Path):
    for report_path in tqdm(all_report_paths, desc="Processing reports for BM25"):
        with open(report_path, 'r', encoding='utf-8') as f:
            report_data = json.load(f)
            
        text_chunks = [chunk['text'] for chunk in report_data['content']['chunks']]
        bm25_index = self.create_bm25_index(text_chunks)
        
        sha1_name = report_data["metainfo"]["sha1_name"]
        output_file = output_dir / f"{sha1_name}.pkl"
        with open(output_file, 'wb') as f:
            pickle.dump(bm25_index, f)   # 用 pickle 序列化
```

BM25 索引用 `pickle` 序列化为 `.pkl` 文件，加载时用 `pickle.load()` 即可。

---

## 7. 整个 Ingestion 流程串联

承接上一篇，完整的数据流如下：

```
chunked_reports/
└── a1b2c3d4e5.json           ← 上一步的输出：带 Chunk 的 JSON 文档
    {
      "metainfo": { "sha1_name": "a1b2c3d4e5", "company_name": "Example Corp", ... },
      "content": {
        "chunks": [
          {"id": 0, "page": 1, "text": "Revenue grew...", "type": "content"},
          {"id": 1, "page": 2, "text": "Net income...", "type": "serialized_table"},
          ...
        ]
      }
    }

        │
        ▼ VectorDBIngestor

vector_dbs/
└── a1b2c3d4e5.faiss           ← 每个 Chunk 的向量（顺序对应 chunks 数组）

        │
        ▼ BM25Ingestor

bm25_dbs/
└── a1b2c3d4e5.pkl             ← BM25 索引对象（分词后的倒排统计）
```

在 `pipeline.py` 中，这两步的调用方式：

```python
# pipeline.py
def create_vector_dbs(self):
    """创建向量数据库"""
    vdb_ingestor = VectorDBIngestor()
    vdb_ingestor.process_reports(
        all_reports_dir=self.paths.documents_dir,   # chunked_reports/
        output_dir=self.paths.vector_db_dir         # vector_dbs/
    )

def create_bm25_db(self):
    """创建 BM25 索引"""
    bm25_ingestor = BM25Ingestor()
    bm25_ingestor.process_reports(
        all_reports_dir=self.paths.documents_dir,   # chunked_reports/
        output_dir=self.paths.bm25_db_path          # bm25_dbs/
    )
```

**运行顺序**：先 `chunk_reports()`，再 `create_vector_dbs()` 和 `create_bm25_db()`。两个索引可以并行创建（互不依赖）。

---

## 8. 常见错误与踩坑指南

### 坑 1：FAISS 向量顺序必须与 Chunk 顺序严格一致

这是最隐蔽的 Bug，不会报错，但检索结果会完全错位。

**错误场景**：你在构建 FAISS 索引之前对 Chunk 列表排序或过滤了，但忘记对向量列表做同样的操作。

```python
# ❌ 错误示例
chunks = report['content']['chunks']
embeddings = get_embeddings([c['text'] for c in chunks])

# 排序了 Chunk，但 embeddings 没有跟着排序
chunks.sort(key=lambda x: x['page'])  # 顺序变了！
index.add(np.array(embeddings))        # embeddings 还是原来的顺序

# 检索时：index 返回位置 3 → chunks[3] 但已经不是对应那段文字了！
```

**正确做法**：Chunk 顺序确定后就不要再改变，Embedding 紧接着按同样顺序调用，两者之间不插入任何排序/过滤操作。

### 坑 2：FAISS 数组类型必须是 float32

```python
# ❌ 错误
embeddings_array = np.array(embeddings)         # 默认 float64

# ✅ 正确
embeddings_array = np.array(embeddings, dtype=np.float32)
```

FAISS 不接受 float64，会直接报 `TypeError`。但更糟糕的是，在某些版本中它会静默转换并给出错误结果，而不是报错。

### 坑 3：Embedding API 的 Token 限制

OpenAI Embedding API 单次请求有 Token 上限（`text-embedding-3-large` 约为 8191 Tokens）。如果你的某个 Chunk 超过了这个限制，整批请求都会失败。

本项目按文档分批处理（每次处理一个文档的所有 Chunk），并在切块阶段控制 Chunk 大小在 300-400 Tokens，因此不会触发这个限制。但如果你修改了切块策略，要注意这一点。

```python
# 发现 Chunk 过大时的简单保护
MAX_TOKENS = 8000
text_chunks = [chunk for chunk in text_chunks if count_tokens(chunk) < MAX_TOKENS]
```

### 坑 4：重试逻辑对 Rate Limit 至关重要

调用 Embedding API 处理大量文档时，必然会遇到 Rate Limit（429 错误）。没有重试机制的代码在大批量处理时会中途崩溃：

```python
# ❌ 没有重试保护的代码，处理到一半可能崩溃
response = openai.embeddings.create(input=chunks, model="text-embedding-3-large")

# ✅ 使用 tenacity 添加重试
from tenacity import retry, wait_fixed, stop_after_attempt

@retry(wait=wait_fixed(20), stop=stop_after_attempt(3))
def get_embeddings_with_retry(chunks):
    return openai.embeddings.create(input=chunks, model="text-embedding-3-large")
```

`wait_fixed(20)` 表示每次重试前等待 20 秒，让 Rate Limit 窗口过去。

### 坑 5：pickle 文件和系统 / 库版本绑定

BM25 索引用 `pickle` 序列化，这意味着：**如果 `rank_bm25` 库的版本升级，旧的 `.pkl` 文件可能无法加载**。同样，在不同操作系统或 Python 版本之间迁移也可能有兼容性问题。

对于生产系统，考虑在文件名或目录名中加入版本标识：

```python
output_file = output_dir / f"{sha1_name}_v1.pkl"
```

### 坑 6：BM25 中文分词问题

再次强调：默认的 `split()` 分词对中文无效。如果你的文档包含中文，务必引入中文分词：

```python
# 安装：pip install jieba
import jieba

tokenized_chunks = [list(jieba.cut(chunk)) for chunk in chunks]
bm25 = BM25Okapi(tokenized_chunks)

# 查询时也必须用同样的分词器！
tokenized_query = list(jieba.cut(query))
scores = bm25.get_scores(tokenized_query)
```

---

## 9. 完整流水线示意图

将前两篇串联起来，完整的 RAG 数据预处理流水线如下：

```
原始 PDF
    │
    ▼ [第一篇]
[步骤1] PDF 解析（Docling）
    │  → debug_data/01_parsed_reports/*.json
    │    （包含原始结构化内容：texts, tables, pictures...）
    │
    ▼
[步骤2] 表格序列化（TableSerializer + LLM）
    │  → 在 parsed_reports 的 JSON 里追加 "serialized" 字段
    │
    ▼
[步骤3] 文本清洗与 Markdown 格式化（PageTextPreparation）
    │  → debug_data/02_merged_reports/*.json
    │    （每页是一段 Markdown 文本）
    │
    ▼
[步骤4] 文本切块（TextSplitter）
    │  → databases/chunked_reports/*.json
    │    （每个文档包含 chunks 数组，每个 chunk 有 id/page/text/type）
    │
    ▼ [本篇]
[步骤5a] 向量化 → FAISS 索引（VectorDBIngestor）
    │  → databases/vector_dbs/*.faiss
    │
[步骤5b] BM25 索引（BM25Ingestor）
    │  → databases/bm25_dbs/*.pkl
    │
    ▼ [下一篇]
[步骤6] 检索（VectorRetriever / BM25Retriever / HybridRetriever）
    │
    ▼
 可用于 RAG 问答的知识库
```

### 运行所有 Ingestion 步骤的代码

```python
from pathlib import Path
from src.pipeline import Pipeline, RunConfig

root_path = Path("./data/test_set")

# 配置：不使用序列化表格（简单模式）
run_config = RunConfig(use_serialized_tables=False)
pipeline = Pipeline(root_path, run_config=run_config)

# 假设 PDF 已经解析完毕（parse_pdf_reports 已运行）
# 只运行 Ingestion 相关的步骤

pipeline.merge_reports()        # 步骤3：文本清洗+格式化
pipeline.chunk_reports()        # 步骤4：切块
pipeline.create_vector_dbs()    # 步骤5a：向量化+FAISS
pipeline.create_bm25_db()       # 步骤5b：BM25 索引

print("Ingestion 完成！")
```

---

## 总结

Ingestion 阶段的核心工作是**把 Chunk 文本转化为两种可检索的索引结构**：

1. **FAISS 向量索引**：捕捉语义相似性，处理同义词、近义词场景
2. **BM25 稀疏索引**：精确关键词匹配，处理数字、日期、专有名词场景

两套索引都以文档的 SHA1 哈希值命名，与 JSON Chunk 文件一一对应。检索时，通过索引返回的位置（下标），从 JSON 里取回对应的 Chunk 文本和页码等元信息。

核心注意事项：

1. **FAISS 向量顺序必须与 Chunk 数组顺序严格一致**，否则检索结果错位
2. **FAISS 只接受 float32**，必须显式指定 `dtype=np.float32`
3. **API 调用必须有重试机制**，大批量处理时 Rate Limit 不可避免
4. **中文文档需要专用分词器**，不能用默认的 `split()`
5. **Embedding API 有 Token 上限**，切块大小要控制在安全范围内

---

> 本系列下一篇：[RAG实战（三）-检索与重排](#)，介绍如何用这两个索引回答用户的问题，以及 LLM Reranking 如何进一步提升检索质量。

*本文基于 RAG-Challenge-2 项目的工程实践总结，参考代码见 `src/ingestion.py`、`src/retrieval.py`、`src/pipeline.py`。*
