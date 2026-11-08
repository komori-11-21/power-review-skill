# `tests/integration/`：端到端全链路集成验证

## 主要实现什么
把 `stage1_db` $\rightarrow$ `stage2_evidence` $\rightarrow$ `stage3_log` $\rightarrow$ `stage4_evolve` 串成完整闭环，每次修改脚本或活文档规范后一键跑通，确保整体不回归。

## 计划验证的核心链路
1. **取证主流程端到端**：传入脱敏真实 `.db`，验证输出包含所有 `Target` 切片、完整 4 列对比表、缺表兜底/线下 `hiperf` 提示及召回卡片；
2. **日志 CLI 全命令跑通**：对配套脱敏日志分别跑 `diff`、`pid-trace`、`search`、`context` 四个动作，验证返回结构合法且能精准定位植入的已知异常日志；
3. **活文档安全追加演练**：在临时目录模拟向 [case_knowledge_base.md](../../references/case_knowledge_base.md) 与 [general_guide.md](../../references/general_guide.md) 归档新条目，验证不破坏原文档结构、不重复写入。
