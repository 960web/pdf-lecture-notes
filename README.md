# pdf-lecture-notes

> Claude Code Skill：将图片型 PDF（教材、讲义、扫描文档）提取为结构化 Markdown 笔记。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](./LICENSE)
[![Skill Format](https://img.shields.io/badge/Skill%20Spec-agentskills.io-blue)](https://agentskills.io/specification)
[![npm](https://img.shields.io/npm/v/@960web/pdf-lecture-notes)](https://www.npmjs.com/package/@960web/pdf-lecture-notes)
[![skills.sh](https://img.shields.io/badge/skills.sh-install-black)](https://skills.sh)

## 这是什么

一个 Claude Code Skill，自动化完成 **PDF → Markdown 笔记** 的完整流程：

- 分析 PDF 目录结构，建立页码映射
- 采样识别内容元素（定义/定理/例题/习题等）
- 与用户交互制定通用笔记方案
- 并行 OCR（PaddleOCR-VL + 后台 Agent）高速提取
- 生成结构化 Markdown 笔记

## 适用场景

- 扫描版/图片型 PDF 教材或讲义
- PDF 过大无法直接读取（>100MB）
- 包含公式、表格等需要保留格式的内容
- 多章节文档需要批量处理

## 安装

### 方式一：skills.sh（推荐，跨平台通用）

```bash
npx skills add 960web/pdf-lecture-notes
```

### 方式二：npm

```bash
npm install -g @960web/pdf-lecture-notes
```

### 方式三：OSM（Open Skills Manager）

```bash
osm install 960web/pdf-lecture-notes
```

### 方式四：手动安装

```bash
# 安装到用户级（所有项目可用）
mkdir -p ~/.claude/skills/pdf-lecture-notes
cp skills/pdf-lecture-notes/SKILL.md ~/.claude/skills/pdf-lecture-notes/

# 或安装到项目级（仅当前项目）
mkdir -p .claude/skills/pdf-lecture-notes
cp skills/pdf-lecture-notes/SKILL.md .claude/skills/pdf-lecture-notes/
```

### 方式五：Cursor

Settings → Rules → 导入 GitHub 仓库 URL：
```
https://github.com/960web/pdf-lecture-notes
```

## 前置依赖

| 依赖 | 用途 | 安装 |
|------|------|------|
| **uv** | Python 包管理器 | `pip install uv` 或参见 [uv 文档](https://docs.astral.sh/uv/) |
| **PyMuPDF (fitz)** | PDF 按章切分、页面分析 | `uv add pymupdf` |
| **PaddleOCR-VL API** | OCR 识别（百度 AI Studio 云端） | 需注册 [百度 AI Studio](https://aistudio.baidu.com/) 获取 Access Token |
| **百度 AI Studio** | OCR 云端服务 | 注册后将 token 写入 `.env` 文件 |

## 快速开始

1. 在 Claude Code 中打开你的项目
2. 说：*"把这个 PDF 转成笔记"*
3. 按 Skill 引导完成：环境配置 → 方案制定 → Pilot 章节 → 批量制作

## 目录结构

```
pdf-lecture-notes/
├── skills/
│   └── pdf-lecture-notes/
│       └── SKILL.md        # 核心 skill 文件（skills.sh 格式）
├── SKILL.md                # 核心 skill 文件（根目录入口，兼容 OSM 等注册中心）
├── package.json            # npm 包清单
├── README.md               # 本文件
├── LICENSE                 # MIT
├── CONTRIBUTING.md         # 贡献指南
├── CHANGELOG.md            # 版本记录
└── .github/                # Issue/PR 模板
```

## 分发渠道

| 渠道 | 安装命令 |
|------|----------|
| **skills.sh** | `npx skills add 960web/pdf-lecture-notes` |
| **npm** | `npm install @960web/pdf-lecture-notes` |
| **OSM** | `osm install 960web/pdf-lecture-notes` |
| **SkillsMP** | 自动收录 |
| **SkillsGate** | 自动收录 |
| **GitHub** | `git clone` + 手动安装 |

## 贡献

欢迎提 Issue 和 PR。详见 [CONTRIBUTING.md](./CONTRIBUTING.md)。

## 许可

[MIT](./LICENSE)
