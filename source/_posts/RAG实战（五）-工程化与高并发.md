---
title: RAG实战（五）-工程化与高并发
date: 2026-03-09 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - Python
  - 并发编程
  - 学习笔记
categories:
  - 技术文章
---

本文是 RAG 实战系列第五篇，也是最后一篇。前四篇覆盖了 RAG 的核心链路：文档解析 → 向量化 → 检索 → 问答生成。但一个"能跑通"的系统和一个"能跑好"的系统之间，差距往往在工程细节上。

这一篇讲的是：**如何让 RAG 系统处理成百上千个问题时依然稳定、高效、可维护。**

> 本系列基于开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) 的工程实践总结。

---

<!-- more -->

## 目录

1. [两种并发模型：Thread vs Async，该用哪个？](#1-两种并发模型thread-vs-async该用哪个)
2. [多线程并行问答：ThreadPoolExecutor 的用法](#2-多线程并行问答threadpoolexecutor-的用法)
3. [异步批量 API 调用：令牌桶限速器](#3-异步批量-api-调用令牌桶限速器)
4. [中间结果实时持久化：防止崩溃丢失进度](#4-中间结果实时持久化防止崩溃丢失进度)
5. [配置管理：用 RunConfig 做 A/B 实验](#5-配置管理用-runconfig-做-ab-实验)
6. [目录结构设计：为什么要分这么多层？](#6-目录结构设计为什么要分这么多层)
7. [debug 文件与引用系统](#7-debug-文件与引用系统)
8. [常见错误与踩坑指南](#8-常见错误与踩坑指南)
9. [系列总结：完整 RAG 流水线一览](#9-系列总结完整-rag-流水线一览)

---

## 1. 两种并发模型：Thread vs Async，该用哪个？

在这个项目里，**两种并发模型同时存在**，各自用在不同的地方：

| 并发模型 | 使用场景 | 项目中的代码位置 |
|---------|---------|---------------|
| 多线程（`ThreadPoolExecutor`） | 并行处理多个独立问题 | `questions_processing.py` |
| 异步（`asyncio` + `aiohttp`） | 高吞吐量 API 调用（如大批量 Embedding） | `api_request_parallel_processor.py` |

**为什么不统一用一种？**

这是一个常见的困惑。选择依据是：

- **多线程**适合"每个任务本身就很慢"的场景。一道 RAG 问题的处理链路包括：检索（I/O）→ LLM 重排（网络）→ LLM 回答（网络），每一步都是外部调用，单题耗时 3-10 秒。用线程让多道题并行，10 个线程理论上缩短为原来的 1/10 时间。

- **异步**适合"需要极高吞吐量"的场景。批量 Embedding 可能要处理数千个 Chunk，每个请求只需几十毫秒，但要同时保持数百个请求在飞，同时还不能触发 Rate Limit。异步 + 令牌桶限速是这种场景的标准解法，线程池在此场景下开销过大（线程本身有内存成本）。

**一句话总结**：任务数量不多但每个很慢 → 线程；任务数量巨大但每个很快 → 异步。

---

## 2. 多线程并行问答：ThreadPoolExecutor 的用法

### 2.1 批量分发机制

`QuestionsProcessor.process_questions_list()` 负责并行处理一批问题：

```python
# src/questions_processing.py
def process_questions_list(self, questions_list, output_path=None, ...):
    parallel_threads = self.parallel_requests  # 例如 10

    if parallel_threads <= 1:
        # 串行模式：调试时用，输出有序，方便追踪
        for question_data in tqdm(questions_with_index):
            result = self._process_single_question(question_data)
            processed_questions.append(result)
            self._save_progress(...)   # 每题处理完立即存档
    else:
        # 并行模式：每次取 parallel_threads 道题，同时处理
        for i in range(0, total_questions, parallel_threads):
            batch = questions_with_index[i : i + parallel_threads]
            
            with ThreadPoolExecutor(max_workers=parallel_threads) as executor:
                # executor.map 保证结果顺序与输入顺序一致（重要！）
                batch_results = list(executor.map(
                    self._process_single_question, batch
                ))
            
            processed_questions.extend(batch_results)
            self._save_progress(...)   # 每批处理完存一次档
```

**为什么用 `executor.map` 而不是 `executor.submit`？**

`executor.map` 保证**结果顺序与输入顺序严格一致**——即使第 3 题比第 1 题先完成，`map` 也会等第 1 题完成后再一起返回。这确保了 `processed_questions` 列表的顺序是稳定的，与原始 `questions_list` 保持对应。

如果用 `executor.submit` + `as_completed`，结果会按完成顺序返回，导致最终答案文件里的问题顺序打乱，难以与原始问题列表对照。

### 2.2 线程安全的答案存储

并行处理时多个线程会同时写入 `answer_details` 列表，需要用锁保护：

```python
self.answer_details = [None] * total_questions  # 预分配列表（固定大小，线程安全地按索引写入）
self._lock = threading.Lock()

def _create_answer_detail_ref(self, answer_dict, question_index):
    """线程安全地存储答案详情"""
    ref_id = f"#/answer_details/{question_index}"
    with self._lock:                                  # 加锁，防止竞态
        self.answer_details[question_index] = {
            "step_by_step_analysis": answer_dict['step_by_step_analysis'],
            "relevant_pages": answer_dict['relevant_pages'],
            ...
            "self": ref_id
        }
    return ref_id
```

**预分配列表 vs `append`**：这里用 `[None] * total_questions` 预分配固定大小的列表，然后按索引写入（`self.answer_details[question_index] = ...`）。这比用 `append` 更安全——不同线程按不同索引写入不同位置，天然无冲突（Python 列表按索引的写入是原子操作），锁只保护极少数需要读-改-写的操作。

### 2.3 比较型问题的内嵌并行

比较型问题内部还有一层并行——对多家公司同时发起子查询：

```python
def process_comparative_question(self, question, companies, schema):
    # 外层：process_questions_list 的线程负责这整道比较题
    # 内层：对每家公司并行检索+回答
    
    with ThreadPoolExecutor() as executor:
        future_to_company = {
            executor.submit(self.get_answer_for_company, company, sub_question, "number"): company
            for company in companies
        }
        for future in as_completed(future_to_company):
            company, answer_dict = future.result()
            individual_answers[company] = answer_dict
```

这形成了**两层并行**：外层 10 个线程并行处理 10 道问题，其中比较型问题内部还会再开若干线程并行处理子公司查询。这在题目数量多且包含大量比较题时，对总耗时的缩减非常显著。

---

## 3. 异步批量 API 调用：令牌桶限速器

### 3.1 为什么需要专门的限速器

OpenAI API 有两个限制维度：

- **RPM（Requests Per Minute）**：每分钟最多发多少次请求
- **TPM（Tokens Per Minute）**：每分钟最多消耗多少 Token

如果你的代码是：

```python
# ❌ 不做限速，直接并发 1000 个请求
async def bad_approach():
    tasks = [call_api(req) for req in 1000_requests]
    await asyncio.gather(*tasks)
```

这会在 1 秒内触发 OpenAI 的 Rate Limit（429 错误），导致大量请求失败并需要重试，实际效率反而更低。

### 3.2 令牌桶算法：核心原理

`api_request_parallel_processor.py` 实现了一个经典的**令牌桶（Token Bucket）**限速器：

```
想象有两个桶：
  桶1（请求桶）：容量 = max_requests_per_minute，每秒补充 max_rpm/60 个"请求令牌"
  桶2（Token桶）：容量 = max_tokens_per_minute，每秒补充 max_tpm/60 个"Token令牌"

发起一个 API 请求时：
  1. 检查请求桶：有没有至少 1 个请求令牌？
  2. 检查 Token 桶：有没有足够的 Token 令牌（等于这个请求预估消耗的 Token 数）？
  3. 两个桶都够 → 同时从两个桶各取所需 → 发起请求
  4. 任意一个桶不够 → 等待，直到桶里积累了足够的令牌
```

代码实现：

```python
# api_request_parallel_processor.py

# 主循环：每 1ms 迭代一次
while True:
    # 根据时间流逝，向两个桶补充令牌
    current_time = time.time()
    seconds_since_update = current_time - last_update_time
    
    available_request_capacity = min(
        available_request_capacity + max_requests_per_minute * seconds_since_update / 60.0,
        max_requests_per_minute   # 桶有上限，不会无限累积
    )
    available_token_capacity = min(
        available_token_capacity + max_tokens_per_minute * seconds_since_update / 60.0,
        max_tokens_per_minute
    )
    last_update_time = current_time
    
    # 检查是否有足够容量发起下一个请求
    if next_request:
        if (available_request_capacity >= 1 
                and available_token_capacity >= next_request.token_consumption):
            # 从两个桶里各取令牌
            available_request_capacity -= 1
            available_token_capacity -= next_request.token_consumption
            # 异步发起请求（不阻塞主循环）
            asyncio.create_task(next_request.call_api(...))
            next_request = None
    
    # 如果触发了 Rate Limit，冷却 15 秒
    if seconds_since_rate_limit_error < 15:
        await asyncio.sleep(remaining_seconds_to_pause)
    
    await asyncio.sleep(0.001)   # 主循环每 1ms 检查一次
```

### 3.3 预估 Token 消耗：发送前就知道这个请求有多贵

限速的关键在于**发送请求之前就知道它会消耗多少 Token**，而不是发送后再统计。`num_tokens_consumed_from_request()` 实现了这个预估：

```python
def num_tokens_consumed_from_request(request_json, api_endpoint, token_encoding_name):
    encoding = tiktoken.get_encoding(token_encoding_name)
    
    if api_endpoint.endswith("completions"):
        # Chat Completions 请求：计算所有 messages 的 token 数 + 最大输出 token
        num_tokens = 0
        for message in request_json["messages"]:
            num_tokens += 4  # 每条消息有固定的格式开销（<im_start>role\ncontent<im_end>）
            for key, value in message.items():
                num_tokens += len(encoding.encode(value))
        num_tokens += 2  # 回复前缀开销
        num_tokens += request_json.get("max_tokens", 15)  # 预估最大输出
        return num_tokens
    
    elif api_endpoint == "embeddings":
        # Embedding 请求：直接计算输入文本的 token 数
        return sum(len(encoding.encode(text)) for text in request_json["input"])
```

这个预估的准确性决定了限速效果：过估（预估比实际多）会让吞吐量低于理论上限，但不会触发限速；欠估会触发 429 并需要降速重试。

### 3.4 失败重试机制

每个 API 请求对象（`APIRequest`）携带 `attempts_left` 计数器，失败时放回重试队列：

```python
@dataclass
class APIRequest:
    task_id: int
    request_json: dict
    token_consumption: int
    attempts_left: int   # 初始值 = max_attempts（默认 5）
    
    async def call_api(self, session, retry_queue, ...):
        try:
            response = await session.post(url, json=self.request_json)
            if "error" in response:
                if "rate limit" in response["error"]["message"].lower():
                    # 记录触发时间，主循环会自动冷却 15 秒
                    status_tracker.time_of_last_rate_limit_error = time.time()
                
                self.attempts_left -= 1
                if self.attempts_left > 0:
                    retry_queue.put_nowait(self)   # 放回队列，等待重试
                else:
                    # 超过最大重试次数，记录错误并放弃
                    append_to_jsonl([self.request_json, errors], save_filepath)
                    status_tracker.num_tasks_failed += 1
        except Exception as e:
            # 网络错误等也进入重试流程
            ...
```

主循环里，重试队列的优先级高于新请求队列：

```python
if not queue_of_requests_to_retry.empty():
    next_request = queue_of_requests_to_retry.get_nowait()  # 优先处理重试
elif file_not_finished:
    next_request = read_next_from_file()                     # 再处理新请求
```

---

## 4. 中间结果实时持久化：防止崩溃丢失进度

处理大批量问题时（竞赛数据集有数百道题），程序可能在中途因网络错误、内存不足或手动中断而崩溃。如果不做持久化，所有已完成的答案就白跑了。

### 4.1 每批完成后立即存档

```python
# 每处理完一批（parallel_threads 道题），立即写入文件
for i in range(0, total_questions, parallel_threads):
    batch_results = list(executor.map(self._process_single_question, batch))
    processed_questions.extend(batch_results)
    
    # 立即保存当前进度（覆盖写，不是追加写）
    if output_path:
        self._save_progress(
            processed_questions,
            output_path,
            submission_file=submission_file,
            ...
        )
```

这意味着如果程序在处理第 50 道题时崩溃，前 40 道题的结果（以每批 10 题为例）已经安全保存在文件里了，损失的只是当前批次的进度。

### 4.2 同时维护两个输出文件

```python
def _save_progress(self, processed_questions, output_path, submission_file=False, ...):
    statistics = self._calculate_statistics(processed_questions)
    
    # 文件1：debug 文件（完整信息，用于调试）
    result = {
        "questions": processed_questions,
        "answer_details": self.answer_details,
        "statistics": statistics
    }
    debug_file = output_file.with_name(output_file.stem + "_debug" + output_file.suffix)
    with open(debug_file, 'w', encoding='utf-8') as f:
        json.dump(result, f, ensure_ascii=False, indent=2)
    
    # 文件2：提交文件（精简格式，用于竞赛提交或对外接口）
    if submission_file:
        submission_answers = self._post_process_submission_answers(processed_questions)
        submission = {
            "answers": submission_answers,
            "team_email": team_email,
            "submission_name": submission_name,
        }
        with open(output_file, 'w', encoding='utf-8') as f:
            json.dump(submission, f, ensure_ascii=False, indent=2)
```

**为什么是覆盖写而不是追加写？**

覆盖写确保文件始终是合法的完整 JSON（包含所有已处理的答案），随时可以读取或提交。追加写在崩溃时可能留下半截 JSON，无法直接解析。

### 4.3 答案后处理：页码从 1-based 转 0-based

提交文件里的页码引用需要做一次转换：程序内部用 1-based 页码（符合人类阅读习惯），竞赛提交要求 0-based：

```python
def _post_process_submission_answers(self, processed_questions):
    for q in processed_questions:
        value = q.get("value")
        references = q.get("references", [])
        
        # N/A 的答案清空引用（不需要溯源）
        if value == "N/A":
            references = []
        else:
            # 1-based → 0-based 转换
            references = [
                {
                    "pdf_sha1": ref["pdf_sha1"],
                    "page_index": ref["page_index"] - 1   # 减 1
                }
                for ref in references
            ]
```

这里有个容易忽视的细节：**不同系统对页码的约定可能不同**。在调试阶段用 1-based 方便对照 PDF 阅读器（PDF 阅读器通常显示 1-based 页码），对外接口再转换为所需格式。在代码里统一维护一种表示，只在输出时转换，能避免很多难以排查的"差一页"错误。

---

## 5. 配置管理：用 RunConfig 做 A/B 实验

### 5.1 RunConfig 的设计

`RunConfig` 是一个 `dataclass`，把所有可调参数集中在一个地方：

```python
@dataclass
class RunConfig:
    use_serialized_tables: bool = False      # 是否使用 LLM 序列化的表格
    parent_document_retrieval: bool = False  # 是否返回整页（而非 Chunk）
    use_vector_dbs: bool = True              # 是否使用向量检索
    use_bm25_db: bool = False                # 是否使用 BM25 检索
    llm_reranking: bool = False              # 是否开启 LLM 重排
    llm_reranking_sample_size: int = 30      # LLM 重排前的候选数量
    top_n_retrieval: int = 10                # 最终返回的上下文数量
    parallel_requests: int = 10             # 并发问题数
    api_provider: str = "openai"             # LLM 提供商
    answering_model: str = "gpt-4o-mini"     # 回答模型
    full_context: bool = False              # 是否用全文档上下文（非 RAG 模式）
    config_suffix: str = ""                  # 输出文件名后缀（区分不同实验）
```

### 5.2 多套预设配置

项目里预定义了多套配置，对应不同的实验目的：

```python
# pipeline.py

# 最简配置（快速验证流程是否通）
base_config = RunConfig(
    parallel_requests=10,
    config_suffix="_base"
)

# 加入 Parent Document Retrieval
parent_document_retrieval_config = RunConfig(
    parent_document_retrieval=True,
    answering_model="gpt-4o-2024-08-06",
    config_suffix="_pdr"
)

# 全套旗舰配置（最终提交用）
max_nst_o3m_config = RunConfig(
    use_serialized_tables=False,
    parent_document_retrieval=True,
    llm_reranking=True,
    llm_reranking_sample_size=30,
    top_n_retrieval=10,
    parallel_requests=25,
    answering_model="o3-mini-2025-01-31",
    config_suffix="_max_nst_o3m"   # 输出文件：answers_max_nst_o3m.json
)

# 大上下文窗口配置
max_nst_o3m_config_big_context = RunConfig(
    parent_document_retrieval=True,
    llm_reranking=True,
    llm_reranking_sample_size=36,  # 候选数增大
    top_n_retrieval=14,            # 最终上下文数增大
    parallel_requests=5,           # 并发减小（单题消耗 token 更多，防止超限）
    answering_model="o3-mini-2025-01-31",
    config_suffix="_max_nst_o3m_bc"
)
```

**`config_suffix` 的妙用**：每个配置输出的文件名包含后缀，所以同一个数据集上跑多套配置，结果文件不会互相覆盖：

```
answers_base.json
answers_pdr.json
answers_max_nst_o3m.json
answers_max_nst_o3m_bc.json
```

可以直接对比这四个文件，看每套配置的准确率差异。这是做系统性 A/B 实验的基础设施。

### 5.3 配置如何影响路径

`PipelineConfig` 根据 `RunConfig` 的设置，自动推导出所有中间目录的路径：

```python
class PipelineConfig:
    def __init__(self, root_path, ..., serialized=False, config_suffix=""):
        # 序列化表格 → 单独的 databases 目录（避免和非序列化版本混用）
        suffix = "_ser_tab" if serialized else ""
        
        self.databases_path = root_path / f"databases{suffix}"
        # 有序列化表格时：databases_ser_tab/
        # 没有时：       databases/
        
        self.answers_file_path = root_path / f"answers{config_suffix}.json"
        # 对应 config_suffix="_max_nst_o3m" 时：answers_max_nst_o3m.json
```

序列化表格版本的数据（Chunk 里包含 LLM 序列化的表格描述）和普通版本的数据存在不同目录，因为两者的 Chunk 内容不同，向量索引也不同，不能混用。`_ser_tab` 后缀让两套数据完全隔离。

---

## 6. 目录结构设计：为什么要分这么多层？

```
data/test_set/
├── pdf_reports/               ← 原始 PDF（输入，只读）
├── subset.csv                 ← 文档元信息（sha1 ↔ 公司名映射）
├── questions.json             ← 问题列表
│
├── debug_data/                ← 中间产物（用于调试，体积大）
│   ├── 01_parsed_reports/         Docling 解析后的结构化 JSON
│   ├── 01_parsed_reports_debug/   Docling 的原始输出（包含所有元数据，极大）
│   ├── 02_merged_reports/         清洗格式化后的页面文本
│   └── 03_reports_markdown/       导出为 Markdown 的可读版本
│
├── databases/                 ← 索引数据（用于检索，频繁读取）
│   ├── chunked_reports/           带 Chunk 的 JSON 文档
│   ├── vector_dbs/                FAISS 向量索引（.faiss）
│   └── bm25_dbs/                  BM25 稀疏索引（.pkl）
│
└── answers_max_nst_o3m.json   ← 最终答案（输出）
    answers_max_nst_o3m_debug.json
```

**`debug_data` 和 `databases` 分开的原因**：

- `debug_data` 存的是中间产物，体积很大（Docling 的原始输出一个文档可能有几 MB），只在调试时才需要，生产环境可以不保留
- `databases` 存的是检索索引，每次问答都要读取，放在独立目录便于单独打包分发（竞赛里就是这样做的：`databases.zip` 单独下载，不需要重新处理 PDF）

**为什么要保留 `01_parsed_reports` 到 `03_reports_markdown` 这几个中间目录？**

每一层中间产物都可以作为重新处理的起点，不需要从头跑：

```
如果只是想调整切块参数：
  跳过解析（步骤1-3），直接从 02_merged_reports 重新切块（步骤4）

如果只是想换一个 Embedding 模型：
  跳过所有解析，直接从 03_chunked_reports 重新向量化

如果只是想测试不同的检索配置：
  跳过所有预处理，直接从 databases 开始问答
```

这让整个流水线的每个阶段都可以独立重跑，大大缩短了实验迭代周期。

---

## 7. debug 文件与引用系统

### 7.1 两个输出文件的分工

每次运行问答，都会同时生成两个文件：

| 文件 | 内容 | 用途 |
|------|------|------|
| `answers_xxx.json` | 精简版答案（value + references） | 对外提交、接口返回 |
| `answers_xxx_debug.json` | 完整版（包含推理过程、Token 统计） | 调试、定位错误 |

### 7.2 `$ref` 引用系统：避免数据冗余

debug 文件里，`questions` 数组里的每个问题不直接嵌入完整的推理过程，而是通过 `$ref` 引用：

```json
// answers_debug.json
{
  "questions": [
    {
      "question_text": "What is the operating margin?",
      "value": 9.9,
      "answer_details": {
        "$ref": "#/answer_details/1"   // ← 引用，不是嵌入
      }
    }
  ],
  "answer_details": [
    null,
    {
      "step_by_step_analysis": "...(很长的推理过程)...",
      "reasoning_summary": "...",
      "relevant_pages": [45, 44],
      "response_data": {
        "model": "o3-mini-2025-01-31",
        "input_tokens": 7168,
        "output_tokens": 1516
      },
      "self": "#/answer_details/1"    // ← 自引用，方便定位
    }
  ]
}
```

**`$ref` 的好处**：`answer_details` 里的推理过程很长（几百词），如果在 `questions` 里直接嵌入，当你想按 `question_text` 快速浏览所有问题和答案时，长篇推理过程会干扰阅读。分离存储后，`questions` 数组轻量可读，需要看推理细节时再通过 `$ref` 跳转。

### 7.3 `response_data`：成本追踪

每个 `answer_details` 里都记录了 Token 消耗：

```json
"response_data": {
    "model": "o3-mini-2025-01-31",
    "input_tokens": 7168,
    "output_tokens": 1516
}
```

这让你能精确计算每道题的 API 成本，以及不同配置的总成本差异。比如：

```python
# 从 debug 文件计算总成本
with open("answers_max_nst_o3m_debug.json") as f:
    debug = json.load(f)

total_input = sum(d["response_data"]["input_tokens"] 
                  for d in debug["answer_details"] if d and "response_data" in d)
total_output = sum(d["response_data"]["output_tokens"]
                   for d in debug["answer_details"] if d and "response_data" in d)

# o3-mini 定价（示例）
cost = total_input * 1.1/1e6 + total_output * 4.4/1e6
print(f"总成本估算：${cost:.4f}")
```

---

## 8. 常见错误与踩坑指南

### 坑 1：并发数设太高触发 Rate Limit

`parallel_requests=10` 意味着同时有 10 个 LLM 调用在进行（每个问题至少调用 1 次 LLM）。如果开启了 LLM 重排（每题额外调用 14 次），实际并发 LLM 调用可能是 10 × 15 = 150 次/分钟。

各模型的 RPM 限制不同，需要根据自己的 API Tier 调整：

```python
# 保守配置（Tier 1 用户）
RunConfig(parallel_requests=3, llm_reranking_sample_size=10)

# 激进配置（Tier 4+ 用户）
RunConfig(parallel_requests=25, llm_reranking_sample_size=30)
```

**判断是否触发限速的信号**：日志里出现大量 429 错误，或者进度条速度突然变慢（重试等待中）。

### 坑 2：debug 文件体积会超出预期

每道题的 `step_by_step_analysis` 平均 150-300 词，加上推理摘要、Token 统计，每道题的 debug 数据约 1-3 KB。处理 1000 道题，debug 文件可能达到 1-3 MB——听起来不大，但加上 `answer_details` 里的完整推理，实际往往超过 10 MB。

如果 debug 文件太大导致难以用文本编辑器打开，可以用 Python 的 `json` 模块直接读取：

```python
import json
with open("answers_debug.json") as f:
    debug = json.load(f)

# 只看统计信息
print(debug["statistics"])

# 查看第 N 道题的推理过程
q = debug["questions"][N]
ref_index = int(q["answer_details"]["$ref"].split("/")[-1])
print(debug["answer_details"][ref_index]["step_by_step_analysis"])
```

### 坑 3：`executor.map` 会吞掉子线程里的异常

`executor.map` 在迭代结果时才抛出异常，不是在提交任务时：

```python
# ❌ 这样写看不到异常，直到取结果时才报错
batch_results = list(executor.map(self._process_single_question, batch))
# 如果某个线程抛了异常，list() 才会重新抛出

# ✅ 项目里的正确处理方式：在 _process_single_question 内部 try/except
def _process_single_question(self, question_data):
    try:
        ...
    except Exception as err:
        return self._handle_processing_error(...)  # 返回错误字典而非抛异常
```

`_process_single_question` 内部把所有异常都 catch 住，返回包含 `"error"` 字段的字典，而不是让异常向上传播。这样即使某道题处理失败，整个批次依然能正常完成，不会因为一道题崩溃而中断全部处理。

### 坑 4：修改了 Chunk 内容后忘记重建索引

如果你修改了切块参数（比如把 chunk_size 从 300 改到 400），需要：

1. 重新运行 `pipeline.chunk_reports()`
2. 重新运行 `pipeline.create_vector_dbs()`（向量必须和当前 Chunk 内容对应）
3. 重新运行 `pipeline.create_bm25_db()`

只做第一步而跳过第 2、3 步，FAISS 索引里的向量数量会和 Chunk 数量不匹配，检索时会出现下标越界或结果错位。

---

## 9. 系列总结：完整 RAG 流水线一览

至此，这个系列覆盖了 RAG 系统从零到生产就绪的全部核心环节。下面是五篇文章对应的完整流水线：

```
┌─────────────────────────────────────────────────────────────────┐
│                    离线预处理阶段（一次性）                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原始 PDF                                                        │
│    │                                                            │
│    ▼  [第一篇] 文档解析与切块                                     │
│  Docling 解析（版式分析 + TableFormer + OCR）                     │
│    → 表格 LLM 序列化（可选）                                      │
│    → Markdown 格式化 + OCR 噪声清洗                               │
│    → 按 Token 切块（300 tokens，按页独立切）                       │
│    → debug_data/chunked_reports/*.json                          │
│                                                                 │
│    ▼  [第二篇] 向量化与索引构建                                   │
│  text-embedding-3-large → FAISS IndexFlatIP                     │
│    → databases/vector_dbs/*.faiss                               │
│  空格分词 → BM25Okapi → pickle                                   │
│    → databases/bm25_dbs/*.pkl                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    在线查询阶段（每次提问）                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户问题 + 公司名                                               │
│    │                                                            │
│    ▼  [第三篇] 检索与重排                                         │
│  查询向量化 → FAISS 召回 Top-30                                   │
│    → LLM 重排（14 批并行评分）→ 加权融合                          │
│    → Top-10 最相关 Chunk 或整页                                   │
│    → 格式化为带页码标注的 RAG Context                             │
│                                                                 │
│    ▼  [第四篇] 问答生成                                           │
│  按题型路由（number/boolean/name/names/comparative）              │
│    → Chain-of-Thought 结构化输出（推理先于答案）                   │
│    → 页码幻觉验证                                                 │
│    → 最终答案 + 引用页码                                          │
│                                                                 │
│    ▼  [第五篇] 工程化                                             │
│  10 题并行（ThreadPoolExecutor）                                  │
│    → 每批完成实时持久化                                            │
│    → debug 文件记录完整推理 + Token 成本                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键技术决策回顾

| 环节 | 关键决策 | 理由 |
|------|---------|------|
| PDF 解析 | Docling（模型驱动）而非 PyPDF2 | 理解版式、正确处理多列和表格 |
| 切块策略 | 按页独立切、Token 计数、Markdown 格式 | 保留页码溯源、多语言一致性、优化切割点 |
| 索引结构 | 每文档单独 FAISS + BM25 双路 | 天然支持按公司过滤；互补覆盖语义和关键词 |
| 检索增强 | Parent Document Retrieval | 小 Chunk 定位精准 + 整页上下文完整 |
| 精排 | LLM Reranking（而非 Cross-Encoder） | 无需训练，利用 LLM 的语言理解能力 |
| 输出约束 | Pydantic Structured Output | 彻底消除 JSON 解析失败，推理字段前置触发 CoT |
| 并发模型 | 多线程问答 + 异步限速 API 调用 | 两种场景的最优匹配 |
| 实验管理 | RunConfig 预设 + config_suffix 区分文件 | 多套配置互不干扰，A/B 对比简单 |

---

*本文基于 RAG-Challenge-2 项目的工程实践总结，参考代码见 `src/questions_processing.py`、`src/api_request_parallel_processor.py`、`src/pipeline.py`。*
