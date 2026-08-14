# 应用流程 (APP_FLOW) — 用户视角的真实流程

## 分析快照

- 分支：`main`；HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时 clean；本任务仅新增 `./docs-analysis/`。
- 子模块状态：无。
- 分析范围：前端路由/鉴权、后端鉴权与工作区、e2e 登录脚本、docker-compose 启动。

## 证据分类

- `[Evidence]` / `[Inference]` / `[Unknown]`（同前）。

## 核心结论

- `[Evidence]` 本项目**存在完整的用户登录/账户认证流程**（邮箱密码、Google/Microsoft OAuth、SAML/OIDC SSO、API Key、模拟登录），登录后进入工作区选择 → 主界面。
- `[Evidence]` 多工作区与单工作区两种域名 shell 并存（`DomainShell` 按 `isMultiWorkspaceEnabled` 分流）。
- `[Evidence]` E2E 登录脚本确证了"Continue with Email"→预填凭据的真实路径。

---

## 1. 安装 / 启动

- `[Evidence]` 自托管：`docker compose up`（服务 `server`/`worker`/`db`/`redis`）。证据：`packages/twenty-docker/docker-compose.yml:3-135`。
- `[Evidence]` 一体化镜像 `twenty-app-dev` 内置 Postgres18 + Redis（s6-overlay）+ `SIGN_IN_PREFILLED=true`。证据：`packages/twenty-docker/twenty/Dockerfile:205-300`。
- `[Evidence]` 本地开发：`yarn start`（并发起 server+front，再起 worker）。证据：`package.json:83`。
- `[Evidence]` 数据库初始化：`database:init`（setup-db + `run-instance-commands --include-slow`），dev 种子 `workspace:seed:dev`。证据：`packages/twenty-server/project.json:224-280`。
- `[Inference]` 首次启动需先跑迁移（不在 server 启动时自动执行，见 [[测试与CI]] / [[BACKEND_Development]]）。

### 启动 → 进入登录（源码链）
```
浏览器加载 index.html → /src/index.tsx
  → hydrateMetadataStore() (IndexedDB 元数据回填) 完成后 renderApp()
  → <App> (App.tsx:19) → DomainShell (按 clientConfig.isMultiWorkspaceEnabled 分流)
  → 未认证 → AuthFlowLayout 下的 SignInUp 页面
```
- `[Evidence]` `DomainShell` 等待 `clientConfigApiStatusState.isLoadedOnce` 后分流。证据：`packages/twenty-front/src/modules/app/components/DomainShell.tsx:14-40`。

## 2. 用户登录 / 无登录模式

- `[Evidence]` 登录方式：邮箱密码（`AUTH_PASSWORD_ENABLED`）、Google/Microsoft OAuth（guards）、SAML/OIDC SSO、API Key（Bearer）。证据：`packages/twenty-server/src/engine/core-modules/auth/guards/{google-oauth,microsoft-oauth,oidc-auth,saml-auth}.guard.ts`；`config-variables.ts` 中 `AUTH_PASSWORD_ENABLED`。
- `[Evidence]` 密码登录服务：`AuthService.validateLoginWithPassword` → `verify`（签发 access+refresh token）。证据：`packages/twenty-server/src/engine/core-modules/auth/services/auth.service.ts:154,374`。

### 登录流程（用户视角 ↔ 源码）
```
前置条件：工作区已存在、用户已是成员
用户操作：在 SignInUp 输入邮箱 → Continue → 密码 → Sign in
系统响应：
  前端：tokenPairState（localStorage，由 migrateTokenPairCookieToLocalStorage 迁移）
  后端：AuthService.validateLoginWithPassword → verify → 签发 token
       JwtAuthStrategy.validate 校验 token + 工作区成员资格(flatWorkspaceMemberMaps)
数据变化：UserSession 记录（Postgres）；可选 cookie session
异常分支：密码错→UNAUTHENTICATED；非成员→FORBIDDEN_EXCEPTION
完成条件：前端拿到 token，Apollo authLink 注入 Bearer，跳转主界面
```
- `[Evidence]` E2E 真实复现此路径：`tests/login.setup.ts` 点 "Continue with Email" → 输 `DEFAULT_LOGIN` → Continue → `DEFAULT_PASSWORD` → Sign in → 等 "Choose a workspace" → 选 "Apple" → 存 `storageState`。证据：`packages/twenty-e2e-testing/tests/login.setup.ts:13-42`、`.env.example`（`DEFAULT_LOGIN=tim@apple.dev`）。

- `[Evidence]` **无登录/匿名模式**：`verifyEmail`/`signUp` 之外的公共端点主要是 webhook（SES、App route trigger）与 MCP 守卫。**未发现产品级"匿名浏览"模式**。

## 3. 工作区创建 / 选择 / 多工作区切换

- `[Evidence]` 单工作区域名 → `WorkspaceApp`；多工作区 → 根域名 `RootApp`、其余 `WorkspaceApp`。证据：`DomainShell.tsx:14-40`。
- `[Evidence]` 工作区创建会动态建 schema（`createWorkspaceDBSchema` → `queryRunner.createSchema('workspace_<base36(uuid)>', true)`）。证据：`packages/twenty-server/src/engine/workspace-datasource/workspace-datasource.service.ts:56-69`。
- `[Evidence]` 预装 App 在工作区创建时自动安装。证据：`packages/twenty-server/src/engine/core-modules/application/pre-installed-apps/pre-installed-apps.service.ts:50-67`。
- `[Evidence]` 切换工作区：前端 `availableWorkspacesState`、`currentWorkspaceState`、`currentWorkspaceMemberState`（localStorage getOnInit）。证据：`packages/twenty-front/src/modules/auth/states/*`。

## 4. 主界面进入 / 核心任务流程

- `[Evidence]` 路由树：`WorkspaceAppProviders → MinimalMetadataGate → DefaultLayout → MainAppLayoutWithSidePanel`，下挂 `RecordIndexPage`/`RecordShowPage`/`PageLayoutPage`/`SettingsCatchAll`。证据：`packages/twenty-front/src/modules/app/hooks/useCreateWorkspaceAppRouter.tsx:139-336`。
- `[Evidence]` Onboarding 步骤在 `BlankLayout + OnboardingStepLayout` 下。证据：同文件。

### 核心流程 A：浏览/创建一条记录（以 company 为例）
```
前置条件：已登录、工作区已选、objectMetadata 已 hydrate
用户操作：打开对象列表 → "New" → 填字段 → 保存
系统响应：
  前端：object-record 模块(最大模块,1891 文件) → ApolloCore mutate /graphql
  后端：动态 workspace resolver → WorkspaceEntityManager.insert
       → RBAC 校验(validateOperationIsPermittedOrThrow)
       → 写入 workspace_<id>.company
持久化：workspace schema 表
实时：object-record-event-publisher → Redis Pub/Sub → onEventSubscription 推送订阅客户端
异常分支：权限不足→PermissionsGraphqlApiExceptionFilter；字段校验失败→GraphQLError
完成条件：列表实时刷新出现新记录
```
- `[Evidence]` 记录访问经 `executeInWorkspaceContext`（设置 schema+entity metadata+auth 上下文）。证据：`packages/twenty-server/src/engine/twenty-orm/repository/global-workspace-orm.manager.ts:70`。

### 核心流程 B：连接邮箱账户并同步邮件
```
前置条件：已登录、有 connected-account 权限
用户操作：Settings → 连接 Google/Microsoft 账户 → OAuth 授权
系统响应：
  后端：connected-account oauth2-client-manager(google|microsoft) → 存 refresh token
       → cron/队列 messagingQueue 拉取邮件 → message-import-manager drivers
       → 写入 message 记录 + 关联 contact(participant)
实时：SES/Graph/Pub-Sub webhook 推送新邮件触发增量同步
异常分支：token 过期 → refresh-tokens-manager 自动刷新
完成条件：Activities/收件箱出现邮件
```
- `[Evidence]` driver-per-provider：`modules/messaging/message-import-manager/drivers/{gmail,microsoft,imap,smtp,inbound-email}`。

### 核心流程 C：安装并使用一个 App（以 Slack 为例）
```
前置条件：工作区存在、marketplace 可达
用户操作：Marketplace → 安装 Slack App → 配置连接(Slack connection provider) → 在 Slack @bot 提问
系统响应：
  安装：ApplicationInstallService.installApplication → ApplicationSyncService.synchronizeFromManifest
       → 26 个 converter 把 App 的 object/field/logic/component/connectionProvider 同步进工作区 metadata
  触发：Slack Events API → POST /webhooks/server/:resolverLogicFunctionUniversalIdentifier
       → ServerRouteTriggerService.handle → 加载 slack-events-resolver logic function
       → LogicFunctionExecutorService 执行(Lambda 或 Local driver) → 校验 Slack 签名
       → enqueueTargetFunction(slack-events-enqueue) 入队 → 异步处理
数据变化：App 注入的对象/字段出现在工作区；logic 函数可被路由/cron/DB 事件触发
异常分支：logic function 失败→按 driver 隔离(Lambda/子进程)
```
- `[Evidence]` 整链：`server-route-trigger.controller.ts:20-28`、`server-route-trigger.service.ts:51,149`、`logic-function-driver.factory.ts:46-112`、`twenty-apps/public/slack/src/logic-functions/slack-events-resolver.ts:23-67`。

## 5. 设置 / 权限 / 多用户

- `[Evidence]` Settings 模块（1027 文件）覆盖 profile/workspace/billing/roles/data-model/api-keys。证据：前端 `modules/settings`。
- `[Evidence]` 角色/权限：`RoleModule` + `PermissionFlag`；前端 `settings/roles`。证据：`core-engine.module.ts`、`metadata-modules/permissions`。
- `[Evidence]` 模拟登录(impersonation)：`generateImpersonationAccessTokenAndRefreshToken`。证据：`auth.service.ts:419`。

## 6. 错误提示 / 异常恢复

- `[Evidence]` 前端：`AppErrorBoundary`（根级 `AppRootErrorFallback`）+ `ExceptionHandlerProvider`；Apollo errorLink 路由 `UNAUTHENTICATED`→续 token、`APP_VERSION_MISMATCH` 等静默、其余→Sentry。证据：`App.tsx:22-25`、`apollo.factory.ts:345-411`。
- `[Evidence]` 后端：全局 `UnhandledExceptionFilter` + Billing/Permissions/Config 专属 filter；GraphQL 错误经 `global-exception-handler.util.ts`（4xx 不上报 Sentry，dev 附堆栈）。证据：`app.module.ts:87-92`、`global-exception-handler.util.ts:46-177`。

## 7. 升级 / 迁移 / 数据恢复

- `[Evidence]` 升级：`run-instance-commands`（先跑 legacy pending TypeORM 迁移，再跑 fast/slow instance command + workspace command）。证据：`packages/twenty-server/src/database/commands/run-instance-commands.command.ts:40-67,170`。
- `[Evidence]` 跨版本升级验证：CI `ci-cross-version-upgrade.yaml`（从 `v1.22` 起跑旧版→构建当前→跑 upgrade→冒烟）。证据：`.github/workflows/ci-cross-version-upgrade.yaml:69-322`。
- `[Evidence]` 工作区导出/导入：`database/commands/workspace_export`（含 `CREATE SCHEMA IF NOT EXISTS`，证据 agent 报告）。

## 8. 退出 / 再次启动

- `[Evidence]` 登出：`updatePassword` 撤销所有 refresh token + session（证据 `auth.service.ts:657`）；前端 `onUnauthenticatedError` 清所有 session atoms 并跳 SignInUp（`useApolloFactory.ts:74-93`）。
- `[Inference]` 再次启动：worker 有 `enableShutdownHooks`，server 进程无显式 shutdown hooks（见 [[运行时架构]]），重启时 Redis/TypeORM 连接靠进程终止回收。

## 已确认事实

- 登录、工作区、记录 CRUD、邮件同步、App 安装与触发均有源码 + 部分有 e2e/integration 证据。
- 用户流程与后端调用链一一对应（见各流程源码引用）。

## 合理推断

- `[Inference]` "无登录模式"不存在产品级匿名浏览（仅 webhook/MCP 等服务端点）。

## Unknown 与待验证事项

- `[Unknown]` Cloud SaaS 的工作区开通/计费真实流程（仓库不可见）。
- `[Unknown]` Onboarding 各步骤的具体业务规则（未逐文件读 onboarding 模块）。

## 批判性评估

- `[Evidence]` server 进程无 `enableShutdownHooks`（仅 worker 有），登出/重启路径的资源回收依赖默认行为，存在连接泄漏风险。证据：`main.ts`（无调用）vs `queue-worker.ts:24`。

## 建设性改善建议

- `[Recommendation]` 为 server 进程补 `app.enableShutdownHooks()` 并显式关闭 Redis/TypeORM 连接。优先级：中；难度：低。依据：`main.ts` 缺失 + `queue-worker.ts:24` 对照。

## 主要证据索引

- `packages/twenty-e2e-testing/tests/login.setup.ts:13-42`；`lib/pom/loginPage.ts:37-56`
- `packages/twenty-front/src/modules/app/components/{App,DomainShell}.tsx`
- `packages/twenty-front/src/modules/app/hooks/useCreateWorkspaceAppRouter.tsx`
- `packages/twenty-server/src/engine/core-modules/auth/services/auth.service.ts`
- `packages/twenty-server/src/engine/workspace-datasource/workspace-datasource.service.ts:56-69`
- `packages/twenty-server/src/engine/core-modules/server-route-trigger/server-route-trigger.{controller,service}.ts`
- `packages/twenty-server/src/database/commands/run-instance-commands.command.ts`
