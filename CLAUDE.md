# pdf-lecture-notes 项目说明

这是一个 Claude Code Skill 的开源项目，用于将图片型 PDF 提取为结构化 Markdown 笔记。

## 核心文件

- `SKILL.md` — Skill 定义文件（根目录入口，兼容 OSM 等注册中心）
- `skills/pdf-lecture-notes/SKILL.md` — Skill 定义文件（skills.sh 格式）
- `package.json` — npm 包清单
- `README.md` — 项目说明和安装指南
- `CONTRIBUTING.md` — 贡献指南

## 开发约定

- 使用中文撰写文档（SKILL.md 使用中文）
- 保持通用性：不针对特定学科
- 路径使用平台无关格式
- 使用 `uv run python` 运行 Python 代码（不直接使用 python/python3）
