---
name: pdf-lecture-notes
description: Use when the user wants to extract image-based PDF content (textbooks, lecture slides, scanned documents) into structured markdown notes. Triggered by requests like "convert this PDF to notes", "extract lectures from PDF", "make study notes from this textbook", or when working with image-heavy PDFs that need OCR.
---

# PDF to Lecture Notes

## Overview

Extract image-based PDF content into structured markdown notes. Uses PyMuPDF (fitz) for page extraction, PaddleOCR-VL MCP for OCR, and parallel background agents for throughput. The workflow is **plan-first**: analyze the document structure, design a universal plan with user approval, pilot one chapter, adjust based on feedback, then batch produce the rest.

## When to Use

- User provides an image-based PDF (textbook, slides, scanned notes) and wants markdown notes
- PDF is too large for direct text extraction (>100MB) or contains only scanned images
- Content includes formulas, tables, or diagrams that need preservation
- Multiple chapters need to be processed from a single PDF
- User says: "make notes from this PDF", "OCR this textbook", "extract lectures", "convert to markdown"

**Do NOT use for:**
- Text-based PDFs that can be read directly (use Read tool with `pages` parameter)
- PDFs small enough for direct reading (<100MB)
- Single-page or very short documents (just use PaddleOCR-VL directly)

## Environment Setup

### Python Dependencies

PyMuPDF (fitz) must be available in the project environment. Detect the package manager first, then install:

| 包管理器 | 检测特征 | 安装命令 |
|----------|----------|----------|
| **uv** | `pyproject.toml` + `uv.lock` | `uv add pymupdf` |
| **pip** | `requirements.txt` | `pip install pymupdf` |
| **conda** | `environment.yml` | `conda install -c conda-forge pymupdf` |

Verify:
```bash
# uv:
uv run python -c "import fitz; print(fitz.__version__)"
# pip/conda:
python -c "import fitz; print(fitz.__version__)"
```

> 📌 `uv` 本身是硬性依赖：PaddleOCR-VL MCP 服务器通过 `uvx` 启动，必须安装。参见 https://docs.astral.sh/uv/getting-started/installation/

### MCP Server: PaddleOCR-VL

OCR is done via the PaddleOCR-VL MCP server backed by Baidu AI Studio. This MCP server must be configured in the project's `.mcp.json` BEFORE any OCR operations.

**Step 1 — Direct the user to register on AI Studio and get an access token:**
> 🔗 Register at https://aistudio.baidu.com/ → create a project → get your access token from the project settings.

**Step 2 — Add the MCP server configuration to `.mcp.json`** in the project root:

```json
{
  "mcpServers": {
    "PaddleOCR-VL": {
      "command": "uvx",
      "args": [
        "--from",
        "paddleocr-mcp",
        "paddleocr_mcp"
      ],
      "env": {
        "PADDLEOCR_MCP_PIPELINE": "PaddleOCR-VL",
        "PADDLEOCR_MCP_PPOCR_SOURCE": "aistudio",
        "PADDLEOCR_MCP_SERVER_URL": "https://w9l3tced96y0w0u9.aistudio-app.com",
        "PADDLEOCR_MCP_AISTUDIO_ACCESS_TOKEN": "<YOUR_AI_STUDIO_ACCESS_TOKEN>"
      }
    }
  }
}
```

**⚠️ Replace `<YOUR_AI_STUDIO_ACCESS_TOKEN>` with the user's actual token. Never hardcode any specific token in the skill or in committed files.**

If `.mcp.json` already exists, merge the `PaddleOCR-VL` entry into the existing `mcpServers` block.

**Step 3 — Verify** that the MCP tool `mcp__PaddleOCR-VL__paddleocr_vl` becomes available after reloading the session. The user may need to restart Claude Code for the MCP server to register.

### Verify Setup

Before starting any PDF work, verify both components:

```bash
# 1. Python + fitz
uv run python -c "import fitz; print('fitz OK', fitz.__version__)"

# 2. MCP tool available — test with a single page OCR:
# (send one OCR call via mcp__PaddleOCR-VL__paddleocr_vl on any test PNG)
```

If either check fails, stop and fix the environment before proceeding.

---

## Core Workflow

```
Phase 0: 环境准备     →  装依赖、配MCP、验证
Phase 1: 分析+方案    →  目录→页码映射→采样→提问→编写通用方案.md
Phase 2: Pilot章节    →  执行第一章→用户反馈→调整方案
Phase 3: 批量制作     →  用户选择节奏→并行/分批→完成剩余章节
Phase 4: 收尾         →  附录→索引→清理→提交
```

---

### Phase 0: 环境准备

**Step 0 — ⚠️ 数据安全确认（必须先做）：**

OCR 处理需要将页面图片发送至百度 AI Studio 云端服务器。在开始任何操作之前，**必须**询问用户：

> ⚠️ 此 PDF 是否包含敏感或机密信息（如未公开财报、个人病历、身份证号、企业商业机密等）？OCR 处理会将页面图片发送至百度 AI Studio 云端。如包含敏感信息，请立即停止，不要继续上传。

**Do NOT proceed unless the user confirms the PDF is safe for cloud OCR.**

**Step 1 — 检查是否中断恢复：**

扫描当前目录下是否存在 `temp_ch*`、`temp_sample` 目录或已输出的笔记文件：

```bash
ls -d temp_ch*/ temp_sample/ temp_lec*/ 讲义/*.md 笔记/*.md 2>/dev/null
```

如果发现已有进度文件：
- 检查是否存在 `notes-plan.md`（已有的通用方案）
- 列出已完成的章节和暂未处理的章节
- 询问用户：从断点继续还是重新开始？

> 发现已有进度：已完成第 1-3 章，第 4-8 章未完成。是从第 4 章继续，还是重新开始？

如果继续，保留已有的 `notes-plan.md` 和临时文件，直接跳到 Phase 3 从断点处执行。

**Step 2 — 打开 PDF，采集基本信息：**

```bash
uv run python -c "
import fitz
doc = fitz.open('<PDF_PATH>')
print(f'总页数: {doc.page_count}')
print(f'加密: {doc.is_encrypted}')
print(f'文件大小: {doc.metadata}')
# 检查前几页的旋转状态
for i in range(min(5, doc.page_count)):
    page = doc[i]
    rot = page.rotation
    txt_len = len(page.get_text())
    print(f'第{i+1}页: 旋转={rot}°, 可提取文字={txt_len}字符')
doc.close()
"
```

根据输出判断：
- **`is_encrypted = True`** → 询问密码，用 `fitz.open(path, password=...)` 重试。没有密码则终止。
- **旋转 ≠ 0** → 记录需要旋转校正的页面
- **可提取文字 < 50 字符的页** → 标记为图片页，需要 OCR
- **混合类型**：如果部分页文字多、部分页文字少 → 提示用户此为混合 PDF，文字页直接读取，图片页走 OCR

**Step 3 — 估算资源需求，告知用户：**

```
📊 PDF 概况：
- 总页数：N 页
- 类型：纯图片 / 纯文字 / 混合（X 页文字 + Y 页图片）
- 需要 OCR 的页数：Y 页
- 预估临时 PNG 空间：约 Z GB（Y × 约 2MB/页 @ 200DPI）
- 预估 OCR 调用次数：Y 次
```

如果临时空间超过 5GB，提示用户可能需要分批清理。询问是否降低 DPI 到 150 以节省空间。

**Step 4 — API 配额预检：**

用 fitz 提取 PDF 第 1 页为 PNG，发送一次 OCR 测试调用：

```bash
uv run python -c "
import fitz
doc = fitz.open('<PDF_PATH>')
doc[0].get_pixmap(dpi=200).save('temp_setup_test.png')
print('test page saved')
"
```

然后调用 `mcp__PaddleOCR-VL__paddleocr_vl` OCR 这一页。如果返回 "402 Insufficient Balance" 或类似余额不足错误 → 提示用户充值。确认 API 可用后再继续。

**Step 5 — 环境检测 + 安装依赖：**

检查项目的包管理工具：

```bash
ls pyproject.toml uv.lock requirements.txt environment.yml 2>/dev/null
```

> 📌 `uv` 是硬性依赖：PaddleOCR-VL MCP 服务器通过 `uvx` 启动（见 `.mcp.json` 中的 `"command": "uvx"`）。如果用户未安装 `uv`，请指导安装：https://docs.astral.sh/uv/getting-started/installation/

安装 `pymupdf`：

| 包管理器 | 安装命令 |
|----------|----------|
| **uv** | `uv add pymupdf` |
| **pip** | `pip install pymupdf` |
| **conda** | `conda install -c conda-forge pymupdf` |

**Step 6 — 验证：**

```bash
uv run python -c "import fitz; print('fitz OK', fitz.__version__)"
# 或 pip: python -c "import fitz; print('fitz OK', fitz.__version__)"
```

确认 `.mcp.json` 包含 `PaddleOCR-VL` 服务器配置。

**Do NOT proceed until ALL checks pass.**

---

### Phase 1: 文档分析与方案制定

#### Step 1.1 — 提取目录

Use the Read tool on the PDF to read the table of contents pages:

```
Read(file_path="path/to/document.pdf", pages="2-5")
```

Extract:
- Chapter/section titles
- Printed page numbers for each chapter
- Any hierarchical structure (parts → chapters → sections)

If the PDF is image-based (TOC pages return garbled text), OCR the TOC pages directly.

**⚠️ 如果 PDF 没有目录（或目录无法解析）：**

按以下优先级自动探测章节边界：

1. **文本密度跳跃法**：用 fitz 遍历所有页面，统计每页的可提取文本量。大幅变化的页面（如前页文字多、后页文字突然归零）往往是章节扉页。
   ```bash
   uv run python -c "
   import fitz
   doc = fitz.open('<PDF_PATH>')
   for i in range(doc.page_count):
       text = doc[i].get_text()
       print(f'{i}: {len(text)} chars')
   doc.close()
   "
   ```
2. **标题样式采样**：每 10 页抽 1 页做 OCR，检测是否有大字/居中/新页起始的标题模式，据此推断章节分界。
3. **页码重置检测**：如果书每章重编页码，fitz 提取的页码重置点就是章节边界。
4. **兜底方案**：以上方法均失败 → 展示 PDF 总页数和每隔 N 页的 OCR 摘要，让用户手动指定每章起止页。

无论用哪种方式，最终都要产出章节→页码的映射表并让用户确认。

#### Step 1.2 — 建立页码映射

Document the mapping between **PDF page numbers** (0-indexed in fitz, 1-indexed in PDF reader) and **printed/book page numbers**.

```
| 章节 | 书页码 | PDF页码(1-indexed) | fitz索引(0-indexed) |
|------|--------|---------------------|---------------------|
| 第1章 | P1-P30  | pp.1-30            | 0-29                |
| 第2章 | P31-P60 | pp.31-60           | 30-59               |
| ...  | ...    | ...                | ...                 |
```

> ⚠️ PDF页码 ≠ 书页码。必须通过目录确认映射关系。

#### Step 1.3 — 采样阅读，识别内容元素

Select the first 1-2 chapters and read a representative sample of pages. The goal is to understand what kinds of content exist and how they're structured.

**Sampling strategy:**
- First 3 pages of chapter 1 (opening, overview, learning objectives)
- 2-3 pages from the middle (main content body)
- 2-3 pages from the end (exercises, summaries)

For image-based PDFs, extract these sample pages to PNG and OCR them:

```bash
uv run python -c "
import fitz, os
doc = fitz.open('document.pdf')
os.makedirs('temp_sample', exist_ok=True)
# Replace indices with actual sample pages
for i in [10, 11, 12, 25, 26, 27, 42, 43, 44]:
    doc[i].get_pixmap(dpi=200).save(f'temp_sample/page_{i:04d}.png')
"
```

Then OCR each sample page with:
```
mcp__PaddleOCR-VL__paddleocr_vl(
  input_data: "absolute/path/to/temp_sample/page_0010.png"
  file_type: "image"
  output_mode: "simple"
)
```

From the samples, identify:

| 识别项 | 说明 |
|--------|------|
| **内容元素类型** | 定义、定理、公式、例题、习题、小结、拓展阅读、边栏提示等 |
| **每类元素的特征** | 在页面上的位置、是否有特殊标记（图标、色块、编号） |
| **章节内部结构** | 各元素出现的顺序和层级关系 |
| **页面排版复杂度** | 单栏/双栏/混合排版？是否有嵌入表格？文字环绕？ |
| **特殊内容** | 大量图表、代码块、化学方程式等需要特殊处理的 |
| **页面类型分布** | 纯文字页 vs 图片页的比例和分布规律 |

> ⚠️ 如果采样发现**多栏排版**或**复杂表格嵌入**，必须在 Phase 1 提问时明确告知用户：OCR 在复杂排版下可能产生阅读顺序混乱，建议用户做好手动校对的心理准备。对于关键页面可以尝试 `output_mode: "detailed"` 获取坐标信息辅助排序。

#### Step 1.4 — 向用户提问，确定通用方案

Based on what you learned from sampling, ask the user a structured set of questions. Present each question with a **recommended default** based on what you observed.

**必须提问的内容：**

**A. 笔记结构** — "每章的笔记文件用什么结构？"
- 推荐基于你的采样观察给出一个结构模板
- 让用户确认板块名称、顺序、是否所有板块都必需
- 示例提问格式：
  ```
  基于采样，我看到每章包含以下元素：[列出元素]。
  我建议每章笔记按此结构组织：
  1. 章节标题 + 页码范围
  2. 考情/概述
  3. 知识结构图
  4. 基础内容精讲（按知识点分节）
  5. 例题精讲
  6. 习题精练
  7. 核心结论速查
  
  这个结构可以吗？有需要增删或调整顺序的吗？
  ```

**B. 内容取舍规则** — "哪些保留、哪些简化、哪些省略？"
- 每种内容元素逐一确认
- 示例：定义定理→完整摘录；例题→保留题目+解答；边栏→视重要性决定

**C. 公式/特殊符号处理规范** — "数学公式、化学式等用什么格式？"
- LaTeX 还是其他？行内/行间公式的写法？特殊符号约定？

**D. 文件命名与输出目录** — "笔记文件存哪里？怎么命名？"

**E. 输出格式** — "最终产出什么格式？"
- 仅 Markdown（`.md` 文件）
- Markdown + PDF（需要将 md 渲染为 PDF）
  - 如果输出 PDF：整个一份 PDF 还是各章节单独 PDF？
  - PDF 生成工具建议：Pandoc + LaTeX / WeasyPrint / 浏览器打印

> 📌 如果用户选择输出 PDF，在方案文档中注明 PDF 生成方式。Phase 4 中执行 PDF 导出。

**F. 其他约定** — 根据文档类型可能有额外问题：
- 图表怎么处理（描述还是跳过）？
- 原文中的交叉引用怎么处理？
- 是否需要保留原书页码标注？

#### Step 1.5 — 编写通用方案文档

After the user answers, write ONE universal plan document that applies to ALL chapters:

**File:** `notes-plan.md`（或用户指定的名称）

This document must include:
1. **页码范围映射表** — 每章的PDF页码和书页码
2. **页面类型标记** — 每章哪些页是图片、哪些页可直接读文字（混合PDF）
3. **笔记结构模板** — 用户确认的板块结构
4. **内容取舍规则** — 每种元素怎么处理
5. **格式规范** — 公式、图表、特殊符号的处理约定
6. **文件命名规则** — 输出目录和文件命名方式
7. **输出格式** — 仅 MD / MD+PDF（整体/分章节）
8. **执行节奏** — 用户选择的全量/分批/逐章（Phase 3 前确认）

Show the document to the user and get explicit approval before proceeding.

---

### Phase 2: Pilot 章节

#### Step 2.1 — 执行第一章

Follow the universal plan exactly. Execute chapter 1 through the full pipeline:

**If image-based PDF (the common case):**

1. **提取页面为 PNG** — use fitz（含旋转校正）:
```bash
uv run python -c "
import fitz, os
doc = fitz.open('<PDF_PATH>')
os.makedirs('temp_ch01', exist_ok=True)
for i in range(<start_0indexed>, <end_0indexed>):
    page = doc[i]
    # 如果有旋转，先校正再提取
    rot = page.rotation
    if rot != 0:
        page.set_rotation(0)
    page.get_pixmap(dpi=200).save(f'temp_ch01/page_{i:04d}.png')
    if rot != 0:
        page.set_rotation(rot)  # 恢复原状
print(f'Extracted {<end_0indexed> - <start_0indexed>} pages')
"
```

2. **并行 OCR** — dispatch background agents (see Phase 3 batch dispatch template below), ~10 pages per agent

3. **整合内容** — wait for all agents, merge their output following the universal plan structure

**If text-based PDF:** Just use the Read tool page by page.

#### Step 2.2 — 展示并收集反馈

Write the completed chapter 1 notes, then ask the user:

```
第一章笔记已完成，请看：[文件路径]

请重点检查：
- 结构是否符合预期？
- 内容的完整性（有没有缺漏？）
- 公式/格式是否正确？
- 有没有需要调整的地方？

需要修改通用方案的地方请告诉我。
```

#### Step 2.3 — 调整方案

Based on user feedback, update `notes-plan.md` in-place. Add a brief **修订记录** at the end:

```markdown
## 修订记录
- 2026-XX-XX：Pilot 第1章后调整 — [具体修改内容]
```

This ensures subsequent chapters follow the refined plan.

---

### Phase 3: 批量制作

#### Step 3.1 — 确认执行节奏

Ask the user to choose the execution rhythm:

> 剩余 X 章需要制作。你希望：
> - (A) 一次性全部并行制作（最快）
> - (B) 分批制作（推荐每批 N 章，中间可以抽查）
> - (C) 逐章制作（最稳，每章做完可微调）

Update the plan document with the chosen strategy.

#### Step 3.2 — 执行批量制作

For each chapter, follow the same pipeline as the pilot:

**通用章节制作流程：**

```
1. ⚠️ 清理章节标题中的非法文件名字符（: / \ * ? " < > | → 替换为 -）
2. 对于混合 PDF：文字页直接 fitz.get_text() 读取，图片页提取 PNG 走 OCR
3. 提取图片页为 PNG with fitz (200 DPI)，处理页面旋转
4. Divide PNGs into batches of ~10 pages WITH 1-page overlap between adjacent batches
5. Dispatch one background Agent per batch (ALL in parallel)
6. Wait for ALL agents to complete — see failure recovery below
7. Merge consolidated markdown + 文字页内容，按页码顺序组装
   - Check for and resolve duplicated boundary content from overlap pages
   - Verify cross-page elements (formulas, tables, paragraphs) are intact
8. Write the chapter notes file
9. 可选：删除该章的临时 PNG（分批清理节省磁盘）
```

**Overlap mechanism — critical for cross-page continuity:**

Adjacent batches MUST overlap by 1 page. For example:
- Agent A: pages 30-40 (primary), page 41 (context-only)
- Agent B: pages 40-49 (primary), page 50 (context-only)
- Agent C: pages 49-58 (primary), ...

Each agent OCRs the overlap page for **context only** — it uses it to complete truncated sentences/formulas/tables, but does NOT include it in its final output. The merge step handles deduplication.

**Agent dispatch template** (dispatch ALL agents simultaneously):

```
Agent(
  description: "OCR ChX Pxxx-xxx"
  subagent_type: "general-purpose"
  run_in_background: true
  prompt: """
OCR these PNG pages using mcp__PaddleOCR-VL__paddleocr_vl with file_type="image".

Your batch (primary pages — output these):
  /absolute/path/to/temp_chXX/page_0030.png
  /absolute/path/to/temp_chXX/page_0031.png
  ...
  /absolute/path/to/temp_chXX/page_0039.png

Overlap page (context only — do NOT include in output):
  /absolute/path/to/temp_chXX/page_0040.png

For each page, call paddleocr_vl with file_type="image" and output_mode="simple".

IMPORTANT: Also OCR the overlap page. Use it to complete any content that is truncated
at the end of your last primary page (split formulas, tables, paragraphs). But only
OUTPUT the consolidated content for your PRIMARY pages — do not include the overlap
page in your final markdown.

Consolidate ALL primary pages into ONE structured markdown response.
Preserve formulas, tables, and special formatting exactly as OCR'd.
"""
)
```

**Critical rules for OCR calls:**
- **`file_type="image"` is REQUIRED** for every PNG file. It is NOT auto-detected.
- Use **absolute paths** (e.g., `/absolute/path/to/file.png`; on Windows use forward slashes: `C:/Users/.../file.png`)
- **~10 primary pages + 1 overlap page per agent** — balance between parallelism and overhead
- **All background agents dispatched in one message** — they run simultaneously
- **Do NOT start merging until ALL agents complete** — wait for all notifications

**Failure recovery:**

If any background agent fails (timeout, rate limit, crash):

1. Note the exact batch (page range) and the error message
2. Wait 30 seconds for rate-limit cooldown
3. Re-dispatch **only the failed batch** as a new background agent (same overlap rule)
4. Do NOT re-dispatch successful batches — their results are already available
5. Continue merging when the retried batch completes
6. If the same batch fails **3 times**, stop and ask the user how to proceed (e.g., reduce batch size, check API quota)

**After each chapter (or batch), report progress:**
> ✅ 第X章完成 → `讲义/第X讲 xxx.md`（Pxxx-Pxxx，N页）

#### Step 3.3 — 抽查（仅分批/逐章模式）

If doing batches, after each batch ask the user if they want a spot check before continuing.

---

### Phase 4: 收尾

#### Step 4.1 — 附录处理

Ask the user whether appendices should also be extracted. If yes, use the same pipeline. Appendices may use a simplified structure — confirm with user.

#### Step 4.2 — 生成总目录索引（可选）

If the user wants a master index file, generate `README.md` in the output directory with:
- Links to all chapter files
- Page ranges for each chapter
- Any cross-references

#### Step 4.3 — PDF 导出（如果用户在 Phase 1 选择了输出 PDF）

根据方案文档中约定的 PDF 方式执行：

- **Pandoc + LaTeX**（推荐，数学公式支持最好）：
  ```bash
  # 单个文件 → PDF
  pandoc notes.md -o notes.pdf --pdf-engine=xelatex -C
  # 合并所有章节 → 一个 PDF
  pandoc ch*.md -o combined.pdf --pdf-engine=xelatex -C
  ```
- **WeasyPrint**（HTML/CSS 渲染，排版灵活）
- **浏览器打印**（最简单，手动操作）

按用户选择的整体/分章节方式导出。

#### Step 4.4 — 清理临时文件

```bash
# Remove all temp extraction directories
rm -rf temp_ch*/ temp_sample/ temp_lec*/ temp_setup_test.png
```

#### Step 4.5 — Git 提交（可选）

Ask the user if they want to commit the output files.

---

## Quick Reference

### fitz Extraction (Single Chapter — with rotation handling)

```bash
uv run python -c "
import fitz, os
doc = fitz.open('<PDF_PATH>')
os.makedirs('<OUTPUT_DIR>', exist_ok=True)
start, end = <START_0INDEXED>, <END_0INDEXED>
for i in range(start, end):
    page = doc[i]
    rot = page.rotation
    if rot != 0:
        page.set_rotation(0)
    page.get_pixmap(dpi=200).save(f'<OUTPUT_DIR>/page_{i:04d}.png')
    if rot != 0:
        page.set_rotation(rot)
print(f'Done: {end-start} pages extracted to <OUTPUT_DIR>/')
"
```

### Filename Sanitization (Python snippet)

```python
import re
def safe_filename(title):
    """Remove characters illegal in filenames across OS."""
    return re.sub(r'[<>:\"/\\|?*]', '-', title).strip()
```

### fitz Extraction (Multiple Chapters / All at Once)

```python
import fitz, os

doc = fitz.open('<PDF_PATH>')

# Chapter definitions: (name, start_0indexed, end_0indexed)
chapters = [
    ('ch01', 10, 45),
    ('ch02', 45, 80),
    # ...
]

for name, start, end in chapters:
    os.makedirs(f'temp_{name}', exist_ok=True)
    for i in range(start, end):
        doc[i].get_pixmap(dpi=200).save(f'temp_{name}/page_{i:04d}.png')
    print(f'{name}: {end-start} pages extracted')

doc.close()
```

### PaddleOCR-VL Call Format

```
mcp__PaddleOCR-VL__paddleocr_vl(
  input_data: "/absolute/path/to/page.png"
  file_type: "image"        # REQUIRED for PNG files — NOT auto-detected
  output_mode: "simple"     # Clean markdown (use "detailed" only for layout analysis)
)
```
> On Windows, use forward slashes: `C:/Users/.../file.png`

### Batch Division Formula

```
Total pages ÷ 10 = number of agents (round up)
Example: 35 pages → 4 agents × ~9 pages
```

### Agent Completion Flow

```
1. Extract pages to PNG, then divide into batches with 1-page overlap
2. Dispatch all OCR agents with run_in_background: true
3. Wait for system notifications for each agent
4. If any agent failed: wait 30s, re-dispatch only the failed batch. Retry up to 3 times.
5. Read each agent's consolidated markdown
6. Merge into final chapter file following the universal plan
   - Remove duplicated boundary content from overlap pages
   - Verify cross-page elements are intact
7. Verify: check page count, spot-check boundary pages for truncation
```

### fitz PDF Info Dump (Phase 0)

```bash
uv run python -c "
import fitz, os
doc = fitz.open('<PDF_PATH>')
print(f'总页数: {doc.page_count}')
print(f'加密: {doc.is_encrypted}')
print(f'文件大小: {os.path.getsize(\"<PDF_PATH>\") / 1024 / 1024:.1f} MB')
for i in range(min(5, doc.page_count)):
    p = doc[i]
    print(f'第{i+1}页: 旋转={p.rotation}°, 可提取文字={len(p.get_text())}字符')
doc.close()
"
```

### Sample Extraction for Analysis

```bash
uv run python -c "
import fitz, os
doc = fitz.open('<PDF_PATH>')
os.makedirs('temp_sample', exist_ok=True)
# Sample: first 3 pages + middle 3 pages + last 3 pages of chapter 1
sample_pages = [10,11,12, 22,23,24, 37,38,39]
for i in sample_pages:
    doc[i].get_pixmap(dpi=200).save(f'temp_sample/page_{i:04d}.png')
print('Sample pages extracted')
"
```

---

## User Interaction Checklist

### Phase 0 必须提问

- [ ] ⚠️ 数据安全确认 — PDF 是否含敏感/机密信息（OCR 走百度 AI Studio 云端）
- [ ] 中断恢复确认 — 如检测到已有进度，从断点继续还是重新开始？
- [ ] PDF 密码（如果加密）
- [ ] 磁盘空间 + API 配额 — 告知预估用量，确认是否继续
- [ ] AI Studio access token（如果 MCP 未配置）

### Phase 1 必须提问

- [ ] **章节划分确认** — 展示从目录（或自动探测）得到的章节映射表，让用户纠正
- [ ] 笔记的板块结构（展示推荐模板，让用户确认/调整）
- [ ] 每种内容元素的取舍规则
- [ ] 公式/特殊符号的格式规范
- [ ] 输出目录和文件命名方式
- [ ] **输出格式** — 仅 MD / MD+PDF（整体/分章节）
- [ ] ⚠️ 复杂排版警告（如果采样发现多栏/混合排版）
- [ ] 通用方案文档 → **必须等用户批准后再继续**

### Phase 2 必须提问

- [ ] Pilot章节完成 → 展示结果，等待反馈
- [ ] 方案调整 → 记录修订内容

### Phase 3 必须提问

- [ ] 执行节奏选择（全量 / 分批 / 逐章）
- [ ] 如果分批：每批几章
- [ ] 如果逐章：是否每章后暂停确认

### Phase 4 必须提问

- [ ] 附录是否处理
- [ ] 是否生成总目录索引
- [ ] PDF 导出（如果在 Phase 1 选择了输出 PDF）
- [ ] 是否清理临时文件
- [ ] 是否 git 提交

---

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Using `pdftoppm` / `pdf2image` | Use fitz (PyMuPDF) |
| Running wrong Python command → ModuleNotFoundError | Detect package manager first (uv/pip/conda), use correct command |
| Forgetting `file_type="image"` in PaddleOCR-VL | Always include `file_type="image"` for PNG inputs — it IS required, NOT auto-detected |
| OCRing sequentially (one page at a time) | Batch ~10 pages per background agent |
| Using sequential rounds of parallel MCP calls | Use background agents, NOT manual parallel call rounds |
| Not using `run_in_background: true` | All OCR agents must be background for parallelism |
| Setting DPI too low (<150) | Use 200 DPI minimum |
| Confusing fitz 0-indexed pages with PDF reader 1-indexed | fitz page 0 = PDF page 1 displayed in reader |
| Writing notes before all agents complete | Wait for ALL agent notifications before integrating |
| Not extracting boundary pages for uncertain chapter borders | Always extract a few extra pages at chapter boundaries |
| Hard page-split at batch boundaries → truncated content | Use 1-page overlap between adjacent batches |
| Merge without deduplication → doubled boundary content | Strip overlap content from each batch's output during merge |
| One agent fails → merge deadlocks forever | Retry failed batch up to 3 times with 30s cooldown; ask user if all retries fail |
| Sending sensitive PDF to cloud OCR without warning | ALWAYS ask privacy question in Phase 0 before any OCR |
| Hardcoding Windows paths in agent prompts | Use platform-agnostic absolute paths (forward slashes) |
| Including appendix content as chapter content | Verify page ranges from TOC |
| Skipping the plan step — jumping straight to extraction | Plan first, extract later |
| Not asking the user about structure/format | Always ask the questions in Phase 1 |
| Writing chapter notes without user-approved plan | Get plan approved before touching any chapter |
| Hardcoding API keys in code or config | Use placeholder `<YOUR_AI_STUDIO_ACCESS_TOKEN>` |
| Making plan per-chapter instead of one universal plan | ONE plan document covers ALL chapters |
| PDF has no TOC → giving up | Use auto-detection: text-density jumps, title sampling, page-number resets |
| All pages treated as images in mixed PDF | Detect text vs image pages; read text pages directly, only OCR image pages |
| Not handling page rotation → sideways PNGs | Check `page.rotation` before extraction, correct if non-zero |
| Chapter title has illegal filename chars → save fails | Sanitize filenames: replace `<>:\"/\\|?*` with `-` |
| Not estimating disk space → runs out mid-extraction | Estimate upfront; warn if >5GB; offer lower DPI; clean up per-chapter |
| API quota runs out mid-way → all work lost | Pre-check with single OCR test; monitor 402 errors; keep successful results |
| Starting fresh when previous progress exists | Check for temp dirs/output files; ask user: resume or restart? |
| Not asking about output format | Always ask: MD only or MD+PDF (combined or per-chapter) |

---

## Red Flags — STOP and Check the Skill

| Thought | Reality |
|---------|---------|
| "I'll skip the plan and just start extracting" | Plan is mandatory. Structure agreed before work begins. |
| "This PDF is probably not sensitive, no need to ask" | You don't know. ALWAYS ask. Cloud OCR = data leaves the machine. |
| "I'll call OCR in parallel rounds myself" | You'll burn context on raw OCR text and take 6× longer. Use background agents. |
| "file_type is auto-detected, I can skip it" | It is NOT auto-detected. Must be `"image"` for every PNG call. |
| "I'll use pdf2image/pdftoppm instead of fitz" | fitz is already in project deps. No system install needed. |
| "5 pages per batch is safer" | 10 pages is the proven batch size. 5 doubles your agent count for no benefit. |
| "I'll process them one at a time to be careful" | 80 pages × 15s each = 20 min sequential vs 3 min with 8 parallel agents. |
| "The batches are exactly 10 pages, no need for overlap" | Adjacent batches WILL cut content. Always include 1-page overlap. |
| "One agent failed, I'll just start over from scratch" | Only retry the failed batch. Successful results are already done. |
| "Agent failed, I'll wait and hope it recovers" | Actively retry. Waiting forever = deadlock. |
| "Let me write notes while agents are running" | Wait for ALL agents. Partial content leads to disorganized notes. |
| "This PDF looks simple, I can skip the sampling" | Even simple PDFs have structural quirks. Always sample. |
| "I'll decide the structure myself, user probably wants X" | NEVER assume. Always ask. |
| "I know what the chapter structure should be" | You don't. The user is the domain expert. Ask. |
| "I'll make a per-chapter plan for each one" | ONE universal plan covers ALL chapters. Don't duplicate work. |
| "I'll use D:/ paths, it'll work on any OS" | Use forward-slashed absolute paths. The OS handles the conversion. |
| "No TOC? I'll just guess the chapter boundaries" | Use text-density + title sampling. Don't guess. |
| "Some pages have text, some are images — I'll just OCR everything" | Read text pages directly. Wasting OCR calls = slower + costs quota. |
| "Page rotation doesn't matter, OCR can handle it" | Rotated pages produce wrong OCR. Fix rotation before extraction. |
| "I don't need to sanitize the filename, the chapter title looks safe" | Always run sanitization. One `:` character breaks the file write. |
| "The PDF is probably not that big, I'll skip the disk space check" | 500 pages × 200 DPI ≈ 2-3 GB. Check before you fill the disk. |
| "I see temp files from before, I'll just delete them and start fresh" | They might be valid progress. Ask the user first. |
| "I'll assume MD-only output, no need to ask about PDF" | Always ask. User may expect a printable PDF. |

---

## Agent Prompt Checklist

Every OCR agent you dispatch MUST be told:
- [ ] Use `mcp__PaddleOCR-VL__paddleocr_vl` with **`file_type="image"`**
- [ ] The exact list of PNG file paths (absolute paths) in page order
- [ ] Which 1 overlap page to OCR for context (and that it must NOT appear in output)
- [ ] To consolidate ALL primary pages into ONE structured markdown response
- [ ] To preserve formulas, tables, and special formatting exactly as OCR'd
- [ ] To organize content by the structure defined in the universal plan

## Boundary Page Handling

When the exact chapter boundary is uncertain (some pages may contain content from both chapters):

1. Extract overlapping pages into a separate temp directory
2. OCR boundary pages separately to determine the exact split point
3. Place overlapping pages with the chapter that has more content on that page
