# 前端开发指南 (FRONTEND_GUIDELINES)

## 分析快照

- 分支：`main`；HEAD：`4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 工作区状态：任务开始时 clean；本任务仅新增 `./docs-analysis/`。
- 子模块状态：无。
- 分析范围：`packages/twenty-front`（入口、路由、状态、数据获取、样式、i18n、测试、构建）与 `packages/twenty-ui`（共享组件/主题）。

## 证据分类

- `[Evidence]` / `[Inference]` / `[Unknown]`（同前）。
- 本文按四类区分规范：**(1) 源码明确存在的规范**；**(2) 可从重复模式推断的约定**；**(3) 仓库无证据的规范**；**(4) 推荐未来采用的规范（[Recommendation]）**。

## 核心结论

- `[Evidence]` React 19 + Vite(SWC) + Jotai + Linaria(@wyw-in-js) + Lingui + Apollo，单 SPA（**无 Module Federation**）。
- `[Evidence]` 路径别名：`@/` → `src/modules`，`~/` → `src`（`vite.config.ts:281-285` 与 tsconfig 同步镜像）。
- `[Evidence]` 三套 Apollo client 对应三个后端 GraphQL 端点，配三套 codegen。

---

## 1. 前端入口

- `[Evidence]` `src/index.tsx`：`ReactDOM.createRoot` → `hydrateMetadataStore()`(IndexedDB) 完成后 `renderApp()`；预渲染副作用 `migrateTokenPairCookieToLocalStorage`（TODO 2026-12-12 移除）；全局 CSS 导入自托管字体 + `twenty-ui/style.css` + `theme-light/dark.css`。证据：`src/index.tsx:1-31`。
- `[Evidence]` `index.html`：`#root`、`/src/index.tsx`、内联 `window._env_`、iOS UA 检测 + viewport pin、FOUC 内联主题变量。证据：`packages/twenty-front/index.html`。
- `[Evidence]` Provider 栈（`src/modules/app/components/App.tsx:19-49`）：`JotaiProvider(store=jotaiStore)` → `AppErrorBoundary` → `I18nActivationGate` → `I18nProvider` → `ApolloDevLogEffect` → `SnackBarComponentInstanceContext` → `IconsProvider` → `ExceptionHandlerProvider` → `HelmetProvider` → `ClickOutsideListenerContext` → `DomainShell`。
- `[Evidence]` Apollo 不在根，而在 `SharedAppProviders`/`WorkspaceAppProviders` 内（根需在未知 auth/clientConfig 前可渲染）。

## 2. 页面 / 路由 / 布局

- `[Evidence]` **React Router v6 数据路由 API**（`createBrowserRouter`+`createRoutesFromElements`），声明式（非文件式），页面 `lazy()` 于 `src/pages/`。证据：`src/modules/app/hooks/useCreateWorkspaceAppRouter.tsx:139-336`。
- `[Evidence]` Shell：`DomainShell`（按 `isMultiWorkspaceEnabled` 分流 `WorkspaceApp`/`RootApp`）。证据：`DomainShell.tsx:14-40`。
- `[Evidence]` 嵌套布局：`WorkspaceAppProviders → MinimalMetadataGate → DefaultLayout → MainAppLayoutWithSidePanel` → `RecordIndexPage/RecordShowPage/PageLayoutPage/SettingsCatchAll/NotFoundWildcard`；auth 走 `AuthFlowLayout`；onboarding 走 `BlankLayout+OnboardingStepLayout`。
- `[Evidence]` 懒加载助手 `LazyRoute` + `lazyWithPreload`；onboarding 页经 route `loader` 预加载。
- `[Evidence]` Settings 用旧版 `<Routes>` API + `SettingsPath` enum，含 legacy `/api-webhooks` 重定向（TODO 2026-08-04 移除）。证据：`SettingsRoutes.tsx`。

## 3. 组件层级与目录组织

- `[Evidence]` 模块位于 `src/modules/`，按功能域组织，每个域内典型子目录：`components/`、`states/`(atoms)、`hooks/`、`utils/`、`schemas/`、`graphql/`、`queries|mutations|fragments/`、`validation/`。
- `[Evidence]` 最大的模块（按文件数）：`object-record`(1891)、`settings`(1027)、`page-layout`(910)、`workflow`(625)、`ui`(583，含 Jotai 状态层)、`side-panel`(416)、`ai`(355)、`views`(298)、`activities`(285)。
- `[Evidence]` 共享 UI 原语在 `packages/twenty-ui`，前端经 `twenty-ui/*` 子路径导入（`twenty-ui/icon`、`twenty-ui/style.css`）。

## 4. 状态管理

- `[Evidence]` **Jotai**，自定义 store `jotaiStore` + `resetJotaiStore()`（清 session localStorage 并重建）。证据：`src/modules/ui/utilities/state/jotai/jotaiStore.ts`。
- `[Evidence]` 原子工厂 `createAtomState`（封装 `atom`/`atomWithStorage`，支持 localStorage/sessionStorage/cookie + `validateInitFn`）；hooks `useAtomState/useAtomStateValue/useSetAtomState`。证据：`.../utils/createAtomState.ts`、`.../hooks/`。
- `[Evidence]` 关键 auth atoms：`currentWorkspaceState`、`currentWorkspaceMemberState`(localStorage getOnInit)、`currentUserState`、`tokenPairState`、`isCookieAuthActiveState`、`availableWorkspacesState`、`isImpersonatingState`。证据：`src/modules/auth/states/`。
- `[Evidence]` object-metadata selectors（20 个派生原子，`objectMetadataItemsSelector` 等）经 `createAtomSelector`。证据：`src/modules/object-metadata/states/`。
- `[Evidence]` **无 Recoil**（旧前提失效）。

## 5. 数据获取

- `[Evidence]` Apollo 链（`apollo.factory.ts:415-424`）：`errorLink → authLink → extraLinks(captcha) → logger? → retryLink → streamingRestLink → restLink(apollo-link-rest) → uploadLink(apollo-upload-client)`。
- `[Evidence]` authLink：cookie 激活时 `credentials:'include'`，否则 Bearer；加 `x-locale`、`X-App-Version`。证据：`apollo.factory.ts:133-172`。
- `[Evidence]` token 续期：RxJS 去重(`renewalPromise`) + `retryWithBackoff`(3 次)，含 cookie-auth 回退（混合舰队滚动）。证据：`apollo.factory.ts:194-285`。
- `[Evidence]` errorLink：`UNAUTHENTICATED`→续期；`APP_VERSION_MISMATCH/NOT_FOUND/FORBIDDEN/CONFLICT/METADATA_VALIDATION_FAILED` 静默；其余→`@sentry/react`。证据：`apollo.factory.ts:345-411`。
- `[Evidence]` 缓存：`InMemoryCache`，`typePolicies:{RemoteTable:{keyFields:['name']}}`，默认 `fetchPolicy:'cache-and-network'`。证据：`useApolloFactory.ts:54-66`。
- `[Evidence]` 三个 client：`ApolloProvider`(/metadata, SharedAppProviders)、`ApolloCoreProvider`(/graphql, WorkspaceAppProviders)、`ApolloAdminProvider`(/admin-panel)。
- `[Evidence]` 三套 codegen → `generated/graphql.ts`(969)、`generated-metadata/graphql.ts`(9694)、`generated-admin/graphql.ts`(1407)；插件 `typescript/typescript-operations/typed-document-node`；nx 目标 `graphql:generate --config=`。

## 6. 表单 / 校验 / 错误处理 / 加载/空状态

- `[Evidence]` 表单校验模式由各域 `validation/` 或 zod schema 提供（`object-record`、`spreadsheet-import` 等含 schema）。
- `[Evidence]` 错误边界：根 `AppErrorBoundary`(`AppRootErrorFallback`)；`ExceptionHandlerProvider` 统一异常处理。
- `[Evidence]` 加载：`react-loading-skeleton`（`index.tsx:13`）；空状态/加载组件散布于各模块（无统一空状态框架）。

## 7. 样式系统 / Design Token / 主题

- `[Evidence]` **Linaria** 经 `@wyw-in-js/vite`（仅处理组件/`*View` 代码，大量 exclude）；babel preset `@babel/preset-typescript + preset-react`。证据：`vite.config.ts:98-135`。
- `[Evidence]` **Design Token 有定义但分三处**：
  1. JS 主题对象（`twenty-ui/src/theme/constants/ThemeLight.ts:15-32` 等，`THEME_LIGHT/DARK/COMMON`）。
  2. CSS 变量镜像 `--t-*`（`twenty-ui/src/theme-constants/themeCssVariables.ts` + `theme-light.css`/`theme-dark.css`），文件头自述"mirrored token-for-token ... kept in sync by the theme parity test"。
  3. `index.html` 内联 FOUC 变量（`--theme-dark/light-background-secondary`）。
- `[Evidence]` 主题应用：`BaseThemeProvider`（读 `persistedColorSchemeState` + `useSystemColorScheme()`，包 `twenty-ui/theme-constants ThemeProvider`）+ `UserThemeProviderEffect`（同步用户偏好）。证据：`src/modules/ui/theme/components/BaseThemeProvider.tsx:17-36`。
- `[Evidence]` parity 仅靠单测 `cornerShapeThemeParity.test.ts`，无单一 schema。

## 8. 响应式 / 移动端 / 可访问性

- `[Evidence]` 移动检测 `useIsMobile`（`react-responsive` `useMediaQuery` 对 `MOBILE_VIEWPORT`）；移动专用组件（`MobileNavigationBar`、`CommandMenuForMobile` 等）。证据：`src/modules/ui/utilities/responsive/hooks/useIsMobile.ts`。
- `[Evidence]` iOS：`index.html:44-60` 检测 iOS 并 `maximum-scale=1.0` 防输入缩放。
- `[Evidence]` a11y：`twenty-ui/src/accessibility/{components,utils}`；前端组件内散布 `aria-*`/`role`。`[Inference]` a11y 是组件级，无统一 a11y 框架/审计。

## 9. 国际化 / 图标 / 静态资源

- `[Evidence]` Lingui（PO 格式，`printLinguiId:true`），31 locale key（en + pseudo-en + 29 真实）。证据：`lingui.config.ts:1-22`、`AppLocales.ts:3-35`。
- `[Evidence]` 激活：`initialI18nActivate()`（`App.tsx:17`）+ `I18nActivationGate`。
- `[Evidence]` 图标：`IconsProvider`(twenty-ui/icon)。
- `[Evidence]` 字体：自托管 `@fontsource/inter|dm-mono`；BlockNote 字体在 `server.fs.allow` 白名单。证据：`index.tsx:7-12`、`vite.config.ts:85-87`。

## 10. 前端测试 / 构建

- `[Evidence]` 单测 Jest（`jest.config.mjs`，swc + lingui）；Storybook + vitest + Playwright（视觉回归经 Argos dispatch）。证据：`nx.json` test/storybook 目标、`.storybook/main.ts`。
- `[Evidence]` 构建：Vite `esbuild` minify，outDir `build`，自定义 `chunk-size-limit`（主 chunk 6.8MB / 其他 5MB，超限构建失败）。证据：`vite.config.ts:174-214`。

## 11. 桌面/移动/浏览器差异

- `[Evidence]` 仅浏览器 SPA（无桌面/移动原生壳）；移动靠响应式 + 移动组件；iOS 特殊处理见上。

## 12. 四类规范区分小结

- **(1) 源码明确存在的规范**：函数组件 + named export + kebab-case 文件 + `.component.tsx/.service.ts` 等后缀（`CLAUDE.md` 声明且代码一致）；Jotai 原子按域 `*/states/` 共置；三 Apollo client 对应三端点；Linaria 仅组件；自托管字体；chunk size 硬上限。
- **(2) 可推断约定**：路由用数据 API + `lazy()`；selectors 用 `createAtomSelector`；errorLink 按错误码分流；token 续期去重。
- **(3) 仓库无证据的规范**：**无统一 Design Token 单一来源**（三处镜像）；**无统一空状态/加载框架**；**无中心 a11y 审计**；**无统一表单库**（各域自选）。
- **(4) [Recommendation] 推荐**：见末尾。

## UI 技术债务

- `[Evidence]` Design Token 三源（JS/CSS 镜像/HTML 内联），靠人工 parity 测试，易漂移。
- `[Evidence]` 元数据三真相源（Apollo `/metadata` cache + object-metadata selectors + `metadata-store` IndexedDB atom family，文件内甚至有 TODO"clarify what really is metadata"）。
- `[Evidence]` Provider 栈深（`WorkspaceAppProviders` 约 12 层，含 effect 组件与 provider 混排，顺序敏感）。
- `[Evidence]` 孤儿 locale 目录（`aa-ER.po`、`hy-AM.po`、`uz-UZ.po` 不在 `APP_LOCALES`，永不编译入）。
- `[Evidence]` 单 SPA bundle（9.7k 行 generated-metadata + 1891 文件 object-record），仅靠 `lazy()` 与 chunk 守卫缓解。

## 前端安全边界

- `[Evidence]` token 存 localStorage（`tokenPairState`），XSS 风险依赖 CSP/sanitize；HTML 块预览经 `sanitizeHtmlPreview`（`HtmlNodeView.tsx:19-30`）。
- `[Evidence]` CSRF 由后端 Origin 校验承担；前端 cookie 模式 `credentials:'include'`。

## 已确认事实 / 合理推断 / Unknown

- 见各节。`[Unknown]` 各域表单/校验库是否统一（未逐域核）。

## 批判性评估

- Design Token 与元数据均存在"多真相源 + 人工 parity 测试"，是前端最脆的两处。

## 建设性改善建议

- `[Recommendation]` 将 Design Token 收敛为单一来源（如代码生成 CSS 变量），消除手写镜像。优先级：高；难度：中。依据：`themeCssVariables.ts:1-2`。
- `[Recommendation]` 明确 `metadata-store`(IndexedDB) 与 object-metadata selectors 的关系（文件内已有 TODO）。优先级：中；难度：中。依据：`metadataStoreState.ts:34`。
- `[Recommendation]` 清理孤儿 locale 或加入 `APP_LOCALES`。优先级：低；难度：低。

## 主要证据索引

- `packages/twenty-front/src/index.tsx:1-31`；`index.html`
- `packages/twenty-front/src/modules/app/components/App.tsx:19-49`；`DomainShell.tsx:14-40`
- `packages/twenty-front/src/modules/app/hooks/useCreateWorkspaceAppRouter.tsx:139-336`
- `packages/twenty-front/src/modules/apollo/services/apollo.factory.ts:133-424`；`hooks/useApolloFactory.ts:54-143`
- `packages/twenty-front/src/modules/ui/utilities/state/jotai/{jotaiStore.ts,utils/createAtomState.ts}`
- `packages/twenty-front/vite.config.ts:98-135,174-214,275-291`
- `packages/twenty-ui/src/theme-constants/themeCssVariables.ts:1-2`；`src/theme/constants/ThemeLight.ts:15-32`
- `packages/twenty-front/lingui.config.ts:1-22`；`packages/twenty-shared/src/translations/constants/AppLocales.ts:3-35`
