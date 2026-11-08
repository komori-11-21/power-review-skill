# `scripts/core/`：公共基础底座

## 主要实现什么
存放所有阶段共用的基础数据结构、配置加载器与只读数据库访问封装，不包含任何具体业务 SQL 或日志规则。

## 计划包含的能力（均不依赖真实 `.db`/日志结构）
1. **统一数据结构定义**（如 `models`）：
   - 电池原始采样点结构（时间戳、充放电、亮灭屏、剩余 mAh / 百分比、电流、温度）；
   - 放电时间切片 `TimeSlice`（异常标记、真实均流、掉电率、均温/最高温）；
   - 4 列对比表行 `MetricRow`、单维度查询结果 `SlotResult`、知识库卡片与最终证据包结构。
2. **配置读取与模板渲染**（如 `config_loader`）：
   - 自动定位 Skill 根目录下的 `assets/config/thresholds.json`、`assets/templates/` 与 `references/` 文档。
3. **只读 SQLite 客户端**（如 `db_client`）：
   - 以只读 URI 模式安全打开本地 `.db` 文件，防误写或破坏原问题单数据库，提供统一查询接口。
