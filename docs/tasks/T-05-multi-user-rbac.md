# 任务 T-05：多用户与权限体系（RBAC）落地

> 项目背景与硬约束见仓库根目录 `AGENTS.md`，本文件只写本轮要做的事。

---

## 1. 目标（一句话）

把"单人使用"的选股系统升级为**多用户 Web 应用**：实现认证、最小 RBAC 授权、
私有数据隔离与审计日志，同时**保持"竞价筛选是系统级全局单次任务"这一架构红线不变**。

## 2. 本轮范围

**要做**：

1. **认证（AuthN）**：JWT 双 Token（access 短时效 + refresh 长时效）、
   密码 Argon2 或 bcrypt 哈希、登录失败计数与账号临时锁定、
   修改密码、管理员重置密码；预置超级管理员（初始密码走环境变量，首次启动种子写入）。
2. **授权（AuthZ）——最小可用 RBAC**：`User ↔ Role ↔ Permission` 多对多；
   角色 `super_admin` / `admin` / `user`；权限点（`user:manage`、`strategy:config`、
   `datasource:manage`、`selection:read`、`selection:run`、`audit:read`）；
   后端以依赖注入做权限校验。
3. **多用户与流水线的关系（红线）**：竞价筛选仍是**系统级全局单次任务**；
   多用户只体现为"共享结果 + 私有关注项 + 角色分级配置"。
4. **私有数据隔离**：用户自选关注、提醒偏好等私有数据，在**数据层**加 owner 过滤。
5. **审计**：登录日志（成功 / 失败、IP、UA、时间）+ 敏感操作审计日志。
6. **前端**：登录页、用户管理、角色管理、审计日志页；
   路由守卫 + 按钮级权限；401 跳登录或刷新、403 提示；WS 连接需带 Token 鉴权。
7. **为既有接口补齐权限**：修改阈值（`strategy:config`）、
   数据源状态（`datasource:manage`）、审计查询（`audit:read`）。

**明确不做**：

1. **不实现 OAuth2 / SSO / 第三方登录**——只预留接口与配置项（`AGENTS.md` 第 12 节）。
2. 不做多租户、组织部门层级与计费。
3. **不改动 `auction_filter` 的筛选规则与算法**；不改调度时点、不改容灾逻辑。
4. **不得把竞价筛选改成"每用户各跑一遍"**（`AGENTS.md` 第 14 节第 1 条硬约束）。
5. 不引入反射热加载、动态插件等过度设计（第 8 节）。

## 3. 文件锚点

- 新增：
  - `backend/app/core/security.py`（JWT 签发 / 校验、密码哈希）
  - `backend/app/core/permissions.py`（权限点常量与校验依赖）
  - `backend/app/models/user.py`（`users` / `roles` / `permissions` / 关联表 /
    `user_watchlist` / `user_notify_config` / `audit_log` / `login_log`）
  - `backend/app/schemas/auth.py`、`backend/app/schemas/user.py`
  - `backend/app/api/auth.py`（login / refresh / logout / me）
  - `backend/app/api/users.py`、`backend/app/api/roles.py`
  - `backend/app/api/audit.py`、`backend/app/api/watchlist.py`
  - `backend/app/services/auth_service.py`、`backend/app/services/user_service.py`
  - `backend/tests/test_auth.py`、`backend/tests/test_permissions.py`、
    `backend/tests/test_data_isolation.py`
  - `frontend/src/views/Login.vue`、`Users.vue`、`Roles.vue`、`AuditLog.vue`
  - `frontend/src/stores/user.ts`、`frontend/src/directives/permission.ts`
  - `frontend/src/api/auth.ts`、`frontend/src/api/users.ts`
- 修改：
  - `backend/app/api/deps.py`（改为 Bearer 认证 + `require_permission` 依赖）
  - `backend/app/main.py`（挂载新路由、收紧 CORS、统一 401 / 403 处理）
  - `backend/app/api/settings.py`、`backend/app/api/datasource.py`（补权限校验）
  - `backend/app/ws/manager.py`（建连时校验 Token）
  - `backend/app/models/__init__.py`（导出新模型）
  - `backend/app/core/config.py`（JWT secret、Token 时效、初始管理员密码，全部走 `.env`）
  - `frontend/src/router/index.ts`（登录守卫、按角色动态路由）
  - `backend/alembic/`（新增权限相关表迁移）

## 4. 实现要求

- **密钥与哈希**：密码必须 Argon2 或 bcrypt 哈希，**严禁明文或可逆加密**；
  JWT secret 只走环境变量，不写日志、不下发前端（第 13 节）。
- **必须后端鉴权**：每个受保护接口独立校验权限；
  前端隐藏按钮只是体验优化，**不构成鉴权**（第 14 节第 8 条）。
- **私有数据在数据层隔离**：查询必须带 owner 条件，不得仅在接口层判断 403；
  严禁越权读取他人数据。
- **401 与 403 严格区分**；统一异常处理；权限校验以依赖注入实现
  （如 `require_permission("selection:read")`），**WS 建连同样校验**。
- **可集成现成方案**：认证可用 FastAPI-Users，授权可用 Casbin
  （策略落库并加缓存，改权限即时生效）；若选择自建，须说明理由与取舍。
- **撤权须即时生效**：不得把权限全集塞进 JWT 导致"撤权后到 Token 过期前仍可操作"；
  角色权限变更应立即可见（策略落库 + 缓存失效）。
- 遵循 `AGENTS.md` 第 13 节工程规范，以及第 12 节"多用户预留原则"。

## 5. 验收标准

1. 命令：
   - `cd backend && pytest tests/ -v`
   - `curl -X POST http://localhost:8000/api/auth/login`（错误密码须返回 401；
     连续失败须触发账号锁定）
   - 未带 Token 访问 `/api/selection/latest` 须 401；
     带 Token 但权限不足访问 `/api/users` 须 403
2. 可判定条件：
   - [ ] 库中密码为哈希值，无任何明文。
   - [ ] 登录 / 刷新 / 注销 / me 全链路可用；access 过期可用 refresh 换取。
   - [ ] 不同角色访问同一接口结果不同，**权限校验在后端生效**（有测试证明）。
   - [ ] 用户 A 无法读取用户 B 的自选与偏好（**数据层隔离**，有测试）。
   - [ ] 系统级筛选仍只执行一次，**未被改造成按用户循环**（有测试或代码结构可证明）。
   - [ ] 登录日志与敏感操作审计落库，可通过接口查询。
   - [ ] 前端路由守卫 + 按钮级权限生效；401 自动跳登录 / 刷新，403 有提示。
   - [ ] WS 连接未携带有效 Token 须被拒绝。
   - [ ] `pytest tests/` 全部通过。

## 6. 交付物

- [ ] 认证与授权模块（含 RBAC 数据模型与 Alembic 迁移）
- [ ] 权限依赖注入，以及既有接口的权限补齐
- [ ] 私有数据（自选 / 提醒偏好）数据层隔离实现
- [ ] 审计日志与登录日志
- [ ] 前端登录页、用户管理、角色管理、审计页，含路由守卫与按钮级权限
- [ ] 单元测试（认证、鉴权、数据隔离）
- [ ] 权限模型与角色说明（可放 README 小节）

## 7. 开始前请先回答（先不要写代码）

> 按 `AGENTS.md` 第 15 节"AI 协作工作方式"，先对齐、再动手。

1. 你打算**自建 RBAC** 还是**集成 FastAPI-Users + Casbin**？理由与取舍是什么？
2. 你如何保证"筛选仍是系统级全局单次任务"，而没有被多用户改造成按用户循环？
3. 私有数据隔离你打算在哪一层拦截？如何测试"越权不可读"确实成立？
4. 角色与权限点如何定义？如何保证**撤权即时生效**（不依赖 JWT 过期）？
5. 有哪些假设或不确定点需要我确认（尤其是初始管理员账号与默认角色）？

## 8. 完成后请按此格式汇报

- 改动文件：
- 验证命令与结果：
- 遗留问题：

---

<!--
维护提示（可保留作团队约定）：
1. 向上晋升：本任务确立的权限模型与"系统级全局单次任务"红线，已预置于 AGENTS.md 第 12 节；
   若实现中产生新的长期约定，回流进 AGENTS.md。
2. 与 T-04 的边界：T-04 明确"不做多用户与权限"，其迁移与接口不得依赖本轮产物；
   T-05 完成后，回测接口 `/api/backtest/*` 需补权限点（可在本轮一并处理或另开任务）。
-->
