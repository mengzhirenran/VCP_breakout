# E1 实验执行日志

实验编号：E1_ATR_Compression
开始时间：2026-11-25T12:26+08:00

---

## Phase 0 — 资料读取与环境确认 ✅ 完成

- 读取 environment_audit.md / environment_audit_summary.json（本次审计产出）。
- 检索 strategy_catalog.md / experiment_design.md / tool_ecosystem.md：文件系统（WorkBuddy 全项目 maxdepth 6）、历史对话（近7日）、资料库/知识库均未找到。按任务书要求**未创建替代文件**；experiment_context.md 仅基于任务书与环境审计形成。
- 目录结构已建立；config.json 参数已冻结（2026-11-25T12:28）。
- 解释器验证：compute=3.13.14/pandas3.0.5/numpy2.5.1；stats=3.13.13/scipy1.18.1/matplotlib3.11.2，与配置一致。

## Phase 1 — 数据质量审计 ⚠️ 发现重大数据问题（已记录，继续）

- 读取 ETF 主库：73 只 ETF，110,682 行，2014-02-26~2026-11-24。
- **重大发现 1（high/low 系统性错位）**：逐 ETF 检查发现，72/73 只 ETF 在 2026-05 之前存在高比例 OHLC 关系异常（high<low 或 low>open/close）。全库按年份统计：2014-2025 年异常率 85%~98%，2026-06 起降为 0。定位原因：历史回填脚本 `backfill_history.py`/`backfill.py` 解析的第三方 kline 输出字段错位，且该数据源脚本已不存在，无法复核原始列顺序。
- **重大发现 2（价格口径未知）**：Wind MCP 核验 510300.SH，2014-03 后复权收盘 2.25 附近，本地库同日收盘 1.36，比例 1.65~1.69 且不恒定；本地价格疑似某种复权口径但无法确认。历史区间 volume/amount 多为 0。不能标记为"已复权"。
- **重大发现 3（指数库干净）**：`bias_model.db.index_daily` 含 7 个宽基/风格指数（000300.SH、000852.SH、000985.CSI、399006.SZ、399370.SZ、399371.SZ、000688.SH），2011-08-02~2026-11-24，共 25,978 行，OHLC 异常率 **0%**，成交量额正常。

### 处理决策（不改变研究假设与参数）

1. **不修改原始数据库**，不使用推测规则"修复"ETF 的 high/low（如把 max/min 重排），因为无法验证真实最高/最低价，违反数据真实性原则。
2. **不擅自把 ATR 改用仅 close 的波动率**（那会改变任务书固定的 True Range 定义）。
3. 按任务书 3.5 的"补充样本单独研究"原则，采用**双层样本**：
   - **主分析样本 = 指数库 7 个指数**（数据干净、ATR 定义可严格执行）。这是本轮可靠结论的来源。
   - **ETF 73 只 = 条件性参考样本**：仅对 high/low 正常的 2026-06 之后区间计算 ATR 事件（样本期过短，不足以独立下结论），历史区间 ATR 结果标记为不可靠、不作为结论依据。
   - 两类样本**严格分开报告，不混合统计**。
4. 价格口径：ETF 与指数的本地价格口径均标记为"未知/未经确认"；Wind 交叉核验结果写入附录；未来收益结果按任务书要求标记为**有条件结果**，不解释为可靠投资收益预测。
5. 该数据问题已如实写入 data_quality_report.md 与价格口径附录。它属于"数据质量不满足主样本要求"，但不构成整体阻塞——因为干净的指数样本可完成 E1 全部研究目标（ATR 收缩是否有独立预测价值）。最终结论等级将据此审慎判定。

### 人工是否需要介入

本问题**不需要**人工授权即可继续（有合法干净样本完成研究）。若要让 73 只 ETF 成为可靠主样本，需人工重建历史行情库（属于环境数据修复，超出本轮授权），已在报告"局限与建议"中说明，本轮不擅自执行。

---

## Phase 2 — 指标计算 ✅ 完成

- `indicators.py`：TR（numpy 实现）、Wilder ATR(10/50)、ATRRatio、每日状态、首次进入事件（含冷却期）。
- `test_indicators.py`：12 项必测 + 3 项附加测试，15/15 PASS。
  - 测试中发现并修复真实缺陷：TR 在缺失 close 时用 `fillna(h−l)` 掩盖缺失 → 改为保持 NaN（首行单独处理）。
- 清理 `first_crossing_events` 死代码 1 处。

## Phase 3 — 未来指标计算 ✅ 完成

- `event_study.py`：生成 daily_states（index 23,979 行 / etf_clean 8,760 行）、events（6,669 行）、forward_returns、volatility_metrics、mfe_mae、control_noncontraction（55,694 行）。
  - 修复 `_forward_metrics` 中 code/name 未传参的缺陷（3 处调用点）。
- `run_cooldown.py`：生成 20/60 日冷却期事件 5,930 行。
- `daily_forward.py`：生成全交易日未来指标表 123,356 行（供 G1-G5 梯度、横截面、分层分析）。
  - 修复市场环境 regime_map 的索引对齐错误（布尔索引与日期索引不匹配）。

## Phase 4-5 — 对照组/统计检验/稳健性 ✅ 完成

- `statistics.py`（VibeTrading 环境，scipy）：Welch t、Mann-Whitney U、按 code 分层 cluster Bootstrap（2,000 次，seed=20261125）、BH-FDR 校正、梯度/分层/冷却期/横截面分析。
  - 首次运行因 `events_cooldown.csv` 缺 `dataset` 列失败 → 用 `sample_group` 推断修复后重跑成功（耗时 286.8s）。
- 输出：statistical_tests.csv、group_stats.csv、gradient_results.csv、yearly_results.csv、etf_level_results.csv、market_regime_results.csv、robustness_results.csv、cross_sectional_comparison.csv、result_summary.json。

## Phase 6 — 结果复核 ✅ 完成

- `verification.py`（独立实现，VibeTrading 环境）：
  - 20 个随机事件逐项核对：ATRRatio 误差 0，收益/RV/MFE 误差 <1e-16，20/20 精确匹配。
  - 首次进入事件 1,334/1,334 正确；无未来函数 10/10 通过；阈值单调性、行数勾稽通过。
  - 独立复算 Welch t：C4 h20 收益差 -0.00688549 与保存值一致，p=0.0106。
  - 修复复算取数缺陷（drop_duplicates 后仅保留 h=5，导致 h=20 为空）→ 从完整 events 表取数后通过。
- 结果：verification_report.json，全部通过，无已知错误带入结论。

## Phase 7 — 报告与图表 ✅ 完成

- `visualization.py`：12 张 PNG 图表（深蓝/橙配色，图例置于绘图区下方，柱状标注数值）。
  - 图8/9 因 gradient_results.csv 列名 `group`≠`g_group` 首次失败 → 修复后重生成。
- `E1_ATR_Compression_Report.md`：16 节完整报告，含 6 个必答问题、ABC 结论（**C 不支持**）、E2 建议。

---

## 最终状态

E1 完整执行完毕。最终结论：**C — 不支持**（ATR Compression 无独立正向收益预测价值）。
主样本：7 个干净指数（原定 73 只 ETF 因历史 high/low 错位降级为条件性参考）。
有效事件：指数去重 C1=6/C2=120/C3=414/C4=794；ETF clean C1=23/C2=96/C3=183/C4=145。
未解决问题：ETF 历史行情需人工重建（超出本轮授权）；强收缩样本不足。
执行纪律：未安装软件包、未改环境/Wind 配置、未修改原始数据库、未输出任何凭证、未开展 E2。
