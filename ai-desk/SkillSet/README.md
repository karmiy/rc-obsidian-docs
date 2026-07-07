# Skill Set Management

本目录记录 AI Desk Skill Set Management 的产品、技术和实现方案。

正式需求以 `specs/` 下的文件为准。旧的汇总草稿已删除，避免后续维护时出现多份文档 drift。

## 阅读顺序

1. `specs/01-product-and-ui.md`：产品交互、页面结构、preview dialog、usage stats。
2. `specs/02-data-model-and-api.md`：数据模型、DB 表、后端 API、delete/sync/save 语义。
3. `specs/03-import-and-versioning.md`：import/sync pipeline、Claude Code sandbox、preview-first、version bump 规则。
4. `specs/04-mcp-and-runtime.md`：`get_platform_entities`、runtime materialization、`manifest.json`、`agentInstructions`。
5. `specs/05-composer-integration.md`：`@Superpowers`、`/Superpowers.Brainstorming`、SlashPromptComposer 集成。
6. `specs/06-implementation-plan.md`：开发拆分、测试点、验收矩阵。

## 目录说明

- `specs/`：唯一正式需求与实现方案。
- `design/`：设计稿与历史 mockup，仅用于辅助理解；如与 `specs/` 冲突，以 `specs/` 为准。
- `solution-draw.md`：讨论过程中的草图/思路记录，仅作背景参考。
