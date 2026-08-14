# 测试与 CI (测试与CI)

## 分析快照

- 分支：`main`；HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时 clean；本任务仅新增 `./docs-analysis/`。
- 子模块状态：无。
- 分析范围：单元/集成/E2E/UI 测试、静态检查、CI workflow、Docker、发布、条件跳过。

## 证据分类

- `[Evidence]` / `[Inference]` / `[Unknown]`（同前）。
- **关键区分**：测试**文件存在** vs **命令配置** vs **CI 执行** vs **跳过/条件禁用**——见 §5。

## 核心结论

- `[Evidence]` 测试体系成熟：server 单元(945 `*.spec.ts`)+集成(528 `*.integration-spec.ts`，16 分片)、front 单元+Storybook(4 分片×scope)、E2E(Playwright)、视觉回归(Argos，dispatch 到 `ci-privileged`)。
- `[Evidence]` CI 有 **drift 守卫**：迁移 drift（`database:migrate:generate` 必须无 diff）+ codegen drift（graphql×3 + client-sdk）+ 升级 mutation guard + 跨版本升级（v1.22→当前）。
- `[Evidence]` 多种条件跳过：changed-files 门控、billing/audit/secure-deployment 目录门控、`ci:auth-cookie-sessions` label、`run-merge-queue` label、cross-version `skip` 输入。

---

## 1. 单元测试

- `[Evidence]` Jest（`@nx/jest:jest`，nx 目标 `test`，cache，coverage `text-summary`）。证据：`nx.json:102-131`。
- `[Evidence]` server 单元：`jest.config.mjs`，`testRegex: '.*\\.spec\\.ts$'`，swc(decorators+`@lingui/swc-plugin`)，`setupFilesAfterEach:['./setupTests.ts']`，CI 仅失败 reporter。证据：`packages/twenty-server/jest.config.mjs:6-67`。
- `[Evidence]` 数量：twenty-server `*.spec.ts` ≈ 945；`*.integration-spec.ts` ≈ 528。twenty-front ≈ 1001 测试文件；twenty-shared ≈ 223；twenty-ui ≈ 11；twenty-cli 0。
- `[Evidence]` `jest.preset.js`：nx jest preset + 空 `testEnvironmentOptions`（绕过 @nx/jest 22.3.3 的 Lingui breakage）。

## 2. 集成测试

- `[Evidence]` `jest-integration.config.ts`：`testRegex: '\\.integration-spec\\.ts$'`，`globalSetup/globalTeardown`（`test/integration/utils/setup-test.ts`/`teardown-test.ts`），`setupFilesAfterEach:[.../setup-wait-for-all-jobs-between-tests.ts]`，`maxWorkers:1`，`maxConcurrency:1`，`testTimeout:20000`。证据：`:20-96`。
- `[Evidence]` 目标 `test:integration`（`NODE_ENV=test`，6144MB heap）+ `with-db-reset`（先 `database:reset`）+ `test:integration:secure`（强制 `SERVER_URL=https://localhost:3000`，仅 `secure-deployment/`）。证据：`project.json:21-48`。
- `[Evidence]` 条件跳过：`billing/` 需 `IS_BILLING_ENABLED`；`audit/` 需 `CLICKHOUSE_URL`；`secure-deployment/` 仅 secure 配置。证据：`jest-integration.config.ts:30-36`。

## 3. 端到端（E2E）

- `[Evidence]` **Playwright**（`@playwright/test`）。`playwright.config.ts`：`setup`(`*.setup.ts`) → `chrome`(`storageState:.auth/user.json`)；webkit/mobile/edge 注释掉；单 worker；CI retries=2。证据：`packages/twenty-e2e-testing/playwright.config.ts:1,22-26,43-82`。
- `[Evidence]` 认证：`tests/login.setup.ts` → "Continue with Email" → `DEFAULT_LOGIN`(`tim@apple.dev`) → Continue → `DEFAULT_PASSWORD` → Sign in → "Choose a workspace" → 选 "Apple" → 存 `storageState`。证据：`:13-42`、`.env.example`、`lib/pom/loginPage.ts:37-56`。
- `[Evidence]` 结构：`tests/`(specs)、`lib/pom/`(page objects)、`lib/fixtures/`、`reporters/`；API token 助手 `lib/utils/getAccessAuthToken.ts`。

## 4. UI / Storybook / 视觉回归

- `[Evidence]` Storybook（`@storybook/react-vite`），stories glob 由 `STORYBOOK_SCOPE`(modules|pages|performance|ui-docs) 选；addons `@storybook/addon-{vitest,coverage}`、`storybook-addon-mock-date`。证据：`packages/twenty-front/.storybook/main.ts:3-66`。
- `[Evidence]` Storybook 测试用 vitest（`storybook:test`，分片）。
- `[Evidence]` 视觉回归 **Argos 经 dispatch 到 `twentyhq/ci-privileged`**（`visual-regression-dispatch.yaml` 触发于 `workflow_run`，解 artifact 名后跑 `post-visual-regression-comment.yaml`）。证据：`visual-regression-dispatch.yaml:11-15,158-207`。

## 5. 静态检查 / 类型 / Lint / 格式 / 覆盖率

- `[Evidence]` Lint：`oxlint --type-aware` + `oxfmt --check`（依赖 `twenty-oxlint-rules:build`）；`lint:diff-with-main`（仅 diff 文件）。证据：`nx.json:42-70`、`project.json:166-200`。
- `[Evidence]` 类型：`tsgo -p tsconfig.json`（TS Go 移植）。证据：`nx.json:88-101`。
- `[Evidence]` 覆盖率：jest coverage（`text-summary`/`lcov`）；storybook coverage（`nyc`）。

## 6. CI 触发条件 / Job / 服务矩阵（精选 workflow）

| workflow | 触发 | 关键 Job / 服务 | Evidence |
| -- | -- | -- | -- |
| `ci-server.yaml` | pull_request, merge_group | server-build; server-lint-typecheck; **server-validation**(pg:18+redis, 迁移/codegen drift 检查); server-test(test:ci, 4 分片); **server-integration-test**(pg:18+redis+clickhouse:25.8.8, 16 分片, shard1 +secure); cross-version-upgrade; discover-public-apps+install-smoke | `.github/workflows/ci-server.yaml:3-5,41-395` |
| `ci-front.yaml` | pull_request, merge_group, push:main | front-sb-build; front-sb-test(4 分片×scope, Playwright+Argos 截图); front-task(lint/typecheck/test/lingui×2); front-build(--max-old-space-size=10240) | `ci-front.yaml:3-7,35-250` |
| `ci-merge-queue.yaml` | merge_group | 仅 `upgrade-mutation-guard`(5min, 无 build/test/services) | `ci-merge-queue.yaml:10-60` |
| `ci-e2e-main.yaml` | push:main 或 PR label `run-merge-queue` | e2e-test(pg:18+redis, build+reset+起 server/front/worker, `nx test twenty-e2e-testing`) | `ci-e2e-main.yaml:3-43,95-119` |
| `cd-deploy-main.yaml` | push:main | **不部署**，dispatch `twenty-infra` auto-deploy-main | `cd-deploy-main.yaml:7-37` |
| `cd-deploy-tag.yaml` | push tags twenty/v* | dispatch `twenty-infra` staging-ci | `cd-deploy-tag.yaml:6-35` |
| `ci-cross-version-upgrade.yaml` | workflow_call/dispatch | Docker pg:16+redis:7；v1.22→当前 upgrade→冒烟；`skip` 输入→no-op | `ci-cross-version-upgrade.yaml:42-46,69-322` |
| `ci-release-create.yaml` | workflow_dispatch | 输入 version/ref/create_release，开 release PR | `ci-release-create.yaml:1-40` |
| 其他 | 各 package 专属(ci-front-component-renderer/ci-sdk/ci-zapier/ci-emails/ci-docs/ci-website/ci-utils/ci-ui/ci-twenty-apps/ci-codex-plugin/ci-shared/ci-create-app*)、i18n push/pull、claude.yml 等 | — | `.github/workflows/` |

`[Evidence]` `AUTH_COOKIE_SESSIONS_ENABLED` 仅当 PR 有 label `ci:auth-cookie-sessions` 时设。证据：`ci-server.yaml:308`。

## 7. 测试存在 vs 配置 vs CI 执行 vs 跳过（明确区分）

- **文件存在**：server 945 spec + 528 integration-spec；front stories + spec；e2e spec+setup。
- **命令配置**：server `test/test:ci/test:debug/jest/test:integration(+with-db-reset)/test:integration:secure`；front `storybook:test`/`test`；e2e `setup`/`test`。
- **CI 执行**：server 单元(`server-test` 4 分片) + 集成(`server-integration-test` 16 分片, shard1+secure)；front 单元/lint/typecheck/lingui(`front-task`) + Storybook(`front-sb-test` 4 分片×scope)→Argos；E2E(`ci-e2e-main`)。
- **跳过/条件**：
  - changed-files 门控（无相关改动→job skip）。证据：`ci-server.yaml:18-43`、`ci-front.yaml:22-34`。
  - 目录门控：billing/audit/secure-deployment。证据：`jest-integration.config.ts:30-36`。
  - `ci:auth-cookie-sessions` label。证据：`ci-server.yaml:308`。
  - secure-deployment 仅 shard1。证据：`ci-server.yaml:352-354`。
  - E2E 仅 push:main 或 label `run-merge-queue`。证据：`ci-e2e-main.yaml:19-22`。
  - cross-version `skip`→no-op。证据：`ci-cross-version-upgrade.yaml:42-46`。
  - Storybook `performance` scope 仅 push(非 PR)。证据：`ci-front.yaml:77`。

## 8. Docker / 部署 / 发布

- `[Evidence]` 多目标 Dockerfile：`twenty-server`/`twenty-server-aws`/`twenty`/`twenty-aws`/`twenty-app-dev`。证据：`packages/twenty-docker/twenty/Dockerfile`。
- `[Evidence]` compose：server(/healthz)/worker(同镜像)/db(pg:16)/redis(noeviction)。证据：`docker-compose.yml:3-135`。
- `[Evidence]` 部署委托 `twenty-infra`（私有）；AWS Lambda 是运行时依赖（执行 logic function），非部署目标。证据：`cd-deploy-*.yaml`、`logic-function-drivers/drivers/lambda.driver.ts`。
- `[Evidence]` 发布：`ci-release-create.yaml`(workflow_dispatch) → release PR；版本 `version:bump`(`scripts/bump-version.ts`，`semver.inc minor`)；**无应用级 CHANGELOG.md**（仅 SDK/zapier/codex/template 有）。证据：`project.json:311-317`、`bump-version.ts:9-49`。
- `[Evidence]` 版本真相：`TWENTY_CURRENT_VERSION='2.29.0'`（`twenty-current-version.constant.ts:10`），非 `package.json` 0.2.1。

## 9. 缓存 / 构建产物 / 签名 / 打包

- `[Evidence]` Nx cache（`cache:true` 多目标，`.cache/jest`、`.vite`）；Storybook build 可缓存。
- `[Evidence]` server build 产物 `dist/`（nest build + 拷贝 client-sdk 到 `dist/assets`）。证据：`project.json:7-20`。
- `[Evidence]` front build 产物 `build/`（Vite，chunk-size 守卫）。证据：`vite.config.ts:174-214`。
- `[Unknown]` 构建签名/可信发布（未在本仓库发现签名步骤，可能在 `twenty-infra`）。

## 10. 测试覆盖矩阵（抽样）

| 模块/功能 | 单元 | 集成 | E2E | CI 执行 | 主要缺口 |
| -- | -- | -- | -- | -- | -- |
| 鉴权(JWT/session/CSRF/OAuth) | ✓ | ✓(secure-deployment) | 部分(登录 setup) | ✓(shard1+secure) | SAML/OIDC 真实 IdP 回归 |
| 工作区/记录 CRUD | ✓ | ✓ | ✓ | ✓ | 跨池事务无专门用例 |
| 工作流引擎 | ✓ | ✓ | — | ✓ | 18 action 全矩阵覆盖度未知 |
| Apps 框架(install/sync/logic) | ✓(converter spec) | ✓(install-smoke per app) | — | ✓(discover-public-apps) | 第三方 App 长尾 |
| 邮件/日历同步 | ✓ | ✓ | — | ✓(条件) | 各 provider 真实 API 回归 |
| AI 流式聊天 | ✓ | ✓ | — | ✓ | 各 LLM provider 真实回归 |
| 实时订阅(matcher) | ✓ | ✓ | — | ✓ | matcher 漏判曾出 bug(commit a47f566eb1) |
| 迁移 drift / codegen drift | — | — | — | ✓(validation) | 仅检测 diff，不验证语义 |

## 11. 测试缺口

- `[Inference]` E2E 仅 chrome、单 worker、retries=2，覆盖面有限（webkit/mobile/edge 注释掉）。
- `[Evidence]` 集成 `maxWorkers:1`（串行，慢但隔离）——扩展性靠 16 分片。
- `[Inference]` 跨 core/workspace 池事务、并发更新竞态无显式用例矩阵。
- `[Unknown]` 性能/安全/混沌测试是否在 `twenty-infra` 私有侧（本仓库不可见）。

## 已确认事实 / 合理推断 / Unknown

- `[Evidence]` 测试体系成熟且 CI 严格（含 drift/mutation/cross-version 守卫）。
- `[Unknown]` 生产部署侧测试/签名/性能基线（`twenty-infra`）。

## 批判性评估 / 建设性改善建议

- `[Recommendation]` 为实时 matcher 加专门的回归矩阵（防止再次"离开过滤视图不通知"类 bug）。优先级：中；难度：低。依据：commit `a47f566eb1`。
- `[Recommendation]` 跨 core/workspace 池事务与并发更新加显式集成用例。优先级：中；难度：中。

## 主要证据索引

- `nx.json:42-131`；`jest.preset.js:1-8`
- `packages/twenty-server/jest.config.mjs:6-67`；`jest-integration.config.ts:20-96`；`jest-integration-secure.config.ts:9-17`
- `packages/twenty-server/project.json:21-48,236-244,311-317`
- `packages/twenty-e2e-testing/playwright.config.ts:1-82`；`tests/login.setup.ts:13-42`；`lib/pom/loginPage.ts:37-56`
- `packages/twenty-front/.storybook/main.ts:3-66`
- `.github/workflows/{ci-server,ci-front,ci-merge-queue,ci-e2e-main,cd-deploy-main,cd-deploy-tag,ci-cross-version-upgrade,ci-release-create,visual-regression-dispatch}.yaml`
- `packages/twenty-docker/twenty/Dockerfile`；`packages/twenty-docker/docker-compose.yml:3-135`
- `packages/twenty-server/src/engine/core-modules/upgrade/constants/twenty-current-version.constant.ts:10`
