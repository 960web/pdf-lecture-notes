---
name: pdf-lecture-notes
description: Use when the user wants to extract image-based PDF content (textbooks, lecture slides, scanned documents) into structured markdown notes. Triggered by requests like "convert this PDF to notes", "extract lectures from PDF", "make study notes from this textbook", or when working with image-heavy PDFs that need OCR.
---

# PDF to Lecture Notes

## Overview

Extract image-based PDF content into structured markdown notes. Uses PyMuPDF (fitz) for PDF splitting and PaddleOCR-VL API for all OCR (per-chapter jobs submitted directly to Baidu AI Studio). All OCR goes through the API — no MCP calls, no PNG conversion. Parallel background agents submit jobs simultaneously. The workflow is **plan-first**: analyze document structure, design a universal plan with user approval, pilot one chapter, adjust based on feedback, then batch produce the rest.

## When to Use

图片型 PDF（教材、幻灯片、扫描笔记）需要提取为结构化 Markdown 笔记时使用。特别是 PDF >100MB、含公式/表格/图表、多章节需要批量处理的情况。

**不适用于：** 文字型 PDF（直接用 Read 工具）、单页或很短的文档。

## Environment Setup

### Python Dependencies

PyMuPDF (fitz) 和 requests 必须可用：

| 包管理器 | 安装命令 |
|----------|----------|
| **uv** | `uv add pymupdf requests` |
| **pip** | `pip install pymupdf requests` |
| **conda** | `conda install -c conda-forge pymupdf requests` |

### API Token 配置

OCR 需要百度 AI Studio access token。

> 🔗 注册 https://aistudio.baidu.com/ → 创建项目 → 从项目设置获取 access token

```bash
# .env 文件格式（KEY=VALUE）
PADDLEOCR_AISTUDIO_ACCESS_TOKEN=<YOUR_TOKEN>
```

Python 脚本从 `.env` 动态读取 token，不硬编码。

> ⚠️ **安全规则**：`.env` 已在 `.gitignore` 中。SKILL.md 内的 Python 代码只从 `.env` 读取，绝不出现真实 token 值。

### Python 脚本约定

所有 Python 操作使用固定命名的脚本文件，AI **禁止**现场写代码，只能从下方模板原样落盘后传参执行：

```
cat > scripts/phaseN_xxx.py << 'PYEOF'
...代码从模板原样复制...
PYEOF

uv run python scripts/phaseN_xxx.py <args>
```

> AI 只替换 `<args>` 命令行参数。脚本内容和文件名都是固定的，不可修改。

### Verify Setup

```bash
mkdir -p scripts

# 1. Python + fitz + requests（首次落盘）
cat > scripts/phase0_check_env.py << 'PYEOF'
import sys
import fitz
import requests

print('fitz OK', fitz.__version__)
print('requests OK', requests.__version__)

if len(sys.argv) > 1:
    pdf_path = sys.argv[1]
    doc = fitz.open(pdf_path)
    print(f'总页数: {doc.page_count}')
    print(f'加密: {doc.is_encrypted}')
    print(f'文件大小: {doc.metadata}')
    for i in range(min(5, doc.page_count)):
        page = doc[i]
        rot = page.rotation
        txt_len = len(page.get_text())
        print(f'第{i+1}页: 旋转={rot}°, 可提取文字={txt_len}字符')
    doc.close()
PYEOF

uv run python scripts/phase0_check_env.py
# 2. token 可用 — Phase 0 Step 4 用单页 PDF 测试 API
```

如果任一检查失败，停止并修复环境后再继续。

---

## Core Workflow

```
Phase 0: 环境准备     →  配 token、验证 API、估算资源
Phase 1: 分析+方案    →  目录→页码映射→采样→提问→编写通用方案.md
Phase 2: Pilot章节    →  执行第一章→用户反馈→调整方案
Phase 3: 批量制作     →  用户选择节奏→并行/分批→完成剩余章节
Phase 4: 收尾         →  附录→索引→清理→提交
```

---

### Phase 0: 环境准备

**Step 0 — ⚠️ 数据安全确认（必须先做）：**

OCR 处理需要将 PDF 页面发送至百度 AI Studio 云端服务器。在开始任何操作之前，**必须**询问用户：

> ⚠️ 此 PDF 是否包含敏感或机密信息（如未公开财报、个人病历、身份证号、企业商业机密等）？OCR 处理会将 PDF 页面发送至百度 AI Studio 云端。如包含敏感信息，请立即停止，不要继续上传。

**Do NOT proceed unless the user confirms the PDF is safe for cloud OCR.**

**Step 0.5 — ⛔ STOP：索要百度 AI Studio access token：**

向用户索要 token，写入 `.env`（格式见上方 Environment Setup）。

**Step 1 — 检查是否中断恢复：**

扫描当前目录下是否存在 `temp_ch*`、`temp_sample` 目录或已输出的笔记文件：

```bash
ls -d temp_ch*/ temp_sample/ temp_lec*/ 讲义/*.md 笔记/*.md 2>/dev/null
# 检查是否已有 OCR Agent 写入磁盘的 chapter_ocr.md 文件：
ls temp_ch*/chapter_ocr.md 2>/dev/null
```

如果发现已有进度文件：
- 检查 `notes-plan.md`、已完成的章节、每个 `temp_chXX/chapter_ocr.md` 是否存在且非空（存在=OCR 已完成，无需重新 OCR）
- 询问用户：从断点继续还是重新开始？

> 发现已有进度：第 1-3 章已完成，temp_ch04/chapter_ocr.md 已存在（OCR 完成可直接合并）。从第 4 章继续还是重新开始？

如果继续，保留已有文件，直接跳到 Phase 3 从断点处执行。

**Step 2 — 打开 PDF，采集基本信息：**

```bash
uv run python scripts/phase0_check_env.py <PDF_PATH>
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
- 章节数：M 章
- 预估 API Job 数量：M 个（每章一个）
- 预估临时空间：约与原 PDF 等大（按章切 PDF + overlap）
- 单章 PDF 超过 50MB 时建议拆分
```

**Step 4 — API 连通性测试：**

用 fitz 提取 PDF 第 1 页为独立 PDF，提交 mini API Job 验证 token 有效：

```bash
cat > scripts/phase0_create_test_pdf.py << 'PYEOF'
import sys, fitz

pdf_path = sys.argv[1]
output = sys.argv[2]

doc = fitz.open(pdf_path)
test = fitz.open()
test.insert_pdf(doc, from_page=0, to_page=0)
test.save(output)
print(f'test page PDF saved -> {output}')
PYEOF

uv run python scripts/phase0_create_test_pdf.py <PDF_PATH> temp_setup_test.pdf
```

然后落盘并运行 `scripts/phase3_submit_job.py`（代码见 Phase 3 Agent 派发模板）提交 `temp_setup_test.pdf` 到 API。如果返回 402 或 token 无效 → 提示用户。确认可用后再继续。

**Step 5 — 验证 .env：**

确认 `.env` 存在且包含 `PADDLEOCR_AISTUDIO_ACCESS_TOKEN`。全部检查通过后才能继续。

---

### Phase 1: 文档分析与方案制定

> ⛔ STOP — Phase 1 禁止直接执行任何操作。必须先完成所有用户提问并获得明确批准，才能进入 Phase 2。

#### Step 1.1 — 提取目录

用 Read 工具读 PDF 的目录页：

```
Read(file_path="path/to/document.pdf", pages="2-5")
```

提取：章节标题、每章书页码、层级结构（篇→章→节）。

如果目录页是图片（返回乱码），直接 OCR 目录页。

**⚠️ 如果 PDF 没有目录（或目录无法解析）：**

按以下优先级自动探测章节边界：

1. **标题样式采样**：每 10 页抽 1 页做 OCR，检测是否有大字/居中/新页起始的标题模式，据此推断章节分界。
2. **页码重置检测**：如果书每章重编页码，fitz 提取的页码重置点就是章节边界。
3. **兜底方案**：以上方法均失败 → 展示 PDF 总页数和每隔 N 页的 OCR 摘要，让用户手动指定每章起止页。

无论用哪种方式，最终都要产出章节→页码的映射表并让用户确认。

> ⛔ STOP — 必须先向用户展示章节映射表并获得确认，禁止直接跳到采样。

#### Step 1.2 — 校准 PDF→书页 偏移量，建立页码映射

前言/目录通常用罗马数字或不编页码，正文 PDF 页码和书页码之间有固定偏移。校准流程：

---

**Step 1.2a — 从目录结束位置开始，定位第 1 章首页**

从「目录最后一页的 fitz 索引 + 1」开始，往后取 8 页候选。下面的脚本对每页裁切页眉（顶部 15%）、页脚（底部 15%）和整页正文，输出 3 个 mini PDF：

```bash
cat > scripts/phase1_seek_chapter1.py << 'PYEOF'
import sys, fitz, os

pdf_path = sys.argv[1]
TOC_END = int(sys.argv[2])   # 目录最后那页的 fitz 索引（来自 Step 1.1）
output_dir = sys.argv[3]

doc = fitz.open(pdf_path)
os.makedirs(output_dir, exist_ok=True)

search_start = TOC_END + 1
search_end = min(doc.page_count - 1, search_start + 7)  # 往后取 8 页

for fitz_idx in range(search_start, search_end + 1):
    page = doc[fitz_idx]
    w, h = page.rect.width, page.rect.height
    
    # 页眉区域（顶部 15%）
    header = fitz.open()
    page.set_cropbox(fitz.Rect(0, 0, w, h * 0.15))
    header.insert_pdf(doc, from_page=fitz_idx, to_page=fitz_idx)
    header.save(f'{output_dir}/page{fitz_idx:04d}_header.pdf')
    
    # 页脚区域（底部 15%）
    footer = fitz.open()
    page.set_cropbox(fitz.Rect(0, h * 0.85, w, h))
    footer.insert_pdf(doc, from_page=fitz_idx, to_page=fitz_idx)
    footer.save(f'{output_dir}/page{fitz_idx:04d}_footer.pdf')
    
    # 整页正文
    page.set_cropbox(page.mediabox)
    full = fitz.open()
    full.insert_pdf(doc, from_page=fitz_idx, to_page=fitz_idx)
    full.save(f'{output_dir}/page{fitz_idx:04d}_full.pdf')
    
    print(f'page {fitz_idx}: header + footer + full PDF saved')

doc.close()
PYEOF

uv run python scripts/phase1_seek_chapter1.py <PDF_PATH> <TOC_END> temp_sample/
```

然后提交这些 mini PDF 到 API Job OCR（`scripts/phase3_submit_job.py`，代码见 Phase 3 Agent 派发模板）。AI 阅读 OCR 结果判断第 1 章首页：正文出现 "第1章"/"第一章" 或 TOC 中的章标题；页眉/页脚有印刷页码；或章节扉页特征（文字少、标题大字居中）。记录：第 1 章首页 fitz 索引 = N，印刷页码 = P。

---

**Step 1.2b — 计算偏移量，建立映射表**

找到第 1 章首页后：

```
偏移量 = (第1章首页的 fitz 索引 + 1) - 第1章的书页码
# 例：第1章首页 fitz=12，书页码 P1 → 偏移量 = 13 - 1 = 12
```

然后用偏移量计算所有章的 fitz 起止索引：

```
某章 fitz 起始 = 该章书页码 + 偏移量 - 1
某章 fitz 结束 = fitz 起始 + 该章页数 - 1
```

> ⚠️ **前提假设**：偏移量全书一致（即书页码和 PDF 页码的差值恒定）。如果书中间有插页（彩图、附录等），需要分段校准。

---

**Step 1.2c — 抽查验证（必须做）**

抽查 2-3 个位置验证偏移量：第 1 章中间页、第 3 章首页、最后一章中间页。OCR 页眉/页脚核对印刷页码是否与预期一致。全部匹配 → 偏移量正确。有不匹配 → 存在分段偏移（书中间有插页），需分段重新校准。

验证用 mini PDF（只裁页眉+页脚，不需要整页）：

```bash
cat > scripts/phase1_verify_offset.py << 'PYEOF'
import sys, fitz, os

pdf_path = sys.argv[1]
# 逗号分隔的 fitz 索引，如 "26,72,180"
check_fitz_indices = [int(x) for x in sys.argv[2].split(',')]
output_dir = sys.argv[3]

doc = fitz.open(pdf_path)
os.makedirs(output_dir, exist_ok=True)

for fitz_idx in check_fitz_indices:
    page = doc[fitz_idx]
    w, h = page.rect.width, page.rect.height
    
    # 页眉
    page.set_cropbox(fitz.Rect(0, 0, w, h * 0.15))
    hdr = fitz.open()
    hdr.insert_pdf(doc, from_page=fitz_idx, to_page=fitz_idx)
    hdr.save(f'{output_dir}/verify_{fitz_idx:04d}_header.pdf')
    
    # 页脚
    page.set_cropbox(fitz.Rect(0, h * 0.85, w, h))
    ftr = fitz.open()
    ftr.insert_pdf(doc, from_page=fitz_idx, to_page=fitz_idx)
    ftr.save(f'{output_dir}/verify_{fitz_idx:04d}_footer.pdf')
    
    page.set_cropbox(page.mediabox)

doc.close()
PYEOF

# 抽查点示例：第1章 P15→fitz=26, 第3章 P61→fitz=72
uv run python scripts/phase1_verify_offset.py <PDF_PATH> 26,72,180 temp_sample/
```

---

**建立最终映射表：**

```
偏移量 = 12（前言+目录共 12 页，经验证确认）

| 章节   | 书页码  | fitz索引(0-indexed) | 验证状态 |
|--------|---------|---------------------|----------|
| 前言   | i-x     | 0-9                 | -        |
| 目录   | -       | 10-11               | -        |
| 第1章  | P1-P30  | 12-41               | ✅ P15 验证通过 |
| 第2章  | P31-P60 | 42-71               | -        |
| 第3章  | P61-P90 | 72-101              | ✅ P61 验证通过 |
| ...    | ...     | ...                 | ✅ P180 验证通过 |
```

> ⚠️ **关键**：后续所有 fitz 操作使用「fitz索引」列。该列经过 TOC 锚点定位 + AI 判断首页 + 多点抽查验证，是可靠的。

#### Step 1.3 — 采样阅读，识别内容元素

取第 1 章的开头 3 页、中间 2-3 页、结尾 2-3 页作为样本，提交 API Job OCR：

```bash
cat > scripts/phase1_sample_pages.py << 'PYEOF'
import sys, fitz, os

pdf_path = sys.argv[1]
# 逗号分隔的 fitz 索引（从 Step 1.2 映射表的"fitz索引"列取）
fitz_indices = [int(x) for x in sys.argv[2].split(',')]
output_path = sys.argv[3]

doc = fitz.open(pdf_path)
os.makedirs(os.path.dirname(output_path) or '.', exist_ok=True)
sample = fitz.open()
for i in fitz_indices:
    sample.insert_pdf(doc, from_page=i, to_page=i)
sample.save(output_path)
print(f'Sample PDF: {sample.page_count} pages -> {output_path}')
PYEOF

# 开头/中间/结尾各 3 页（索引来自映射表的"fitz索引"列，本例第1章 fitz=12-41）
uv run python scripts/phase1_sample_pages.py <PDF_PATH> 12,13,14,26,27,28,39,40,41 temp_sample/sample_pages.pdf
```

提交 `temp_sample/sample_pages.pdf` 到 API Job（`scripts/phase3_submit_job.py`，代码见 Phase 3 Agent 派发模板）。从 OCR 结果中识别：

| 识别项 | 说明 |
|--------|------|
| **内容元素类型** | 定义、定理、公式、例题、习题、小结、边栏等 |
| **每类元素的特征** | 位置、特殊标记（图标、色块、编号） |
| **章节内部结构** | 各元素出现的顺序和层级 |
| **页面排版复杂度** | 单栏/双栏/混合、嵌入表格、文字环绕 |
| **特殊内容** | 大量图表、代码块、化学方程式等 |
| **页面类型分布** | 纯文字页 vs 图片页的比例 |

> ⚠️ 如果采样发现**多栏排版**或**复杂表格嵌入**，必须在 Phase 1 提问时明确告知用户：OCR 在复杂排版下可能产生阅读顺序混乱，建议用户做好手动校对的心理准备。

#### Step 1.4 — 向用户提问，确定通用方案

> ⛔ STOP — 采样完成。现在必须向用户逐一提问以下全部 6 类问题。禁止在用户回答之前编写方案文档或开始制作笔记。每类问题都必须等用户回复后才能问下一个。

Based on what you learned from sampling, ask the user a structured set of questions. Present each question with a **recommended default** based on what you observed.

**必须提问的内容（全部 6 类，缺一不可）：**

**C. 详略风格** — "详细精读笔记，还是简约速查笔记？"

- **详细解释版**：保留原书完整叙述，段落式呈现定义、定理、推导，知识体系以 `├─` / `└─` 连线符号树形图呈现，例题保留完整解答。适合初次学习。
- **简约呈现版**：高度提炼，AI 根据内容类型自动选用格式（概念对比→表格，要点→列表，知识体系→纯文本树形图（`├─` / `└─` 连线符号 + 缩进，不用 mermaid 也不用代码块），例题→只留思路和答案）。适合考前复习。

示例提问：
  ```
  笔记详略偏好：(A) 详细解释版 — 完整叙述，适合初学 / (B) 简约呈现版 — 提炼要点，格式自动匹配，适合复习
  我建议：[根据 PDF 类型推荐]
  ```

> ⚠️ 风格必须最先确定，因为后续的结构和元素处理方式都由风格基调推导。

**A+B. 笔记结构与元素处理方案** — 基于已选的风格，一次性给出完整方案。**不要重新列 Step 1.3 已识别的元素清单**，直接展示每个结构板块下各类元素怎么处理：

示例提问：
  ```
  基于详细解释版的风格，以下是我对每章的完整方案：

  1. 章节标题 + 页码范围（Pxx-Pxx）
  2. 考情/概述
  3. 知识结构图（`├─` / `└─` 连线符号 + 缩进表示层级，含具体概念名称及知识点关系，不用代码块）
  4. 知识点梳理（按知识点分节）
     - 定义      → 完整摘录原文
     - 定理/定律 → 完整摘录 + 推导过程
     - 公式图表   → 保留，公式用 LaTeX
  5. 例题精讲
     - 例题      → 完整题目 + 详细解答步骤
  6. 习题精练
     - 思考题    → 仅列题目
  7. 核心结论速查
     - 本章小结  → 保留原文

  结构、每类元素的处理方式都在这了。有需要调整的吗？
  ```

**D. 公式/特殊符号处理规范** — "数学公式、化学式等用什么格式？"
- LaTeX 还是其他？行内/行间公式的写法？特殊符号约定？

**E. 文件命名与输出目录** — "笔记文件存哪里？怎么命名？"

**F. 输出格式** — "最终产出什么格式？"
- 仅 Markdown（`.md` 文件）
- Markdown + PDF（使用 Pandoc + XeLaTeX 渲染）
  - 如果输出 PDF：整个一份 PDF 还是各章节单独 PDF？
  - 是否需要 PDF 目录？Pandoc `--toc` 参数可自动生成 PDF 书签目录
  - 需要安装 pandoc 和 texlive（或 MiKTeX）

> 📌 如果用户选择输出 PDF，在方案文档中注明 PDF 生成方式。Phase 4 中执行 PDF 导出。

**G. 其他约定** — 根据文档类型可能有额外问题：
- 图表怎么处理（描述还是跳过）？
- 其他学科特有的约定（如化学方程式、代码块等）

#### Step 1.5 — 编写通用方案文档

用户回答后，编写一份适用于所有章节的通用方案文档：

**File:** `notes-plan.md`（或用户指定名称），必须包含：
1. **页码范围映射表** — 每章 PDF 页码和书页码
2. **页面类型标记** — 图片页/文字页（混合 PDF）
3. **详略风格** — 详细解释版 / 简约呈现版
4. **笔记结构与元素处理方案** — 结构板块 + 每类元素处理方式（来自 C + A+B 的答案）
5. **格式规范** — 写作时必须遵守的格式约定：
   - **格式由内容类型决定**：对比关系→表格，并列要点→列表，层级体系→树形图（纯文本 `├─`/`└─` 连线符号，不用代码块），定义/推导→段落
   - **分隔符规则**：页面间内容合并用两个换行分隔，禁止 `---` 水平线
   - **树形图/结构图**：纯文本缩进 + 连线符号模拟树状结构。同级最后一项用 `└─`，其余用 `├─`，父级延续用 `│`。**绝对不要用代码块包裹**（会阻止 LaTeX 公式渲染）。示例：
     ```
     - ├─ 力学基础
     - │  ├─ 牛顿第一定律（$F=0$ 时物体保持静止或匀速直线运动）
     - │  └─ 牛顿第二定律（$F=ma$）
     - └─ 运动学
     ```
   - **公式/特殊符号**：LaTeX 写法、行内/行间约定
   - **交叉引用**：原文中"见第X章"等引用如何处理
   - **页码标注**：是否保留原书页码
   - **空行规则**：列表、代码块、表格等块级元素前必须加空行，否则 Pandoc 转 PDF 后格式丢失变为正文段落
6. **文件命名规则** — 输出目录和文件命名
7. **输出格式** — 仅 MD / MD+PDF（Pandoc + XeLaTeX）
8. **执行节奏** — 全量/分批/逐章（Phase 3 前确认）

> ⛔ STOP — 通用方案文档已生成，但禁止继续执行。必须向用户展示 notes-plan.md 的全部内容，等待用户明确说"批准"/"可以"/"继续"之后，才能进入 Phase 2。用户说"看看"或"嗯"不算批准。

---

### Phase 2: Pilot 章节

> ⛔ 只有 Phase 1 方案获得用户明确批准后，才能进入 Phase 2。禁止跳过 Phase 1 直接执行 OCR。

#### Step 2.1 — 执行第一章

严格按通用方案执行第一章的完整流程：

1. **按章切 PDF** — 用 Phase 3 的 fitz 命令提取第 1 章为 `temp_ch01/ch01.pdf`（含 overlap）
2. **提交 API Job** — 按 Phase 3 Agent 派发模板派发后台 Agent
3. **整合内容** — 等待 Agent 完成，从 `temp_ch01/chapter_ocr.md` 读取，剔除 overlap 页，按 notes-plan.md 的结构（第 4 项）和格式规范（第 5 项）整理

文字型 PDF 直接用 Read 工具逐页读取即可。

#### Step 2.2 — 展示并收集反馈

完成第一章笔记后，向用户展示并提问：

```
第一章笔记已完成 → [文件路径]
请检查：结构、内容完整性、公式/格式。需要调整的地方请告诉我。
```

> ⛔ STOP — 必须等用户反馈后再决定下一步。用户说"继续"之前，禁止开始 Phase 3。

#### Step 2.3 — 调整方案

根据用户反馈就地更新 `notes-plan.md`，末尾追加**修订记录**：

```markdown
## 修订记录
- 2026-XX-XX：Pilot 第1章后调整 — [具体修改内容]
```

---

### Phase 3: 批量制作

> ⛔ 只有 Phase 2 Pilot 通过用户验收、方案修订完成后，才能进入 Phase 3。

#### Step 3.1 — 确认执行节奏

> ⛔ STOP — 必须向用户提问执行节奏，获得明确选择（A/B/C）后才能派发 Agent。禁止默认选 A 直接开始。

剩余 X 章需要制作。你希望：
- (A) 一次性全部并行制作（最快）
- (B) 分批制作（推荐每批 N 章，中间可以抽查）
- (C) 逐章制作（最稳，每章做完可微调）

Update the plan document with the chosen strategy.

#### Step 3.2 — 执行批量制作

每章流程与 Pilot 相同：

```
1. ⚠️ 清理章节标题中的非法文件名字符（: / \ * ? " < > | → 替换为 -）
2. 混合 PDF：文字页直接 fitz.get_text() 读取，图片页走 API OCR
3. 按章切 PDF（含 overlap，处理页面旋转校正）
4. 所有章同时派发后台 Agent → 提交 API Job → 轮询 → 下载 JSONL → 解析 markdown → Write 到 temp_chXX/chapter_ocr.md
5. 等待所有 Agent 完成（失败恢复见下方）
6. Read 每个 chapter_ocr.md（缺失/空 → 用 Agent 返回消息 fallback → 仍无则重试）
7. 剔除 overlap 页（JSONL 头尾），按页码组装，检查跨页元素完整性
8. 写入章节笔记文件（严格按 notes-plan.md 第 4、5 项的结构和格式规范）
9. 可选：删除该章临时 PDF（分批清理）
```

**fitz 按章切 PDF 命令：**

> ⚠️ **页码来源**：`chap_start` 和 `chap_end` 必须直接从 Phase 1 Step 1.2 **映射表的「fitz索引(0-indexed)」列**取值，该列已通过页脚 OCR 校准了前言偏移量。**禁止**用 PDF 阅读器页码、书页码、或任何估算值。

例如映射表中第3章 fitz索引 = 72-101（偏移量 12，书页码 P61-P90）：

```bash
cat > scripts/phase2_split_chapter.py << 'PYEOF'
import sys, fitz

pdf_path = sys.argv[1]
chap_start = int(sys.argv[2])  # 映射表 fitz索引(0-indexed) 列
chap_end   = int(sys.argv[3])  # 映射表 fitz索引(0-indexed) 列
output_path = sys.argv[4]

doc = fitz.open(pdf_path)
start_overlap = max(0, chap_start - 1)
end_overlap = min(doc.page_count - 1, chap_end + 1)
chapter = fitz.open()
chapter.insert_pdf(doc, from_page=start_overlap, to_page=end_overlap)
chapter.save(output_path)
print(f'{output_path}: {chapter.page_count} pages (含 overlap)')
PYEOF

# ⚠️ 以下数字来自 Phase 1 映射表的「fitz索引(0-indexed)」列，已含偏移量校准
uv run python scripts/phase2_split_chapter.py <PDF_PATH> 72 101 temp_ch03/ch03.pdf
```

**Overlap：** 每章 PDF 含 ±1 页 overlap。合并时剔除 JSONL 头尾各 1 页。Overlap 仅让 API 看到跨章上下文，防止截断。

**大章拆分（>50 页或 >50MB）：**
如果单章太大，按 ~30 页拆成子批，每批一个 Job。例如第 3 章拆为 `temp_ch03/batch_01/`、`temp_ch03/batch_02/`，每批独立提交 API Job，合并时按顺序拼接。

**Agent dispatch template** (dispatch ALL agents simultaneously):

```
Agent(
  description: "OCR 第X章 API Job"
  subagent_type: "general-purpose"
  run_in_background: true
  prompt: """
提交 PDF 到 PaddleOCR API Job，轮询直到完成，解析 JSONL，保存 OCR 结果和图片到磁盘。

Step 1 — 用 Read 工具读 .env 获取 PADDLEOCR_AISTUDIO_ACCESS_TOKEN。

Step 2 — 落盘并运行 API Job 脚本：

```bash
mkdir -p scripts

cat > scripts/phase3_submit_job.py << 'PYEOF'
import json, os, requests, time, sys

# Read token from .env (NEVER hardcode the real token here)
with open('.env') as f:
    for line in f:
        line = line.strip()
        if line and not line.startswith('#') and '=' in line:
            key, val = line.split('=', 1)
            os.environ.setdefault(key.strip(), val.strip())
TOKEN = os.environ['PADDLEOCR_AISTUDIO_ACCESS_TOKEN']

JOB_URL = 'https://paddleocr.aistudio-app.com/api/v2/ocr/jobs'
MODEL = 'PaddleOCR-VL-1.6'
PDF_PATH = sys.argv[1]
OUTPUT_MD = sys.argv[2]
OUTPUT_IMG_DIR = sys.argv[3]

headers = {'Authorization': f'bearer {TOKEN}'}
optional_payload = {
    'useDocOrientationClassify': False,
    'useDocUnwarping': False,
    'useChartRecognition': False,
}

# === Submit job ===
print(f'Submitting: {PDF_PATH}')
if not os.path.exists(PDF_PATH):
    print(f'Error: File not found at {PDF_PATH}')
    sys.exit(1)

data = {'model': MODEL, 'optionalPayload': json.dumps(optional_payload)}
with open(PDF_PATH, 'rb') as f:
    resp = requests.post(JOB_URL, headers=headers, data=data, files={'file': f})

print(f'Response status: {resp.status_code}')
if resp.status_code != 200:
    print(f'Error response: {resp.text}')
assert resp.status_code == 200, f'Job submit failed: {resp.text}'

job_id = resp.json()['data']['jobId']
print(f'Job submitted. job_id={job_id}')

# === Poll until done ===
while True:
    resp = requests.get(f'{JOB_URL}/{job_id}', headers=headers)
    assert resp.status_code == 200, f'Poll failed: {resp.text}'
    state = resp.json()['data']['state']
    if state == 'pending':
        print('State: pending...')
    elif state == 'running':
        try:
            tp = resp.json()['data']['extractProgress']['totalPages']
            ep = resp.json()['data']['extractProgress']['extractedPages']
            print(f'State: running, {ep}/{tp} pages')
        except KeyError:
            print('State: running...')
    elif state == 'done':
        ep = resp.json()['data']['extractProgress']['extractedPages']
        st = resp.json()['data']['extractProgress']['startTime']
        et = resp.json()['data']['extractProgress']['endTime']
        print(f'Job done: {ep} pages, {st} ~ {et}')
        jsonl_url = resp.json()['data']['resultUrl']['jsonUrl']
        break
    elif state == 'failed':
        err = resp.json()['data']['errorMsg']
        print(f'Job failed: {err}')
        sys.exit(1)
    time.sleep(5)

# === Download JSONL and parse ===
jsonl_text = requests.get(jsonl_url).text
os.makedirs(OUTPUT_IMG_DIR, exist_ok=True)

with open(OUTPUT_MD, 'w', encoding='utf-8') as out:
    for line in jsonl_text.strip().split('\n'):
        line = line.strip()
        if not line:
            continue
        result = json.loads(line)['result']
        for res in result['layoutParsingResults']:
            # Write markdown (pages separated by double newline, NO ---)
            out.write(res['markdown']['text'])
            out.write('\n\n')
            # Download embedded images
            for img_path, img_url in res['markdown']['images'].items():
                full_img_path = os.path.join(OUTPUT_IMG_DIR, img_path)
                os.makedirs(os.path.dirname(full_img_path), exist_ok=True)
                with open(full_img_path, 'wb') as img_file:
                    img_file.write(requests.get(img_url).content)
                print(f'Image saved: {full_img_path}')
            # Download output images (charts, diagrams)
            for img_name, img_url in res.get('outputImages', {}).items():
                filename = os.path.join(OUTPUT_IMG_DIR, f'{img_name}.jpg')
                img_resp = requests.get(img_url)
                if img_resp.status_code == 200:
                    with open(filename, 'wb') as f:
                        f.write(img_resp.content)
                    print(f'Image saved: {filename}')

print(f'OCR done -> {OUTPUT_MD}')
PYEOF

uv run python scripts/phase3_submit_job.py temp_chXX/chXX.pdf temp_chXX/chapter_ocr.md temp_chXX/images/
```

Step 3 — 验证：Read temp_chXX/chapter_ocr.md 确认非空。空/缺失 = 失败，报告错误。

返回消息包含：章节号、页数、输出文件路径、下载图片数。
"""
)
```

每个 Agent 产出 `temp_chXX/chapter_ocr.md` + `temp_chXX/images/`。页之间用两个换行分隔，不使用 `---` 水平线。

**关键规则：**
- 所有后台 Agent 在同一消息中派发（全部并行）
- 所有 Agent 完成后才开始合并
- 单章 >50 页或 >50MB → 切成 ~30 页子批

**失败恢复：**

Agent 失败（超时/限流/崩溃）时：
1. 记录章节号和错误信息（提交失败 / Job failed / JSONL 空）
2. 等待 30 秒冷却，仅重派失败章节（相同 overlap 规则，会覆盖旧的 chapter_ocr.md）
3. 不重派已成功的章节（chapter_ocr.md 已存在）
4. 重试完成后确认 chapter_ocr.md 存在且非空，再继续合并
5. 同一章失败 3 次 → 拆成 ~15 页子批重试，或手动排查

**进度报告：**
> ✅ 第X章完成 → `讲义/第X讲 xxx.md`（Pxxx-Pxxx，N页）

#### Step 3.3 — 抽查（仅分批/逐章模式）

每批完成后询问用户是否需要抽查。

---

### Phase 4: 收尾

> ⛔ 所有章节 OCR 和合并完成后，才能进入 Phase 4。逐项向用户确认后再执行。

#### Step 4.1 — 附录处理

询问用户是否需要提取附录。如需要，用相同流程，结构可简化。

#### Step 4.2 — 生成总目录索引（可选）

如需要，在输出目录生成 `README.md`，包含各章节文件链接、页码范围、交叉引用。

#### Step 4.3 — PDF 导出（Phase 1 选择输出 PDF 时）

先创建 LaTeX 导言区文件（控制字体、数学、列表紧凑度）：

```bash
cat > pandoc-header.tex << 'TEXEOF'
\usepackage{xeCJK}
\setCJKmainfont{SimSun}
\setCJKsansfont{Microsoft YaHei}
\setCJKmonofont{FangSong}
\usepackage{unicode-math}
\setmathfont{Latin Modern Math}
\usepackage{amsmath}
\usepackage{amssymb}
\usepackage{enumitem}
\setlist[itemize,1]{itemsep=0.15em,topsep=0.15em,parsep=0.05em}
\setlist[itemize,2]{itemsep=0.1em,topsep=0.1em,parsep=0.03em}
\setlist[enumerate,1]{itemsep=0.15em,topsep=0.15em,parsep=0.05em}
\setlist[enumerate,2]{itemsep=0.1em,topsep=0.1em,parsep=0.03em}
\setlength{\parskip}{0.1em}
TEXEOF
```

**单章 → PDF：**
```bash
pandoc "讲义/第X章-标题.md" -o "讲义/第X章-标题.pdf" \
  --pdf-engine=xelatex --include-in-header=pandoc-header.tex
```

**合并所有章 → 一个 PDF：**
（必须按章节号显式排序，避免 glob 字典序导致"第10章"排在"第2章"前）

```bash
# 按章节号升序拼接为中间文件
cat 讲义/第1章*.md 讲义/第2章*.md 讲义/第3章*.md ... > 讲义/完整笔记.md
# 中间文件 → PDF
pandoc 讲义/完整笔记.md -o 讲义/完整笔记.pdf \
  --pdf-engine=xelatex --include-in-header=pandoc-header.tex
```

> 如果用户需要目录，单章和合并命令都加 `--toc`。
> 需要安装 texlive（或 MiKTeX）和 pandoc。

#### Step 4.4 — 清理临时文件

```bash
rm -rf temp_ch*/ temp_sample/ temp_lec*/ scripts/ temp_setup_test.pdf
```

#### Step 4.5 — Git 提交（可选）

询问用户是否提交输出文件。

## 常见错误与纠正

| ❌ 错误做法 / 危险想法 | ✅ 正确做法 |
|---------|---------|
| 用 MCP 逐页调而非 API Job | 统一使用 API Job 提交 PDF，不转 PNG，不调 MCP |
| fitz 拆 PNG 而非直接切 PDF | fitz 输出格式是 PDF（insert_pdf + save），不调 get_pixmap |
| AI 现场生成 Python 脚本或改写代码逻辑 | 使用 SKILL.md 固定脚本：cat > scripts/phaseN_xxx.py 原样落盘，AI 只替换 CLI 参数，不改脚本内容 |
| Token 硬编码在 Python 里 | 从 `.env` 动态读取 |
| JSONL 页之间用 `---` 分隔 | 用两个换行分隔，禁用水平线 |
| 合并时未剔除 overlap 页 | API 返回的 JSONL 头尾各 1 页是 overlap，合并时剔除 |
| 切章 PDF 忘记 overlap 页 | fitz insert_pdf 时范围扩展 ±1 页（首章无前 overlap，末章无后 overlap）|
| Job 失败整章重来 3 次仍失败 | 拆成 ~15 页子批重试，或手动排查该章 |
| PDF 导出不用 Pandoc，或用 glob 通配符合并 PDF | 用 pandoc --pdf-engine=xelatex --include-in-header=pandoc-header.tex，合并时按章节号显式排序（cat 第1章*.md 第2章*.md ...），按需加 --toc |
| 通篇套用同一种格式（全段落或全列表） | 格式由内容类型决定：对比→表格，要点→列表，体系→树形图，定义→段落 |
| AI 跳过 Phase 1 提问直接干活 | 看到 ⛔ STOP 标记必须停下来问用户 |
| 混淆 fitz 0-indexed 和 PDF 阅读器 1-indexed | fitz page 0 = PDF 阅读器第 1 页 |
| Agent 未全部完成就开始合并 | 等所有 Agent 通知后再整合 |
| 发送敏感 PDF 到云端 OCR 未警告用户 | Phase 0 必须先问隐私问题 |
| 硬编码 Windows 路径 | 使用正斜杠绝对路径（`C:/Users/.../file.png`）|
| 跳过方案直接干活 / "PDF 简单不用采样" / "我自己决定格式" | Plan first, extract later. 必须采样、必须提问、必须等用户批准方案后才开始做笔记 |
| 每章单独写方案而非一个通用方案 | 一份 notes-plan.md 覆盖所有章节 |
| PDF 没有目录就放弃或猜测章节边界 | 用标题采样 OCR + 页码重置检测自动探测 |
| 混合 PDF 把所有页当图片处理 | 检测文字/图片页，文字页直接读，只 OCR 图片页 |
| 未处理页面旋转 | 提取前检查 `page.rotation`，非零则校正 |
| 章节标题含非法文件名字符导致保存失败 | 替换 `<>:\"/\\|?*` → `-` |
| API 配额中途用完导致全部丢失 | Phase 0 先做单页 OCR 测试；监控 402；保留已完成结果 |
| 仅依赖 Agent 返回消息获取 OCR 结果 | Agent 必须通过 Write 工具写入 temp_chXX/chapter_ocr.md |
| 忽略 chapter_ocr.md：中断恢复没检查，或合并时没用 Read 工具读取 | 恢复时：chapter_ocr.md 已存在 → OCR 已完成，跳过。合并时：必须 Read 每个 chapter_ocr.md（Agent 返回消息仅做 fallback）|
| 有旧进度却重新开始 | 检查 temp 目录/输出文件，询问用户：恢复还是重来？ |
| 没问输出格式 | 必须问：仅 MD 还是 MD+PDF（Pandoc + XeLaTeX）|

---

## Agent Prompt Checklist

Every OCR agent you dispatch MUST be told:
- [ ] 从 `.env` 读取 token（代码固定，不硬编码）
- [ ] 使用固定脚本：cat > scripts/phaseN_xxx.py 原样落盘，不自己写代码
- [ ] 只替换 CLI 参数（文件路径），不修改脚本内容
- [ ] 页之间用两个换行分隔，不使用 `---` 水平线
- [ ] 下载 JSONL 中的图片到 temp_chXX/images/
- [ ] 结果写入 temp_chXX/chapter_ocr.md（固定路径）
- [ ] 返回消息包含：章节号、页数、文件路径、下载图片数
- [ ] Job 失败时报告具体 errorMsg
