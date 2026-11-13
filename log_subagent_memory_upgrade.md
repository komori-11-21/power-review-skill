# 阶段三 & 阶段四架构升级指南：日志调查子代理 + SQLite 关键字/行为对比记忆库

> **文档定位**：本文档用于替换原方案中“主模型死板调用 4 个固定脚本查日志 + 仅向 Markdown 追加静态卡片”的旧设计，升级为 **“子代理自主查日志（种子关键字引导 + 时间围栏内自由探索） + SQLite 结构化记忆库（首次记录 $\rightarrow$ 下次对比确认/更正 + 自动导出可读快照）”** 闭环架构。  
> 全文保持**通用性与指导性**，不绑定任何内部真实表名、包名或私有日志格式；文末附有可直接发给内网 AI 自动完成改造的指令模板。

---

## 一、 核心升级背景与整体流转图

### 1. 为什么要改？
- **解绑死脚本，释放大模型原生日志推理能力**：固定脚本只能处理预设好的刷屏格式；改成**独立子代理（Subagent）**后，既能先按已知关键字秒级定位，又能在未命中时利用原生搜索与上下文阅读能力顺着 `PID / Tag / IPC 上下游 / 状态机切换` 自由探索，还不污染主对话上下文窗口。
- **从“只增不改的文本堆砌”升级为“可对比、可纠偏、可回滚的结构化记忆”**：利用 Python 标准库自带的 `sqlite3`（零第三方依赖）建立轻量记忆库 `power_memory.db`——第 1 次遇到某模块时记录初版关键字与日志行为（`provisional`），下一次再遇到时与实测日志**对比验证**，一致则升级坐实（`verified`），发现旧关键字失效或行为理解有偏差则**原地更正（`CORRECT`）并留存修改前快照**。

### 2. 存储边界划分（什么进 SQLite，什么留 Markdown？）

| 知识资产类别 | 存储载体 | 原因与维护方式 |
| :--- | :--- | :--- |
| **模块级查日志经验**：包名/锁名/IRQ $\leftrightarrow$ 真实日志 `Tag` 映射、有效定位关键字、无害噪音/失效避坑词、正常与异常日志行为特征、归属团队、历次修改记录 | **SQLite 库**<br>`assets/memory/power_memory.db` | 条目随跑单持续增长；按嫌犯名毫秒级精准召回 2~5 条喂给子代理，省 Token、防重复、支持状态流转与回滚。 |
| **记忆库人工可读视图**：全量模块关键字与行为总览表 | **自动生成的只读 Markdown**<br>`references/log_memory_snapshot.md` | 每次写库后由脚本自动覆盖导出，消除二进制黑盒，方便人眼审阅与 `git diff` 检查。 |
| **全局宪法级审单规则**：真/模电量分层、豁免白名单、亮/灭屏看表法则、5 类通用死法、结案三问自检 | **人工/复盘维护的 Markdown**<br>`general_guide.md`、`screen_off_rules.md`、`screen_on_rules.md` | 条目精炼且每单必读，留在 `.md` 中便于随时直接打开增删几句话。 |

*(注：原有的 `case_knowledge_base.md` 可仅保留少量经典示范卡片供跨模块“照猫画虎”教学，具体模块词条统一收归 `power_memory.db` 管理。)*

### 3. 四步闭环运行流转

```text
[.db 锁定异常时间窗 + 嫌犯名单]
        │
        ▼
① 派单前精准召回（脚本读 power_memory.db）
   ├─ 精确匹配：按包名 / 锁名 / IRQ / 别名查出历史记录（含有效词、避坑词、已知行为、状态）
   └─ 同类示范：若为全新模块，按异常槽位类别（slot_category）召回 1~2 条已验证（verified）同类记录作参考
        │
        ▼
② 子代理自主调查（独立上下文窗口，只读不写库）
   ├─ 第一步【对比核验】：跳过避坑噪音词，先用历史有效关键字查异常窗，与旧记录中的行为描述逐项对比
   ├─ 第二步【自由探索】：未命中或旧记录失准时，在 [T_start - 10min, T_end] 围栏内自由检索 Tag/PID/IPC/报错上下文
   ├─ 第三步【Ref 反证】：所有抓到的疑似异常日志，必须在同库正常低耗 Ref 段核验频次，剔除常驻背景打印
   └─ 第四步【结构化回传】：返回「根因结论 + 2~4 行原日志铁证」+「记忆库变更提案 JSON（CREATE / CONFIRM / CORRECT）」
        │
        ▼
③ 主模型“结案三问自检”把关
   └─ 核查：时间贯穿性、物理硬件可解释性、Ref 正常段反证；通过后放行写库提案（无日志或未查清时禁止覆写旧记录）
        │
        ▼
④ 统一写库留痕 + 导出快照（主线程单次调用写库脚本）
   ├─ 更新 module_log_memory 主表（首次记 provisional，二次吻合升 verified，有偏差原地更正）
   ├─ 写入 memory_revision_log 留痕表（保存修改前 JSON 快照与对比差异原因，随时可回滚）
   └─ 自动刷新 references/log_memory_snapshot.md 可读总览表
```

---

## 二、 SQLite 记忆库表结构设计（`power_memory.db`）

数据库文件建议存放在 `.claude/skills/power-review/assets/memory/power_memory.db`（纳入版本管理或内网持久化目录）。仅需 **2 张核心表**，结构通用清晰：

### 1. 主表：`module_log_memory`（模块关键字与日志行为字典）

```sql
CREATE TABLE IF NOT EXISTS module_log_memory (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key          TEXT NOT NULL UNIQUE, -- 核心唯一标识（如包名 com.xxx.sync、锁名 SyncWake、中断名 hall_irq）
    entity_type         TEXT NOT NULL,        -- 对象类型：app | wakelock | irq | thread | subsystem | peripheral
    slot_category       TEXT NOT NULL,        -- 异常维度分类（如 off_wakelock / off_irq / off_blindspot / on_high_refresh / on_bg_run）
    aliases_and_tags    TEXT DEFAULT '[]',    -- JSON 数组：关联进程名、真实日志 Tag、服务名（打通 .db 名字与日志 Tag 差异）
    effective_keywords  TEXT DEFAULT '[]',    -- JSON 数组：实测精准有效的检索关键字或正则片段（按推荐优先级排列）
    noise_keywords      TEXT DEFAULT '[]',    -- JSON 数组：已证实搜不到、或正常 Ref 段也高频刷屏的误导词/无害噪音词（下次直接避开）
    normal_behavior     TEXT DEFAULT '',      -- 正常态日志行为总结（如：正常起止成对打印特征、单次耗时范围、正常心跳频率）
    abnormal_behavior   TEXT DEFAULT '',      -- 异常高耗态日志行为总结（如：卡在何处不释放、何种报错死循环、状态乒乓特征及上下游依赖）
    owner_team_hint     TEXT DEFAULT '',      -- 建议转交研发模块/团队方向（通用占位，按内部实际组织填写）
    status              TEXT DEFAULT 'provisional', -- 置信状态：provisional（仅记录一次待验证）| verified（经对比验证坐实）
    hit_count           INTEGER DEFAULT 1,    -- 累计实战命中/核验次数
    last_ticket_id      TEXT DEFAULT '',      -- 最近一次更新来源单号或样本标识
    updated_at          TEXT NOT NULL         -- 最后更新时间（ISO-8601）
);

CREATE INDEX IF NOT EXISTS idx_memory_slot_status
    ON module_log_memory (slot_category, status);
```

### 2. 留痕表：`memory_revision_log`（对比修改审计与回滚表）

```sql
CREATE TABLE IF NOT EXISTS memory_revision_log (
    rev_id          INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key      TEXT NOT NULL,        -- 关联主表的模块标识
    action          TEXT NOT NULL,        -- 动作类型：CREATE（首次记录）| CONFIRM（对比吻合）| CORRECT（对比发现偏差并更正）| MANUAL（人工修订）
    diff_reason     TEXT NOT NULL,        -- 对比结论与修改原因（写明上次记了什么、本次实测看到了什么、为何更正）
    old_snapshot    TEXT DEFAULT NULL,    -- 修改前主表整行数据的 JSON 备份（CREATE 时为 NULL；改错时可一键还原）
    new_snapshot    TEXT NOT NULL,        -- 修改后主表整行数据的 JSON 快照
    log_sample      TEXT DEFAULT '',      -- 支撑本次记录/更正的 1~3 行关键原始日志样本（脱敏保留 Tag 与动作）
    ticket_id       TEXT DEFAULT '',      -- 触发本次变更的单号
    created_at      TEXT NOT NULL         -- 变更时间
);

CREATE INDEX IF NOT EXISTS idx_rev_entity
    ON memory_revision_log (entity_key, rev_id DESC);
```

---

## 三、 阶段三：日志调查子代理（Subagent）协同规范

### 1. 主模型派发给子代理的《日志调查任务单》输入契约
当存在设备日志时，主模型针对每个待审异常簇（或 `Primary Target` 切片）启动一个独立的只读调查子代理，传入以下 **5 项结构化上下文**（严禁把整份 `.db` 原始大表塞给子代理，保持子代理上下文干净）：

1. **【时间围栏（强制边界）】**：
   - **异常调查窗口**：`[T_start - 10分钟, T_end]`（往前多扩 10 分钟，专门捕捉“灭屏前或切后台前申请了锁/开启了任务但一直未释放”的起点日志）；
   - **正常对照窗口（`Ref`）**：`[Ref_start, Ref_end]`（若全库无正常参照段则标 `Ref = 无`）。
2. **【待解释的 `.db` 物理铁证与头号嫌犯】**：
   - 电池真实均流 $I_{\text{real}}$ 与异常类型（灭屏不深睡 / 灭屏静默外设漏电 / 亮屏高刷常驻 / 亮屏后台陪跑等）；
   - 纯物理指标排名靠前或跨表共振的嫌犯名称列表（包名、锁名、线程名、`Wakeup IRQ` 名）。
3. **【日志目录路径与可选大文件辅助工具】**：
   - 设备日志所在路径；若日志为数百 MB 压缩包或跨多天巨型文件，提示子代理可先按时间窗切片提取，再做精细检索。
4. **【从 `power_memory.db` 召回的历史记忆】**：
   - **精确命中记录**：该嫌犯上次记录的 `status (provisional/verified)`、`aliases_and_tags`、`effective_keywords`、`noise_keywords`、`normal_behavior`、`abnormal_behavior`；
   - **同类示范记录**（当嫌犯首次出现时）：同属一个 `slot_category` 的 1~2 条历史已验证记录，供子代理参考排查手法。
5. **【子代理执行纪律与返回格式要求】**（见下节）。

---

### 2. 子代理内部查日志四步法（种子引导 + 自由探索 + 对比纠偏）

子代理在自身独立上下文中按以下顺序自主开展调查：

- **第 1 步：核对历史记忆（有记录先验证，避开已知坑）**
  - 若任务单带有历史记录：**直接跳过 `noise_keywords` 中的已知噪音词**，优先用 `effective_keywords` 和 `aliases_and_tags` 在异常时间窗内检索（`Grep`）；
  - 将实际看到的日志上下文与历史记录中的 `normal_behavior / abnormal_behavior` 对比，判断属于“完全吻合”、“正常无辜行为”还是“旧关键字失效 / 实际行为与旧记录不符”。
- **第 2 步：时间围栏内自由探索（旧关键字未命中或新模块时全面放开）**
  - 不受任何固定脚本限制，自主组合运用正则搜索（`Grep`）与前后上下文阅读（`Read`）：
    1. **找真名与 PID**：用包名片段、服务名或功能词搜进程启动/绑定行或常规打印行，锁定真实 `PID` 与模块专属日志 `Tag`；
    2. **追生命周期与 5 类死法**：围绕异常时间窗排查 **① 有借无还**（有 `Acquire/Start/Open/Vote` 无对应释放）、**② 报错死循环重试**（高频 `fail/retry/timeout`）、**③ 状态机乒乓震荡**（如 `99% <-> 100%`、`Connect <-> Disconnect`）、**④ 跨进程 IPC/Binder 阻塞**（A 持代理锁等下游 B 服务响应，顺着日志继续去搜 B 模块）、**⑤ 灭屏/冻结拦截失效**；
    3. **无明确嫌犯时的全局扫描**：若属软件模拟覆盖率极低/物理静默的外设盲区漏电，直接在异常窗内检索内核唤醒源、外设充电/霍尔/连接状态切换及高频 `W/E` 级日志。
- **第 3 步：`Ref` 正常参照段反证（子代理内部必做自检！）**
  - 一旦锁定某条疑似异常关键字或刷屏日志，**只要存在正常低耗 `Ref` 段，子代理必须用同一关键字去 `[Ref_start, Ref_end]` 统计出现频次**：
    - 若 `Ref` 段同样同频刷屏 $\rightarrow$ 判定为常驻背景噪音，子代理自行推翻该假设继续探索，并把该词记入本次拟提交的 `noise_keywords`！
    - 若异常段高频出现（或长持锁不释放）而 `Ref` 段为 `0`（或极少且成对秒放） $\rightarrow$ 锁定为实锤证据。
- **第 4 步：向主模型返回结构化调查结果（严禁子代理直接写库）**
  - 无论某个被调查模块最终是**“确诊真凶”**还是**“排除嫌疑的无辜模块”**，只要摸清了它的日志 Tag、关键字或正常/异常行为，都可以生成记忆提案一并返回：

```json
{
  "investigation_result": {
    "verdict": "CONFIRMED_ROOT_CAUSE | SUSPECT_INCONCLUSIVE | RULED_OUT_NORMAL",
    "culprit_entity": "模块/包名/锁名",
    "failure_archetype": "有借无还 | 报错死循环 | 状态乒乓 | 跨进程阻塞 | 状态拦截失效 | 正常行为",
    "root_cause_summary": "大白话讲清来龙去脉（含上下游链路）",
    "target_ vs_ref_count": "异常段出现 N 次（持续 X 分钟） vs Ref 正常段出现 0 次",
    "key_log_evidence": [
      "保留原始时间戳、PID、Tag 的 2~4 行最核心日志原文"
    ],
    "owner_team_suggestion": "建议转交的研发方向"
  },
  "memory_proposals": [
    {
      "action": "CREATE | CONFIRM | CORRECT",
      "entity_key": "com.xxx.example",
      "entity_type": "app | wakelock | irq | subsystem | peripheral",
      "slot_category": "off_wakelock",
      "aliases_and_tags": ["ExampleSvc", "ExampleSyncTag"],
      "effective_keywords": ["AcquireLock.*Example", "SyncTask timeout"],
      "noise_keywords": ["Example Heartbeat ok"],
      "normal_behavior": "正常同步时 Acquire -> Sync finish -> Release 在 3 秒内成对出现",
      "abnormal_behavior": "网络异常或远端无响应时卡在 SyncTask timeout 每 5 秒重试，期间一直不走 Release",
      "owner_team_hint": "云同步模块团队",
      "diff_reason": "首次记录该模块 Tag 与持锁重试特征（若为 CORRECT 则写明：原记录关键字为 XXX，本次对比实测日志发现实际打印为 YYY，故更正）",
      "log_sample": "02:15:03.112  1842  1901 E ExampleSyncTag: SyncTask timeout, retry..."
    }
  ]
}
```

---

## 四、 阶段四：防污染护栏与“记录 $\rightarrow$ 下次对比修改”写库规则

为了防止大模型某次没查清就把好不容易攒对的知识库改坏，主模型在更新 `power_memory.db` 前必须执行 **4 条硬护栏**：

1. **子代理只读、主线程单点写库（防锁库与并发冲突）**：
   - 多个异常簇可并行启动多个子代理查日志，但所有子代理只读不写；主模型汇总结果并通过**结案三问自检**后，统一调用一次写库脚本落盘。
2. **缺日志单严禁改库**：
   - 当用户仅传入 `.db` 未提供日志，或日志时间段缺失导致归因状态为 `疑似对象（尚缺日志验证）` 时，**禁止触发任何 `CREATE / CONFIRM / CORRECT` 写库操作**。
3. **状态分级保护（`provisional` vs `verified`）**：
   - **首次遇到（`CREATE`）**：状态一律标为 `provisional`（待二次验证），`hit_count = 1`。下次子代理召回看到 `provisional` 时，会知道“这只是上次记过一次的初稿，需重点核对真伪”；
   - **二次遇到且实测吻合（`CONFIRM`）**：`hit_count` 加 1，状态自动晋升为 `verified`（已坐实），并合并新发现的补充别名或噪音词；
   - **二次遇到发现偏差（`CORRECT`）**：
     - 若原记录为 `provisional`：有明确日志样本即可直接更正字段，并在 `memory_revision_log` 记录差异原因；
     - 若原记录已是多次验证的 `verified`：只有本次通过了主模型“三问自检”拿到实锤日志（或在报告末尾经人工确认）时才允许覆盖更正，同时自动将修改前整行 JSON 备份进 `old_snapshot`，确保随时可一键回滚。
4. **“无辜模块”的客观经验照样沉淀**：
   - 某模块在 `.db` 里模拟电量虚高被列为嫌疑人，子代理查完日志证实它只是正常心跳或短时合法任务（`RULED_OUT_NORMAL`）——此时将其真实的 `Tag`、`normal_behavior` 和无害刷屏词 `noise_keywords` 记入库中，**下次再遇到它虚高时，子代理 10 秒就能核对排除，大幅提速！**

---

## 五、 零依赖记忆库管理脚本参考蓝图（`scripts/stage4_evolve/memory_store.py`）

以下脚本仅使用 Python 标准库（`sqlite3`、`json`、`argparse`、`pathlib`、`datetime`），不依赖任何外部包或私有数据格式，可直接作为 `scripts/stage4_evolve/memory_store.py` 落地使用：

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
功耗审单日志关键字与行为记忆库管理工具（零第三方依赖）
支持子命令：
  1. init      初始化 SQLite 表结构
  2. recall    按嫌犯标识/别名精准召回 + 同槽位类别示范召回（供喂给日志子代理）
  3. apply     主模型结案把关后批量执行 CREATE / CONFIRM / CORRECT 并自动导出 Markdown 快照
  4. rollback  将指定 entity_key 回滚到上一次修改前的快照
  5. export-md 手动导出 references/log_memory_snapshot.md 可读总览表
"""

import argparse
import json
import sqlite3
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, List

SKILL_ROOT = Path(__file__).resolve().parents[2]
DEFAULT_DB_PATH = SKILL_ROOT / "assets" / "memory" / "power_memory.db"
DEFAULT_SNAPSHOT_MD = SKILL_ROOT / "references" / "log_memory_snapshot.md"

DDL_SCRIPT = """
CREATE TABLE IF NOT EXISTS module_log_memory (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key          TEXT NOT NULL UNIQUE,
    entity_type         TEXT NOT NULL,
    slot_category       TEXT NOT NULL,
    aliases_and_tags    TEXT DEFAULT '[]',
    effective_keywords  TEXT DEFAULT '[]',
    noise_keywords      TEXT DEFAULT '[]',
    normal_behavior     TEXT DEFAULT '',
    abnormal_behavior   TEXT DEFAULT '',
    owner_team_hint     TEXT DEFAULT '',
    status              TEXT DEFAULT 'provisional',
    hit_count           INTEGER DEFAULT 1,
    last_ticket_id      TEXT DEFAULT '',
    updated_at          TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_memory_slot_status
    ON module_log_memory (slot_category, status);

CREATE TABLE IF NOT EXISTS memory_revision_log (
    rev_id          INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_key      TEXT NOT NULL,
    action          TEXT NOT NULL,
    diff_reason     TEXT NOT NULL,
    old_snapshot    TEXT DEFAULT NULL,
    new_snapshot    TEXT NOT NULL,
    log_sample      TEXT DEFAULT '',
    ticket_id       TEXT DEFAULT '',
    created_at      TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_rev_entity
    ON memory_revision_log (entity_key, rev_id DESC);
"""


def now_iso() -> str:
    return datetime.now(timezone.utc).astimezone().isoformat(timespec="seconds")


def merge_unique_list(old_json: str, new_items: List[str], replace: bool = False) -> str:
    if replace:
        cleaned = [x.strip() for x in new_items if x and x.strip()]
        return json.dumps(list(dict.fromkeys(cleaned)), ensure_ascii=False)
    try:
        base = json.loads(old_json) if old_json else []
    except json.JSONDecodeError:
        base = []
    merged = list(dict.fromkeys([x.strip() for x in (base + new_items) if x and x.strip()]))
    return json.dumps(merged, ensure_ascii=False)


def row_to_dict(row: sqlite3.Row) -> Dict[str, Any]:
    d = dict(row)
    for k in ("aliases_and_tags", "effective_keywords", "noise_keywords"):
        if isinstance(d.get(k), str):
            try:
                d[k] = json.loads(d[k])
            except json.JSONDecodeError:
                d[k] = []
    return d


def connect_db(db_path: Path) -> sqlite3.Connection:
    db_path.parent.mkdir(parents=True, exist_ok=True)
    conn = sqlite3.connect(str(db_path), timeout=10.0)
    conn.row_factory = sqlite3.Row
    conn.executescript(DDL_SCRIPT)
    return conn


def cmd_recall(db_path: Path, entities: List[str], slot_category: str = "") -> Dict[str, Any]:
    conn = connect_db(db_path)
    exact_matches: List[Dict[str, Any]] = []
    seen_keys = set()

    for ent in entities:
        ent_clean = ent.strip()
        if not ent_clean:
            continue
        rows = conn.execute(
            """
            SELECT * FROM module_log_memory
            WHERE LOWER(entity_key) = LOWER(?)
               OR LOWER(aliases_and_tags) LIKE LOWER(?)
            """,
            (ent_clean, f'%"{ent_clean}"%'),
        ).fetchall()
        for r in rows:
            if r["entity_key"] not in seen_keys:
                seen_keys.add(r["entity_key"])
                exact_matches.append(row_to_dict(r))

    category_examples: List[Dict[str, Any]] = []
    if slot_category:
        rows = conn.execute(
            """
            SELECT * FROM module_log_memory
            WHERE slot_category = ? AND status = 'verified'
            ORDER BY hit_count DESC, updated_at DESC
            LIMIT 2
            """,
            (slot_category,),
        ).fetchall()
        for r in rows:
            if r["entity_key"] not in seen_keys:
                category_examples.append(row_to_dict(r))

    conn.close()
    return {
        "exact_matches": exact_matches,
        "category_examples": category_examples,
    }


def export_markdown_snapshot(conn: sqlite3.Connection, md_path: Path) -> None:
    rows = conn.execute(
        "SELECT * FROM module_log_memory ORDER BY status DESC, hit_count DESC, entity_key ASC"
    ).fetchall()
    md_path.parent.mkdir(parents=True, exist_ok=True)

    lines = [
        "# 模块日志关键字与行为记忆库快照（自动生成，请勿直接手改）",
        "",
        f"> **最后同步时间**：`{now_iso()}` | **累计沉淀模块数**：`{len(rows)}`  ",
        "> **说明**：本文件由 `scripts/stage4_evolve/memory_store.py` 从 `assets/memory/power_memory.db` 自动导出，供日常审阅与版本对比。如需人工修正，请通过 `memory_store.py` 更新数据库。",
        "",
        "| 模块标识 (`entity_key`) | 类型 / 维度 | 状态 (命中数) | 关联 Tag / 别名 | 有效关键字 (`✅`) | 噪音/避坑词 (`❌`) | 正常日志行为 | 异常高耗行为 & 转交方向 |",
        "| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |",
    ]

    for r in rows:
        d = row_to_dict(r)
        cell = lambda s: str(s).replace("|", "\\|").replace("\n", "<br>")
        tags = ", ".join(f"`{x}`" for x in d["aliases_and_tags"]) or "-"
        eff = ", ".join(f"`{x}`" for x in d["effective_keywords"]) or "-"
        noise = ", ".join(f"`{x}`" for x in d["noise_keywords"]) or "-"
        abn = d["abnormal_behavior"] or "-"
        if d["owner_team_hint"]:
            abn = f"{abn}（→ {d['owner_team_hint']}）"
        lines.append(
            f"| `{cell(d['entity_key'])}` | `{cell(d['entity_type'])}`<br>`{cell(d['slot_category'])}` "
            f"| `{cell(d['status'])}` ({d['hit_count']}) | {cell(tags)} | {cell(eff)} | {cell(noise)} "
            f"| {cell(d['normal_behavior'] or '-')} | {cell(abn)} |"
        )

    md_path.write_text("\n".join(lines) + "\n", encoding="utf-8")


def cmd_apply(db_path: Path, md_path: Path, proposals: List[Dict[str, Any]], ticket_id: str = "") -> List[Dict[str, str]]:
    conn = connect_db(db_path)
    ts = now_iso()
    results = []

    with conn:
        for p in proposals:
            key = p["entity_key"].strip()
            req_action = p.get("action", "CREATE").upper()
            existing = conn.execute(
                "SELECT * FROM module_log_memory WHERE entity_key = ?", (key,)
            ).fetchone()

            if existing is None:
                # 首次写入：状态一律定为 provisional
                actual_action = "CREATE"
                old_snap_str = None
                conn.execute(
                    """
                    INSERT INTO module_log_memory (
                        entity_key, entity_type, slot_category,
                        aliases_and_tags, effective_keywords, noise_keywords,
                        normal_behavior, abnormal_behavior, owner_team_hint,
                        status, hit_count, last_ticket_id, updated_at
                    ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 'provisional', 1, ?, ?)
                    """,
                    (
                        key,
                        p.get("entity_type", "app"),
                        p.get("slot_category", "general"),
                        merge_unique_list("[]", p.get("aliases_and_tags", [])),
                        merge_unique_list("[]", p.get("effective_keywords", [])),
                        merge_unique_list("[]", p.get("noise_keywords", [])),
                        p.get("normal_behavior", ""),
                        p.get("abnormal_behavior", ""),
                        p.get("owner_team_hint", ""),
                        ticket_id,
                        ts,
                    ),
                )
            elif req_action == "CONFIRM":
                # 二次对比吻合：命中数 +1，状态自动晋升为 verified，并增量合并新发现的 Tag/关键字/噪音词
                actual_action = "CONFIRM"
                old_snap_str = json.dumps(row_to_dict(existing), ensure_ascii=False)
                new_hits = int(existing["hit_count"]) + 1
                conn.execute(
                    """
                    UPDATE module_log_memory
                    SET aliases_and_tags   = ?,
                        effective_keywords = ?,
                        noise_keywords     = ?,
                        normal_behavior    = COALESCE(NULLIF(?, ''), normal_behavior),
                        abnormal_behavior  = COALESCE(NULLIF(?, ''), abnormal_behavior),
                        owner_team_hint    = COALESCE(NULLIF(?, ''), owner_team_hint),
                        status             = 'verified',
                        hit_count          = ?,
                        last_ticket_id     = ?,
                        updated_at         = ?
                    WHERE entity_key = ?
                    """,
                    (
                        merge_unique_list(existing["aliases_and_tags"], p.get("aliases_and_tags", [])),
                        merge_unique_list(existing["effective_keywords"], p.get("effective_keywords", [])),
                        merge_unique_list(existing["noise_keywords"], p.get("noise_keywords", [])),
                        p.get("normal_behavior", ""),
                        p.get("abnormal_behavior", ""),
                        p.get("owner_team_hint", ""),
                        new_hits,
                        ticket_id,
                        ts,
                        key,
                    ),
                )
            else:
                # CORRECT（或重复提交时原地更正）：用新实测的有效关键字与行为覆盖偏差项，噪音词累加保留
                actual_action = "CORRECT"
                old_snap_str = json.dumps(row_to_dict(existing), ensure_ascii=False)
                new_hits = int(existing["hit_count"]) + 1
                replace_eff = bool(p.get("effective_keywords"))
                conn.execute(
                    """
                    UPDATE module_log_memory
                    SET entity_type        = COALESCE(NULLIF(?, ''), entity_type),
                        slot_category      = COALESCE(NULLIF(?, ''), slot_category),
                        aliases_and_tags   = ?,
                        effective_keywords = ?,
                        noise_keywords     = ?,
                        normal_behavior    = COALESCE(NULLIF(?, ''), normal_behavior),
                        abnormal_behavior  = COALESCE(NULLIF(?, ''), abnormal_behavior),
                        owner_team_hint    = COALESCE(NULLIF(?, ''), owner_team_hint),
                        hit_count          = ?,
                        last_ticket_id     = ?,
                        updated_at         = ?
                    WHERE entity_key = ?
                    """,
                    (
                        p.get("entity_type", ""),
                        p.get("slot_category", ""),
                        merge_unique_list(existing["aliases_and_tags"], p.get("aliases_and_tags", [])),
                        merge_unique_list(existing["effective_keywords"], p.get("effective_keywords", []), replace=replace_eff),
                        merge_unique_list(existing["noise_keywords"], p.get("noise_keywords", [])),
                        p.get("normal_behavior", ""),
                        p.get("abnormal_behavior", ""),
                        p.get("owner_team_hint", ""),
                        new_hits,
                        ticket_id,
                        ts,
                        key,
                    ),
                )

            updated_row = conn.execute(
                "SELECT * FROM module_log_memory WHERE entity_key = ?", (key,)
            ).fetchone()
            new_snap_str = json.dumps(row_to_dict(updated_row), ensure_ascii=False)

            conn.execute(
                """
                INSERT INTO memory_revision_log (
                    entity_key, action, diff_reason, old_snapshot, new_snapshot, log_sample, ticket_id, created_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                """,
                (
                    key,
                    actual_action,
                    p.get("diff_reason", f"执行 {actual_action} 更新"),
                    old_snap_str,
                    new_snap_str,
                    p.get("log_sample", ""),
                    ticket_id,
                    ts,
                ),
            )
            results.append({"entity_key": key, "action": actual_action, "status": updated_row["status"]})

        export_markdown_snapshot(conn, md_path)

    conn.close()
    return results


def cmd_rollback(db_path: Path, md_path: Path, entity_key: str) -> Dict[str, Any]:
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
        return {"ok": False, "reason": f"未找到 {entity_key} 可回滚的历史快照"}

    old = json.loads(rev["old_snapshot"])
    ts = now_iso()
    with conn:
        conn.execute(
            """
            UPDATE module_log_memory
            SET entity_type = ?, slot_category = ?, aliases_and_tags = ?,
                effective_keywords = ?, noise_keywords = ?, normal_behavior = ?,
                abnormal_behavior = ?, owner_team_hint = ?, status = ?,
                hit_count = ?, last_ticket_id = ?, updated_at = ?
            WHERE entity_key = ?
            """,
            (
                old["entity_type"],
                old["slot_category"],
                json.dumps(old["aliases_and_tags"], ensure_ascii=False),
                json.dumps(old["effective_keywords"], ensure_ascii=False),
                json.dumps(old["noise_keywords"], ensure_ascii=False),
                old["normal_behavior"],
                old["abnormal_behavior"],
                old["owner_team_hint"],
                old["status"],
                old["hit_count"],
                old["last_ticket_id"],
                ts,
                entity_key,
            ),
        )
        conn.execute(
            """
            INSERT INTO memory_revision_log (
                entity_key, action, diff_reason, old_snapshot, new_snapshot, ticket_id, created_at
            ) VALUES (?, 'MANUAL', '触发回滚至上一版本快照', ?, ?, '', ?)
            """,
            (entity_key, rev["new_snapshot"], rev["old_snapshot"], ts),
        )
        export_markdown_snapshot(conn, md_path)

    conn.close()
    return {"ok": True, "restored_entity": entity_key}


def main() -> None:
    parser = argparse.ArgumentParser(description="功耗审单日志关键字与行为 SQLite 记忆库管理 CLI")
    parser.add_argument("--db", type=Path, default=DEFAULT_DB_PATH, help="SQLite 数据库文件路径")
    parser.add_argument("--snapshot", type=Path, default=DEFAULT_SNAPSHOT_MD, help="导出的 Markdown 快照路径")
    sub = parser.add_subparsers(dest="cmd", required=True)

    sub.add_parser("init", help="初始化数据库与快照")

    p_recall = sub.add_parser("recall", help="按嫌犯名列表与槽位类别召回历史记忆")
    p_recall.add_argument("--entities", nargs="+", required=True, help="嫌犯包名/锁名/IRQ 列表")
    p_recall.add_argument("--slot", default="", help="异常维度类别（如 off_wakelock）")

    p_apply = sub.add_parser("apply", help="传入变更提案 JSON 文件或字符串更新记忆库")
    p_apply.add_argument("--proposals-json", required=True, help="JSON 文件路径或内联 JSON 数组字符串")
    p_apply.add_argument("--ticket", default="", help="来源单号标识")

    p_rb = sub.add_parser("rollback", help="将指定模块回滚到修改前状态")
    p_rb.add_argument("--entity", required=True, help="要回滚的 entity_key")

    sub.add_parser("export-md", help="重新导出 Markdown 快照表")

    args = parser.parse_args()
    if args.cmd == "init":
        conn = connect_db(args.db)
        export_markdown_snapshot(conn, args.snapshot)
        conn.close()
        print(json.dumps({"ok": True, "db": str(args.db), "snapshot": str(args.snapshot)}, ensure_ascii=False))
    elif args.cmd == "recall":
        res = cmd_recall(args.db, args.entities, args.slot)
        print(json.dumps(res, ensure_ascii=False, indent=2))
    elif args.cmd == "apply":
        raw = args.proposals_json
        p_path = Path(raw)
        data = json.loads(p_path.read_text(encoding="utf-8") if p_path.is_file() else raw)
        proposals = data if isinstance(data, list) else data.get("memory_proposals", [data])
        res = cmd_apply(args.db, args.snapshot, proposals, args.ticket)
        print(json.dumps({"ok": True, "applied": res}, ensure_ascii=False, indent=2))
    elif args.cmd == "rollback":
        res = cmd_rollback(args.db, args.snapshot, args.entity)
        print(json.dumps(res, ensure_ascii=False, indent=2))
    elif args.cmd == "export-md":
        conn = connect_db(args.db)
        export_markdown_snapshot(conn, args.snapshot)
        conn.close()
        print(json.dumps({"ok": True, "snapshot": str(args.snapshot)}, ensure_ascii=False))


if __name__ == "__main__":
    main()
```

---

## 六、 仓库对应文件具体修改映射表

将这套机制合入 `.claude/skills/power-review/` 时，各文件的具体改动如下：

| 序号 | 目标文件路径 | 具体修改内容 |
| :--- | :--- | :--- |
| 1 | `scripts/stage4_evolve/memory_store.py` *(新增)* | 落地第五节的零依赖 SQLite 记忆库管理 CLI 脚本，并运行 `python memory_store.py init` 生成初始库与快照文件。 |
| 2 | `.claude/skills/power-review/SKILL.md` | • **阶段二末尾**：增加调用 `memory_store.py recall` 按 `.db` 嫌犯列表精准召回 SQLite 历史关键字与行为记忆；<br>• **阶段三第 3 步**：删除“仅调用固定脚本查日志”的表述，改为**派发独立【日志调查子代理】**（带时间围栏 `[T_start - 10min, T_end]`、`Ref` 对照窗与历史记忆，执行“先对比验证旧关键字 $\rightarrow$ 再自由探索 $\rightarrow$ `Ref` 段反证”）；<br>• **阶段四**：加入主模型过完三问自检后调用 `memory_store.py apply` 执行首次记录（`CREATE`）、二次确认（`CONFIRM`）或偏差更正（`CORRECT`），并在审单报告末尾展示本次记忆库变更摘要。 |
| 3 | `.claude/skills/power-review/references/general_guide.md` | 更新第四章“查日志通用破案思路”，补充子代理自由探索的四步纪律（先避开 `noise_keywords` 核验 `effective_keywords`、时间围栏约束、`Ref` 正常段频次反证、连同无辜模块的正常行为一并沉淀）。 |
| 4 | `.claude/skills/power-review/assets/templates/iteration_patch_card.md` | 将原有的纯 Markdown 卡片追加模板，升级为 **【一、SQLite 关键字与行为记忆库变更明细（`CREATE / CONFIRM / CORRECT`）】** + **【二、全局通用规则补充建议（可选）】** 双栏结构，清晰展示“本次新记了哪些关键字/行为，或更正了上次哪条偏差记录”。 |
| 5 | `.claude/skills/power-review/scripts/stage3_log/README.md` | 将定位从“大模型唯一查日志入口”降级调整为 **“供日志调查子代理按需选用的大文件时间窗裁剪与脱敏预处理辅助工具”**（普通体积日志由子代理直接用原生检索/阅读自由探索；若内网已删除旧查日志工具脚本，仅保留时间格式对齐与大压缩包切片能力即可）。 |
| 6 | `.claude/skills/power-review/scripts/stage4_evolve/README.md` | 更新职责说明：核心负责维护 `assets/memory/power_memory.db`（首次记录、二次对比确认/更正、快照备份回滚）及自动导出 `references/log_memory_snapshot.md`。 |

---

## 七、 附：回内网一键交给 AI 自动落地改造的指令模板

将本文档放入项目根目录后，可直接把下面这段话发给内网的 Coding Agent，让它一次性把 SQLite 记忆库与子代理流程接好：

```text
请完整阅读根目录下的 `log_subagent_memory_upgrade.md`，按文档第六节的“文件修改映射表”对 `.claude/skills/power-review/` 完成阶段三与阶段四升级：
1. 新建 `.claude/skills/power-review/scripts/stage4_evolve/memory_store.py`（实现 `init / recall / apply / rollback / export-md` 五个子命令），运行 `init` 初始化 `assets/memory/power_memory.db` 与 `references/log_memory_snapshot.md`，并用一条模拟数据测试 `CREATE -> CONFIRM -> CORRECT -> rollback` 全链路无报错；
2. 更新 `.claude/skills/power-review/SKILL.md`、`references/general_guide.md`、`assets/templates/iteration_patch_card.md` 以及 `scripts/stage3_log/README.md`、`scripts/stage4_evolve/README.md`：
   - 将阶段三查日志正式切换为“主模型按嫌犯从 SQLite 召回记忆 -> 派发带时间围栏与 Ref 对照窗的【日志调查子代理】（先对比验证种子关键字、再自由探索、必做 Ref 反证、只读回传 JSON）”；
   - 将阶段四切换为“主模型三问自检把关后单点调用 `memory_store.py apply` 完成首次记录或二次对比更正，并自动刷新可读快照表”。
注意保持所有规则与说明的通用性，不要写入任何硬编码的虚构私有业务阈值。
```
