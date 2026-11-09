# 任务 T-01：Mock 模式下跑通集合竞价筛选全链路

> 项目背景与硬约束见仓库根目录 `AGENTS.md`，本文件只写本轮要做的事。

---

## 1. 目标（一句话）

在**不依赖任何外部 API Key** 的前提下，用内置 Mock 数据跑通
"取数 → 筛选 → 落库 → 推送 → 前端展示"完整链路，并能在看板看到候选股结果。

## 2. 本轮范围

**要做**：

1. 搭出 `backend` 目录骨架（`main.py` / `core` / `api` / `engine` / `datasource` /
   `models` / `schemas`），按 `AGENTS.md` 第 6 节的目录结构落地。
2. 实现 `MockProvider`（`datasource/`）与 `auction_filter` 策略
   （`engine/strategies/`），并搭好 `Strategy` 基类与注册表（第 8 节）。
3. 实现 `POST /api/selection/run` 手动触发、`GET /api/selection/latest` 读取结果，
   以及 `/ws/selection` 推送（第 9 节接口契约）。
4. 前端做一个最小看板页，接入 `/ws/selection` 并渲染候选股表格。

**明确不做**：

1. 不接任何真实数据源（`TushareProvider` 与商业竞价 Provider 留到 T-02）。
2. 不实现多源容灾切换、不接定时调度（APScheduler 留到 T-03）。
3. 不做多用户与权限（属 `AGENTS.md` 第 12 节的后期预留）。

## 3. 文件锚点

- 新增：
  - `backend/app/main.py`
  - `backend/app/core/{config.py, logging.py, calendar.py}`
  - `backend/app/datasource/{base.py, mock.py}`
  - `backend/app/engine/strategies/{base.py, auction_filter.py}`
  - `backend/app/engine/runner.py`
  - `backend/app/api/selection.py`、`backend/app/ws/selection.py`
  - `backend/app/models/selection.py`、`backend/app/schemas/selection.py`
  - `backend/tests/test_auction_filter.py`
  - `frontend/src/views/SelectionBoard.vue`、`frontend/src/api/selection.ts`
- 修改：
  - `backend/requirements.txt`（按 `AGENTS.md` 第 5 节锁定 pandas / numpy 版本）

## 4. 实现要求

- `MockProvider` 返回的 DataFrame 字段必须与 `AGENTS.md` 第 9 节
  "选股结果字段"完全一致（`code` / `name` / `auction_price` / `auction_chg` /
  `auction_amount` / `matched_vol` / `buy_unmatch_vol` / `sell_unmatch_vol` /
  `total_mv` / `score`）。
- `auction_filter` 必须是**纯函数**：入参 DataFrame、出参 DataFrame，与 IO 完全解耦
  （`AGENTS.md` 第 5 节）。
- 筛选阈值一律从配置读取，不得写死在函数里；每个阈值加注释说明含义、单位与来源
  （`AGENTS.md` 第 3 节）。
- Mock 数据需覆盖边界情形：空数据、缺字段、脏数据、全部被过滤为空的情况，
  用于验证防御逻辑。
- 遵循 `AGENTS.md` 第 13 节（工程规范）：全量类型注解、`.env` 配置、
  结构化日志、非交易日跳过、无 TODO 与伪代码。

## 5. 验收标准

1. 命令：
   - `cd backend && uvicorn app.main:app --reload`
   - `cd frontend && npm run dev`
   - `cd backend && pytest tests/ -v`
2. 可判定条件：
   - [ ] 后端与前端均可正常启动，无报错。
   - [ ] `POST /api/selection/run` 返回非空候选股列表，字段与第 9 节一致。
   - [ ] 前端页面能实时收到 `/ws/selection` 推送并渲染表格。
   - [ ] `pytest tests/` 全部通过，覆盖算法主流程与边界情形。
   - [ ] 空数据 / 全被过滤时接口不报 500，返回空列表。

## 6. 交付物

- [ ] 可运行的 backend / frontend 最小工程
- [ ] 单元测试文件
- [ ] 本轮启动说明（如何以 Mock 模式跑起来）

## 7. 开始前请先回答（先不要写代码）

1. 你理解的本轮链路与分层是什么？各层边界如何划分？
2. 你打算分几步做？每步产出什么、如何验证？
3. 字段定义与阈值有哪些需要我确认？Mock 数据你打算怎么造？

## 8. 完成后请按此格式汇报

- 改动文件：
- 验证命令与结果：
- 遗留问题：
