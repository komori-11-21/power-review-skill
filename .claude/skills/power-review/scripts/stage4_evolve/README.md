# `scripts/stage4_evolve/`：阶段四（报告归档与双文档自我迭代助手）

## 主要实现什么
负责在阶段四人工确认或复盘后，安全规范地把新沉淀的知识追加回 `references/` 活文档，保证越用越聪明、同时不破坏既有文档格式（不依赖真库结构）。

## 计划包含的能力
1. **新案例卡片校验与归档**：
   - 校验 AI 生成或用户整理的新卡片是否具备核心字段（`① DUBAI 怎么看`、`② 日志怎么看`、`③ 结论与转交团队`），规范追加到 [case_knowledge_base.md](../../references/case_knowledge_base.md)。
2. **通用方法论与分支规则补充**：
   - 将日常复盘确认的新思路条款或阈值/误判边界，按标准条目格式追加到 [general_guide.md](../../references/general_guide.md) 沉淀区（或对应 `screen_off_rules.md` / `screen_on_rules.md` 补充区）。
3. **防重复/安全写入检查**：
   - 写入前校验不重复覆盖已有案例，避免多次追加弄乱原文档结构。
