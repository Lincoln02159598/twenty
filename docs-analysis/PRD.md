# 产品需求视角 (PRD) — 当前代码实际实现的产品

## 分析快照

- 分支：`main`；HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时 clean；本任务仅新增 `./docs-analysis/`。
- 子模块状态：无任何 Git 子模块。
- 应用版本：`2.29.0`（`twenty-current-version.constant.ts:10`）。
- 分析范围：README、twenty-shared 类型、server modules/engine、twenty-apps、SDK。
- 本文区分：(1) 当前已实现产品；(2) README 愿景；(3) 源码可推断方向；(4) 改善建议。

## 证据分类

- `[Evidence]` / `[Inference]` / `[Unknown]`（同前）。

## 核心结论

- `[Evidence]` Twenty 是一个**元数据驱动的可扩展 CRM 平台**：核心是"对象(Object)+字段(Field)+视图(View)+工作流(Workflow)+AI Agent+Apps 扩展"的构建块模型，面向"想自建并像代码一样版本化 CRM 的技术团队"。
- `[Evidence]` README 标语"The #1 Open-Source CRM"为营销声明，不可在仓库内验证。
- `[Evidence]` README 的三个入口（Cloud 注册、`create-twenty-app`、自托管 Docker Compose）中，后两者在本仓库有源码支撑；Cloud 为外部 SaaS，仓库不可见。

---

## 1. 项目定位 / 目标用户 / 解决的问题

- `[Evidence]` 定位（README）："gives technical teams the building blocks for a custom CRM ... the CRM you build, ship, and version like the rest of your stack." 证据：`README.md:23-25`。
- `[Inference]` 目标用户：有工程能力、需要超越开箱即用 SaaS CRM 灵活性的技术团队。推导依据：README 强调"as code / version control / CLI 脚手架"（`README.md:37-69`）+ 仓库存在 `create-twenty-app`、`twenty-sdk`、`twenty-apps` 等开发者工具链。

## 2. 产品边界与核心功能

`[Evidence]` 核心构建块（与代码对应）：

| 构建块 | 源码依据 | 状态 |
| -- | -- | -- |
| 对象(Object)/字段(Field) | `engine/metadata-modules/object-metadata`、`field-metadata` 实体 + 动态建表 | 已实现 |
| 视图(View)/布局(PageLayout)/Widget | `metadata-modules/view`、`page-layout` + 类型 `page-layout-widget-configuration.type.ts` | 已实现 |
| 工作流(Workflow) | `modules/workflow/workflow-executor` + 18 种 action | 已实现 |
| AI Agent / Chat | `metadata-modules/ai/{ai-agent,ai-chat,...}` + Vercel AI SDK | 已实现 |
| Apps 扩展（对象/字段/组件/logic/连接器/角色/agent） | `engine/core-modules/application` + `twenty-sdk` + `twenty-apps` | 已实现（运行时插件系统） |
| 消息通道（邮件/日历） | `modules/messaging`、`calendar`、`connected-account`（Google/Microsoft/IMAP） | 已实现 |
| 数据导入/导出、API Key、权限角色 | `engine/core-modules/{api-key,role,workspace}`、`spreadsheet-import` 模块 | 已实现 |
| 实时（记录变更推送） | `engine/subscriptions` + GraphQL Subscriptions | 已实现 |
| 分析/事件日志 | ClickHouse + `event-logs` | 已实现（可选） |

## 3. 用户角色

- `[Evidence]` 鉴权上下文类型（`JwtTokenTypeEnum`）：`API_KEY`、`WORKSPACE_AGNOSTIC`、`ACCESS`/`PLAYGROUND`、`APPLICATION_ACCESS`，并支持**模拟(impersonation)**。证据：`packages/twenty-server/src/engine/core-modules/auth/strategies/jwt.auth.strategy.ts:121-127,243-311,390-396`。
- `[Evidence]` 角色与权限：`RoleModule` + `PermissionFlag`（如 `SSO_BYPASS`）。证据：`core-engine.module.ts` 注册 `RoleModule`；`auth.service.ts:331-335`。
- `[Evidence]` 实例管理员：独立 `/admin-panel` GraphQL 端点 + 前端 `settings/admin-panel`。证据：`app.module.ts:75,154-159`。
- `[Inference]` 用户层级：平台用户(User) → 工作区成员(UserWorkspace) → 角色/权限；工作区(Workspace)是租户根（携带 `databaseSchema`）。

## 4. 输入 / 输出 / 外部依赖

- `[Evidence]` 输入：Web UI（GraphQL/REST/MCP）、Webhook（SES/Google Pub-Sub/Microsoft Graph/App route trigger）、CLI（`twenty` SDK CLI、`nest-commander` 实例命令）、API Key。
- `[Evidence]` 输出：GraphQL/REST 响应、实时订阅推送、出站邮件(SES/SMTP)、出站 HTTP（workflow `HTTP_REQUEST`）、LLM 调用、文件存储(S3/local)、Lambda 调用（Logic Function）。
- `[Evidence]` 外部依赖：PostgreSQL、Redis、（可选）ClickHouse、S3、SES、Lambda、LLM Provider、Google/Microsoft OAuth、身份提供方(SAML/OIDC)。

## 5. 功能实现状态矩阵

| 功能 | 文档/README 声明 | 源码状态 | 运行时入口 | 测试证据 | 最终判断 |
| -- | ---- | ---- | ----- | ---- | ---- |
| 自托管 CRM（对象/字段/视图/记录） | README "self-hosting" | `metadata-modules` + `workspace-schema-manager` | `/graphql` core | 大量 integration-spec | 已完整实现 |
| 工作流引擎 | README "workflows" | `workflow-executor` 18 action | `workflowQueue` | integration | 已实现 |
| AI Agent / Chat | README "AI agents and chats" | `metadata-modules/ai/*` + streaming | `/graphql` + `aiStreamQueue` | integration | 已实现 |
| Apps 扩展（SDK define*） | README `defineObject`/`app:publish` | `twenty-sdk/src/cli/commands/app/{publish,install,deploy,uninstall}.ts` + server `application-manifest` 26 converters | `ApplicationSyncService.synchronizeFromManifest` | fixtures + slack app | 已实现（真实运行时插件） |
| `create-twenty-app` 脚手架 | README `npx create-twenty-app` | `packages/create-twenty-app`（workspace 成员） | npm bin | 有 jest config | 已实现 |
| Slack 集成 | 隐含（近期 commit） | **非核心模块**，以 App 形式 `twenty-apps/public/slack` | `/webhooks/server/:id` → logic function | app 内测试 | 已实现（作为 App） |
| 邮件通道（Gmail/Microsoft/IMAP/SMTP） | 隐含 | `modules/messaging` driver-per-provider | SES webhook + cron | integration | 已实现 |
| 日历（Google/Microsoft/CalDAV） | 隐含 | `modules/calendar` + connected-account | cron + webhook | integration | 已实现 |
| 通话录音转写/摘要 widget | 近期 commit `3cbcf999d6` | 类型 `CALL_RECORDING_SUMMARY/TRANSCRIPT` + `call-recording` 标准对象 | 记录页 widget | — | 类型+对象已落地；**渲染组件 commit 自述为后续 PR**，仅类型定义风险 |
| MCP（Model Context Protocol） | 未在 README | `engine/api/mcp` + 真 SSE | `/mcp` | — | 已实现（README 未宣传） |
| ClickHouse 分析 | 未在 README | `database/clickHouse` + `event-logs-live` | 可选 | 条件测试 | 已实现（可选） |
| Cloud SaaS 注册 | README "Sign up at twenty.com" | **仓库不可见**（外部） | — | — | 无法在仓库验证 |
| Webhook 出站投递子系统 | 一般 CRM 期望 | **未发现专用子系统**；仅 workflow `HTTP_REQUEST` | — | — | 仅通过 workflow action 实现，无独立投递/重试框架 |

## 6. 仅存在类型/配置/占位的功能

- `[Evidence]` `DataSourceEntity`（`@deprecated`，仅保留表）：`packages/twenty-server/src/engine/metadata-modules/data-source/data-source.entity.ts:15-17`。代码迁移到 `workspace.databaseSchema` 进行中（`1-22-instance-command-fast-...-drop-object-metadata-data-source-fk.ts`）。
- `[Evidence]` 通话录音 widget 类型已加，但 commit 自述"widget rendering components are a follow-up PR" → `CALL_RECORDING_SUMMARY/TRANSCRIPT` 当前可能仅有类型+对象，前端渲染待补。

## 7. README 声称但仓库未验证 / 已废弃

- `[Unknown]` Cloud 注册/工作区创建（外部 SaaS，仓库不可见）。
- `[Evidence]` 旧 `twenty-cli` 已 **DEPRECATED**，README 的 `npx create-twenty-app` 与 `app:publish` 实际由 `twenty-sdk` 提供（`packages/twenty-cli/package.json:3`）。

## 8. 非目标 / 当前限制

- `[Evidence]` 非 Single-Page-Only：单工作区与多工作区两种域名 shell（`DomainShell` 依据 `isMultiWorkspaceEnabled` 分流）。证据：`packages/twenty-front/src/modules/app/components/DomainShell.tsx:14-40`。
- `[Inference]` 限制：每工作区独立 schema，多租户规模受 `pg_namespace` 增长与每上下文 entity metadata 重建影响（见 [[SOURCE_ARCHITECTURE]] 风险章节）。
- `[Evidence]` REST API 无 OpenAPI/Swagger 文档（src 无 import）。

## 9. 数据与隐私边界

- `[Evidence]` `sendDefaultPii: true`（Sentry 默认发送用户 PII）。证据：`instrument.ts:60`。对合规（GDPR/DPA）敏感，需在自托管时关闭或配置区域。
- `[Evidence]` CSRF 对 SAML 回调豁免（跨源 IdP POST）。证据：`app.module.ts:134-137`。
- `[Evidence]` Logic Function 在 **AWS Lambda（subhosting）** 或本地子进程执行用户代码——第三方 App 代码隔离依赖 Lambda/子进程边界。证据：`logic-function-driver.factory.ts:46-112`。

## 10. 产品风险

1. `[Evidence]` 自研 ORM + 动态 DDL：SQL 注入面依赖人工 `escapeIdentifier`（有测试但新增 manager 方法易遗漏）。证据：`workspace-schema-table-manager.service.ts`、`remove-sql-injection.util.ts`。
2. `[Evidence]` 迁移不在启动时执行（`migrationsRun:false`）：忘记跑 `run-instance-commands` 会静默 schema drift。证据：`core.datasource.ts:55-56`。
3. `[Evidence]` `UnhandledExceptionFilter` 在 `headersSent` 时静默 return，异常被吞（不记日志/不上报 Sentry）。证据：`unhandled-exception.filter.ts:20-22`。
4. `[Evidence]` Sentry `sendDefaultPii:true` + dev 栈泄漏（`global-exception-handler.util.ts:157-160`）。

## 11. 源码可推断的发展方向（非已实现声明）

- `[Inference]` "Apps as first-class"：`pre-installed-apps.service.ts` 自动给每个工作区装预装 App、Slack 以 App 形态存在——产品正把更多能力从核心迁移到 App 层。
- `[Inference]` AI-first：`workflow-tools`（约 20 个 LLM 可调用工具）让 AI Agent 直接构建/编辑工作流。

## 改善建议（针对已确认问题）

- `[Recommendation]` 给 `run-instance-commands` 增加启动期 schema-version 守卫（启动时校验并强提示，而非静默 drift）。优先级：高；难度：中。依据：`core.datasource.ts:55-56`。
- `[Recommendation]` 默认关闭 `sendDefaultPii` 或按区域配置，并在文档显式标注。优先级：高；难度：低。依据：`instrument.ts:60`。
- `[Recommendation]` 修复 `UnhandledExceptionFilter` 静默吞异常分支（至少记日志 + 上报）。优先级：高；难度：低。依据：`unhandled-exception.filter.ts:20-22`。

## 已确认事实 / 合理推断 / Unknown

- 见各节内联标签。关键 Unknown：Cloud SaaS 实现、生产部署拓扑、通话录音 widget 渲染是否已合入。

## 主要证据索引

- `README.md:23-148`；`packages/twenty-shared/src/types/{ConnectedAccountProvider,MessageChannelType}.ts`
- `packages/twenty-server/src/engine/core-modules/auth/strategies/jwt.auth.strategy.ts`
- `packages/twenty-server/src/engine/core-modules/application/application-manifest/application-sync.service.ts`
- `packages/twenty-sdk/src/cli/commands/app/{publish,install,deploy,uninstall}.ts`
- `packages/twenty-shared/src/types/page-layout/page-layout-widget-configuration.type.ts:171,175`
- `packages/twenty-server/src/filters/unhandled-exception.filter.ts:20-22`
- `packages/twenty-server/src/instrument.ts:60`
