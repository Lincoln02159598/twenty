# 技术栈 (TECH_STACK)

## 分析快照

- 分支：`main`（== `origin/main` == `origin/HEAD`）
- HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时为 **clean**；本任务仅新增 `./docs-analysis/` 文件，未修改任何源码/配置/锁文件。
- 子模块状态：**仓库无任何 Git 子模块**（`.gitmodules` 不存在）。
- 应用版本：`TWENTY_CURRENT_VERSION = '2.29.0'`（证据：`packages/twenty-server/src/engine/core-modules/upgrade/constants/twenty-current-version.constant.ts:10`）。注意：根 `package.json` 的 `"version": "0.2.1"` 已失效，不用于应用版本（证据：`package.json:76`）。
- 分析范围：根配置 + 22 个 packages + 关键运行时依赖。
- 未覆盖范围：`node_modules/`、构建产物、第三方依赖内部源码。

## 证据分类说明

- `[Evidence]`：由源码/配置/测试/CI 直接证明。
- `[Inference]`：由多个可验证事实合理推导。
- `[Unknown]`：当前仓库证据不足或无法在不运行的情况下确认。

## 核心结论

- `[Evidence]` 全栈为 TypeScript 单语言 monorepo（Nx + Yarn 4），主应用是 NestJS 后端 + React 前端 + BullMQ worker 三进程模型。
- `[Evidence]` 数据层为 PostgreSQL（单 `core` schema 容纳 core+metadata 实体 + 每工作区独立 schema），ORM 是 **自研的 TwentyORM（基于 TypeORM 内部 API 的子类）**，而非纯 TypeORM。
- `[Evidence]` 运行时强依赖：PostgreSQL、Redis（队列 + Pub/Sub）、（可选）ClickHouse（分析）、（可选）AWS S3/SES/Lambda（存储、邮件入站、Logic Function 执行）。
- `[Evidence]` LLM 通过 Vercel AI SDK 多 Provider 接入（OpenAI/Anthropic/Google/Mistral/xAI/Bedrock/Azure/OpenAI-compatible）。

---

## 1. 编程语言与各语言用途

- `[Evidence]` **TypeScript** 为主语言，覆盖前端、后端、worker、SDK、CLI、e2e、docs 脚本。证据：`tsconfig.base.json`、各 package 的 `tsconfig.json`。
- `[Evidence]` **SQL（DDL）** 以字符串形式由 `WorkspaceSchemaManagerService` 动态生成（建表/加列/外键/枚举/索引），标识符统一经过 `escapeIdentifier`。证据：`packages/twenty-server/src/engine/twenty-orm/workspace-schema-manager/services/workspace-schema-table-manager.service.ts`、`.../utils/remove-sql-injection.util.ts`。
- `[Evidence]` **YAML**：CI（`.github/workflows/*.yaml`，45 个）、Helm Chart（`packages/twenty-docker/helm/`）、docker-compose。
- `[Evidence]` **PO/JSON**：Lingui 翻译目录（`packages/twenty-front/src/locales/*.po`）。
- `[Evidence]` 少量 **Shell/Makefile**：`packages/twenty-docker/Makefile`、`packages/twenty-utils/setup-dev-env.sh`。

## 2. 构建工具 / 包管理器 / Monorepo

| 维度 | 实际使用 | 证据 |
| -- | -- | -- |
| 包管理器 | **Yarn 4**（`packageManager: yarn@4.13.0`），禁用 npm（`npm: please-use-yarn`） | `package.json:22-27` |
| Node 版本 | `^24.5.0`（`.nvmrc` 与 Dockerfile `node:24.18.0-alpine` 一致） | `package.json:20-23`、`packages/twenty-docker/twenty/Dockerfile` |
| Monorepo/任务编排 | **Nx 22.7.5**（`appsDir`/`libsDir` 均为 `packages`） | `nx.json:2-5`、`package.json:4-18` |
| workspace | 18 个 yarn workspaces（`packages/twenty-front` 等），另有 `twenty-apps`、`twenty-docker` 为非 workspace 基础设施目录 | `package.json:85-106` |
| 类型检查 | **`tsgo`**（TypeScript 原生 Go 移植），非 `tsc` | `nx.json:88-101` |
| Lint/格式化 | **`oxlint`（含 `--type-aware`）+ `oxfmt`**，并依赖自研 `twenty-oxlint-rules` 插件 | `nx.json:42-70`、`packages/twenty-server/project.json:166-200` |
| 后端编译 | `nest build`（SWC） | `packages/twenty-server/project.json:7-20` |
| 前端构建 | **Vite + @vitejs/plugin-react-swc**，minify `esbuild`，outDir `build` | `packages/twenty-front/vite.config.ts:90-146,174-176` |
| 代码生成 | GraphQL codegen（前端三套：core/metadata/admin）、`database:migrate:generate`（实例命令） | `packages/twenty-front/codegen*.cjs`、`packages/twenty-server/project.json:236-244` |

`[Evidence]` 根 `package.json` 的 `resolutions` 含大量安全/去重 pin（注释明确说明每条是 load-bearing 的 CVE 修复或类型一致性去重）。证据：`package.json:28-75`。这是依赖治理的实际证据，但不应机械复制。

## 3. 前端框架与运行时

- `[Evidence]` **React 19**（`@types/react ^19.2.0`）。证据：`package.json:10`。
- `[Evidence]` **Vite** 单页应用（SPA），**无 Module Federation**（`vite.config.ts` 无 `moduleFederation`，仅 `VITE_HOST` 为 dev server host）。证据：`packages/twenty-front/vite.config.ts:23-49,275-291`。
- `[Evidence]` 路由：**React Router v6 数据路由 API**（`createBrowserRouter` + `createRoutesFromElements`），声明式非文件式，页面 `lazy()` 懒加载于 `src/pages/`。证据：`packages/twenty-front/src/modules/app/hooks/useCreateWorkspaceAppRouter.tsx`。
- `[Evidence]` 状态管理：**Jotai**（自定义 `jotaiStore` + `createAtomState` 工厂，支持 localStorage/sessionStorage/cookie 后端）。**无 Recoil**（旧前提已失效）。证据：`packages/twenty-front/src/modules/ui/utilities/state/jotai/jotaiStore.ts`、`.../utils/createAtomState.ts`。
- `[Evidence]` 数据获取：**Apollo Client**（`apollo-link-rest` + `apollo-upload-client`），按 schema 拆 3 个 client（`/graphql`、`/metadata`、`/admin-panel`）。证据：`packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:415-424`。
- `[Evidence]` 样式：**Linaria（@wyw-in-js/vite）** 零运行时 CSS-in-JS，仅处理组件代码（大量 exclude）。证据：`packages/twenty-front/vite.config.ts:98-135`。
- `[Evidence]` i18n：**Lingui**（PO 格式，31 个 locale key = en + pseudo-en + 29 真实语言）。证据：`packages/twenty-front/lingui.config.ts`、`packages/twenty-shared/src/translations/constants/AppLocales.ts:3-35`。
- `[Evidence]` 富文本：BlockNote + TipTap（`advanced-text-editor`、`blocknote-editor` 模块），HTML 块预览经 `sanitizeHtmlPreview`。证据：`packages/twenty-front/src/modules/advanced-text-editor/extensions/blocks/HtmlNodeView.tsx:19-30`。
- `[Evidence]` 图表：nivo（`@nivo/core|pie|line|arcs`），构建期预打包。证据：`packages/twenty-front/vite.config.ts:159-171`。
- `[Evidence]` 字体：**自托管** `@fontsource/inter` + `@fontsource/dm-mono`（不请求 Google Fonts）。证据：`packages/twenty-front/src/index.tsx:7-12`。

## 4. 后端框架与运行时

- `[Evidence]` **NestJS**（NestExpress 平台），GraphQL 走 **@graphql-yoga/nestjs（YogaDriver）**，code-first（`autoSchemaFile: true`）。证据：`packages/twenty-server/src/app.module.ts:58-62`、`packages/twenty-server/src/engine/api/graphql/graphql-config/graphql-config.service.ts:83-94`。
- `[Evidence]` **三个独立 GraphQL 端点**：`/graphql`（core，scope=`CoreEngineModule`）、`/metadata`、`/admin-panel`。证据：`packages/twenty-server/src/app.module.ts:73-75,140-163`。
- `[Evidence]` **REST**（`RestApiModule`）与 **MCP**（Model Context Protocol，`McpModule`，含真 SSE 端点）共存。证据：`packages/twenty-server/src/engine/api/mcp/controllers/mcp-core.controller.ts:79-82`。
- `[Evidence]` 鉴权：Passport JWT 策略 + 可选 cookie session（express-session）+ CSRF（Origin 校验）+ SAML/OIDC/Google/Microsoft OAuth。证据：`packages/twenty-server/src/engine/core-modules/auth/strategies/jwt.auth.strategy.ts:35-436`、`packages/twenty-server/src/main.ts:69-71`。
- `[Evidence]` CLI 命令运行器：**nest-commander**（实例/工作区命令、数据库迁移）。证据：`packages/twenty-server/src/command/command.ts:1,20-32`。

## 5. 数据库 / ORM / 数据访问

- `[Evidence]` **PostgreSQL**（`type: 'postgres'`，单 `core` schema + 每工作区 schema `workspace_<base36(uuid)>`）。证据：`packages/twenty-server/src/database/typeorm/core/core.datasource.ts:40-57`、`packages/twenty-server/src/engine/workspace-datasource/utils/get-workspace-schema-name.util.ts:3`。
- `[Evidence]` **ORM：自研 TwentyORM**。core/metadata 表用标准 TypeORM repository（`@InjectRepository`）；工作区记录用 `GlobalWorkspaceDataSource`（`extends DataSource`，`entities: []`，按 AsyncLocalStorage 上下文动态解析 entity metadata）+ `WorkspaceEntityManager extends EntityManager`（直接 import TypeORM 私有路径）。证据：`packages/twenty-server/src/engine/twenty-orm/global-workspace-datasource/global-workspace-datasource.ts:40-72`、`.../entity-manager/workspace-entity-manager.ts:30-88`。
- `[Evidence]` **TypeORM 迁移系统已冻结**：`synchronize:false`、`migrationsRun:false`（不在启动时跑），新结构变更必须写 fast/slow instance command。证据：`packages/twenty-server/src/database/typeorm/core/core.datasource.ts:55-73`。
- `[Evidence]` **ClickHouse**（可选，分析/事件日志，独立迁移路径）。证据：`packages/twenty-server/src/database/clickHouse/clickHouse.module.ts`、`project.json:281-294`。

## 6. API 与通信协议

- `[Evidence]` GraphQL（Yoga，HTTP + WebSocket Subscriptions）、REST、MCP（HTTP + SSE）。
- `[Evidence]` 实时：**GraphQL Subscriptions over Redis Pub/Sub**（前端 `sse-db-event` 模块实际跑在 WebSocket Subscriptions 上），记录变更经 `EventStreamResolver`。证据：`packages/twenty-server/src/engine/subscriptions/event-stream.resolver.ts:49-169`、`.../subscription.service.ts:34-67`。
- `[Evidence]` Webhook 入站：SES 邮件、Google Pub/Sub push、Microsoft Graph 通知、App route trigger（`POST /webhooks/server/:resolverLogicFunctionUniversalIdentifier`）。证据：`packages/twenty-server/src/engine/core-modules/server-route-trigger/server-route-trigger.controller.ts:20-28`。

## 7. 队列 / 缓存 / 调度

- `[Evidence]` **BullMQ**（自研 driver 抽象 `MessageQueueDriverType: Sync | BullMQ`，**未使用 `@nestjs/bull`**）。**17 个命名队列**。证据：`packages/twenty-server/src/engine/core-modules/message-queue/message-queue.constants.ts:5-23`、`.../message-queue.module-factory.ts:22-33`。
- `[Evidence]` **Redis**：3 个懒加载客户端（general / queue / pubsub），**仅用于队列与 Pub/Sub**；会话存 Postgres，应用缓存为进程内本地缓存（`WorkspaceCacheModule`/`CoreEntityCacheModule`）。证据：`packages/twenty-server/src/engine/core-modules/redis-client/redis-client.service.ts:17-62`。
- `[Evidence]` 定时任务：cron job 注册（worker 进程 `DISABLE_CRON_JOBS_REGISTRATION` 由 server 拥有）。证据：`packages/twenty-docker/docker-compose.yml:61-71`。

## 8. 可观测性 / 日志 / 监控

- `[Evidence]` **Sentry**（`@sentry/node` + profiling-node；集成 redis/http/express/graphql/postgres/vercelAI/nodeRuntimeMetrics；`sendDefaultPii: true`；tracesSampleRate 0.1）。证据：`packages/twenty-server/src/instrument.ts:27-86`。
- `[Evidence]` **OpenTelemetry metrics**（MeterProvider，按 `METER_DRIVER` 选 Console/OTLP/Prometheus:9464）。证据：`packages/twenty-server/src/instrument.ts:88-119`。
- `[Evidence]` 自研 `LoggerService`（Nest Logger 实现）。证据：`packages/twenty-server/src/main.ts:47,77`。
- `[Evidence]` Grafana provisioning + otel-collector 配置随仓库提供（`packages/twenty-docker/grafana/`、`otel-collector/`）。

## 9. LLM / AI

- `[Evidence]` **Vercel AI SDK**（`ai` 包），多 Provider 工厂：OpenAI/Anthropic/Azure/Google/Mistral/xAI/Bedrock/OpenAI-compatible，实例缓存于 `Map`。证据：`packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/sdk-provider-factory.service.ts:3-10,38-115`。

## 10. 部署 / 容器

- `[Evidence]` **Docker 多目标 Dockerfile**（`twenty-server`、`twenty-server-aws`、`twenty`、`twenty-aws`、`twenty-app-dev` 一体化镜像）。证据：`packages/twenty-docker/twenty/Dockerfile`。
- `[Evidence]` docker-compose 顶层服务：`server`、`worker`（同镜像 + `yarn worker:prod`）、`db`（postgres:16）、`redis`（`--maxmemory-policy noeviction`）。证据：`packages/twenty-docker/docker-compose.yml:3-135`。
- `[Evidence]` **Helm Chart + k8s manifests + podman** + Spilo HA Postgres。证据：`packages/twenty-docker/helm/`、`k8s/`、`podman/`、`twenty-postgres-spilo/`。
- `[Evidence]` CD 实际部署委托给私有仓库 `twentyhq/twenty-infra`（本仓库仅 `gh workflow run` 触发）。证据：`.github/workflows/cd-deploy-main.yaml:7-37`、`cd-deploy-tag.yaml:6-35`。

## 11. 关键运行时依赖（精选，按"声明/import/初始化/路径"核对）

| 依赖 | 实际用途 | 初始化/调用位置 | 关键路径 | Evidence |
| -- | -- | -- | -- | -- |
| `@nestjs/core` `platform-express` | 后端框架 | `main.ts:34` | 是 | `main.ts:1-2,34` |
| `typeorm` | ORM 基座（被自研 TwentyORM 子类化） | `core.datasource.ts:87` | 是 | `core.datasource.ts:4,87` |
| `@nestjs/graphql` + `@graphql-yoga/nestjs` | GraphQL | `app.module.ts:58` | 是 | `app.module.ts:8,14,58` |
| `bullmq` + `ioredis` | 队列 | `message-queue.module-factory.ts:22` | 是 | 同文件 |
| `graphql-redis-subscriptions` | 实时 Pub/Sub | `redis-client.service.ts:5,55` | 是 | 同文件 |
| `express-session` | 会话 | `main.ts:71` | 是（可选） | `main.ts:9,71` |
| `@sentry/node` `@sentry/profiling-node` | 错误/性能 | `instrument.ts:12-13,27` | 是（可选） | 同文件 |
| `ai`（Vercel AI SDK）+ 各 provider 包 | LLM | `sdk-provider-factory.service.ts:3-10` | 是（可选） | 同文件 |
| `passport` `@nestjs/passport` `jsonwebtoken` | 鉴权 | `jwt.auth.strategy.ts` | 是 | 同文件 |
| AWS SDK（S3/SES/Lambda） | 存储/邮件/Logic Function | `s3.driver.ts`、`aws-ses-client.provider.ts`、`lambda-aws-client.service.ts` | 是（可选） | commit `f5a42cdbae` |

## 12. 可选 / 平台专用 / Feature Flag 依赖

- `[Evidence]` 计费（Stripe 系，`IS_BILLING_ENABLED`）、ClickHouse（`CLICKHOUSE_URL`）、Sentry（`EXCEPTION_HANDLER_DRIVER=SENTRY`）、OTEL/Prometheus（`METER_DRIVER`）、AWS（`STORAGE_TYPE=s3`、`LOGIC_FUNCTION_TYPE=lambda`）均为**环境开关型可选依赖**。
- `[Evidence]` `FeatureFlag` 实体 + `FeatureFlagModule` 提供运行时特性开关。证据：`packages/twenty-server/src/engine/core-modules/feature-flag/`。

## 13. 依赖状态分类（声明 vs import vs 运行时 vs 仅测试）

- `[Evidence]` `helmet`：**仓库 src 中无 import**（agent grep 与本人核对 `main.ts` 确认仅 `applyCredentialedCors`，无 helmet）—— 即使曾被声明也未进入运行时。`[Unknown]` 是否曾在 `package.json` 声明未单独核对，但运行时确定未启用。
- `[Evidence]` `@nestjs/swagger`：**仓库 src 中无 import**——REST 无 OpenAPI 文档。
- `[Evidence]` `twenty-cli`：**已废弃**（`description: "[DEPRECATED] Use twenty-sdk instead"`），实际 CLI 为 `twenty-sdk`（bin `twenty`）。证据：`packages/twenty-cli/package.json:3-4`、`packages/twenty-sdk/package.json:5`。
- `[Evidence]` `@module-federation/dts-plugin`：仅出现在根 `resolutions`（为 storybook/component-renderer 间接依赖的安全 pin），**前端构建无 Module Federation**。

## 已确认事实

- 单语言 TS monorepo；Nx + Yarn 4；oxlint/oxfmt/tsgo；NestJS+React+Vite+BullMQ；PostgreSQL + 自研 TwentyORM；Redis 仅队列/Pub/Sub；多 LLM Provider via Vercel AI SDK。
- 版本真实来源是 TS 常量 `2.29.0`，非 `package.json`。

## 合理推断

- `[Inference]` 类型检查用 `tsgo`、lint 用 `oxlint --type-aware`，表明团队追求构建速度与类型安全并重，且愿意采用非主流/前沿工具链。
- `[Inference]` 自研 TwentyORM 的存在说明：动态多租户 schema + 每行 RBAC 注入 + 热升级期实体可用性，是纯 TypeORM 无法满足的硬需求。

## Unknown 与待验证事项

- `[Unknown]` 生产实际部署拓扑（ECS/EKS/Lambda/其他）位于私有仓库 `twenty-infra`，本仓库不可见。
- `[Unknown]` `helmet` 是否曾在依赖清单中声明（运行时确定未启用）。
- `[Unknown]` 各 LLM Provider 的生产实际启用集合（取决于 `AI_PROVIDERS` 配置，运行时数据不可见）。

## 批判性评估

- `[Evidence]` 安全 pin 数量庞大且高度依赖人工维护（`package.json:28-75` 的超长注释），任何一条被误删都可能重新引入 CVE——治理强但脆弱。
- `[Evidence]` 自研 TwentyORM 直接 import TypeORM 私有路径（`workspace-entity-manager.ts:30-32`），TypeORM 升级有破坏风险，无 semver 保护。

## 建设性改善建议

- `[Recommendation]` 为关键安全 pin 增加"移除条件"的自动检测（如 dependabot/yarn 专属脚本），减少人工遗忘。优先级：中；难度：中。
- `[Recommendation]` 为自研 TwentyORM 对 TypeORM 私有 API 的使用建立适配层 + 升级回归测试矩阵。优先级：高；难度：高。

## 主要证据索引

- `package.json:20-106`；`nx.json:1-284`；`packages/twenty-server/project.json:1-319`
- `packages/twenty-server/src/main.ts:1-117`；`app.module.ts:55-171`
- `packages/twenty-server/src/database/typeorm/core/core.datasource.ts:40-89`
- `packages/twenty-server/src/engine/core-modules/message-queue/message-queue.constants.ts:5-23`
- `packages/twenty-server/src/engine/core-modules/redis-client/redis-client.service.ts:17-77`
- `packages/twenty-server/src/instrument.ts:1-119`
- `packages/twenty-server/src/engine/metadata-modules/ai/ai-models/services/sdk-provider-factory.service.ts:3-115`
- `packages/twenty-shared/src/types/ConnectedAccountProvider.ts:1-9`；`MessageChannelType.ts:1-5`
- `packages/twenty-front/vite.config.ts:23-291`；`src/modules/app/components/App.tsx:19-49`
- `packages/twenty-docker/twenty/Dockerfile`；`packages/twenty-docker/docker-compose.yml:3-135`
