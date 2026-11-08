# 【待审异常切片 #{index} 概览】（本分支共检出 {total_targets} 个异常段，逐段独立分析）
- **异常判定依据**：`{abnormal_reasons}`
- **当前异常段 (Target)**：`{target_start} ~ {target_end}` ({branch_label} `{target_hours}h`) | 电池真实掉电：`{target_mah}` -> 真实均流：**`{target_i_real}`** (掉电率 `{target_drop_rate}%/h`)
- **同库最低段参考 (Ref)**：`{ref_summary_or_无}`
  *(注：若全库仅 1 段则 Ref 显示为“无”；若 Ref 均流同样偏高，说明整份数据全程持续漏电，请直接看第 ① 列实测占比)*

> [槽位跳过提示] 若真实 `.db` 未包含某维度表，在此输出跳过说明与上位兜底/线下 `hiperf` 补抓提示，不中断流水线。

## {section_num}. [{dim_id}] {dim_title}
> {slot_note_e.g._coverage_or_multiwindow_tag}

| 排名与实体名称 (实测大户 ∪ 突增Top3) | ① 当前异常段实测 (Target) | ② 同库最低段参考 (Ref) | ③ 对比变化 (Target - Ref) |
| :--- | :--- | :--- | :--- |
| **`{top_entity}`** | **`[{metric_tag}] {target_val}`** | `{ref_val}` | **`{diff_val}`** |
| **`[突增入选] {surge_entity}`** | **`[{metric_tag}] {target_val}`** | `{ref_val}` | **`{diff_val}`** |
| **Others (其余 {n} 项合计)** | `[{metric_tag}] {others_target}` | `{others_ref}` | `{others_diff}` |

*(注：若某段没采到该项数据请填 `无数据`，不要写 `0`；仅靠电量 `%` 反推的电流请标 `[估算值]`)*
