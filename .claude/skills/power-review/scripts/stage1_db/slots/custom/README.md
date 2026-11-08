# `scripts/stage1_db/slots/custom/`：内部专有新表万能扩展槽位

## 主要实现什么
用于对接真实 `.db` 中存在、但不在上面灭/亮屏预设维度中的公司内部专有新表（例如 NPU 功耗表、星闪 NearLink 统计、软总线唤醒表等），无需改动主流程即可输出统一的 4 列对比表。

## 3 种万能物理形态转换逻辑（自适应扩展规范）
1. **模拟能量型**（含 energy / power / mAs / mAh）：计算时间窗内各实体占该表总能量的相对占比 `Share %`，接入双榜合并；
2. **真实时长型**（含 duration / time / run_time / ms）：统一换算秒后除以切片总时长 $\Delta t$，输出物理时长占比 `%`；
3. **真实频次/极值型**（含 count / times / packets / bytes / temp / level）：计算 `次/h` 频次或时间窗内 `AVG / MAX`。
