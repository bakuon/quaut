# 任务 T-04：历史回放与策略回测

> 项目背景与硬约束见仓库根目录 `AGENTS.md`，本文件只写本轮要做的事。

---

## 1. 目标（一句话）

新增**历史竞价回放**能力，并用它与**回测模块**在历史数据上验证 `auction_filter` 策略的表现
（胜率、盈亏比、最大回撤），结果在前端可视化，为是否用于实盘提供依据。

## 2. 本轮范围

**要做**：

1. 实现 **`ReplayProvider`**（`AGENTS.md` 第 7 节）：按指定交易日回放历史竞价数据，
   与 `MockProvider` 共用统一 `DataProvider` 接口，可被 `runner` 透明替换。
2. 实现**回测引擎**：复用 `auction_filter` **同一套纯函数**（不得复制一份改写的算法），
   在历史区间上逐日执行筛选。
3. 实现**绩效指标**：胜率、盈亏比、最大回撤、平均收益、交易次数等，
   并明确每项指标的**计算口径**（收益基准用开盘价 / 收盘价须写明）。
4. 新增回测触发与查询接口：`POST /api/backtest/run`（提交回测任务）、
   `GET /api/backtest/result`（读取回测结果）、`GET /api/backtest/tasks`（回测任务列表）。
5. 前端新增**回测页**：选择日期区间与参数、触发回测、用 ECharts 展示净值曲线与指标卡片。
6. 回测结果落库持久化，支持历史回测记录查询。

**明确不做**：

1. **不做实盘下单与模拟交易撮合**——回测只做"选出后按规则统计表现"，不接券商（第 12 节自动交易为后期预留）。
2. 不做多用户与权限（属第 12 节后期预留）。
3. **不引入 Backtrader / VNPY / RQAlpha 等外部回测框架**——当前只有一个策略，
   直接复用本项目的纯函数式引擎，**不得为回测引入重型依赖**。
4. 不改动 `auction_filter` 的筛选规则本身；若回测发现规则需调整，**另开任务处理**。
5. 不做超大规模数据的性能优化（单次回测规模为若干交易日的全市场快照）。

## 3. 文件锚点

- 新增：
  - `backend/app/datasource/replay_provider.py`
  - `backend/app/engine/backtest/__init__.py`
  - `backend/app/engine/backtest/engine.py`（区间回放 + 逐日执行策略）
  - `backend/app/engine/backtest/metrics.py`（胜率 / 盈亏比 / 最大回撤等）
  - `backend/app/api/backtest.py`
  - `backend/app/schemas/backtest.py`
  - `backend/app/models/backtest.py`（`backtest_task` / `backtest_result` 两张表）
  - `backend/tests/test_backtest_metrics.py`
  - `backend/tests/test_replay_provider.py`
  - `frontend/src/views/Backtest.vue`、`frontend/src/api/backtest.ts`
- 修改：
  - `backend/app/datasource/registry.py`（注册 `ReplayProvider`）
  - `backend/app/scheduler/calendar_guard.py`（回测按交易日历遍历区间，跳过非交易日）
  - `backend/app/models/__init__.py`（导出新模型）
  - `frontend/src/router/index.ts`（新增回测路由）
  - `backend/alembic/`（新增回测表迁移）

## 4. 实现要求

- **算法必须复用，不得复制**：回测引擎调用的是 `auction_filter` 的**同一个纯函数**
  （`AGENTS.md` 第 5 节：入参 DataFrame、出参 DataFrame、与 IO 解耦）；
  严禁为回测另写一份筛选逻辑，否则两边规则一旦分叉，回测结论即失效。
- **绩效指标必须写明计算口径**：每项指标在代码注释与前端说明中都要交代——
  收益基准（开盘价 / 收盘价 / 次日开盘等）、持有周期、是否剔除涨跌停无法成交的样本、
  最大回撤按净值序列还是单笔收益计算。**口径不清的回测数字没有参考价值。**
- **回测须防未来函数**：仅使用当日 9:20 之后、且决策时点（9:25）之前可得的数据
  （第 2 节）；严禁让回测使用当日收盘等未来信息。
- **回测参数走配置 / 接口入参**，不得硬编码；`ReplayProvider` 的数据来源需可配置
  （本地快照库，或经容灾链路取历史数据）。
- `ReplayProvider` 必须与 `MockProvider` 实现**同一接口**，可被 `runner` 无感替换（第 7 节）。
- 遵循 `AGENTS.md` 第 13 节工程规范：全量类型注解、`.env` 配置、结构化日志 + `trace_id`、
  无 TODO 与伪代码；外部依赖全部有 Mock 兜底（第 14 节第 2 条）。

## 5. 验收标准

1. 命令：
   - `cd backend && pytest tests/ -v`
   - `cd backend && pytest tests/test_backtest_metrics.py tests/test_replay_provider.py -v`
   - `curl -X POST http://localhost:8000/api/backtest/run`（以 Mock / 本地历史快照数据）
2. 可判定条件：
   - [ ] 回测引擎调用的是与实盘**同一个** `auction_filter` 函数（有测试或代码引用可证明）。
   - [ ] `ReplayProvider` 与 `MockProvider` 可互换，`runner` 无需改动（有对应测试）。
   - [ ] 绩效指标（胜率 / 盈亏比 / 最大回撤 / 平均收益 / 交易次数）计算正确（以人工构造样本校验）。
   - [ ] 回测不跳过非交易日、不使用未来数据（有对应测试）。
   - [ ] 回测结果落库，可通过接口查询到历史回测记录。
   - [ ] 前端回测页可选择区间、触发回测并展示净值曲线与指标。
   - [ ] `pytest tests/` 全部通过。

## 6. 交付物

- [ ] `ReplayProvider` 历史回放实现
- [ ] 回测引擎与绩效指标模块（含口径说明）
- [ ] 回测接口与结果落库
- [ ] 前端回测页（参数选择 + 净值曲线 + 指标卡片）
- [ ] 单元测试（指标计算、回放、防未来函数）
- [ ] 回测指标口径说明文档（可放在页面内或 README 小节）

## 7. 开始前请先回答（先不要写代码）

1. 你打算怎么保证回测用的是与实盘**同一个**策略函数，而不是复制一份？
2. 绩效指标你打算都算哪些？每一项的**计算口径**分别是什么（收益基准、持有周期、样本剔除规则）？
3. 你打算怎么规避**未来函数**？决策时点与可用数据的时间边界如何界定？
4. `ReplayProvider` 的历史数据从哪来、如何落库与读取？数据缺口如何处理？
5. 有哪些假设或不确定点需要我确认（尤其是回测的默认区间与持有周期）？

## 8. 完成后请按此格式汇报

- 改动文件：
- 验证命令与结果：
- 遗留问题：
