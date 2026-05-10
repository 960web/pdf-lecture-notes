# 贡献指南

感谢你考虑为 `pdf-lecture-notes` 贡献代码或建议！

## 如何贡献

### 报告 Bug

1. 使用 [Bug Report](./.github/ISSUE_TEMPLATE/bug_report.md) 模板
2. 描述：做了什么、期望什么、实际发生了什么
3. 附上环境信息：OS、Claude Code 版本、PDF 类型

### 功能建议

1. 使用 [Feature Request](./.github/ISSUE_TEMPLATE/feature_request.md) 模板
2. 描述使用场景和期望效果
3. 如果是新增边界情况处理，说明触发条件

### Pull Request 流程

1. Fork 本仓库
2. 创建分支：`git checkout -b feat/your-feature`
3. 修改 `SKILL.md`
4. **测试**：用真实 PDF 跑一次完整流程，确认改动生效
5. 提交：`git commit -m "feat: 描述改动"`
6. 推送并创建 PR

### SKILL.md 修改规范

- 保持通用性：不针对特定学科或文档类型
- 遵循 [agentskills.io 规范](https://agentskills.io/specification)
- `description` 字段聚焦触发条件，不总结工作流
- 新增内容考虑放到对应 Phase 章节
- 同步更新 `CHANGELOG.md`

### 代码风格

- Markdown 使用中文（README 除外，中英双语）
- Python 代码示例使用 `uv run python`
- 路径使用平台无关格式（`/absolute/path/`）
- 不在 skill 中硬编码任何 API key 或 token
