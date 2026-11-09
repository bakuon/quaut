# 任务 T-02：接入真实数据源（基础面层 + 竞价层）

> 项目背景与硬约束见仓库根目录 `AGENTS.md`，本文件只写本轮要做的事。

---

## 1. 目标（一句话）

把数据源层从"只有 Mock"扩展为"Mock + TushareProvider（基础面）+ 商业竞价 Provider 模板"**三者按统一接口可插拔**；填入真实 Key 后能拉到真实数据，未填 Key 时**自动降级到 Mock 且不报错崩溃**。

## 2. 本轮范围

**要做**：

1. 按 `AGENTS.md` 第 7 节实现 `TushareProvider`，负责**基础面层**：股票列表、`daily_basic`（市值 / 换手率 / 量比）、交易日历。
2. 实现**商业竞价 Provider 模板**（StockApi / TickDB 风格），负责 `fetch_auction`：鉴权、请求封装、字段映射，Key 走环境变量。
3. 建立 **Provider 注册与优先级路由**（按"数据类型 × 市场"配置链路），`runner` 只依赖 `DataProvider` 接口，不感知具体来源。
4. **无 Key / 接口不可用时的自动降级**到 `MockProvider`，且日志给出明确提示（不是静默降级）。
5. 实现各 Provider 的 `health_check`，以及**数据新鲜度（时间戳）校验**；补上 `GET /api/datasource/status` 与 `/ws/datasource`。
6. **基础面数据盘后落库**到 `stock_base` 表（第 10 节），次日从本地读取，**不实时现拉**（第 7 节分层取数原则）。

**明确不做**：

1. **不实现完整容灾机制**（熔断器、冷却隔离、主备切换滞后判定、定期健康探测、跨源语义比对）——留到 T-03。
2. 不接 APScheduler 定时调度（9:20 / 9:24:50 自动触发）——留到 T-03。
3. 不做历史回放与回测——留到 T-04。
4. 不把任何付费 Key 作为默认启动路径；**默认仍必须能纯 Mock 跑通**。

## 3. 文件锚点

- 新增：
  - `backend/app/datasource/tushare_provider.py`
  - `backend/app/datasource/auction_provider.py`（商业竞价 Provider 模板）
  - `backend/app/datasource/registry.py`（Provider 注册与优先级路由）
  - `backend/app/datasource/field_mapping.py`（各源字段 → 统一字段映射）
  - `backend/app/services/stock_base_sync.py`（盘后基础面落库）
  - `backend/app/api/datasource.py`、`backend/app/ws/datasource.py`
  - `backend/app/scripts/sync_stock_base.py`（手动触发同步的入口）
  - `backend/tests/test_datasource_fallback.py`
  - `backend/tests/test_field_mapping.py`
- 修改：
  - `backend/app/datasource/base.py`（补齐并固化统一接口）
  - `backend/app/datasource/mock.py`（对齐接口，明确其作为降级目标）
  - `backend/app/engine/runner.py`（改为经注册路由取数，不直接实例化具体 Provider）
  - `backend/app/core/config.py`（新增数据源相关配置项）
  - `backend/.env.example`（新增 `TUSHARE_TOKEN`、`AUCTION_API_KEY` 等占位与注释）
  - `backend/requirements.txt`（新增数据源 SDK 依赖，版本须显式锁定）

## 4. 实现要求

- 所有 Provider 必须实现 `AGENTS.md` 第 7 节的**统一签名**：`fetch_auction(codes)`、`fetch_base_info()`、`health_check()`；上层只依赖接口。
- **字段映射集中管理**：各源原始字段一律经 `field_mapping` 统一映射到第 9 节"选股结果字段"（`code` / `name` / `auction_price` / `auction_chg` / `auction_amount` / `matched_vol` / `buy_unmatch_vol` / `sell_unmatch_vol` / `total_mv` / `score`）。**未知字段不得猜测，缺失字段必须显式处理**（标记缺失或降级），不得静默填 0。
- **能力边界必须如实标注**：`TushareProvider` 只承担基础面层；免费版在竞价时段**拿不到竞价明细**，`fetch_auction` 不得假装能返回竞价字段。竞价层由商业竞价 Provider 负责。
- 所有 Key 走 `.env`（第 13 节），**不得硬编码、不得写入日志或前端响应**。
- **降级链路**：竞价层优先级为「商业竞价 Provider → MockProvider」；基础面层为「TushareProvider → 本地 `stock_base` 缓存 → MockProvider」。任一环无 Key 或调用失败，降级而非中断。
- 竞价数据**必须是 9:20 之后的数据**（第 2 节时间铁律）；对 9:20 前的数据直接丢弃，不得入库、不得参与筛选。
- 基础面数据落库 `stock_base`（第 10 节，带索引）；同步失败不影响 Mock 模式可用。
- 遵循 `AGENTS.md` 第 13 节工程规范：全量类型注解、`.env` 配置、结构化日志 + `trace_id`、无 TODO 与伪代码；外部服务全部有 Mock 兜底（第 14 节第 2 条）。

## 5. 验收标准

1. 命令：
   - `cd backend && pytest tests/ -v`
   - `cd backend && python -m app.scripts.sync_stock_base`（未配 Token 时应降级并给出明确日志）
   - `curl http://localhost:8000/api/datasource/status`
2. 可判定条件：
   - [ ] **未配置任何 Key 时**，系统仍能跑通筛选全流程（降级到 Mock），无报错、无崩溃。
   - [ ] 配置 `TUSHARE_TOKEN` 后，`stock_base` 表能同步到真实股票列表与市值，且日期与交易日历一致。
   - [ ] `GET /api/datasource/status` 返回各源健康状态，含 `active` / `degraded` 标记与数据时间戳。
   - [ ] 字段映射测试通过：各源原始字段能正确映射到统一字段，缺失字段按约定显式处理。
   - [ ] 9:20 之前的数据被丢弃，不进入筛选链路（有对应测试）。
   - [ ] 密钥不出现在日志、接口响应与前端中（有对应测试或检查）。
   - [ ] `pytest tests/` 全部通过，覆盖降级路径与映射异常。

## 6. 交付物

- [ ] 三个 Provider 实现（Mock 对齐 + Tushare + 商业竞价模板）与注册路由
- [ ] 字段映射模块
- [ ] 盘后基础面同步脚本
- [ ] 数据源状态接口与推送
- [ ] 单元测试（降级路径 + 字段映射 + 时间过滤）
- [ ] `.env.example` 更新说明：如何接真实数据源、各 Key 从哪获取

## 7. 开始前请先回答（先不要写代码）

1. 你理解的数据源**分层**（基础面层 / 竞价层）与各自的**降级链路**是什么？各层边界如何划分？
2. 你确认的 `TushareProvider` 能力边界是什么？竞价字段缺失时你打算怎么处理，才不会让上层误以为拿到了竞价数据？
3. 商业竞价 Provider 模板你打算按哪种字段结构和鉴权方式搭？字段映射表怎么设计才便于以后换源？
4. 有哪些假设或不确定点需要我确认（尤其是请求频率限制与批量取数的粒度）？

## 8. 完成后请按此格式汇报

- 改动文件：
- 验证命令与结果：
- 遗留问题：
