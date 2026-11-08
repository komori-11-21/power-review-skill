# `scripts/stage1_db/slots/screen_off/`：灭屏待机分支查询槽位

## 主要实现什么
对应 [screen_off_rules.md](../../../../references/screen_off_rules.md)，在真实 `.db` 中提取灭屏异常段与 `Ref` 段的 7 个排查维度数据：

1. **槽位 1（L1 必选）**：AP 深睡时长、子系统（AOC/WiFi/BT/Modem）低功耗时长、全器件模拟能量总和 $\rightarrow$ 算深睡率与模拟覆盖率，供流水线识别 `< 80%` 深睡异常；覆盖率 `< 35%` 时先排查是否缺器件表/未换算单位，确认无误再标记存在未解释耗电（盲区/外设漏电）；
2. **槽位 2（L2 硬证据）**：后台持锁（RunningLock / Wakelock）按锁/包名/代理标识统计持锁时长占比，及后台冻结失败次数；
3. **槽位 3（L2 硬证据）**：内核硬件唤醒源（Wakeup IRQ）频次（`次/h`）与应用层 Alarm 定时器唤醒频次；
4. **槽位 4（L2 归因）**：全硬件器件（CPU/WiFi/BT/Audio/Sensor 等）与后台应用灭屏模拟占比 `Share %`；
5. **槽位 5（L3 深挖）**：后台线程（`Thread_*`）真实 CPU 时长占比、灭屏收发包率与 Wi-Fi/BLE/GPS 扫描频次（缺表附 `hiperf` 提示）；
6. **槽位 6（L3 平板专属）**：手写笔吸附/无线充/充断切换、磁吸键盘连接、皮套霍尔中断、Type-C OTG 反向供电状态；
7. **槽位 7（L3 直通车）**：时间窗内端侧捕获的 `AbnormalEvents` 与调用栈摘录。
