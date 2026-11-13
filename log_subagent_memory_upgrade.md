# 阶段二~四实战升级方案：轻量日志漏斗 + 子代理探索 + SQLite 双层知识库

> **文档定位**：承接你内网**已实现的阶段一（`.db` 切片与 4 列对比表）**，专门解决后续查日志与知识沉淀环节的三大实战痛点：
> 1. **防盲搜与防硬编**：几百 MB 原始日志不再让子代理大海捞针，先过一层**“时间覆盖止损 + 异常窗切片 + Top 10 暴增 Tag”薄漏斗**（日志被冲掉立刻停手不瞎编，日志还在则缩小成几 MB 小文件交给子代理自由推理）；
> 2. **防同模块案例互相覆盖 & 防错题本自污染**：SQLite 拆为 **“客观查证地图（`1对1` 自动对比更正）” + “具体故障死法库（`1对多` 候选隔离，人工确认转正）”** 两张表；
> 3. **防死板匹配查不出**：采用 **“词根分词模糊匹配 + 全库轻量目录兜底”**，解决 `.db` 名字与日志名字对不上的失忆问题。

---

## 一、 整体衔接链路（阶段一不动，后三步彻底跑实）

```text
【阶段一：你已实现，保持不动】
  输入 .db  ──▶ 输出：异常时间段 [T_start, T_end]、参照段 [Ref_start, Ref_end]、4 列对比表头号嫌犯
        │
        ▼
【阶段二：SQLite 双层召回（memory_store.py recall）】
  ├─ 词根模糊匹配：拆解嫌犯名词根，查出表 1《模块客观查证地图》+ 表 2《该模块历史故障案例列表》
  └─ 轻量目录兜底：附带全库已存模块一句话目录（防名称缩写不一致漏召回，主模型可点名补调）
        │
        ▼
【阶段三：薄漏斗预检 + 子代理自由破案】
  ├─ 第一步：跑轻量漏斗脚本（log_funnel.py）
  │    ├─ 若日志最早时间 > T_end（已被环形缓冲区冲掉）──▶ 立即止损！不启子代理，直接按 .db 物理指标给疑似结论
  │    └─ 若日志覆盖该时段 ──▶ 切出 [T_start - 10min, T_end] 小文件 + 算出比 Ref 暴增的 Top 10 Tag/PID 榜单
  │
  └─ 第二步：派发独立【日志调查子代理】（在小切片文件内作业，不撑爆主窗口）
       ├─ ① 按表 1《查证地图》跳过已知噪音词，搜真实 Tag 与追上游跳板词，核对是否命中表 2 老案例
       ├─ ② 未命中老案例时，顺着 Top 10 暴增 Tag/PID 用 Grep/Read 自由展开上下文推理根因
       └─ ③ 去 Ref 正常段核验同关键字频次（排除常驻背景打印），只读回传「实锤日志结论 + 分层写库提案」
        │
        ▼
【阶段四：分层安全写回 SQLite（memory_store.py）】
  ├─ 表 1《模块查证地图》（客观 Tag / 跳板词 / 实锤背景噪音词）──▶ 主模型三问自检通过后【自动写入或对比更正】（带快照可回滚）
  ├─ 表 2《模块故障死法》（新发现的具体 Bug 因果）──▶ 一律存为【candidate 待确认候选】（绝不覆盖同模块旧案例，人工确认后转 confirmed）
  └─ 自动刷新 references/log_memory_snapshot.md 可读总览表（方便肉眼检查与 git diff）
```

---

## 二、 SQLite 双层知识库设计（`power_memory.db`，彻底替代原 `case_knowledge_base.md`）

数据库存放路径建议：`.claude/skills/power-review/assets/memory/power_memory.db`（仅用 Python 自带 `sqlite3`，零外部依赖）。

> **⚠️ 彻底废除双轨制（删除原 `references/case_knowledge_base.md`）**：  
> 上线 `power_memory.db` 后，**不要再同时维护原有的 `case_knowledge_base.md`**，否则两边查、两边记必然打架。回内网初始化时，把你现有的历史排查笔记一次性导入 `power_memory.db`（直接标为 `confirmed`）后即可删掉旧的 `case_knowledge_base.md`。此后全套资料只有两块，分工零重叠：
> 1. **`general_guide.md`（及亮/灭屏规则 `.md`）**：只存跟具体 App 无关的**全局通用审单规矩**（真/模分层、豁免白名单、三问自检）；
> 2. **`power_memory.db`（自动导出只读视图 `references/log_memory_snapshot.md`）**：唯一承接所有**跟具体模块有关**的查法地图与历史故障案例。

### 为什么必须拆成两张业务表？
同一个系统模块（例如音频服务、系统代理锁、网络管理、某常驻核心进程）在实战中往往有 **3~5 种完全不同的耗电死法**。
- **模块叫什么 Tag、怎么顺藤摸瓜找上游 App、哪句打印是平时也有的正常噪音** $\rightarrow$ 这是**模块固有属性（1 对 1）**，如果以前记错了或版本改名了，应该直接**原地更正**；
- **这次是因为什么 Bug 导致耗电** $\rightarrow$ 这是**故障案例（1 对多）**，新死法必须**并列追加**，且未经人工/结案确认前只能标为 `candidate`（防止大模型两次猜错自己洗白成真理）。

### 表 1：`module_lookup_map`（模块客观查证地图 —— `1 对 1`，支持自动对比更正）

```sql
CREATE TABLE IF NOT EXISTS module_lookup_map (
    entity_key           TEXT PRIMARY KEY,     -- 核心标识（如 com.xxx.audio / ProxyRunningLock / hall_irq）
    entity_type          TEXT NOT NULL,        -- 类型：app | wakelock | irq | thread | subsystem | peripheral
    aliases_and_tags     TEXT DEFAULT '[]',    -- JSON 数组：日志真实 Tag、进程名、常见简称（解决 .db 名与日志名不一致）
    log_source_hint      TEXT DEFAULT 'hilog', -- 该去哪类日志查：hilog | kernel | hiview_event | 专有日志说明
    trace_keywords       TEXT DEFAULT '[]',    -- JSON 数组：追责跳板词 & 生命周期起止词（如怎么揪出代理锁背后的真实调用者 UID/PID）
    noise_keywords       TEXT DEFAULT '[]',    -- JSON 数组：经 Ref 正常段比对证实的无害常驻刷屏词（下次直接跳过不看）
    normal_baseline_note TEXT DEFAULT '',      -- 正常态下的日志规律一句话备忘
    updated_at           TEXT NOT NULL         -- 最后更新时间
);
```

### 表 2：`module_fault_cases`（模块故障死法库 —— `1 对多`，候选隔离 + 人工转正）

```sql
CREATE TABLE IF NOT EXISTS module_fault_cases (
    case_id              INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key           TEXT NOT NULL,        -- 归属模块标识（关联 module_lookup_map.entity_key，一对多）
    slot_category        TEXT NOT NULL,        -- 异常维度（如 off_wakelock / off_irq / off_blindspot / on_high_refresh / on_bg_run）
    case_title           TEXT NOT NULL,        -- 故障一句话标题（如：某通路切换失败每 2 秒重试导致持锁不释放）
    symptom_in_db        TEXT DEFAULT '',      -- .db 典型数据表象
    trigger_log_pattern  TEXT NOT NULL,        -- 确诊日志特征（搜到什么报错/状态横跳/有借无还组合即可坐实）
    root_cause_detail    TEXT NOT NULL,        -- 根因逻辑解释与上下游链路
    owner_team           TEXT DEFAULT '',      -- 建议转交研发团队
    status               TEXT DEFAULT 'candidate', -- 状态：candidate（AI 新发现待确认）| confirmed（人工/研发结案已确认）
    hit_count            INTEGER DEFAULT 1,    -- 累计命中次数
    sample_log           TEXT DEFAULT '',      -- 1~3 行典型脱敏原始日志样本
    last_ticket_id       TEXT DEFAULT '',      -- 来源单号
    updated_at           TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_cases_entity ON module_fault_cases (entity_key, status);
CREATE INDEX IF NOT EXISTS idx_cases_slot   ON module_fault_cases (slot_category, status);
```

### 表 3：`memory_revision_log`（查证地图修改留痕与回滚表）

```sql
CREATE TABLE IF NOT EXISTS memory_revision_log (
    rev_id        INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key    TEXT NOT NULL,
    action        TEXT NOT NULL,        -- CREATE_MAP | UPDATE_MAP | ROLLBACK_MAP
    diff_reason   TEXT NOT NULL,        -- 修改原因（如：实测日志 Tag 已由 OldTag 变更为 NewTag）
    old_snapshot  TEXT DEFAULT NULL,    -- 修改前 module_lookup_map 行 JSON 备份
    new_snapshot  TEXT NOT NULL,        -- 修改后 JSON 快照
    ticket_id     TEXT DEFAULT '',
    created_at    TEXT NOT NULL
);
```

---

## 三、 阶段三：1 个薄漏斗预检脚本 + 子代理自主破案规范

### 1. 薄漏斗脚本（`scripts/stage3_log/log_funnel.py`）只干哪 3 件事？
不再写限制大模型的复杂查询脚本，只保留这一个预处理漏斗（回内网只需对齐真实日志行开头的时间戳正则即可）：

1. **先验日志覆盖率（1 秒止损）**：
   - 读取日志目录各文件首尾有效时间戳 `[Log_min, Log_max]`，与异常段 `[T_start, T_end]` 求交集时长占比；
   - **若交集时长 `< 5%`（掉电时段日志已被冲掉）**：直接输出 `"covered": false`，**主模型不启动子代理**，直接在报告写明*“异常掉电时段日志已被环形缓冲区覆盖，以下基于 `.db` 纯物理真值给出疑似归因”*。
2. **切出异常小窗口文本**：
   - 将落在 `[T_start - 10分钟, T_end]` 内的日志行按时间顺序写入单个临时文件（如 `tmp_target_slice.log`，往前多扩 10 分钟专门抓灭屏前申请未释放的锁起点）。
3. **输出 Top 10 异常活跃/暴增 `Tag (PID)` 靶子榜**：
   - 顺手统计异常小窗口内各 `Tag` 的打印行数；若同库存在正常 `Ref` 段且日志也覆盖 `Ref` 段，则折算每小时打印速率，输出**相比 `Ref` 段净增/倍数最高的前 10 个 `Tag (PID)` 及其最高频的 1 条脱敏代表行**。

---

### 2. 日志调查子代理（Subagent）输入与三步探索法

当漏斗返回 `"covered": true` 时，主模型启动独立的日志调查子代理，传入：
- 切好的异常窗口日志路径 `tmp_target_slice.log`（及原始日志目录，供查 `Ref` 段对照）；
- `.db` 异常段摘要（真实均流、物理铁证、头号嫌犯名）与 `Ref` 正常时间段；
- 漏斗算出的 **Top 10 异常活跃/暴增 `Tag` 榜单**；
- 从 `power_memory.db` 召回的 **《模块查证地图》+《历史故障案例（区分 `confirmed` 与 `candidate`）》**。

**子代理内部按以下 3 步自主运行（只读不写库）：**

1. **第一步：对照《查证地图》定向核查（有经验先用经验）**
   - 自动避开 `noise_keywords` 中的已知无害刷屏词；
   - 若嫌犯是系统代理服务/公共锁，先搜 `trace_keywords` 揪出真正发起请求的业务进程 `UID/PID`；
   - 对照该模块名下的 `module_fault_cases`：若命中 `confirmed` 案例特征词则直接锁定；若命中 `candidate` 候选案例，仍需严格核验上下文是否真能解释本次耗电。
2. **第二步：在小切片内抓现行 + 必要时拿精准标识回原目录往前追起点**
   - **主战场在 `tmp_target_slice.log` 自由探索**：使用 `Grep` 和 `Read` 重点排查 **Top 10 暴增 `Tag/PID`** 为什么在该时段狂刷，以及跨进程 `IPC/Binder` 是否卡在等下游服务；
   - **锁/任务在更早之前（如 3 小时前）就已经开启怎么办？（精准回溯机制）**：
     - 严禁一开始就在几百 MB 原日志目录里搜宽泛词（如 `lock`、`error`）；
     - 但当子代理在小切片里已经锁定了持续活动的**具体锁编号、唯一实例 ID 或嫌犯进程 `PID / 专属 Tag`**，却在小切片（含前 10 分钟）内没看到它的首次申请/开启起点时，**允许子代理拿着这个精准标识回到原始全量日志目录往前检索**，定位数小时前是谁、在什么场景下触发了申请却一直未释放。
3. **第三步：`Ref` 正常段反证 + 结构化回传**
   - 凡是新抓到的疑似异常日志，若 `Ref` 段有日志覆盖，**必须去 `Ref` 段检索一次频次**——确认正常段没有同频狂打后才可定案（若正常段也在狂打，则推翻重查，并顺手将该词放入拟更新的 `noise_keywords`！）；
   - 向主模型返回调查结论与两条独立的记忆提案：
     - `map_update`（可选）：本次新摸清或需更正的真实日志 `Tag`、追责跳板词、实锤背景噪音词；
     - `case_proposal`（可选）：若匹配已有案例则返回 `{"action": "HIT_EXISTING", "case_id": ...}`；若发现了新死法则返回 `{"action": "ADD_CANDIDATE", ...}`。

---

## 四、 零依赖双层知识库管理脚本（`scripts/stage4_evolve/memory_store.py`）

以下完整脚本可直接落盘使用，已内置**词根分词模糊召回**、**全库轻量目录输出**、**表 1 自动对比更正 + 快照回滚**、**表 2 候选隔离与一键转正/剔除**，以及 **Markdown 总览快照自动导出**：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
功耗审单 SQLite 双层知识库管理工具（零第三方依赖）
表 1: module_lookup_map  —— 模块客观查证地图（1对1，支持自动记录与对比更正 + 回滚）
表 2: module_fault_cases —— 模块具体故障死法（1对多，AI 新增默认存 candidate，人工确认转 confirmed）

支持子命令：
  1. init          初始化数据库与可读快照
  2. recall        词根模糊召回嫌犯地图与案例 + 输出全库轻量目录（供阶段二/三使用）
  3. apply         结案把关后写入/更正客观地图，并追加新 candidate 候选案例或累加命中数
  4. confirm-case  人工确认某条 candidate 案例转正为 confirmed
  5. reject-case   删除误判的 candidate 候选案例
  6. rollback-map  将指定模块的查证地图回滚至上一次修改前快照
  7. export-md     手动刷新 references/log_memory_snapshot.md 可读总览文件
"""

import argparse
import json
import re
import sqlite3
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, List

SKILL_ROOT = Path(__file__).resolve().parents[2]
DEFAULT_DB_PATH = SKILL_ROOT / "assets" / "memory" / "power_memory.db"
DEFAULT_SNAPSHOT_MD = SKILL_ROOT / "references" / "log_memory_snapshot.md"

STOP_TOKENS = {
    "com", "cn", "org", "net", "app", "system", "service", "server",
    "manager", "mgr", "daemon", "process", "thread", "lock", "wakelock",
    "runninglock", "irq", "wakeup", "ohos", "harmony", "android", "core",
}

DDL_SCRIPT = """
CREATE TABLE IF NOT EXISTS module_lookup_map (
    entity_key           TEXT PRIMARY KEY,
    entity_type          TEXT NOT NULL,
    aliases_and_tags     TEXT DEFAULT '[]',
    log_source_hint      TEXT DEFAULT 'hilog',
    trace_keywords       TEXT DEFAULT '[]',
    noise_keywords       TEXT DEFAULT '[]',
    normal_baseline_note TEXT DEFAULT '',
    updated_at           TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS module_fault_cases (
    case_id              INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key           TEXT NOT NULL,
    slot_category        TEXT NOT NULL,
    case_title           TEXT NOT NULL,
    symptom_in_db        TEXT DEFAULT '',
    trigger_log_pattern  TEXT NOT NULL,
    root_cause_detail    TEXT NOT NULL,
    owner_team           TEXT DEFAULT '',
    status               TEXT DEFAULT 'candidate',
    hit_count            INTEGER DEFAULT 1,
    sample_log           TEXT DEFAULT '',
    last_ticket_id       TEXT DEFAULT '',
    updated_at           TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_cases_entity ON module_fault_cases (entity_key, status);
CREATE INDEX IF NOT EXISTS idx_cases_slot   ON module_fault_cases (slot_category, status);

CREATE TABLE IF NOT EXISTS memory_revision_log (
    rev_id        INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key    TEXT NOT NULL,
    action        TEXT NOT NULL,
    diff_reason   TEXT NOT NULL,
    old_snapshot  TEXT DEFAULT NULL,
    new_snapshot  TEXT NOT NULL,
    ticket_id     TEXT DEFAULT '',
    created_at    TEXT NOT NULL
);
"""


def now_iso() -> str:
    return datetime.now(timezone.utc).astimezone().isoformat(timespec="seconds")


def connect_db(db_path: Path) -> sqlite3.Connection:
    db_path.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(str(db_path), timeout=10.0)
    conn.row_factory = sqlite3.Row
    conn.executescript(DDL_SCRIPT)
    return conn


def parse_json_list(raw: Any) -> List[str]:
    if isinstance(raw, list):
        return [str(x).strip() for x in raw if str(x).strip()]
    if isinstance(raw, str) and raw.strip():
        try:
            val = json.loads(raw)
            if isinstance(val, list):
                return [str(x).strip() for x in val if str(x).strip()]
        except json.JSONDecodeError:
            return [raw.strip()]
    return []


def merge_list_json(old_raw: Any, new_items: List[str], replace: bool = False) -> str:
    cleaned_new = [str(x).strip() for x in (new_items or []) if str(x).strip()]
    if replace:
        return json.dumps(list(dict.fromkeys(cleaned_new)), ensure_ascii=False)
    base = parse_json_list(old_raw)
    return json.dumps(list(dict.fromkeys(base + cleaned_new)), ensure_ascii=False)


def map_row_to_dict(row: sqlite3.Row) -> Dict[str, Any]:
    d = dict(row)
    for k in ("aliases_and_tags", "trace_keywords", "noise_keywords"):
        d[k] = parse_json_list(d.get(k))
    return d


def extract_tokens(name: str) -> List[str]:
    """将包名、下划线锁名或驼峰名称拆分为有辨识度的词根，用于模糊召回"""
    camel_split = re.sub(r"([a-z0-9])([A-Z])", r"\1 \2", name)
    parts = re.split(r"[^a-zA-Z0-9\u4e00-\u9fa5]+", camel_split.lower())
    tokens = [p for p in parts if len(p) >= 3 and p not in STOP_TOKENS]
    return list(dict.fromkeys(tokens))


def cmd_recall(db_path: Path, entities: List[str], slot_category: str = "") -> Dict[str, Any]:
    conn = connect_db(db_path)
    matched_keys: List[str] = []

    for ent in entities:
        ent_clean = ent.strip()
        if not ent_clean:
            continue
        # 1) 精确或包含匹配
        rows = conn.execute(
            """
            SELECT entity_key FROM module_lookup_map
            WHERE LOWER(entity_key) = LOWER(?)
               OR LOWER(aliases_and_tags) LIKE LOWER(?)
            """,
            (ent_clean, f"%{ent_clean}%"),
        ).fetchall()
        for r in rows:
            if r["entity_key"] not in matched_keys:
                matched_keys.append(r["entity_key"])

        # 2) 词根拆分模糊匹配
        for tok in extract_tokens(ent_clean):
            rows = conn.execute(
                """
                SELECT entity_key FROM module_lookup_map
                WHERE LOWER(entity_key) LIKE ? OR LOWER(aliases_and_tags) LIKE ?
                """,
                (f"%{tok}%", f"%{tok}%"),
            ).fetchall()
            for r in rows:
                if r["entity_key"] not in matched_keys:
                    matched_keys.append(r["entity_key"])

        # 3) 兜底查仅存在于故障表中的记录
        rows = conn.execute(
            "SELECT DISTINCT entity_key FROM module_fault_cases WHERE LOWER(entity_key) = LOWER(?)",
            (ent_clean,),
        ).fetchall()
        for r in rows:
            if r["entity_key"] not in matched_keys:
                matched_keys.append(r["entity_key"])

    recalled_modules = []
    for key in matched_keys:
        m_row = conn.execute(
            "SELECT * FROM module_lookup_map WHERE entity_key = ?", (key,)
        ).fetchone()
        cases = conn.execute(
            """
            SELECT * FROM module_fault_cases
            WHERE entity_key = ?
            ORDER BY CASE WHEN status = 'confirmed' THEN 0 ELSE 1 END, hit_count DESC
            """,
            (key,),
        ).fetchall()
        recalled_modules.append({
            "entity_key": key,
            "lookup_map": map_row_to_dict(m_row) if m_row else None,
            "fault_cases": [dict(c) for c in cases],
        })

    # 同维度已确认示范案例（供未知新模块照猫画虎）
    slot_examples = []
    if slot_category:
        rows = conn.execute(
            """
            SELECT * FROM module_fault_cases
            WHERE slot_category = ? AND status = 'confirmed'
            ORDER BY hit_count DESC LIMIT 2
            """,
            (slot_category,),
        ).fetchall()
        for r in rows:
            if r["entity_key"] not in matched_keys:
                slot_examples.append(dict(r))

    # 全库轻量目录兜底（防止奇怪缩写漏召回，大模型扫一眼目录即可补查）
    catalog_rows = conn.execute(
        """
        SELECT m.entity_key, m.entity_type, m.aliases_and_tags,
               COUNT(c.case_id) AS case_cnt
        FROM module_lookup_map m
        LEFT JOIN module_fault_cases c ON m.entity_key = c.entity_key
        GROUP BY m.entity_key
        ORDER BY m.entity_key ASC
        """
    ).fetchall()
    catalog_index = [
        f"{r['entity_key']} ({r['entity_type']}) | 别名/Tag: {', '.join(parse_json_list(r['aliases_and_tags'])) or '-'} | 案例数: {r['case_cnt']}"
        for r in catalog_rows
    ]

    conn.close()
    return {
        "recalled_modules": recalled_modules,
        "slot_confirmed_examples": slot_examples,
        "catalog_index": catalog_index,
    }


def export_markdown_snapshot(conn: sqlite3.Connection, md_path: Path) -> None:
    maps = conn.execute("SELECT * FROM module_lookup_map ORDER BY entity_key ASC").fetchall()
    cases = conn.execute(
        """
        SELECT * FROM module_fault_cases
        ORDER BY CASE WHEN status = 'confirmed' THEN 0 ELSE 1 END, entity_key ASC, hit_count DESC
        """
    ).fetchall()
    md_path.parent.mkdir(parents=True, exist_ok=True)

    cell = lambda s: str(s).replace("|", "\\|").replace("\n", "<br>")
    lines = [
        "# 功耗日志知识库可读快照（自动导出，请通过 `memory_store.py` 维护）",
        "",
        f"> **同步时间**：`{now_iso()}` | **查证地图模块数**：`{len(maps)}` | **故障案例数**：`{len(cases)}`",
        "",
        "## 一、 模块客观查证地图（`module_lookup_map`）",
        "",
        "| 模块标识 (`entity_key`) | 类型 / 日志源 | 真实 Tag / 别名 | 追责跳板与起止关键字 (`🔍`) | 已知无害噪音词 (`🔇`) | 正常态规律备忘 |",
        "| :--- | :--- | :--- | :--- | :--- | :--- |",
    ]
    for r in maps:
        d = map_row_to_dict(r)
        tags = ", ".join(f"`{x}`" for x in d["aliases_and_tags"]) or "-"
        trace = ", ".join(f"`{x}`" for x in d["trace_keywords"]) or "-"
        noise = ", ".join(f"`{x}`" for x in d["noise_keywords"]) or "-"
        lines.append(
            f"| `{cell(d['entity_key'])}` | `{cell(d['entity_type'])}`<br>`{cell(d['log_source_hint'])}` "
            f"| {cell(tags)} | {cell(trace)} | {cell(noise)} | {cell(d['normal_baseline_note'] or '-')} |"
        )

    lines.extend([
        "",
        "## 二、 模块具体故障死法库（`module_fault_cases`）",
        "",
        "> `confirmed` = 人工/结案已坐实；`candidate` = AI 审单新发现待人工确认（可用 `confirm-case` 转正或 `reject-case` 剔除）。",
        "",
        "| ID | 状态 (命中) | 归属模块 | 维度 | 故障标题 | 确诊日志特征 | 根因解释 & 转交方向 |",
        "| :--- | :--- | :--- | :--- | :--- | :--- | :--- |",
    ])
    for c in cases:
        team_str = f"（→ {c['owner_team']}）" if c["owner_team"] else ""
        lines.append(
            f"| `{c['case_id']}` | `{c['status']}` ({c['hit_count']}) | `{cell(c['entity_key'])}` "
            f"| `{cell(c['slot_category'])}` | {cell(c['case_title'])} "
            f"| `{cell(c['trigger_log_pattern'])}` | {cell(c['root_cause_detail'])}{cell(team_str)} |"
        )

    md_path.write_text("\n".join(lines) + "\n", encoding="utf-8")


def cmd_apply(db_path: Path, md_path: Path, payload: Dict[str, Any], ticket_id: str = "") -> Dict[str, Any]:
    """
    payload 结构示例：
    {
      "map_updates": [
        {
          "entity_key": "ProxyRunningLock",
          "entity_type": "wakelock",
          "aliases_and_tags": ["PowerMgrLock", "RunningLockMgr"],
          "log_source_hint": "hilog",
          "trace_keywords": ["AcquireRunningLock.*uid="],
          "replace_trace_keywords": false,
          "noise_keywords": ["UpdateLockState unchanged"],
          "normal_baseline_note": "正常申请释放成对出现且单次 < 3s",
          "diff_reason": "补充代理锁查真实调用方 uid 跳板词与无害噪音"
        }
      ],
      "case_updates": [
        {
          "action": "HIT_EXISTING",
          "case_id": 1
        },
        {
          "action": "ADD_CANDIDATE",
          "entity_key": "com.xxx.sync",
          "slot_category": "off_wakelock",
          "case_title": "远端鉴权超时每 5s 重试导致长持锁",
          "symptom_in_db": "灭屏深睡率低，SyncWake 持锁 > 80%",
          "trigger_log_pattern": "Auth token expired, retry in 5000ms",
          "root_cause_detail": "Token 失效后未走异常释放锁分支，死循环重试阻塞休眠",
          "owner_team": "云同步团队",
          "sample_log": "02:14:05.110 1920 1988 E SyncTag: Auth token expired, retry in 5000ms"
        }
      ]
    }
    """
    conn = connect_db(db_path)
    ts = now_iso()
    summary = {"map_applied": [], "case_applied": []}

    with conn:
        # 1) 更新表 1：模块客观查证地图（支持首次创建或对比更正）
        for m in payload.get("map_updates", []):
            key = m["entity_key"].strip()
            existing = conn.execute(
                "SELECT * FROM module_lookup_map WHERE entity_key = ?", (key,)
            ).fetchone()
            replace_trace = bool(m.get("replace_trace_keywords", False))

            if existing is None:
                action = "CREATE_MAP"
                old_snap = None
                conn.execute(
                    """
                    INSERT INTO module_lookup_map (
                        entity_key, entity_type, aliases_and_tags, log_source_hint,
                        trace_keywords, noise_keywords, normal_baseline_note, updated_at
                    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                    """,
                    (
                        key,
                        m.get("entity_type", "app"),
                        merge_list_json("[]", m.get("aliases_and_tags", [])),
                        m.get("log_source_hint", "hilog"),
                        merge_list_json("[]", m.get("trace_keywords", [])),
                        merge_list_json("[]", m.get("noise_keywords", [])),
                        m.get("normal_baseline_note", ""),
                        ts,
                    ),
                )
            else:
                action = "UPDATE_MAP"
                old_snap = json.dumps(map_row_to_dict(existing), ensure_ascii=False)
                conn.execute(
                    """
                    UPDATE module_lookup_map
                    SET entity_type          = COALESCE(NULLIF(?, ''), entity_type),
                        aliases_and_tags     = ?,
                        log_source_hint      = COALESCE(NULLIF(?, ''), log_source_hint),
                        trace_keywords       = ?,
                        noise_keywords       = ?,
                        normal_baseline_note = COALESCE(NULLIF(?, ''), normal_baseline_note),
                        updated_at           = ?
                    WHERE entity_key = ?
                    """,
                    (
                        m.get("entity_type", ""),
                        merge_list_json(existing["aliases_and_tags"], m.get("aliases_and_tags", [])),
                        m.get("log_source_hint", ""),
                        merge_list_json(existing["trace_keywords"], m.get("trace_keywords", []), replace=replace_trace),
                        merge_list_json(existing["noise_keywords"], m.get("noise_keywords", [])),
                        m.get("normal_baseline_note", ""),
                        ts,
                        key,
                    ),
                )

            new_row = conn.execute(
                "SELECT * FROM module_lookup_map WHERE entity_key = ?", (key,)
            ).fetchone()
            conn.execute(
                """
                INSERT INTO memory_revision_log (
                    entity_key, action, diff_reason, old_snapshot, new_snapshot, ticket_id, created_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    key,
                    action,
                    m.get("diff_reason", action),
                    old_snap,
                    json.dumps(map_row_to_dict(new_row), ensure_ascii=False),
                    ticket_id,
                    ts,
                ),
            )
            summary["map_applied"].append({"entity_key": key, "action": action})

        # 2) 更新表 2：故障死法库（一对多，AI 新增固定为 candidate，导入历史人工笔记可用 IMPORT_CONFIRMED）
        for c in payload.get("case_updates", []):
            c_act = c.get("action", "ADD_CANDIDATE").upper()
            if c_act == "HIT_EXISTING" and c.get("case_id") is not None:
                conn.execute(
                    """
                    UPDATE module_fault_cases
                    SET hit_count = hit_count + 1,
                        last_ticket_id = COALESCE(NULLIF(?, ''), last_ticket_id),
                        updated_at = ?
                    WHERE case_id = ?
                    """,
                    (ticket_id, ts, int(c["case_id"])),
                )
                summary["case_applied"].append({"action": "HIT_EXISTING", "case_id": int(c["case_id"])})
            else:
                init_status = "confirmed" if c_act == "IMPORT_CONFIRMED" else "candidate"
                cur = conn.execute(
                    """
                    INSERT INTO module_fault_cases (
                        entity_key, slot_category, case_title, symptom_in_db,
                        trigger_log_pattern, root_cause_detail, owner_team,
                        status, hit_count, sample_log, last_ticket_id, updated_at
                    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, 1, ?, ?, ?)
                    """,
                    (
                        c["entity_key"].strip(),
                        c.get("slot_category", "general"),
                        c["case_title"].strip(),
                        c.get("symptom_in_db", ""),
                        c["trigger_log_pattern"].strip(),
                        c["root_cause_detail"].strip(),
                        c.get("owner_team", ""),
                        init_status,
                        c.get("sample_log", ""),
                        ticket_id,
                        ts,
                    ),
                )
                summary["case_applied"].append({
                    "action": c_act if c_act == "IMPORT_CONFIRMED" else "ADD_CANDIDATE",
                    "case_id": cur.lastrowid,
                    "entity_key": c["entity_key"].strip(),
                    "status": init_status,
                })

        export_markdown_snapshot(conn, md_path)

    conn.close()
    return summary


def cmd_set_case_status(db_path: Path, md_path: Path, case_id: int, confirm: bool) -> Dict[str, Any]:
    conn = connect_db(db_path)
    with conn:
        if confirm:
            conn.execute(
                "UPDATE module_fault_cases SET status = 'confirmed', updated_at = ? WHERE case_id = ?",
                (now_iso(), case_id),
            )
            res = {"ok": True, "case_id": case_id, "new_status": "confirmed"}
        else:
            conn.execute("DELETE FROM module_fault_cases WHERE case_id = ?", (case_id,))
            res = {"ok": True, "case_id": case_id, "new_status": "deleted"}
        export_markdown_snapshot(conn, md_path)
    conn.close()
    return res


def cmd_rollback_map(db_path: Path, md_path: Path, entity_key: str) -> Dict[str, Any]:
    conn = connect_db(db_path)
    rev = conn.execute(
        """
        SELECT * FROM memory_revision_log
        WHERE entity_key = ? AND old_snapshot IS NOT NULL
        ORDER BY rev_id DESC LIMIT 1
        """,
        (entity_key,),
    ).fetchone()
    if rev is None:
        conn.close()
        return {"ok": False, "reason": f"未找到 {entity_key} 可回滚的历史修改快照"}

    old = json.loads(rev["old_snapshot"])
    ts = now_iso()
    with conn:
        conn.execute(
            """
            UPDATE module_lookup_map
            SET entity_type = ?, aliases_and_tags = ?, log_source_hint = ?,
                trace_keywords = ?, noise_keywords = ?, normal_baseline_note = ?, updated_at = ?
            WHERE entity_key = ?
            """,
            (
                old["entity_type"],
                json.dumps(old["aliases_and_tags"], ensure_ascii=False),
                old["log_source_hint"],
                json.dumps(old["trace_keywords"], ensure_ascii=False),
                json.dumps(old["noise_keywords"], ensure_ascii=False),
                old["normal_baseline_note"],
                ts,
                entity_key,
            ),
        )
        conn.execute(
            """
            INSERT INTO memory_revision_log (
                entity_key, action, diff_reason, old_snapshot, new_snapshot, ticket_id, created_at
            ) VALUES (?, 'ROLLBACK_MAP', '人工回滚至修改前快照', ?, ?, '', ?)
            """,
            (entity_key, rev["new_snapshot"], rev["old_snapshot"], ts),
        )
        export_markdown_snapshot(conn, md_path)
    conn.close()
    return {"ok": True, "restored_entity": entity_key}


def main() -> None:
    parser = argparse.ArgumentParser(description="功耗审单 SQLite 双层知识库管理 CLI")
    parser.add_argument("--db", type=Path, default=DEFAULT_DB_PATH)
    parser.add_argument("--snapshot", type=Path, default=DEFAULT_SNAPSHOT_MD)
    sub = parser.add_subparsers(dest="cmd", required=True)

    sub.add_parser("init", help="初始化双层表与可读快照")

    p_recall = sub.add_parser("recall", help="分词模糊召回模块查证地图与故障案例")
    p_recall.add_argument("--entities", nargs="+", required=True, help="嫌犯名称列表")
    p_recall.add_argument("--slot", default="", help="异常维度分类")

    p_apply = sub.add_parser("apply", help="写入/更正查证地图并记录候选故障案例")
    p_apply.add_argument("--payload-json", required=True, help="JSON 文件路径或 JSON 字符串")
    p_apply.add_argument("--ticket", default="", help="来源单号")

    p_conf = sub.add_parser("confirm-case", help="人工确认候选案例转正为 confirmed")
    p_conf.add_argument("--case-id", type=int, required=True)

    p_rej = sub.add_parser("reject-case", help="删除误判的候选案例")
    p_rej.add_argument("--case-id", type=int, required=True)

    p_rb = sub.add_parser("rollback-map", help="回滚指定模块的查证地图到上一版")
    p_rb.add_argument("--entity", required=True)

    sub.add_parser("export-md", help="刷新 Markdown 快照文件")

    args = parser.parse_args()
    if args.cmd == "init":
        conn = connect_db(args.db)
        export_markdown_snapshot(conn, args.snapshot)
        conn.close()
        print(json.dumps({"ok": True, "db": str(args.db), "snapshot": str(args.snapshot)}, ensure_ascii=False))
    elif args.cmd == "recall":
        print(json.dumps(cmd_recall(args.db, args.entities, args.slot), ensure_ascii=False, indent=2))
    elif args.cmd == "apply":
        raw = args.payload_json
        p = Path(raw)
        payload = json.loads(p.read_text(encoding="utf-8") if p.is_file() else raw)
        print(json.dumps({"ok": True, **cmd_apply(args.db, args.snapshot, payload, args.ticket)}, ensure_ascii=False, indent=2))
    elif args.cmd == "confirm-case":
        print(json.dumps(cmd_set_case_status(args.db, args.snapshot, args.case_id, True), ensure_ascii=False))
    elif args.cmd == "reject-case":
        print(json.dumps(cmd_set_case_status(args.db, args.snapshot, args.case_id, False), ensure_ascii=False))
    elif args.cmd == "rollback-map":
        print(json.dumps(cmd_rollback_map(args.db, args.snapshot, args.entity), ensure_ascii=False))
    elif args.cmd == "export-md":
        conn = connect_db(args.db)
        export_markdown_snapshot(conn, args.snapshot)
        conn.close()
        print(json.dumps({"ok": True, "snapshot": str(args.snapshot)}, ensure_ascii=False))


if __name__ == "__main__":
    main()
```

---

## 五、 内网落地修改清单（仅改阶段二~四，阶段一完全不动）

| 序号 | 内网目标文件 | 具体修改内容 |
| :--- | :--- | :--- |
| 1 | `scripts/stage4_evolve/memory_store.py` *(新增/替换)*<br>& `references/case_knowledge_base.md` *(迁移后删除)* | 落地第四节的 SQLite 双层知识库脚本，运行 `init` 生成 `power_memory.db` 与 `references/log_memory_snapshot.md`；将原 `case_knowledge_base.md` 中的已有实战案例用 `"action": "IMPORT_CONFIRMED"` 一次性导入 SQLite 后，**删除原 `case_knowledge_base.md`**，杜绝双库并存打架。 |
| 2 | `scripts/stage3_log/log_funnel.py` *(新增单文件)* | 对照内网真实日志时间格式，实现 100 行左右的薄漏斗：**① 检查 `[T_start, T_end]` 是否被冲掉（冲掉则直接止损退出）**；**② 切出 `[T_start - 10min, T_end]` 临时小文件**；**③ 输出 Top 10 异常活跃/暴增 `Tag (PID)`**。 |
| 3 | `.claude/skills/power-review/SKILL.md` | • 移除对旧 `case_knowledge_base.md` 的引用，统一指向 `power_memory.db`（及只读快照 `references/log_memory_snapshot.md`）；<br>• **阶段二**：调用 `memory_store.py recall` 词根模糊召回嫌犯的《查证地图》+《故障案例》+《全库轻量目录》；<br>• **阶段三**：先跑 `log_funnel.py` 预检切片；确认日志存在后，派发**【日志调查子代理】**——主战场在小切片文件内核对地图、顺 Top 10 暴增 Tag 自由探索；**若小切片内未见锁/任务开启起点，允许拿着已锁定的精准锁名/PID 回原始日志目录往前追溯起点**；最后做 `Ref` 段反证；<br>• **阶段四**：主模型过完三问自检后调用 `memory_store.py apply`——客观 Tag/跳板词/噪音词自动更新进表 1，新发现的死法以 `candidate` 存入表 2 并在报告末尾提示人工确认编号（`case_id`）。 |
| 4 | `.claude/skills/power-review/assets/templates/iteration_patch_card.md` | 报告末尾迭代块简化为两行清爽提示：<br>1) 本次自动更新了哪个模块的客观查证地图（Tag/噪音词）；<br>2) 若新增了候选故障案例（`case_id = X, status = candidate`），提示你确认无误后执行 `confirm-case --case-id X` 转正（或猜错时 `reject-case` 删掉）。 |

---

## 六、 附：回内网直接发给 AI 执行改造的指令模板

```text
请阅读根目录下的 `log_subagent_memory_upgrade.md`，保持我现有的阶段一 `.db` 解析代码不动，按文档第五节清单完成阶段二~四升级：
1. 创建/更新 `.claude/skills/power-review/scripts/stage4_evolve/memory_store.py`（实现 `module_lookup_map` 客观查证地图与 `module_fault_cases` 一对多候选案例双层 SQLite 表，支持词根模糊召回、轻量目录兜底、`IMPORT_CONFIRMED` 批量导入、`candidate -> confirmed` 转正与快照导出），运行 `init` 初始化；随后把原 `references/case_knowledge_base.md` 中的正式案例导入进 `power_memory.db`（标为 `confirmed`），并删除旧的 `case_knowledge_base.md`；
2. 对照我仓库现有的真实日志格式，在 `scripts/stage3_log/` 下实现轻量预处理漏斗 `log_funnel.py`（检查异常时间段日志是否已被环形缓冲区冲掉、切出 `[T_start - 10min, T_end]` 窗口日志小文件、统计相比 Ref 暴增的 Top 10 Tag/PID）；
3. 更新 `.claude/skills/power-review/SKILL.md`、`references/general_guide.md` 和 `assets/templates/iteration_patch_card.md`（同步清理旧 `case_knowledge_base.md` 引用），将查日志与沉淀流程串成：“漏斗预检止损与切片 -> 子代理基于查证地图与暴增 Tag 在小文件内自由探索（若缺起点可拿精准锁名/PID 回原日志目录往前追溯）+ Ref 反证 -> 客观查证地图自动对比更正 + 新故障案例存为 candidate 待人工确认”。
```
