# 仓库分析基线快照 (Repository Analysis Baseline)

## 任务性质
只读式系统架构审计。任务产生的文件仅写入 `./docs-analysis/`。

## 环境基线 (任务开始前)
- 工作目录: `/home/lin/agent/twenty`
- 当前分支: `main` (== `origin/main`, == `origin/HEAD`)
- HEAD commit: `4dbc5a567ef4f8465ca69632cc9fec8eff388732`
- 最近提交信息: `feat(slack): strip the bot's own mention anywhere in the request text (#23837)`
- 工作区状态: **clean** (`git status --porcelain` = 0 行)
- 任务开始前用户未提交修改: **无**
- 仓库类型: monorepo (Nx workspace, Yarn 4)
- 分析时间: 2026-08-07

## Git 子模块
- `.gitmodules`: 不存在
- `git submodule status --recursive`: 无输出 (没有子模块)
- 结论: 本仓库**没有任何 Git 子模块**。后续文档中"子模块"相关章节将统一记录为 `[Evidence] 当前仓库未使用 Git 子模块`。

## 跟踪文件规模
- `git ls-files | wc -l` = 27714 个被 Git 跟踪的文件
- 顶层目录 (tracked): `.claude`, `.cursor`, `.github`, `.vscode`, `.yarn`, `packages` 以及若干根级配置文件

## packages 子目录 (22 个)
create-twenty-app, twenty-apps, twenty-claude-skills, twenty-cli, twenty-client-sdk,
twenty-codex-plugin, twenty-docker, twenty-docs, twenty-e2e-testing, twenty-emails,
twenty-front, twenty-front-component-renderer, twenty-oxlint-rules, twenty-sdk,
twenty-server, twenty-shared, twenty-ui, twenty-utils, twenty-website, twenty-zapier

## 写入边界
- 本任务新增文件仅位于 `./docs-analysis/` 及其 `evidence/`, `scripts/` 子目录。
- 未对任何源码、配置、测试、CI、依赖清单、锁文件、数据库、migration、子模块、构建产物做修改。
- 未执行 git add / commit / checkout / restore / reset / clean / stash / switch / rebase / merge。

## 备注
- 由于工作区在任务开始时为 clean, 任务期间若 `git status` 出现变化, 必然全部由本任务在 `./docs-analysis/` 内产生。
- 后续每份正式文档将在开头注明本基线。
