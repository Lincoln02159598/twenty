# 后端开发指南 (BACKEND_Development)

## 分析快照

- 分支：`main`；HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时 clean；本任务仅新增 `./docs-analysis/`。
- 子模块状态：无。
- 分析范围：`packages/twenty-server`（入口、配置、API、服务、数据层、迁移、鉴权、队列、扩展）。

## 证据分类

- `[Evidence]` / `[Inference]` / `[Unknown]`（同前）。

## 核心结论

- `[Evidence]` 后端为 NestJS（NestExpress）单进程 HTTP + 同代码库独立 worker 进程 + nest-commander CLI；三入口共享同一 `CoreEngineModule`。
- `[Evidence]` 数据访问分两套：core/metadata 用标准 TypeORM repository；工作区记录用自研 `WorkspaceEntityManager`（强制 RBAC）。
- `[Evidence]` 结构变更**不**靠 TypeORM 迁移，而靠 fast/slow instance command + workspace command，CLI `run-instance-commands` 执行。

---

## 1. 后端入口与初始化流程

- `[Evidence]` HTTP：`src/main.ts:31-117`（`NestFactory.create<NestExpressApplication>`）。顺序：`setPgDateTypeParser` → 创建 app → 取 logger/config/exceptionHandler → 注册 `unhandledRejection` → `trust proxy` → `applyCredentialedCors` → `session` → `useContainer`(class-validator) → body parsers → `graphql-upload`(/graphql,/metadata) → `generateFrontConfig` → keepAliveTimeout → `listen(NODE_PORT)`。
- `[Evidence]` `import './instrument'`（`main.ts:25`）作为副作用导入，先于 bootstrap 执行 Sentry/OTel 初始化。
- `[Evidence]` Worker：`src/queue-worker/queue-worker.ts:9-34`（`createApplicationContext`，无 HTTP，有 `enableShutdownHooks`）。
- `[Evidence]` CLI：`src/command/command.ts:9-35`（`nest-commander` `CommandFactory`）。

### 关键：main API handler 模板
```
入口：@Resolver/@Mutation/@Query (engine/api/graphql 动态或手写)
→ 参数：@Args/@InputType（class-validator 校验，DI via useContainer）
→ 鉴权：JwtAuthStrategy → WorkspaceAuthContext(AsyncLocalStorage) → @Guard(IsUserAuthContext)
→ 服务：WorkspaceService / 业务 service (CoreEngineModule 提供)
→ 数据访问：
   - core/metadata：coreDataSource repository
   - 工作区记录：executeInWorkspaceContext → WorkspaceEntityManager.find/insert/...
→ 事务边界：coreDataSource.transaction(...) 或 workspace 内单连接
→ 错误类型：GraphQLError(经 global-exception-handler.util) / HttpException(filter)
→ 调用方：前端 ApolloCore(/graphql) / Apollo(/metadata) / ApolloAdmin(/admin-panel) / REST / MCP
→ Evidence：app.module.ts:128-170；jwt.auth.strategy.ts；workspace-entity-manager.ts
```

## 2. 配置加载

- `[Evidence]` 无 `@nestjs/config`。自研 `TwentyConfigService.get(key)`：env-only 走 `EnvironmentConfigDriver`，否则 DB driver（`IS_CONFIG_VARIABLES_IN_DB_ENABLED`）后 env。证据：`twenty-config.service.ts:56-70`、`twenty-config.module.ts:10-34`。
- `[Evidence]` schema：`config-variables.ts`（`@ConfigVariablesMetadata({group,description,type,isSensitive?,isEnvOnly?})` + class-validator）。组枚举：`SERVER_CONFIG/RATE_LIMITING/STORAGE_CONFIG/GOOGLE_AUTH/MICROSOFT_AUTH/EMAIL_SETTINGS/LOGGING/ADVANCED_SETTINGS/BILLING_CONFIG/CAPTCHA_CONFIG/CLOUDFLARE_CONFIG/LLM/LOGIC_FUNCTION_CONFIG/CODE_INTERPRETER_CONFIG/SSL/SUPPORT_CHAT_CONFIG/ANALYTICS_CONFIG/TOKENS_DURATION/AWS_SES_SETTINGS`。
- `[Evidence]` 配置可运行时变更（DB）并有刷新 cron（`config-variables-refresh-cron-interval.constants.ts`）。

## 3. 服务注册 / 路由注册

- `[Evidence]` 模块：`AppModule`（顶层）+ `CoreEngineModule`（聚合根，约 80 模块）+ `ModulesModule`（6 业务模块）。证据：`app.module.ts:55-86`、`core-engine.module.ts:87-184`。
- `[Evidence]` 中间件 `configure()`（`app.module.ts:128-170`）：`CookieSessionCsrfMiddleware`（全路由除 SAML 回调）、`GraphQLHydrateRequestFromTokenMiddleware`+`WorkspaceAuthContextMiddleware`（graphql/metadata/admin-panel）、`McpMethodGuardMiddleware`（/mcp）、`RestCoreMiddleware`+`WorkspaceAuthContextMiddleware`（REST）。
- `[Evidence]` 全局 filter：`UnhandledExceptionFilter`（APP_FILTER）+ Billing/Permissions GraphQL filter（CoreEngineModule 注册）。

## 4. API 接口面

- `[Evidence]` **GraphQL（code-first, Yoga）**：`/graphql`（core, scope=CoreEngineModule, `autoSchemaFile:true`）、`/metadata`（`metadataModuleFactory`）、`/admin-panel`。Yoga 插件：复杂度限制（`GRAPHQL_MAX_FIELDS/MAX_ROOT_RESOLVERS`）、生产关闭 introspection/suggestions、Sentry tracing。证据：`graphql-config.service.ts:55-94`。
- `[Evidence]` **动态 workspace schema**：`workspace-resolver-builder` 从 object metadata 动态生成 resolver；`workspace-graphql-schema-sdl` 生成 SDL。证据：`engine/api/graphql/workspace-resolver-builder/`。
- `[Evidence]` **REST**：`RestApiModule`（迁移中，部分走 TwentyORM）。**无 Swagger/OpenAPI**。
- `[Evidence]` **MCP**：`McpModule`，`/mcp`，含真 SSE（`text/event-stream`）。证据：`engine/api/mcp/controllers/mcp-core.controller.ts:79-82`。

## 5. 应用服务 / 领域逻辑 / 数据访问层

- `[Evidence]` core/metadata 数据访问：`@InjectRepository`/`@InjectDataSource` over `coreDataSource`。证据：`workspace.service.ts` 等用 `coreDataSource.createQueryRunner()/.transaction()`。
- `[Evidence]` 工作区记录访问：`WorkspaceEntityManager extends EntityManager`（import TypeORM 私有 `EntityPersistExecutor`/`PlainObjectToDatabaseEntityTransformer`），每次操作 `validateOperationIsPermittedOrThrow` + 文件字段同步 + 嵌套关系处理；入口 `executeInWorkspaceContext`。证据：`workspace-entity-manager.ts:30-88`、`global-workspace-orm.manager.ts:70`。
- `[Evidence]` 自定义 query builder：`workspace-select-query-builder.ts`、`workspace-insert-query-builder.ts`。

## 6. 数据库 / 数据模型 / Schema

- `[Evidence]` 单 `core` schema（core+metadata 表）+ 每工作区 `workspace_<base36(uuid)>` schema。证据：`core.datasource.ts:44,57`、`get-workspace-schema-name.util.ts:3`。
- `[Evidence]` 实体 77 个（36 core-modules + 41 metadata-modules）。核心：`WorkspaceEntity`(带 `databaseSchema`)、`UserEntity`、`UserWorkspaceEntity`(成员)、`UserSessionEntity`、`ApiKeyEntity`、`FeatureFlagEntity`、`ObjectMetadata`、`FieldMetadata`、`View`、`Role`、`PermissionFlag`、`Workflow`/`WorkflowVersion`、`File`、Billing 系列、`DataSourceEntity`(deprecated)。
- `[Evidence]` 两 DataSource：`coreDataSource`（唯一 `TypeOrmModule.forRootAsync`）+ `GlobalWorkspaceDataSource`（子类，`entities:[]`，按上下文解析）。replica 仅 workspace pool 支持（`PG_DATABASE_REPLICA_URL`）。

## 7. 迁移与升级命令（重点）

- `[Evidence]` **TypeORM 迁移已冻结**：`synchronize:false`、`migrationsRun:false`，历史迁移在 `legacy-typeorm-migrations-do-not-add/`（185 个），**禁止新增**。证据：`core.datasource.ts:55-73`。
- `[Evidence]` **实例/工作区命令框架**：
  - 装饰器 `@RegisteredInstanceCommand(version,timestamp,{type:'slow'?})` / `@RegisteredWorkspaceCommand(version,timestamp)`。证据：`engine/core-modules/upgrade/decorators/registered-{instance,workspace}-command.decorator.ts`。
  - 发现：`UpgradeCommandRegistryService.onModuleInit` 遍历 `DiscoveryService.getProviders()` 读 Reflect metadata，按版本分桶(fast/slow instance + workspace)。证据：`upgrade-command-registry.service.ts:74-92`。
  - 执行：`run-instance-commands` CLI（先 `runLegacyPendingTypeOrmMigrations` 再 instance command）。证据：`run-instance-commands.command.ts:40-67,170`。
  - 约 279 个命令文件于 `database/commands/upgrade-version-command/`（版本目录 `1-21`...`2-27`）。
  - 生成：`database:migrate:generate` → `generate:instance-command`，用 `dataSource.driver.createSchemaBuilder().log()` diff 生成 up/down SQL 并自动注册。证据：`instance-command-generation.service.ts:26-30`、`project.json:236-244`。
  - 文档：`packages/twenty-server/docs/UPGRADE_COMMANDS.md`（fast=up/down；slow=先 `runDataMigration` 再 up，`--include-slow` 门控）。
- `[Evidence]` workspace schema DDL：`WorkspaceSchemaManagerService` 发裸 `CREATE TABLE/ALTER TABLE`，标识符经 `escapeIdentifier`。证据：`workspace-schema-table-manager.service.ts`、`workspace-schema-column-manager.service.ts`。
- `[Evidence]` ClickHouse 独立迁移：`database/clickHouse/migrations/run-migrations.ts`。

## 8. 事务 / 并发

- `[Evidence]` core 写：`coreDataSource.transaction()`（单连接）。证据：`workspace.service.ts` 等。
- `[Evidence]` workspace 记录：`WorkspaceEntityManager` 内单连接；migration runner 用 `coreDataSource.createQueryRunner()` 并经 `WorkspaceSchemaManagerService` 在同一 connection 发 workspace DDL（migration 内原子性较好）。
- `[Risk]` 两连接池（core vs workspace）**无跨池事务**：请求路径上"先写 core.objectMetadata 再 alter workspace schema"无跨池原子保证。证据：`global-workspace-datasource.service.ts` + `workspace-migration-runner.service.ts:266`。
- `[Evidence]` DDL 热升级锁：`WORKSPACE_SCHEMA_DDL_LOCKED` 时 `createWorkspaceDBSchema`/migration runner 抛 `DDL_LOCKED`（升级期建工作区直接失败，不排队）。证据：`workspace-datasource.service.ts:30-38`、`workspace-migration-runner.service.ts:251-258`。

## 9. 鉴权 / 权限

- `[Evidence]` JWT（`APP_SECRET` 签名，支持 `JWT_SUPPORTED_VERIFY_ALGORITHMS` 多算法 + 签名密钥轮换 cron `RotateSigningKeysCronJob`）。证据：`jwt.module.ts:22-53`。
- `[Evidence]` `JwtAuthStrategy.validate` 按 `JwtTokenTypeEnum` 分派（API_KEY/WORKSPACE_AGNOSTIC/ACCESS/PLAYGROUND/APPLICATION_ACCESS），强校验工作区成员资格，支持 impersonation 与 legacy API key。证据：`jwt.auth.strategy.ts:121-127,182-202,243-311,390-396`。
- `[Evidence]` `WorkspaceAuthContext`（apiKey/user/application/pending-activation）经全局 `AsyncLocalStorage`。证据：`workspace-auth-context.middleware.ts:19-80`、`workspace-auth-context.storage.ts:5-25`。
- `[Evidence]` Guards：`is-{api-key,application,system,user}-auth-context.guard.ts`、`google/microsoft/oidc/saml-auth.guard.ts`、`enterprise-features-enabled.guard.ts`。
- `[Evidence]` RBAC：`PermissionsService` + `PermissionsGraphqlApiExceptionFilter`；`AuthService.canUserBypassAuthProvider` 查 `PermissionFlagType.SSO_BYPASS`。证据：`metadata-modules/permissions/`、`auth.service.ts:331-335`。
- `[Evidence]` CSRF：`cookie-session-csrf.middleware.ts`（Origin 校验，Bearer 豁免，缺 Origin fail-closed，SAML 回调豁免）。

## 10. 队列 / 后台任务 / 缓存

- `[Evidence]` BullMQ（自研 driver 抽象 `Sync|BullMQ`，**非 @nestjs/bull**），17 命名队列。证据：`message-queue.constants.ts:5-23`、`message-queue.module-factory.ts:22-33`。
- `[Evidence]` `InjectMessageQueue(name)` DI；`JobsModule` 注册各 job processor（cron/billing/app-upgrade/email-sender/campaign/webhook...）。
- `[Evidence]` Redis 3 client（general/queue/pubsub），仅队列+Pub/Sub。证据：`redis-client.service.ts:17-62`。
- `[Evidence]` 应用缓存为进程内本地（`WorkspaceCacheModule`/`CoreEntityCacheModule`），**非 Redis**。
- `[Evidence]` cron 注册由 server 进程拥有（worker `DISABLE_CRON_JOBS_REGISTRATION=true`）。证据：`docker-compose.yml:61-71`。

## 11. 外部服务 / 文件系统

- `[Evidence]` AWS：S3（`STORAGE_TYPE=s3`）、SES（邮件入/出站 webhook）、Lambda（Logic Function，`LOGIC_FUNCTION_TYPE=lambda`）；近期加显式 `requestTimeout`（commit `f5a42cdbae`）。证据：`logic-function-drivers/drivers/lambda.driver.ts`、`aws-ses-client.provider.ts`、`s3.driver.ts`。
- `[Evidence]` 本地存储：`STORAGE_LOCAL_PATH`。

## 12. 错误处理 / 日志 / 监控

- `[Evidence]` 错误：`UnhandledExceptionFilter`（HTTP，`headersSent` 静默吞 + `exception.response ?? exception.message` 可能泄漏原始结构）；GraphQL 错误经 `global-exception-handler.util.ts`（4xx 不上报，dev 附堆栈）。证据：`unhandled-exception.filter.ts:20-22,43`、`global-exception-handler.util.ts:62-97,157-174`。
- `[Evidence]` 日志：`LoggerService`；ORM 查询日志由 `ORM_QUERY_LOGGING`(disabled/server-only/always) 控制。证据：`core.datasource.ts:16-36`。
- `[Evidence]` 监控：Sentry（`sendDefaultPii:true`、vercelAI/profiling 集成）+ OTel metrics（Prometheus:9464/OTLP/Console）。证据：`instrument.ts`。

## 13. 扩展方式（后端）

- `[Evidence]` 新增业务模块：放进 `modules/`，若需 DI 注册则加入 `ModulesModule` 或 `CoreEngineModule`。
- `[Evidence]` 新增 API：core GraphQL 装饰器（@ObjectType/@InputType/@Resolver）或 REST controller；动态对象走 metadata。
- `[Evidence]` 新增队列：在 `MessageQueue` enum 加项 + `@Processor`/`InjectMessageQueue`。
- `[Evidence]` 第三方扩展（App）：经 `application-manifest` + logic-function driver（见 [[扩展机制]]）。

## 14. 测试方式 / 调试入口 / 已知限制

- `[Evidence]` 单元 `*.spec.ts`（jest.config.mjs，swc）；集成 `*.integration-spec.ts`（jest-integration.config.ts，globalSetup/Teardown，maxWorkers:1，testTimeout:20000）。证据：`jest.config.mjs`、`jest-integration.config.ts:20-96`。
- `[Evidence]` 调试：`start:debug`（`nest start --watch --debug`）、`test:debug`（`--inspect-brk`）。
- `[Evidence]` 已知限制：REST 无 OpenAPI；server 无 shutdown hooks；自研 ORM 依赖 TypeORM 私有 API；迁移不 boot-enforced。

## 后端技术债务

- 自研 ORM 私有 API 依赖；两池无跨池事务；`ModulesModule` 注册语义分散；`UnhandledExceptionFilter` 吞异常；`sendDefaultPii` 默认开。

## 已确认事实 / 合理推断 / Unknown

- 见各节。`[Unknown]` REST 迁移 TwentyORM 的完成度（注释标 TODO，未量化）。

## 批判性评估 / 建设性改善建议

- `[Recommendation]` 为 REST 补 OpenAPI（@nestjs/swagger）或文档生成，降低集成成本。优先级：中；难度：中。
- `[Recommendation]` `UnhandledExceptionFilter` 静默分支至少记日志 + Sentry。优先级：高；难度：低。依据：`unhandled-exception.filter.ts:20-22`。
- `[Recommendation]` 启动期加 schema-version 守卫，避免静默 drift。优先级：高；难度：中。依据：`core.datasource.ts:55-56`。

## 主要证据索引

- `packages/twenty-server/src/main.ts:31-117`；`queue-worker/queue-worker.ts:9-34`；`command/command.ts:9-35`
- `packages/twenty-server/src/app.module.ts:55-170`
- `packages/twenty-server/src/database/typeorm/core/core.datasource.ts:40-89`
- `packages/twenty-server/src/engine/twenty-orm/entity-manager/workspace-entity-manager.ts:30-88`
- `packages/twenty-server/src/engine/core-modules/upgrade/{decorators,services}/*`
- `packages/twenty-server/src/database/commands/run-instance-commands.command.ts:40-67`
- `packages/twenty-server/src/engine/core-modules/auth/strategies/jwt.auth.strategy.ts`
- `packages/twenty-server/src/engine/core-modules/message-queue/message-queue.constants.ts:5-23`
- `packages/twenty-server/src/filters/unhandled-exception.filter.ts:13-45`
