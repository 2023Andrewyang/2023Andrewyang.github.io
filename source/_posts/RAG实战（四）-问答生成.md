---
title: RAG实战（四）-问答生成
date: 2026-03-08 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - Python
  - Prompt Engineering
  - 学习笔记
categories:
  - 技术文章
---

本文是 RAG 实战系列第四篇，承接第三篇的检索与重排。上一篇结束时，我们已经拿到了最相关的若干 Chunk（或整页），拼成了一段 RAG Context。这一篇讲的是：**把 Context 和问题交给 LLM，如何设计 Prompt、如何处理不同类型的问题、如何拿到结构化答案、以及怎么让 LLM 不瞎编。**

> 本系列基于开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) 的工程实践总结。

---

<!-- more -->

## 目录

1. [问答生成的核心挑战](#1-问答生成的核心挑战)
2. [按题型路由：五种问题五套 Prompt](#2-按题型路由五种问题五套-prompt)
3. [Chain-of-Thought + 结构化输出：强制 LLM 先推理再回答](#3-chain-of-thought--结构化输出强制-llm-先推理再回答)
4. [各题型 Prompt 的关键设计细节](#4-各题型-prompt-的关键设计细节)
5. [比较型问题：Query Routing 模式](#5-比较型问题query-routing-模式)
6. [多 LLM 提供商的统一适配](#6-多-llm-提供商的统一适配)
7. [非 OpenAI 模型的结构化输出容错](#7-非-openai-模型的结构化输出容错)
8. [完整问答流程串联](#8-完整问答流程串联)
9. [常见错误与踩坑指南](#9-常见错误与踩坑指南)

---

## 1. 问答生成的核心挑战

很多新手以为到了问答阶段只需要一个 Prompt：

```python
# "够用但不好"的写法
response = llm.chat("根据以下内容回答问题：\n{context}\n\n问题：{question}")
```

这在 Demo 里能跑通，但在生产中会遇到一系列问题：

**问题一：LLM 会直接输出自由文本，无法程序化处理**  
你需要从回答里判断是 `True`/`False`，还是提取一个数字，自由文本格式极难解析。

**问题二：LLM 会跳步推理，直接给结论**  
如果不强制推理过程，LLM 倾向于快速给出答案，在复杂语义判断题上容易出错。

**问题三：不同题型需要截然不同的约束**  
"营收是多少"和"是否发生了并购"需要完全不同的 Schema 约束和指导语。对数字题你要严格禁止 LLM 做数学计算；对布尔题你要防止它把"会计政策中提到了并购处理方式"误判为"发生了并购"。

**问题四：LLM 会幻觉出不存在的信息**  
Context 里没有的内容，LLM 有时会"想象"出来填充答案。

本项目用两个核心机制解决这些问题：**按题型分发不同 Prompt** + **Chain-of-Thought 结构化输出**。

---

## 2. 按题型路由：五种问题五套 Prompt

项目把所有问题分为五种类型（`kind` 字段），每种类型有独立的 Prompt 和 Output Schema：

```
kind = "name"        → 提取单个名称（CEO姓名、产品名等）
kind = "number"      → 提取单个数值（营收、利润率等）
kind = "boolean"     → 是/否判断（是否发生了某事件）
kind = "names"       → 提取名称列表（所有新任高管姓名等）
kind = "comparative" → 多公司比较（哪家公司营收更高）
```

路由逻辑在 `api_requests.py` 的 `APIProcessor._build_rag_context_prompts()` 里：

```python
def _build_rag_context_prompts(self, schema):
    """根据题型返回对应的 (system_prompt, response_format, user_prompt) 三元组"""
    
    # IBM/Gemini 不支持原生 Structured Output，需要在 Prompt 里附上 JSON Schema
    use_schema_prompt = True if self.provider in ("ibm", "gemini") else False
    
    if schema == "name":
        system_prompt = (AnswerWithRAGContextNamePrompt.system_prompt_with_schema 
                        if use_schema_prompt else AnswerWithRAGContextNamePrompt.system_prompt)
        response_format = AnswerWithRAGContextNamePrompt.AnswerSchema
        user_prompt = AnswerWithRAGContextNamePrompt.user_prompt
        
    elif schema == "number":
        ...
    elif schema == "boolean": ...
    elif schema == "names": ...
    elif schema == "comparative": ...
    
    return system_prompt, response_format, user_prompt
```

这个设计的核心价值在于：**每种题型的约束完全隔离**，修改数字题的 Prompt 不会影响布尔题，新增题型也不需要改动现有逻辑。

---

## 3. Chain-of-Thought + 结构化输出：强制 LLM 先推理再回答

### 3.1 Output Schema 的设计

所有题型的 Output Schema 都遵循同一个模式——**先推理，后结论**：

```python
class AnswerSchema(BaseModel):
    step_by_step_analysis: str   # 字段1：详细推理过程（至少5步、150词）
    reasoning_summary: str       # 字段2：推理摘要（约50词）
    relevant_pages: List[int]    # 字段3：答案所在页码
    final_answer: ...            # 字段4：最终答案（类型因题型而异）
```

**为什么把 `step_by_step_analysis` 放在 `final_answer` 前面？**

这不是偶然的顺序。OpenAI 的 Structured Output 会按照 Schema 字段定义的顺序依次生成内容。把推理过程放前面，意味着 LLM 在填写 `final_answer` 之前**已经完成了完整的推理链**。

这利用了大语言模型的自回归特性：生成 `final_answer` 时，模型的注意力机制会"回顾"前面已生成的推理过程，这和直接问"答案是什么"相比，等效于做了一次思维链提示（Chain-of-Thought Prompting），但无需在 Prompt 里显式写出"请一步一步思考"。

```python
# Field 描述对 LLM 的约束力非常强
step_by_step_analysis: str = Field(description="""
Detailed step-by-step analysis of the answer 
with at least 5 steps and at least 150 words.
Pay special attention to the wording of the question 
to avoid being tricked.
""")
```

`at least 5 steps and at least 150 words` 这个约束防止 LLM 用两句话走完推理，强迫它展开分析。

### 3.2 使用 OpenAI Structured Output

```python
# src/api_requests.py
completion = self.llm.beta.chat.completions.parse(
    model=model,
    temperature=0,              # temperature=0 保证确定性输出
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": user_prompt.format(context=rag_context, question=question)},
    ],
    response_format=AnswerSchema   # 直接传 Pydantic 类，保证 100% 符合 Schema
)

response = completion.choices[0].message.parsed   # 直接得到 Pydantic 对象
answer_dict = response.dict()
```

`beta.chat.completions.parse()` 与普通的 `chat.completions.create()` 的区别：前者在 API 层面就强制约束输出格式，如果 LLM 生成的内容不符合 Schema，OpenAI 服务端会自动修正，直到符合为止。这彻底消除了"JSON 解析失败"的可能性（对 OpenAI 而言）。

---

## 4. 各题型 Prompt 的关键设计细节

### 4.1 number 题：最严格的约束

数字类问题最容易出错，因为 LLM 倾向于在 Context 中找到"相关数字"后直接给出，而不是找"精确匹配的指标"。

`AnswerWithRAGContextNumberPrompt` 的核心约束写在 `step_by_step_analysis` 的 Field 描述里：

```python
step_by_step_analysis: str = Field(description="""
Detailed step-by-step analysis with at least 5 steps and at least 150 words.
**Strict Metric Matching Required:**

1. Determine the precise concept the question's metric represents.
2. Examine potential metrics in the context.
3. Accept ONLY if: The context metric's meaning *exactly* matches the target metric.
   Synonyms are acceptable; conceptual differences are NOT.
4. Reject (and use 'N/A') if:
   - The context metric covers more or less than the question's metric.
   - The context metric is a related concept but not the exact equivalent.
   - Answering requires calculation, derivation, or inference.
   - Aggregation Mismatch: question needs a single value but context offers only aggregated total
5. No Guesswork: If any doubt exists about the metric's equivalence, default to N/A.
""")
```

**"不允许计算"这条规则是关键**。考虑这个场景：

```
问题：  "Ritter Pharmaceuticals 的研发设备原始成本是多少？"
Context："Property and equipment, net: $12,500"
         "Accumulated Depreciation: $110,000"
```

LLM 可能会尝试：原始成本 = 净值 + 折旧 = 12,500 + 110,000 = 122,500。但这是错误的——问题要求的是原始成本，Context 里根本没有这个数字，计算出来的结果并不可信。正确答案是 `N/A`。

`final_answer` 的 Field 描述里还有另一个重要的数字处理规则：

```python
final_answer: Union[float, int, Literal['N/A']] = Field(description="""
Pay special attention to any mentions about whether metrics are reported 
in units, thousands, or millions:
- Value from context: 4970.5 (in thousands $)
  Final answer: 4970500          ← 要换算成实际值，补三个零

Pay attention if value wrapped in parentheses, it means NEGATIVE:
- Value from context: (2,124,837) CHF
  Final answer: -2124837         ← 括号 = 负数（财务报表惯例）
""")
```

这两条规则处理的是财务报表的惯用表达，新手极容易在此出错：
- **千元单位**：财务报表里 "4,970.5 (in thousands)" 的真实值是 4,970,500，不是 4,970.5
- **括号负数**：财务报表里 `(2,124,837)` 是 `-2,124,837`，不是正数

### 4.2 boolean 题：防止"政策描述"误判为"事件发生"

布尔题最常见的错误是：报告里有一章叫"Business Combinations 会计政策"，LLM 误以为这代表"发生了并购"。

Prompt 里特别强调了这个区别：

```python
step_by_step_analysis: str = Field(description="""
Detailed step-by-step analysis with at least 5 steps and at least 150 words.
Pay special attention to the wording of the question to avoid being tricked.
Sometimes it seems that there is an answer in the context, but this might 
be not the requested value, but only a similar one.
""")
```

配合 Example 里的反面案例（这是 Prompt 里最有价值的部分之一）：

```python
example = r"""
Question: "Did W. P. Carey Inc. announce any changes to its dividend policy?"

Answer:
{
  "step_by_step_analysis": "...
  4. Consistent, incremental increases throughout the year, with explicit mentions 
     of maintaining a 'steady and growing' dividend, indicates no changes to *policy*, 
     though the *amount* increased as planned within the existing policy.",
  
  "final_answer": False    ← 股息金额变了，但政策没变，答案是 False
}
"""
```

这个 Example 展示了一个关键的语义区分：**"股息金额变化"** ≠ **"股息政策变化"**。Example 在 Prompt 里的作用是：让 LLM 知道"我应该做多细粒度的区分"，这比任何文字描述都更直观。

### 4.3 names 题：位置名称 vs 人名的区分

`names` 题（复数）专门用于"列举多个名称"的场景，比如"哪些高管发生了职位变动"。

```python
final_answer: Union[List[str], Literal["N/A"]] = Field(description="""
If question asks about POSITIONS (e.g., changes in positions), 
return ONLY position titles, WITHOUT names or additional info.
Appointments on new leadership positions also count as changes.
If several changes related to position with same title, return title only once.
Position title should always be in SINGULAR form.

Example: ['Chief Technology Officer', 'Board Member', 'Chief Executive Officer']

If question asks about NAMES, return ONLY full names exactly as in context.
Example: ['Carly Kennedy', 'Brian Appelgate Jr.']
""")
```

这个约束解决了一个具体问题：如果问的是"哪些职位发生了变动"，LLM 如果回答 `["Carly Kennedy - EVP", "Brian Appelgate - Interim COO"]` 就是错的，正确答案是 `["Executive Vice President", "Chief Operations Officer"]`。

### 4.4 name 题和 names 题的区别

| 题型 | 适用场景 | `final_answer` 类型 |
|------|---------|-------------------|
| `name` | 单个名称（CEO是谁） | `Union[str, "N/A"]` |
| `names` | 名称列表（所有新高管） | `Union[List[str], "N/A"]` |

两者共享基础 Prompt 框架，但 Schema 不同，且 `names` 题的 Field 描述对"返回位置名称还是人名"有额外的规则判断。

### 4.5 共享的基础 Prompt

所有题型的 `instruction` 部分来自同一个基类：

```python
class AnswerWithRAGContextSharedPrompt:
    instruction = """
You are a RAG (Retrieval-Augmented Generation) answering system.
Your task is to answer the given question based only on information 
from the company's annual report, which is uploaded in the format 
of relevant pages extracted using RAG.

Before giving a final answer, carefully think out loud and step by step.
Pay special attention to the wording of the question.
- Keep in mind that the content containing the answer may be worded 
  differently than the question.
- The question was autogenerated from a template, so it may be 
  meaningless or not applicable to the given company.
"""
```

最后一句 **"The question was autogenerated from a template, so it may be meaningless"** 是竞赛场景特有的提示——问题是模板生成的，某些问题对特定公司不适用（比如"公司的主要产品是什么"，但这家公司是金融机构没有实体产品），这时候正确答案是 `N/A` 而不是强行找一个答案。

---

## 5. 比较型问题：Query Routing 模式

### 5.1 什么是比较型问题

```
"Which company had higher revenue in 2022, 'Apple' or 'Microsoft'?"
```

这类问题涉及多个公司，不能直接套单公司的检索流程，需要分别查询每家公司的年报再做比较。这是 RAG 系统中典型的 **Query Routing（查询路由）** 场景。

### 5.2 四步处理流程

```python
# src/questions_processing.py
def process_comparative_question(self, question, companies, schema):
    
    # Step 1: 用 LLM 把比较题拆成独立子问题
    rephrased_questions = self.openai_processor.get_rephrased_questions(
        original_question=question,
        companies=companies
    )
    # 输入: "Which had higher revenue, 'Apple' or 'Microsoft'?"
    # 输出: {
    #   "Apple": "What was Apple's revenue in 2022?",
    #   "Microsoft": "What was Microsoft's revenue in 2022?"
    # }
    
    # Step 2: 对每家公司并行执行独立的 RAG 问答
    individual_answers = {}
    with ThreadPoolExecutor() as executor:
        futures = {
            executor.submit(self.get_answer_for_company, company, sub_question, "number"): company
            for company, sub_question in rephrased_questions.items()
        }
        for future in as_completed(futures):
            company, answer_dict = future.result()
            individual_answers[company] = answer_dict
    
    # Step 3: 把各公司的独立答案汇总，再调用一次 LLM 做比较判断
    comparative_answer = self.openai_processor.get_answer_from_rag_context(
        question=question,           # 原始比较问题
        rag_context=individual_answers,  # 各公司独立答案作为上下文
        schema="comparative",
        model=self.answering_model
    )
    
    # Step 4: 聚合所有公司的引用页码
    comparative_answer["references"] = aggregated_references
    return comparative_answer
```

### 5.3 第一步：问题拆解的 Prompt

问题拆解本身也是一次 LLM 调用，使用 `RephrasedQuestionsPrompt`：

```python
instruction = """
You are a question rephrasing system.
Your task is to break down a comparative question into individual 
questions for each company mentioned.
Each output question must be:
- self-contained（完全独立，不依赖原始问题的上下文）
- maintain the same intent and metric（保留原始问题的度量指标）
- specific to the respective company（改写为针对单个公司的问题）
"""
```

**"self-contained" 这个约束至关重要**。如果直接对每家公司发"Which had higher revenue?"这个问题，LLM 会困惑——改写后的子问题必须是完全独立的，比如 "What was Apple's total revenue in fiscal year 2022?"。

### 5.4 第三步：比较汇总的 Prompt

`ComparativeAnswerPrompt` 的输入不是原始 RAG Context，而是**各公司独立问答的答案**：

```python
user_prompt = """
Here are the individual company answers:
\"\"\"
{context}     ← 这里是 {"Apple": {"final_answer": 394.3}, "Microsoft": {"final_answer": 198.3}}
\"\"\"

Here is the original comparative question:
"{question}"
"""
```

专门的比较规则：

```python
instruction = """
Important rules for comparison:
- When the question asks to choose one company, return the company name 
  EXACTLY as it appears in the original question
- If a company's metric is in a different currency than what is asked, 
  EXCLUDE that company from comparison
- If all companies are excluded, return 'N/A'
"""
```

**货币不匹配排除规则**防止了一类典型错误：题目问"哪家公司美元资产更多"，但某家公司的资产是加元，不能直接比较，应该排除该公司。

### 5.5 为什么子问题强制用 "number" 类型

```python
answer_dict = self.get_answer_for_company(
    company_name=company,
    question=sub_question,
    schema="number"    # 强制 number 类型，即使原题是 comparative
)
```

因为比较型问题的本质是比较数值（"哪家更高/更低/更多"），所以每家公司的子问题答案必须是数字，才能在第三步做数值比较。强制 `schema="number"` 确保每个子答案都是可比较的格式。

### 5.6 完整的比较题数据流

```
原始问题: "Which had higher revenue, 'Apple' or 'Microsoft'?"
    │
    ▼ Step 1: LLM 拆解
┌─────────────────────────────────┐
│ "Apple":    "Apple's revenue?"  │
│ "Microsoft": "Microsoft revenue?"│
└─────────────────────────────────┘
    │
    ▼ Step 2: 并行 RAG 问答（各自独立检索）
┌────────────────────┐  ┌─────────────────────────┐
│ Apple 年报检索      │  │ Microsoft 年报检索        │
│ → final_answer: 394│  │ → final_answer: 198      │
└────────────────────┘  └─────────────────────────┘
    │
    ▼ Step 3: LLM 汇总比较
{"Apple": 394, "Microsoft": 198}
    + "Which had higher revenue?"
    → final_answer: "Apple"
```

---

## 6. 多 LLM 提供商的统一适配

### 6.1 为什么需要多 Provider 适配

RAG 系统在选择回答 LLM 时往往需要灵活切换：
- **OpenAI GPT-4o / o3-mini**：精度高，支持原生 Structured Output，但成本较高
- **Gemini**（Google）：超长上下文窗口（100万 token），适合全文档塞进去直接回答
- **开源模型（如 Llama）**：成本低，可私有化部署，但不支持 Structured Output

本项目用**策略模式（Strategy Pattern）**封装了三种 Provider 的差异：

```python
class APIProcessor:
    def __init__(self, provider: Literal["openai", "ibm", "gemini"] = "openai"):
        if self.provider == "openai":
            self.processor = BaseOpenaiProcessor()
        elif self.provider == "ibm":
            self.processor = BaseIBMAPIProcessor()
        elif self.provider == "gemini":
            self.processor = BaseGeminiProcessor()
    
    def get_answer_from_rag_context(self, question, rag_context, schema, model):
        # 对调用方完全透明，不需要知道底层是哪个 Provider
        system_prompt, response_format, user_prompt = self._build_rag_context_prompts(schema)
        answer_dict = self.processor.send_message(...)
        return answer_dict
```

调用方（`QuestionsProcessor`）只需要传入 `provider="openai"` 或 `provider="gemini"`，其余逻辑完全一致：

```python
# questions_processing.py
self.openai_processor = APIProcessor(provider=api_provider)
# api_provider 来自 RunConfig，可以是 "openai" / "ibm" / "gemini"
```

### 6.2 三种 Provider 的关键差异

| 特性 | OpenAI | IBM (Llama) | Gemini |
|------|--------|-------------|--------|
| 原生 Structured Output | ✅ `beta.chat.completions.parse()` | ❌ 不支持 | ❌ 不支持 |
| Prompt 里需要附 Schema | ❌ 不需要 | ✅ 必须 | ✅ 必须 |
| 输出格式保证 | API 层强制 | 靠 Prompt 约束，可能失败 | 靠 Prompt 约束，可能失败 |
| 容错修复机制 | 不需要 | `json_repair` + 二次调用重解析 | `json_repair` + 二次调用重解析 |
| 上下文窗口 | 128K tokens | ~128K tokens | 1M tokens |
| temperature 支持 | ✅（o3-mini 除外） | ✅ | ✅ |

**o3-mini 的特殊处理**：o3-mini 是推理模型，不支持 `temperature` 参数。代码里专门做了判断：

```python
# api_requests.py
params = {
    "model": model,
    "messages": [...]
}
# 推理模型不支持 temperature，条件性添加
if "o3-mini" not in model:
    params["temperature"] = temperature
```

### 6.3 Gemini 的全文档模式

Gemini 的超大上下文窗口（100万 token）支持一种特殊的"非 RAG"模式：不做检索，直接把**整本年报的所有页面**塞进 Context：

```python
# questions_processing.py
if self.full_context:
    # 跳过检索，直接取所有页面
    retrieval_results = retriever.retrieve_all(company_name)
    # retrieve_all() 返回整本文档的所有页面，按页码排序
else:
    # 正常 RAG 检索流程
    retrieval_results = retriever.retrieve_by_company_name(...)
```

`retrieve_all()` 返回的是整本文档按页排序的全部内容，对应 `gemini_thinking_config` 里的 `full_context=True`。这实际上已经不是 RAG（检索增强生成）了，更接近"超长上下文理解"——但对于上下文窗口足够大的模型，这有时比 RAG 检索更准确。

---

## 7. 非 OpenAI 模型的结构化输出容错

### 7.1 问题背景

IBM、Gemini 等不支持原生 Structured Output。Prompt 里虽然附上了 JSON Schema，但 LLM 输出的内容可能是：

```
# 常见的"不符合格式"输出形式

# 形式1：有多余的 Markdown 代码块标记
```json
{"step_by_step_analysis": "...", "final_answer": "Robert E. Jordan"}
```

# 形式2：字段名拼写错误
{"step_by_step_analsis": "..."}   ← 少了一个 'y'

# 形式3：JSON 格式有语法错误
{"final_answer": "Robert E. Jordan",}  ← 多了尾逗号
```

这些都会导致 `json.loads()` 直接报错，整个问题处理失败。

### 7.2 两层容错机制

**第一层：`json_repair` 自动修复**

```python
from json_repair import repair_json

raw_response = llm_output  # 可能有各种格式问题
repaired = repair_json(raw_response)   # 自动修复常见 JSON 格式错误
parsed_dict = json.loads(repaired)
validated = ResponseSchema.model_validate(parsed_dict)  # Pydantic 验证字段
```

`json_repair` 能处理大多数常见问题：多余的 Markdown 标记、尾逗号、单引号替代双引号等。

**第二层：让 LLM 自己重新格式化**

如果 `json_repair` 也失败了，调用另一个 LLM（专门的 JSON 格式化助手）来重新解析：

```python
def _reparse_response(self, response, system_content):
    """让 LLM 把自己的输出重新格式化为合法 JSON"""
    user_prompt = AnswerSchemaFixPrompt.user_prompt.format(
        system_prompt=system_content,   # 原始 System Prompt，告诉它 Schema 是什么
        response=response               # LLM 原始输出
    )
    
    reparsed = self.send_message(
        system_content=AnswerSchemaFixPrompt.system_prompt,
        human_content=user_prompt,
        is_structured=False  # 不用结构化，只要干净的 JSON 字符串
    )
    return reparsed
```

对应的 `AnswerSchemaFixPrompt`：

```python
class AnswerSchemaFixPrompt:
    system_prompt = """
You are a JSON formatter.
Your task is to format raw LLM response into a valid JSON object.
Your answer should always start with '{' and end with '}'
Your answer should contain only json string, 
without any preambles, comments, or triple backticks.
"""
```

这个两层容错的效果：第一层处理格式错误（~90% 的情况），第二层用 LLM 兜底（剩余 ~9%），极少情况（~1%）才真正失败。

### 7.3 OpenAI vs 非 OpenAI 的 Prompt 差异

由于 IBM/Gemini 需要在 Prompt 里附上 Schema 才能约束输出，`build_system_prompt()` 函数提供了两个版本：

```python
def build_system_prompt(instruction, example, pydantic_schema=""):
    schema_block = (
        f"Your answer should be in JSON and strictly follow this schema:\n"
        f"```\n{pydantic_schema}\n```"
    )
    # 组装：instruction + schema（可选）+ example
    return instruction + schema_block + example

# 两个版本
system_prompt             = build_system_prompt(instruction, example)
                            # 没有 schema 块，适合 OpenAI（原生支持）
system_prompt_with_schema = build_system_prompt(instruction, example, pydantic_schema)  
                            # 附上 schema 块，适合 IBM/Gemini
```

`pydantic_schema` 是直接从 Python 源代码提取的类定义文字：

```python
# 用 inspect.getsource() 把 Pydantic 类的源码字符串化，直接嵌入 Prompt
pydantic_schema = re.sub(r"^ {4}", "", inspect.getsource(AnswerSchema), flags=re.MULTILINE)
```

这确保了 Prompt 里的 Schema 描述和实际 Pydantic 类定义始终保持同步——修改类定义时，Prompt 里的 Schema 自动同步，不需要手动维护两份定义。

---

## 8. 完整问答流程串联

将四篇串联起来，以一道 number 题为例，展示端到端的完整数据流：

```
用户问题：
"What is the operating margin for Tradition in 2022?"
kind = "number"

    │
    ▼ [第三篇] 检索（HybridRetriever）
召回 Top-28 候选 Chunk → LLM 重排 → Top-10 最相关 Chunk（或整页）

    │ 格式化为 RAG Context
    ▼
"Text retrieved from page 45:
 \"\"\"
 Operating margin: 9.9% (2022 reported)
 Adjusted operating margin: 11.4%
 ...
 \"\"\""

    │
    ▼ [本篇] 题型路由
schema = "number" → AnswerWithRAGContextNumberPrompt

    │
    ▼ LLM 调用（temperature=0，Structured Output）
{
  "step_by_step_analysis": 
    "1. 问题要求 Tradition 2022 年的 Operating margin
     2. 第45页有三种 margin：Reported(9.9%)、Adjusted(11.4%)、Adjusted underlying(12.7%)
     3. 题目未指定 Adjusted，应取 Reported operating margin
     4. Reported operating margin = 9.9%
     5. 无需任何计算，直接引用原文数值",
  
  "reasoning_summary": "第45页明确列出 2022 年 Reported operating margin 为 9.9%",
  
  "relevant_pages": [45, 44],
  
  "final_answer": 9.9
}

    │
    ▼ 页码验证（防幻觉）
claimed_pages = [45, 44]
retrieved_pages = [45, 44, 12, 8, ...]  ← 来自检索结果
validated_pages = [45, 44]              ← 都在检索结果里，通过验证

    │
    ▼ 最终输出
{
  "question_text": "What is the operating margin for Tradition in 2022?",
  "kind": "number",
  "value": 9.9,
  "references": [
    {"pdf_sha1": "2779336b845a41544348abb7b3e6e5bd2ff893a2", "page_index": 44},
    {"pdf_sha1": "2779336b845a41544348abb7b3e6e5bd2ff893a2", "page_index": 43}
  ]
}
```

### 快速上手：最小可运行的问答示例

```python
from src.api_requests import APIProcessor

processor = APIProcessor(provider="openai")

# 已经格式化好的 RAG Context（来自检索阶段）
rag_context = """Text retrieved from page 45:
\"\"\"
Operating Review
Reported operating margin: 9.9% (2022)
Adjusted operating margin: 11.4% (2022)
\"\"\"
"""

answer = processor.get_answer_from_rag_context(
    question="What is the operating margin for Tradition in 2022?",
    rag_context=rag_context,
    schema="number",
    model="gpt-4o-mini-2024-07-18"
)

print(answer["final_answer"])         # 9.9
print(answer["relevant_pages"])       # [45]
print(answer["step_by_step_analysis"]) # 完整推理过程...
```

---

## 9. 常见错误与踩坑指南

### 坑 1：number 题的单位换算是 LLM 的责任，不是程序的责任

财务数据的单位换算（千元 → 元，百万 → 元）应该在 Prompt 里明确指导 LLM 完成，而不是在代码里对 `final_answer` 做后处理。

原因：Context 里可能有多个数值，程序不知道哪个数值需要换算，但 LLM 读了上下文后知道。如果在程序里统一乘以 1000，会把那些已经是完整单位的数值也错误地放大。

```python
# ❌ 错误做法：程序后处理单位
if "thousands" in rag_context:
    final_answer *= 1000

# ✅ 正确做法：在 Field 描述里告诉 LLM 自己处理
final_answer: Union[float, int, Literal['N/A']] = Field(description="""
Pay attention to mentions of thousands/millions and adjust accordingly.
Value from context: 4970.5 (in thousands $) → Final answer: 4970500
""")
```

### 坑 2：比较型问题里子问题必须强制 "number" 类型

```python
# ❌ 错误：用原始的 schema（比如 "comparative"）
answer = get_answer_for_company(company, sub_question, schema=schema)

# ✅ 正确：子问题强制 number 类型
answer = get_answer_for_company(company, sub_question, schema="number")
```

如果子问题不强制 number 类型，可能返回自然语言描述，第三步 LLM 比较时无法做数值判断。

### 坑 3：Example 在 Prompt 里比 Instruction 更有效

当你想约束 LLM 的某种行为时，**一个具体的 Example 往往比几段文字描述更有效**。

本项目中 boolean 题防止"政策描述误判为事件"的方式，是在 Example 里展示了一个典型的反面案例并给出正确答案。这比在 Instruction 里写"注意区分政策描述和实际事件"效果好得多。

```python
# 对于容易出错的场景，在 Example 里专门展示反面案例
example = r"""
Question: "Did W. P. Carey Inc. announce dividend POLICY changes?"

Answer: {
    "step_by_step_analysis": "... dividend *amount* changed, but *policy* didn't ...",
    "final_answer": False   ← 关键：明确展示这种情况应该回答 False
}
"""
```

### 坑 4：temperature=0 对生产系统很重要

所有问答调用都用 `temperature=0`，这保证了：
- **可复现性**：相同输入 → 相同输出，方便调试
- **稳定性**：不会因为随机采样导致同一个问题有时答对有时答错
- **评估可靠性**：对比不同配置的效果时，排除了随机性因素

只有在需要多样化生成（如头脑风暴）时才考虑提高 temperature。

### 坑 5：不要在 Prompt 里用中文约束英文文档的回答

如果你的文档是英文，Prompt 用中文写，LLM 可能会把答案翻译成中文再输出——而财务数字、公司名称、人名经过翻译后往往就错了。

**规则**：Prompt 的语言应该与文档和预期输出的语言一致，或使用英文（通用性最强）。

---

## 总结

问答生成是 RAG 系统的最后一公里，也是最容易被忽视的环节。本篇的核心思路：

1. **按题型路由 Prompt**：不同问题类型需要截然不同的约束，一个通用 Prompt 无法兼顾所有场景
2. **Chain-of-Thought 先于 final_answer**：在 Schema 里把推理字段排在答案字段之前，让 LLM 先完成推理再给结论，系统性提升准确率
3. **Field 描述是 Prompt 的一部分**：Pydantic Field 的 `description` 参数直接影响 LLM 的输出，细节约束（如"不允许计算"、"括号代表负数"）写在这里比写在 Instruction 里更精准
4. **比较型问题拆解为三步**：问题分解 → 并行子查询 → 汇总比较，这是处理多实体对比的标准模式
5. **多 Provider 用策略模式封装**：调用方完全不感知 Provider 差异，切换模型只需改一个参数
6. **非 OpenAI 模型需要两层容错**：`json_repair` 自动修复 + LLM 二次重格式化，保证结构化输出的鲁棒性

---

> 本系列下一篇：[RAG实战（五）-工程化与高并发](#)，介绍如何让 RAG 系统处理大批量问题时依然稳定、高效、可维护。

*本文基于 RAG-Challenge-2 项目的工程实践总结，参考代码见 `src/api_requests.py`、`src/prompts.py`、`src/questions_processing.py`。*
