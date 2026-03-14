---
title: RAG实战（一）-文档解析与文本切块
date: 2026-03-15 10:00:00
tags:
  - RAG
  - LLM
  - AI
  - Python
  - 学习笔记
categories:
  - 技术文章
---

本文是 RAG 实战系列第一篇，介绍 RAG 流水线的第一个环节：如何把原始 PDF 文档解析成可检索的文本片段。很多人忽视这一步，但实际上文档解析和切块的质量直接决定了整个 RAG 系统的上限。

> 本系列基于开源项目 [IlyaRice/RAG-Challenge-2](https://github.com/IlyaRice/RAG-Challenge-2) 的工程实践总结。

---

<!-- more -->

## 目录

1. [整体流程概览](#1-整体流程概览)
2. [第一步：PDF 解析 —— 选对工具是成功的一半](#2-第一步pdf-解析--选对工具是成功的一半)
3. [补充：Docling 为什么比其他库解析得更好？](#3-补充docling-为什么比其他库解析得更好)
4. [第二步：元信息提取与保存](#4-第二步元信息提取与保存)
5. [第三步：文档结构化 —— 不只是提取文字](#5-第三步文档结构化--不只是提取文字)
6. [第四步：表格的专项处理](#6-第四步表格的专项处理)
7. [第五步：文本清洗 —— 容易被忽视的关键环节](#7-第五步文本清洗--容易被忽视的关键环节)
8. [第六步：文本切块策略](#8-第六步文本切块策略)
9. [第七步：切块后的元信息挂载](#9-第七步切块后的元信息挂载)
10. [常见错误汇总与避坑指南](#10-常见错误汇总与避坑指南)
11. [完整数据结构参考](#11-完整数据结构参考)

---

## 1. 整体流程概览

很多新手拿到一堆 PDF 文件后，第一反应是直接用 `PyPDF2` 提取文本，然后按固定字符数切块，最后灌进向量数据库。这种做法在 Demo 阶段勉强能跑通，但在生产环境中会带来一系列问题：**检索质量差、表格信息丢失、上下文断裂、元信息缺失**。

一个经过打磨的 RAG 文档处理流程应该是这样的：

```
原始 PDF
    │
    ▼
[步骤1] PDF 解析（Layout分析 + OCR + 表格识别）
    │  → 输出：结构化 JSON（文本块、表格、图片，含坐标和页码）
    │
    ▼
[步骤2] 表格专项序列化（用 LLM 将表格转为自包含的文本块）
    │  → 输出：每个表格行/列的独立语义描述
    │
    ▼
[步骤3] 文本清洗与格式化（修复 OCR 噪声、统一 Markdown 格式）
    │  → 输出：按页面组织的干净文本（含 Markdown 结构标记）
    │
    ▼
[步骤4] 文本切块（语义感知切块，按 Token 数控制大小）
    │  → 输出：带元信息的 Chunk 列表
    │
    ▼
[步骤5] 向量化 + 索引（向量数据库 + BM25 双路索引）
    │
    ▼
 可检索的知识库
```

下面我们逐步拆解每个环节。

---

## 2. 第一步：PDF 解析 —— 选对工具是成功的一半

### 2.1 为什么不能直接用 PyPDF2？

`PyPDF2`、`pdfplumber` 等基础工具只能做简单的文本提取，它们有一个致命缺陷：**不理解文档版式（Layout）**。对于包含多列排版、嵌套表格、页眉页脚的商业文档（如年报、合同），这类工具提取的文本顺序往往是错乱的。

**典型问题场景：**

```
原始 PDF 双列布局：
┌──────────────┬──────────────┐
│ 第一列第1段  │ 第二列第1段  │
│ 第一列第2段  │ 第二列第2段  │
└──────────────┴──────────────┘

PyPDF2 提取结果（顺序混乱）：
"第一列第1段 第二列第1段 第一列第2段 第二列第2段"

正确的提取结果应该是：
"第一列第1段 第一列第2段" 和 "第二列第1段 第二列第2段"（独立段落）
```

### 2.2 推荐工具：Docling

对于企业级文档解析，推荐使用 [Docling](https://github.com/DS4SD/docling)。它基于深度学习模型，能够：

- 理解文档版式，正确区分多列、标题、正文、页眉页脚
- 识别表格结构（行列关系）
- 处理扫描件（OCR）
- 输出结构化的文档模型

**基础使用示例：**

```python
from docling.document_converter import DocumentConverter, FormatOption
from docling.datamodel.pipeline_options import PdfPipelineOptions, TableFormerMode, EasyOcrOptions
from docling.datamodel.base_models import InputFormat
from docling.pipeline.standard_pdf_pipeline import StandardPdfPipeline
from docling.backend.docling_parse_v2_backend import DoclingParseV2DocumentBackend

# 配置解析选项
pipeline_options = PdfPipelineOptions()
pipeline_options.do_ocr = True                                    # 开启OCR（扫描件必须）
pipeline_options.ocr_options = EasyOcrOptions(
    lang=['en'],
    force_full_page_ocr=False                                     # 仅对无文字区域做OCR，节省时间
)
pipeline_options.do_table_structure = True                        # 开启表格结构识别
pipeline_options.table_structure_options.do_cell_matching = True  # 精确匹配单元格
pipeline_options.table_structure_options.mode = TableFormerMode.ACCURATE  # 精确模式

format_options = {
    InputFormat.PDF: FormatOption(
        pipeline_cls=StandardPdfPipeline,
        pipeline_options=pipeline_options,
        backend=DoclingParseV2DocumentBackend
    )
}

converter = DocumentConverter(format_options=format_options)
result = converter.convert("your_document.pdf")
```

> **坑1**：`force_full_page_ocr=False` 是合理的默认值。如果对非扫描件开启全页OCR，处理速度会慢 5-10 倍，且结果不一定更好。只有当你确认文档是扫描件时才开启。

> **坑2**：`TableFormerMode.ACCURATE` 比 `FAST` 模式慢，但对复杂表格（合并单元格、多级表头）的识别准确率显著更高。如果你的文档包含财务报表等复杂表格，建议用 `ACCURATE`。

### 2.3 解析结果的数据结构

Docling 解析后，通过 `export_to_dict()` 可以得到一个结构化字典，包含：

```python
{
  "origin": {"filename": "report.pdf"},
  "pages": [...],          # 页面尺寸信息
  "texts": [...],          # 所有文本块，含label（标题/正文/页眉等）
  "tables": [...],         # 表格数据（含行列结构）
  "pictures": [...],       # 图片区域
  "body": {                # 文档逻辑树（体现层级关系）
    "children": [...]
  }
}
```

每个文本块（`texts` 中的元素）都包含非常有价值的信息：

```python
{
  "text": "Revenue increased by 15% year-over-year",
  "label": "text",          # 类型：text/section_header/page_header/footnote/list_item等
  "prov": [{
    "page_no": 3,           # 所在页码
    "bbox": {               # 边界框坐标
      "l": 72.0, "t": 150.0, "r": 540.0, "b": 165.0
    }
  }]
}
```

**这些信息非常重要，后续处理都依赖它们，一定要在处理过程中保留。**

---

## 3. 补充：Docling 为什么比其他库解析得更好？

这一节专门讲 Docling 的内部工作原理，帮助你理解它和传统工具在本质上的差异，而不只是知道"用 Docling 效果更好"。

### 3.1 传统工具的工作方式：字符流提取

`PyPDF2`、`pdfminer`、`pdfplumber` 等传统工具的工作原理本质上都是**从 PDF 的底层字节流中读取字符和坐标**。

PDF 文件格式在内部存储的并不是"段落"或"表格"，而是一条条绘图指令，类似：

```
# PDF 内部指令（简化示意）
BT                          % 开始文本块
/F1 12 Tf                   % 设置字体
72 720 Td                   % 移动到坐标 (72, 720)
(Revenue grew by) Tj        % 绘制文字
150 720 Td                  % 移动到坐标 (150, 720)
(15%) Tj                    % 绘制文字
72 700 Td                   % 移动到下一行坐标
(in fiscal year 2023.) Tj   % 继续绘制
ET                          % 结束文本块
```

传统工具做的事情是：**把这些字符按坐标排序，然后拼成字符串**。它们并不知道哪些字符属于"标题"，哪些属于"表格的第二列第三行"，哪些是"页眉"。

这就是双列布局会出错的根本原因：两列的 Y 坐标交错，按坐标排序后左右列的内容就混在一起了。

```
坐标示意（Y值越大越靠上）：
(72, 720): "左列第一行"
(320, 720): "右列第一行"     ← 同一Y高度
(72, 700): "左列第二行"
(320, 700): "右列第二行"     ← 同一Y高度

按Y排序后：左列第一行 右列第一行 左列第二行 右列第二行  ← 顺序混乱
```

### 3.2 Docling 的工作方式：模型驱动的版式理解

Docling 采用完全不同的思路，它的处理流水线分为多个阶段，每个阶段都使用专门的模型：

```
PDF 原始数据
    │
    ▼
[阶段1] PDF 后端解析（DoclingParseV2）
    │  提取原始字符、字体、坐标、图片等底层元素
    │  → 输出：原始元素列表（字符、图形、图片区域）
    │
    ▼
[阶段2] 版式分析模型（Layout Model）
    │  对每一页截图，用目标检测模型识别版式区域
    │  → 输出：带标签的区域列表（标题/段落/表格/图片/页眉/页脚...）
    │         每个区域有精确的边界框（bounding box）
    │
    ▼
[阶段3] 内容分配（Cell Matching）
    │  将阶段1的原始字符，分配到阶段2识别出的各个区域中
    │  → 输出：每个区域内的有序文字内容
    │
    ▼
[阶段4] 表格结构识别（TableFormer 模型）
    │  针对被识别为"表格"的区域，专门识别行列结构
    │  → 输出：结构化的表格（知道哪个单元格属于第几行第几列）
    │
    ▼
[阶段5] OCR（可选，EasyOCR / Tesseract）
    │  对无文字的图片区域（扫描件）运行 OCR
    │  → 输出：扫描内容的文字
    │
    ▼
[阶段6] 文档模型组装
       将上述结果组合为层级化的文档模型（DoclingDocument）
       → 输出：带语义标签的结构化文档
```

**关键差异在阶段2**：Docling 会对每一页生成一张截图，然后用类似目标检测（Object Detection）的深度学习模型来识别版式区域。这个模型是在大量学术论文和商业文档上训练过的，它"看到"的是整张页面的视觉布局，而不是一个个孤立的字符。

### 3.3 各类文档元素的识别能力对比

| 文档元素 | PyPDF2/pdfplumber | Docling |
|---------|------------------|---------|
| 单列正文文本 | ✅ 基本正确 | ✅ 正确 |
| 双列/多列布局 | ❌ 顺序混乱 | ✅ 正确分列 |
| 章节标题识别 | ❌ 无法区分（仅字体大小） | ✅ 模型识别，输出 `section_header` 标签 |
| 页眉页脚识别 | ❌ 混入正文 | ✅ 识别为 `page_header`/`page_footer` 并可单独过滤 |
| 表格结构（行列） | ❌ 表格变成乱序文本 | ✅ TableFormer 识别完整行列结构 |
| 合并单元格（rowspan/colspan） | ❌ 完全丢失 | ✅ 保留并输出 HTML |
| 多级表头 | ❌ 完全丢失 | ✅ 保留层级关系 |
| 脚注识别 | ❌ 混入正文 | ✅ 识别为 `footnote` 标签 |
| 列表项识别 | ❌ 无法区分 | ✅ 识别为 `list_item` 标签 |
| 图片区域识别 | ❌ 无法识别 | ✅ 识别区域和边界框 |
| 扫描件 OCR | ❌ 不支持 | ✅ 集成 EasyOCR/Tesseract |
| 公式识别 | ❌ 不支持 | ✅ 部分支持，输出 `equation` 标签 |

### 3.4 `label` 字段：Docling 最值得利用的输出

Docling 给每个文本块打上语义标签（`label` 字段），这是它相比传统工具最大的优势之一。可用的标签包括：

```python
# Docling 文本块的 label 类型
"section_header"     # 章节标题（##）
"page_header"        # 页眉（通常要过滤掉）
"page_footer"        # 页脚（通常要过滤掉）
"text"               # 普通正文
"paragraph"          # 段落（有时与 text 混用）
"list_item"          # 列表项（- ）
"footnote"           # 脚注
"caption"            # 图表标题
"formula"            # 数学公式
"checkbox_selected"  # 选中的复选框 [x]
"checkbox_unselected"# 未选中的复选框 [ ]
# 非 texts 数组中的元素：
# "table"            # 表格（在 tables 数组中）
# "picture"          # 图片（在 pictures 数组中）
```

这些标签让后处理变得极为精确。例如，过滤页眉页脚只需一行代码：

```python
ignored_types = {"page_footer", "picture"}
filtered_blocks = [b for b in blocks if b.get("type") not in ignored_types]
```

而用 PyPDF2，你要通过坐标位置猜测哪些是页眉（"Y 坐标大于页面高度的 90%？也许是页眉？"），极不可靠。

### 3.5 表格识别：TableFormer 模型

Docling 的表格识别使用了 **TableFormer** 模型（IBM Research 开发），这是专门针对文档表格设计的 Transformer 架构。

它解决的核心问题是：PDF 里的表格在底层只是一堆坐标随机排列的文字，没有任何"行"和"列"的概念。TableFormer 通过理解视觉上的对齐关系来推断表格结构：

```
PDF 底层存储的表格（只有坐标和文字）：
(100, 500): "Metric"   (200, 500): "Q1"    (300, 500): "Q2"
(100, 480): "Revenue"  (200, 480): "4521"  (300, 480): "4890"
(100, 460): "Net Inc"  (200, 460): "412"   (300, 460): "456"

TableFormer 推断出的结构：
行0（表头）: ["Metric", "Q1", "Q2"]
行1:         ["Revenue", "4521", "4890"]
行2:         ["Net Inc", "412", "456"]
```

更重要的是，它能处理**合并单元格**，这是传统工具完全无法做到的：

```
带合并单元格的表头：
┌─────────┬────────────────────────┐
│         │   Financial Results    │  ← 这个单元格跨越2列（colspan=2）
│ Metric  ├────────────┬───────────┤
│         │     Q1     │    Q2     │
├─────────┼────────────┼───────────┤
│ Revenue │   4,521    │   4,890   │

PyPDF2 提取结果：混乱的字符串，完全失去表头层级
TableFormer 输出：正确的 rowspan/colspan HTML，语义完整
```

`do_cell_matching=True` 这个配置项的作用是：让 TableFormer 推断出的行列结构与底层的字符坐标进行精确匹配，确保每个单元格内的文字内容正确对应到正确的格子里，而不是靠坐标猜测。

### 3.6 两种 PDF 后端的差异

| 后端 | 说明 | 适用场景 |
|------|------|---------|
| `DoclingParseV2DocumentBackend`（推荐） | Docling 自研的高精度解析器，能正确处理复杂字体编码和特殊字符 | 大多数商业 PDF，尤其含特殊字符/字体时 |
| `DoclingParseDocumentBackend`（v1） | 老版本，精度略低 | 兼容性回退 |
| `PyPdfiumDocumentBackend` | 基于 pdfium 库，速度快但精度一般 | 对速度要求高且 PDF 格式简单时 |

**为什么 DoclingParseV2 更好**：它在处理字体映射时更精确，这直接影响特殊字符（如货币符号 `€`、数学符号 `±`）和非英文字符的提取质量。用 PyPdfium 后端时，这类字符可能被替换为乱码或 `?`，而 DoclingParseV2 能正确解析字体编码表并还原原始字符。

### 3.7 性能取舍：什么时候可以降级？

Docling 的完整流水线比 PyPDF2 慢得多（主要消耗在版式模型推理上），大概慢 10-30 倍。如果处理量很大，有几个可调节的旋钮：

```python
# 最快配置（牺牲表格识别精度）
pipeline_options.table_structure_options.mode = TableFormerMode.FAST  # 快速模式

# 如果文档是原生 PDF（非扫描件），关闭 OCR 可大幅提速
pipeline_options.do_ocr = False

# 如果不需要识别表格结构（已知没有重要表格），关闭它
pipeline_options.do_table_structure = False

# 多线程处理（见 parse_and_export_parallel）
parser.parse_and_export_parallel(
    input_doc_paths=pdf_paths,
    optimal_workers=4,    # 根据CPU核数调整
    chunk_size=5          # 每个进程处理5个文件
)
```

**决策建议**：
- 文档含财务表格、对表格精度要求高 → 用完整配置（`ACCURATE` 模式）
- 文档以叙述性文字为主、表格不重要 → 可关闭 `do_table_structure`，速度提升明显
- 文档是扫描件 → OCR 必须开启，不可妥协
- 原生数字 PDF → 关闭 OCR，速度提升显著

---

## 4. 第二步：元信息提取与保存

### 4.1 什么是元信息，为什么重要？

元信息（Metadata）是关于文档本身的描述性信息，例如：

- 文档来源（公司名称、文档类型）
- 文档统计（总页数、表格数量、图片数量）
- 每个 Chunk 所在的页码

**元信息的核心作用：**

1. **过滤检索范围**：用户问"A公司2023年的营收"时，可以先通过元信息过滤只在A公司的文档里检索
2. **溯源**：检索到的内容能追溯到原文的具体页码，方便人工验证
3. **调试**：出现问题时快速定位是哪个文档哪一页的数据

### 4.2 文档级元信息的设计

```python
def assemble_metainfo(self, data):
    metainfo = {}
    
    # 文档唯一标识（建议用文件SHA1哈希，避免文件名冲突）
    sha1_name = data['origin']['filename'].rsplit('.', 1)[0]
    metainfo['sha1_name'] = sha1_name
    
    # 文档统计信息（用于后续质量检查）
    metainfo['pages_amount'] = len(data.get('pages', []))
    metainfo['text_blocks_amount'] = len(data.get('texts', []))
    metainfo['tables_amount'] = len(data.get('tables', []))
    metainfo['pictures_amount'] = len(data.get('pictures', []))
    metainfo['equations_amount'] = len(data.get('equations', []))
    metainfo['footnotes_amount'] = len(
        [t for t in data.get('texts', []) if t.get('label') == 'footnote']
    )
    
    # 来自外部CSV的业务元信息（如公司名称）
    if sha1_name in self.metadata_lookup:
        metainfo['company_name'] = self.metadata_lookup[sha1_name]['company_name']
    
    return metainfo
```

> **坑3**：不要用文件名作为文档唯一标识。在批量处理时，不同来源的文档可能有相同文件名。用文件内容的 SHA1 哈希作为标识符，既能去重，又能稳定索引。

### 4.3 外部元信息的关联

实际项目中，文档的业务信息（如所属公司、年份、分类）往往不在 PDF 里，而在一个外部 CSV 或数据库中。常见做法是用文档的哈希值作为 key 关联：

```python
@staticmethod
def _parse_csv_metadata(csv_path: Path) -> dict:
    """从CSV加载元信息，以sha1为key建立查找表"""
    import csv
    metadata_lookup = {}
    with open(csv_path, 'r', encoding='utf-8') as csvfile:
        reader = csv.DictReader(csvfile)
        for row in reader:
            company_name = row.get('company_name', row.get('name', '')).strip('"')
            metadata_lookup[row['sha1']] = {
                'company_name': company_name
            }
    return metadata_lookup
```

---

## 5. 第三步：文档结构化 —— 不只是提取文字

### 5.1 按页面组织内容

Docling 的输出是一个逻辑文档树（`body.children`），不是按页面组织的。我们需要将其转换为按页面组织的结构，这对后续的切块和检索非常有用：

```python
# 处理后的内容结构
{
  "content": [
    {
      "page": 1,
      "content": [
        {"type": "page_header", "text": "Annual Report 2023"},
        {"type": "section_header", "text": "Financial Highlights"},
        {"type": "text", "text": "Revenue grew by 15%..."},
        {"type": "table", "table_id": 0},
        {"type": "text", "text": "Note: All figures in USD millions"}
      ],
      "page_dimensions": {"l": 0, "t": 842, "r": 595, "b": 0}
    },
    // ...更多页面
  ]
}
```

### 5.2 处理页面序号缺口

**这是一个非常容易踩到的坑**。Docling 在某些情况下会跳过没有文本内容的页面（如全图页面），导致页面序号不连续（1, 2, 4, 5...）。如果后续按索引访问页面，会产生越界或错位问题。

正确做法是填充空页面，保持序号连续：

```python
def _normalize_page_sequence(self, data: dict) -> dict:
    """填充缺失页面，确保页码序列连续"""
    existing_pages = {page['page'] for page in data['content']}
    max_page = max(existing_pages)
    
    new_content = []
    for page_num in range(1, max_page + 1):
        page_content = next(
            (page for page in data['content'] if page['page'] == page_num),
            {"page": page_num, "content": [], "page_dimensions": {}}  # 空页面占位
        )
        new_content.append(page_content)
    
    data['content'] = new_content
    return data
```

### 5.3 处理文档逻辑树中的 Group 引用

Docling 的 `body.children` 中有时会出现 `$ref` 引用（指向 `groups`、`texts`、`tables` 等），需要递归展开。初学者容易忽略 `groups` 层级的处理：

```python
def expand_groups(self, body_children, groups):
    """展开body中的group引用，将group的子项直接平铺到父级"""
    expanded_children = []
    for item in body_children:
        if isinstance(item, dict) and '$ref' in item:
            ref = item['$ref']
            ref_type, ref_num = ref.split('/')[-2:]
            ref_num = int(ref_num)
            
            if ref_type == 'groups':
                # Group 不是内容，需要展开其子项
                group = groups[ref_num]
                for child in group['children']:
                    child_copy = child.copy()
                    # 保留 group 信息，便于后续判断内容归属
                    child_copy['group_id'] = ref_num
                    child_copy['group_name'] = group.get('name', '')
                    child_copy['group_label'] = group.get('label', '')
                    expanded_children.append(child_copy)
            else:
                expanded_children.append(item)
        else:
            expanded_children.append(item)
    return expanded_children
```

---

## 6. 第四步：表格的专项处理

### 6.1 表格是 RAG 的噩梦

表格是 RAG 中最难处理的内容类型。直接把 Markdown 表格灌进向量库，检索时有两个问题：

1. **语义稀疏**：向量模型不擅长理解表格的行列关系，"Q3 Revenue: $5.2B" 和 "Revenue in the third quarter was 5.2 billion dollars" 语义相似度可能很低
2. **上下文缺失**：一行数据脱离表头就失去了意义，比如 "5,234" 单独看毫无意义，需要知道"这是营收数据，单位是百万美元，Q3 2023"

### 6.2 表格序列化：用 LLM 转化为自包含文本块

解决方案是用 LLM 将每一行（或逻辑相关的几行）转化为一段自包含的自然语言描述：

**原始 Markdown 表格：**

```markdown
| Metric | Q1 2023 | Q2 2023 | Q3 2023 |
|--------|---------|---------|---------|
| Revenue (USD M) | 4,521 | 4,890 | 5,234 |
| Net Income (USD M) | 412 | 456 | 501 |
```

**序列化后的文本块（每行一个）：**

```
Revenue for the company:
- Q1 2023: USD 4,521 million
- Q2 2023: USD 4,890 million  
- Q3 2023: USD 5,234 million
(Source: Quarterly Financial Summary table, amounts in USD millions)

Net Income for the company:
- Q1 2023: USD 412 million
- Q2 2023: USD 456 million
- Q3 2023: USD 501 million
(Source: Quarterly Financial Summary table, amounts in USD millions)
```

### 6.3 关键：利用表格上下文

表格旁边的文字（标题、注释、脚注）对理解表格至关重要。在调用 LLM 序列化时，要把这些上下文一并传入：

```python
def _get_table_context(self, json_report, target_table_index):
    """提取目标表格在页面中的上下文文字"""
    table_info = next(t for t in json_report["tables"] if t["table_id"] == target_table_index)
    page_num = table_info["page"]
    page_content = next(
        (page["content"] for page in json_report["content"] if page["page"] == page_num), []
    )
    
    # 找到目标表格在页面中的位置
    current_pos = next(
        i for i, block in enumerate(page_content)
        if block["type"] == "table" and block.get("table_id") == target_table_index
    )
    
    # 获取相邻表格的位置，限制上下文范围
    prev_table_pos = next(
        (i for i in range(current_pos-1, -1, -1) if page_content[i]["type"] == "table"),
        -1
    )
    next_table_pos = next(
        (i for i in range(current_pos+1, len(page_content)) if page_content[i]["type"] == "table"),
        -1
    )
    
    # 提取上下文文字
    start = prev_table_pos + 1 if prev_table_pos != -1 else 0
    context_before = "\n".join(
        b.get("text", "") for b in page_content[start:current_pos] if "text" in b
    )
    
    end = min(current_pos + 4, next_table_pos) if next_table_pos != -1 else current_pos + 4
    context_after = "\n".join(
        b.get("text", "") for b in page_content[current_pos+1:end] if "text" in b
    )
    
    return context_before, context_after
```

**LLM Prompt 设计要点：**

```python
system_prompt = """你是一个表格序列化代理。
你的任务是基于提供的表格和周围文字，创建一组上下文无关的信息块。
这些信息块必须完全独立，因为它们将作为独立的 Chunk 存入数据库。
每个 Chunk 必须包含：
1. 行头信息（核心主体）
2. 所有列头信息（维度说明）
3. 单位、货币等说明
4. 表格名称或来源说明
跳过任何有价值的信息将受到严重惩罚！"""

user_prompt = f"""
表格前的上下文：
"{context_before}"

HTML 格式的表格：
"{table_html}"

表格后的上下文：
"{context_after}"
"""
```

> **坑4**：不要只传 Markdown 格式的表格给 LLM。**HTML 格式**保留了合并单元格信息（`rowspan`/`colspan`），对于理解复杂的多级表头非常关键。Markdown 格式在表示合并单元格时会信息丢失。

### 6.4 序列化结果的存储策略

序列化后的表格信息有两种使用方式：

**方式一（推荐）：Markdown + 序列化并行存储**

```python
# 在切块时，普通文本页面仍使用 Markdown 表格
# 同时将序列化结果作为独立的 Chunk 追加
for page in file_content['content']['pages']:
    page_chunks = split_page(page)  # 包含 Markdown 表格的普通切块
    chunks.extend(page_chunks)
    
    # 追加序列化表格 Chunk（独立类型，便于区分处理）
    if page['page'] in tables_by_page:
        for table_chunk in tables_by_page[page['page']]:
            table_chunk['type'] = 'serialized_table'
            chunks.append(table_chunk)
```

**方式二：替换 Markdown 表格**

直接用序列化文本替换原始 Markdown 表格，更简洁，但丢失了原始表格格式。

---

## 7. 第五步：文本清洗 —— 容易被忽视的关键环节

### 7.1 OCR 噪声问题

对于扫描 PDF，OCR 会引入各种噪声。一个常见但极其隐蔽的问题是 **PDF 字体命令残留**。某些 PDF 文件在提取时会保留字体排版指令，例如：

```
Revenue was /five/period/two/three billion dollars in Q3
```

实际应该是：

```
Revenue was 5.23 billion dollars in Q3
```

必须用正则清洗这类噪声：

```python
command_mapping = {
    'zero': '0', 'one': '1', 'two': '2', 'three': '3', 'four': '4',
    'five': '5', 'six': '6', 'seven': '7', 'eight': '8', 'nine': '9',
    'period': '.', 'comma': ',', 'colon': ':', 'hyphen': '-',
    'percent': '%', 'dollar': '$', 'slash': '/', 'space': ' ',
    'plus': '+', 'minus': '-', 'asterisk': '*',
    'lparen': '(', 'rparen': ')', 'parenright': ')', 'parenleft': '(',
}

recognized_commands = "|".join(command_mapping.keys())
slash_command_pattern = rf"/({recognized_commands})(\.pl\.tnum|\.tnum\.pl|\.pl|\.tnum|\.case|\.sups)?"

text = re.sub(slash_command_pattern, lambda m: command_mapping.get(m.group(1), m.group(0)), text)
text = re.sub(r'glyph<[^>]*>', '', text)        # 清理未识别字形
text = re.sub(r'/([A-Z])\.cap', r'\1', text)    # 修复大写字母编码
```

### 7.2 用 Markdown 结构化文本，提升切块质量

将文档的视觉结构映射到 Markdown 语义，可以显著提升后续切块的效果：

```python
def _apply_formatting_rules(self, blocks):
    """将结构化块转为 Markdown 格式的文本"""
    final_blocks = []
    
    for i, block in enumerate(blocks):
        block_type = block.get("type")
        text = block.get("text", "").strip()
        
        # 忽略页脚和图片
        if block_type in {"page_footer", "picture"}:
            continue
        
        # 页面主标题 → H1
        if block_type == "page_header":
            prefix = "\n# " if i < 3 else "\n## "
            final_blocks.append(f"{prefix}{text}\n")
        
        # 章节标题 → H2
        elif block_type == "section_header":
            final_blocks.append(f"\n## {text}\n")
        
        # 段落标题 → H3（尾部有冒号的段落，视为小标题）
        elif block_type == "paragraph":
            final_blocks.append(f"\n### {text}\n")
        
        # 列表项 → Markdown 列表
        elif block_type == "list_item":
            final_blocks.append(f"- {text}\n")
        
        # 表格：插入 Markdown 表格
        elif block_type == "table":
            table_md = self._get_table_markdown(block["table_id"])
            final_blocks.append(f"\n{table_md}\n")
        
        # 脚注和普通文本
        elif block_type in {"text", "footnote", "caption"}:
            if text:
                final_blocks.append(f"{text}\n")
    
    return final_blocks
```

**为什么要转成 Markdown？**

RecursiveCharacterTextSplitter 等工具会优先在 Markdown 分隔符（`#`、`\n\n` 等）处切割，避免在句子中间切断。章节标题 `## Revenue Overview` 会成为天然的切割点，确保每个 Chunk 以完整的语义单元为边界。

### 7.3 表格与脚注的关联处理

**这是一个非常重要但常被忽视的细节**。表格下方的脚注（如"* 金额以百万美元为单位"）往往对理解表格数据至关重要。要将表格及其相关脚注作为一个整体处理，不能被切断：

```python
# 识别"表格组"：表头文字 + 表格 + 相关脚注
if block_type == "table":
    group_blocks = []
    
    # 如果上一个块以冒号结尾，说明它是这个表格的标题
    if prev_block_ends_with_colon:
        group_blocks.append(prev_block)
    group_blocks.append(table_block)
    
    # 收集紧跟在表格后面的脚注
    while next_block.type == "footnote":
        group_blocks.append(next_block)
        advance()
    
    # 将整个表格组渲染为一个不可切割的单元
    group_text = render_table_group(group_blocks)
```

---

## 8. 第六步：文本切块策略

### 8.1 切块的核心矛盾

切块大小是一个权衡：
- **太小**（< 100 tokens）：单个 Chunk 信息不完整，缺乏上下文，检索到了也没用
- **太大**（> 1000 tokens）：向量语义被稀释，精确信息被淹没，检索精度下降

**实践经验**：对于企业文档（年报、合同、技术文档），**200-400 tokens** 的 Chunk 大小是较好的平衡点，重叠（overlap）设为 Chunk 大小的 15%-20%。

### 8.2 使用 Token 计数而非字符数

**不要用字符数控制切块大小。** 中文、英文、代码的字符密度差异巨大，1000 个字符可能是 500 tokens（英文），也可能是 300 tokens（中文）。应该直接按 Token 计数：

```python
import tiktoken
from langchain.text_splitter import RecursiveCharacterTextSplitter

def count_tokens(text: str, encoding_name="o200k_base") -> int:
    """精确计算 Token 数（使用与目标模型一致的分词器）"""
    encoding = tiktoken.get_encoding(encoding_name)
    return len(encoding.encode(text))

# 使用 from_tiktoken_encoder 确保按 Token 切割
text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    model_name="gpt-4o",    # 使用与你的 Embedding/LLM 一致的分词器
    chunk_size=300,          # Token 数
    chunk_overlap=50         # 重叠 Token 数
)
```

> **坑5**：编码器的选择要与你使用的 Embedding 模型保持一致。OpenAI 的 `text-embedding-3-large` 使用 `cl100k_base` 分词器，`gpt-4o` 使用 `o200k_base`。两者对同一段文本的 Token 计数可能有差异。

### 8.3 RecursiveCharacterTextSplitter 的切割优先级

这是 LangChain 中最常用的切块工具，它按以下优先级寻找切割点（从高到低）：

```
1. \n\n    （段落间空行 —— 对应 Markdown 块间距）
2. \n      （换行）
3. 。/. /!/?  （句子结束标点）
4. 空格
5. 字符（最后手段）
```

这就是为什么我们要把文档格式化为 Markdown —— Markdown 的结构恰好与这些切割优先级对齐，能最大概率在"好的位置"切割。

### 8.4 按页面切块 vs 全文切块

**强烈推荐按页面切块**，而不是将整个文档拼接后再切块。原因：

1. **页码元信息**：每个 Chunk 可以知道自己来自第几页，便于溯源
2. **避免跨页混淆**：两个相邻页面的内容可能没有任何逻辑关联，强行重叠会产生语义噪声
3. **性能**：更小的处理单元，并行更容易

```python
def split_all_pages(file_content: dict, chunk_size: int = 300, chunk_overlap: int = 50):
    """按页面切块，每个Chunk携带页码信息"""
    all_chunks = []
    chunk_id = 0
    
    for page in file_content['content']['pages']:
        text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
            model_name="gpt-4o",
            chunk_size=chunk_size,
            chunk_overlap=chunk_overlap
        )
        chunks = text_splitter.split_text(page['text'])
        
        for chunk_text in chunks:
            all_chunks.append({
                "id": chunk_id,
                "page": page['page'],              # 页码元信息
                "text": chunk_text,
                "length_tokens": count_tokens(chunk_text),
                "type": "content"
            })
            chunk_id += 1
    
    return all_chunks
```

---

## 9. 第七步：切块后的元信息挂载

### 9.1 每个 Chunk 应携带的信息

一个完整的 Chunk 对象应该包含以下字段：

```python
{
    # 必须字段
    "id": 42,                              # 在本文档内的唯一编号
    "text": "Revenue grew by 15%...",      # 实际文本内容
    "page": 3,                             # 来源页码
    "type": "content",                     # Chunk类型（content/serialized_table）
    
    # 质量监控字段
    "length_tokens": 287,                  # Token 数量（用于调试和质量分析）
    
    # 可选的文档级信息（在检索时从metainfo注入）
    # "sha1_name": "abc123...",
    # "company_name": "Example Corp",
}
```

### 9.2 文档级元信息 vs Chunk 级元信息的存储策略

有两种方案：

**方案A：扁平化（每个Chunk都存文档元信息）**

```python
# 每个 Chunk 包含所有元信息
chunk = {
    "id": 42,
    "text": "...",
    "page": 3,
    "sha1_name": "abc123",
    "company_name": "Example Corp",
    "document_pages": 48,
    ...
}
```

优点：检索后无需额外查询；缺点：存储冗余。

**方案B：分层存储（推荐）**

```python
# 文档级别
document = {
    "metainfo": {
        "sha1_name": "abc123",
        "company_name": "Example Corp",
        "pages_amount": 48,
        ...
    },
    "content": {
        "chunks": [
            {"id": 42, "page": 3, "text": "...", "type": "content"},
            ...
        ]
    }
}
```

检索时，通过 `sha1_name` 关联文档的 `metainfo`。存储更紧凑，且文档元信息修改不需要更新所有 Chunk。

---

## 10. 常见错误汇总与避坑指南

### 错误清单

| # | 错误 | 后果 | 正确做法 |
|---|------|------|----------|
| 1 | 用 PyPDF2 处理多列/复杂PDF | 文本顺序混乱，检索内容错误 | 使用 Docling 等支持 Layout 分析的工具 |
| 2 | 不处理 OCR 噪声 | 数字和特殊字符显示为乱码 | 用正则清洗字体命令残留 |
| 3 | 直接将 Markdown 表格作为 Chunk | 表格行脱离上下文无意义 | 用 LLM 序列化表格为自包含文本块 |
| 4 | 按字符数切块 | 中英文切块大小不一致 | 用 tiktoken 按 Token 数切块 |
| 5 | 全文拼接后再切块 | 丢失页码信息，跨页内容混淆 | 按页面独立切块 |
| 6 | 不保留页码信息 | 无法溯源，用户无法验证 | 每个 Chunk 必须携带页码 |
| 7 | 不处理页码缺口 | 页面数组访问越界 | 用空页面填充确保序号连续 |
| 8 | 不关联表格的脚注 | 表格数据单位/货币信息丢失 | 将表格+脚注作为整体处理 |
| 9 | 用文件名作为文档唯一标识 | 文件名冲突导致数据错乱 | 用文件内容的 SHA1 哈希 |
| 10 | 切块大小太小（< 100 tokens） | 单个 Chunk 信息不完整 | 200-400 tokens 是合理范围 |
| 11 | 切块大小太大（> 800 tokens） | 向量语义稀释，检索精度下降 | 根据实际文档类型调整 |
| 12 | 不保存表格的 HTML 格式 | 合并单元格信息丢失，LLM 无法正确序列化 | 同时保存 Markdown、HTML、JSON 三种格式 |

### 调试技巧

1. **质量检查**：统计各类型 Chunk 的 token 数分布，发现异常大或异常小的 Chunk

```python
import numpy as np

token_lengths = [chunk['length_tokens'] for chunk in chunks]
print(f"平均: {np.mean(token_lengths):.1f}")
print(f"中位数: {np.median(token_lengths):.1f}")
print(f"最大: {np.max(token_lengths)}")
print(f"最小: {np.min(token_lengths)}")
print(f"超过500 tokens的Chunk比例: {sum(1 for t in token_lengths if t > 500) / len(token_lengths):.1%}")
```

2. **可视化检查**：将处理后的文档导出为 Markdown 文件，人工抽查几篇看格式是否正确

3. **OCR 质量检查**：统计噪声修正次数，修正量过大说明 PDF 质量差，需要检查

---

## 11. 完整数据结构参考

### 处理后的 JSON 结构

```json
{
  "metainfo": {
    "sha1_name": "a1b2c3d4e5",
    "company_name": "Example Corporation",
    "pages_amount": 48,
    "text_blocks_amount": 312,
    "tables_amount": 15,
    "pictures_amount": 8,
    "equations_amount": 0,
    "footnotes_amount": 23
  },
  "content": {
    "chunks": [
      {
        "id": 0,
        "page": 1,
        "text": "# Annual Report 2023\n\nExample Corporation delivered strong results...",
        "length_tokens": 287,
        "type": "content"
      },
      {
        "id": 1,
        "page": 2,
        "text": "Revenue for the company in Q3 2023: USD 5,234 million...",
        "length_tokens": 145,
        "type": "serialized_table",
        "table_id": 3
      }
    ],
    "pages": [
      {
        "page": 1,
        "text": "# Annual Report 2023\n\n## Financial Highlights\n\nExample Corporation delivered strong results..."
      }
    ]
  }
}
```

### 流水线代码骨架

```python
from pathlib import Path
from pdf_parsing import PDFParser
from tables_serialization import TableSerializer
from parsed_reports_merging import PageTextPreparation
from text_splitter import TextSplitter

# 目录配置
RAW_PDF_DIR = Path("./data/raw_pdfs")
PARSED_DIR = Path("./data/parsed")
SERIALIZED_DIR = Path("./data/serialized_tables")
MERGED_DIR = Path("./data/merged")
CHUNKED_DIR = Path("./data/chunked")

# Step 1: PDF 解析
parser = PDFParser(
    output_dir=PARSED_DIR,
    csv_metadata_path=Path("./metadata.csv")
)
parser.parse_and_export(doc_dir=RAW_PDF_DIR)

# Step 2: 表格序列化（可选，但强烈推荐）
serializer = TableSerializer()
serializer.process_directory_parallel(PARSED_DIR, max_workers=5)

# Step 3: 文本清洗与格式化
preparation = PageTextPreparation(
    use_serialized_tables=True,
    serialized_tables_instead_of_markdown=False  # 保留Markdown + 追加序列化描述
)
preparation.process_reports(reports_dir=PARSED_DIR, output_dir=MERGED_DIR)

# Step 4: 文本切块
splitter = TextSplitter()
splitter.split_all_reports(
    all_report_dir=MERGED_DIR,
    output_dir=CHUNKED_DIR
)

print("文档处理完成！")
```

---

## 总结

文档解析和切块是 RAG 系统的地基。地基不稳，上层再精妙的检索策略和 Prompt 工程也无法挽救。

核心原则回顾：

1. **用合适的工具解析 PDF**：Docling > pdfplumber > PyPDF2，选择能理解版式的工具
2. **元信息是一等公民**：页码、文档来源、文档标识符，从一开始就要设计好并贯穿整个处理流程
3. **表格需要特殊对待**：序列化为自包含文本块，结合上下文（标题、脚注）处理
4. **清洗不能省**：OCR 噪声、字体命令残留，这些"看不见的垃圾"会悄悄污染检索结果
5. **按 Token 切块，按页面划分**：控制精度，保留溯源能力
6. **用 Markdown 结构化中间表示**：让切块工具能够"理解"文档结构，在合理位置切割

做好这些，你的 RAG 系统就有了一个坚实的起点。

---

> 本系列下一篇：[RAG实战（二）-向量化与索引构建](#)，介绍如何把切好的 Chunk 变成可被快速检索的向量索引。

*本文基于 RAG-Challenge-2 项目的工程实践总结，参考代码见 `src/pdf_parsing.py`、`src/tables_serialization.py`、`src/parsed_reports_merging.py`、`src/text_splitter.py`。*
