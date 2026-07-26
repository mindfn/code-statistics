---
feature_ids: [F001, F002, F003]
topics: [architecture, maintenance, testing, release, github, gitcode]
doc_kind: guide
created: 2026-07-25
---

# 维护与迭代指南

本文面向继续开发本项目的人，记录当前实现的模块边界、不可破坏的统计口径、常见改动的落点、验证方法和发布流程。

用户操作、Token 权限、统计口径及已知限制见 [README.md](README.md)。本文件与生产代码 `index.html`、生产代码测试 `test.html` 共同构成后续迭代的仓库内真相源。

## 1. 项目边界

当前交付物是一个可直接打开的静态单页应用：

- 所有 HTML、CSS、JavaScript 和 favicon 都内联在 `index.html`。
- 无服务端、数据库、构建工具、运行时依赖或 CDN。
- `localStorage` 只保存 Token、仓库和用户配置。
- 当前查询投影只存在于页面内存；完整已合入 PR 在 `http(s)` 下按 PR 持久化到浏览器 IndexedDB，`file://` 下使用页面内存缓存。
- CSV 是统计结果的可移植持久交付物。
- GitHub Pages 只托管静态文件；浏览器直接请求 GitHub / GitCode API。

除非需求明确改变产品形态，否则不要为局部功能引入后端、框架、第三方依赖或新的持久化服务。

### 当前发布基线

| 阶段 | 最终 SHA | 浏览器测试 | Review / 发布证据 |
|---|---|---|---|
| F001 PR 代码变更统计 | `6523cc0` | 65 passed | sol APPROVE |
| F002 Comments / Approve 与三类详情 | `b4b57a5` | 122 passed | sol APPROVE，Pages 验收通过 |
| 最终 UI、导出诊断、Token 保存与隐私说明 | `876bec3` | 137 passed | kimi APPROVE，Pages 验收通过 |

本文创建时线上生产基线为 `876bec3`。后续发布应把新的最终 SHA、测试数量和线上验收结果补到本表或后续变更记录中。

## 2. 代码地图

`index.html` 仍是单文件，但 JavaScript 用固定 section comment 划分模块。查找这些标题即可定位：

| Section | 职责 |
|---|---|
| `Utilities` | HTML 转义、仓库 URL 解析、日期半开区间、并发限制器 |
| `Storage` | 版本化 `localStorage` 读写 |
| `CacheStore` | 版本化 merged PR cache；HTTP IndexedDB / file Memory 后端 |
| `ProviderRequestBudget` | 所有真实网络请求的平台级并发预算（GitHub 8 / GitCode 16） |
| `CSV Helper` | RFC 4180 解析/生成、BOM、公式注入防护、下载 |
| `GitHub Provider` | 默认分支、merged PR、代码量、Comments、Reviews |
| `GitCode Provider` | 默认分支、merged PR、代码量、Comments、当前有效 Approve |
| `Statistics Engine` | 仓库去重、用户映射、活动聚合、筛选、完整性 |
| `Query Controller` | 查询编排、进度、取消、错误与部分统计传播 |
| `CSV Export` | 从当前可见投影导出汇总或明细 |
| `Token Management` | 保存、清除、测试并保存 Token |
| `User CRUD` / `Repo Management` | 本地配置维护 |
| `Initialization` | 恢复配置、默认日期、事件绑定 |

主数据流：

```text
localStorage（配置、用户）
  → collectFilters / buildRepoList
  → Provider.getDefaultBranch
  → Provider.fetchMergedPRs（仓库级发现 → PR cache hit / core hydration）
  → fetchCollaborationForPR（cache hit / Provider.fetchCollaboration → full-only cache write）
  → 规范化 activity
  → mapActivitiesToUsers
  → applyDisplayFilters
  → aggregate
  → 页面 / 汇总 CSV / 明细 CSV
```

页面和导出必须共享同一批筛选后的活动及汇总数据。不要在导出函数中另写一套业务筛选。

## 3. 持久化契约

配置使用两个版本化 localStorage Key：

```text
code-statistics.config.v1
code-statistics.users.v1
```

等价数据形态：

```js
// code-statistics.config.v1
{
  githubToken: "…",
  gitcodeToken: "…",
  repositories: [
    "https://github.com/owner/repo",
    "https://gitcode.com/owner/repo"
  ]
}

// code-statistics.users.v1
[
  {
    user_key: "alice",
    display_name: "Alice",
    email: "alice@example.com",
    github_login: "alice-gh",
    gitcode_login: "alice-gc",
    enabled: true
  }
]
```

完整已合入 PR 使用 IndexedDB：

```text
Database: code-statistics.pr-cache.v1
Object store: prs
Key: platform:repository-lowercase#pr-number
```

```js
{
  schemaVersion: 1,
  cachedAt,
  platform,
  repository,
  prNumber,
  core,
  activities,       // raw activity；禁止 user_key / matched / display_name
  completeness: "full",
  partialReason: ""
}
```

缓存规则：

1. 只有当前 schema、结构一致且 `core/activities` 全部为 `full` 的条目可命中。
2. schema 不匹配、结构损坏、partial 或 failed 一律 miss；损坏/旧 schema 条目在读取时删除。
3. 已合入 PR 的 core + collaboration 整体不设时间 TTL；产品明确不追踪少数 merge 后评论/审批变化。
4. `http(s)` 使用 IndexedDB；初始化失败时降级为当前页面 Memory，不影响统计正确性。`file://` 不尝试 IndexedDB。
5. IndexedDB 写入只在 readwrite transaction `complete` 后返回成功。
6. 用户资料不是 cache key；用户变化只重新投影当前 raw activities，不使 PR cache 失效。

演进规则：

1. 新增可选字段时必须为旧数据提供默认值。
2. 破坏性结构变更必须使用新 Key（如 `.v2`）并实现可回滚迁移。
3. 迁移完整成功前不得覆盖或删除旧 Key。
4. Token 不得进入 URL、日志、CSV 或错误详情；界面只显示掩码。
5. 清除浏览器站点数据会清除全部配置与 PR cache，此行为必须继续在界面和 README 中说明。

## 4. Provider 契约

每个平台 Provider 都必须实现以下边界：

```js
testConnection(token)
getDefaultBranch(token, owner, repo, signal)
fetchMergedPRs(token, owner, repo, defaultBranch, startDate, endDate, signal, onProgress, cacheStore?)
fetchCollaboration(token, owner, repo, pr, signal, onProgress)
```

`fetchMergedPRs` 返回：

```js
{
  prs: [
    {
      platform, repo, number, title, author,
      mergedAt, baseBranch,
      additions, deletions, changedFiles,
      completeness, url
    }
  ],
  partial: false,
  partialReason: "",
  cacheHits: 0,
  networkPRs: 0
}
```

`fetchCollaboration` 返回：

```js
{
  activities: [/* 规范化 activity */],
  partial: false,
  partialReason: ""
}
```

规范化 activity 的稳定字段：

```js
{
  activity_type: "development" | "comment" | "approval",
  platform,
  repository,
  pr_number,
  pr_title,
  pr_url,
  merged_at,
  actor_login,
  user_key,
  matched,
  display_name,
  event_at,
  source_type,
  source_id,
  activity_url,
  body_summary,
  additions,
  deletions,
  changed_files,
  completeness: "full" | "partial" | "failed",
  partial_reason
}
```

新增平台时应先适配这两个返回结构，不要让平台特有字段泄漏到 `StatsEngine`。

## 5. 统计不变量

下列规则属于产品契约，修改前必须先更新需求与测试：

1. 只统计合入仓库默认分支的已合并 PR。
2. 日期按用户本地日期解释，内部使用 `[开始日 00:00, 结束日次日 00:00)`。
3. 三类活动都随 PR 的 `merged_at` 归入查询周期。
4. 用户归属只按 `platform + actor_login` 匹配；邮箱不参与归属。
5. 禁用用户不参与匹配，未匹配账号保留为独立诊断行。
6. 开发 PR 的代码量只归 PR 作者；Comment 和 Approve 不增加代码量。
7. `变更行 = 新增行 + 删除行`，`净增行 = 新增行 - 删除行`。
8. GitHub Comment 来源包含 issue comment、review comment、非空 review body。
9. GitHub Approve 取同一 PR、同一用户最后一个决定性 Review 状态。
10. GitCode Approve 只表示查询时 `accept === true` 的当前有效状态。
11. 不同来源按 `activity_type + source_type + source_id` 去重。
12. 失败、限流、字段不可解析或 `too_large` 不得静默当作 0；必须传播为“部分”或“失败”。
13. 同一查询每个仓库只执行一次 PR 发现流程；协作来源按每个入选 PR 拉取一次。
14. 汇总页面、详情和两种 CSV 必须遵守相同的平台、用户、仓库及用户范围筛选。
15. cache hit 必须是同 schema、同 PR identity 的完整条目；partial/failed 不能命中。
16. 缓存只保存 raw activity；用户归属永远从当前 users 重新投影。
17. 所有实际网络请求必须经过 Provider `_fetch` 的共享预算；PR 内部 fan-out 不得绕过 GitHub 8 / GitCode 16 上限。
18. 进度完成数只在 cache hit 或网络任务 settle 后递增；最终标题和仓库行不得保留“正在获取”。

## 6. 安全边界

- 所有 API 或用户输入写入 `innerHTML` 前必须经过 `escapeHtml`。
- 外部链接保持 `target="_blank"` 与 `rel="noopener"`。
- CSV 文本继续通过 `CsvHelper.sanitize` 防止公式注入。
- Token 只能放在认证请求头，不得放进 query string。
- 不在仓库、截图、测试夹具或文档中写入真实 Token。
- `file://` 页面直接请求平台 API；新增网络目标时必须在配置页和 README 明示。
- API 失败信息可以展示状态和平台返回摘要，但不得拼入认证请求头或 Token。

## 7. 常见迭代怎么改

### 新增一个平台

1. 扩展 `parseRepoUrl` 和仓库平台展示。
2. 增加 Token 配置、测试连接和版本化存储字段。
3. 实现完整 Provider 契约，输出规范化 PR 与 activity。
4. 在 `runQuery` 中只增加 Provider 选择，不复制查询编排。
5. 为用户模型增加该平台账号字段及 CSV 导入兼容。
6. 增加分页、限流、字段缺失、部分失败和用户匹配测试。
7. 更新 README 的权限、网络目标、口径和限制。

### 新增一种活动或指标

1. 先定义归属用户、时间口径、去重键和完整性传播。
2. Provider 只负责生成新的规范化 activity。
3. 在 `StatsEngine.aggregate` 增加聚合，不在 UI 临时计算业务口径。
4. 同步汇总列、详情、CSV 字段和空态。
5. 用固定夹具覆盖“只有该活动、没有开发 PR”的用户。

### 修改筛选或导出

1. 优先修改 `StatsEngine.applyDisplayFilters` 或可见数据投影。
2. 页面、详情和 CSV 都消费该投影。
3. 至少覆盖 `matched / all / unmatched × platform × user × repo`。
4. 检查 partial/failed 诊断只跟随当前可见仓库，不生成空平台行。

### 修改本地数据结构

1. 写旧数据加载测试。
2. 增加迁移，不直接改变旧 Key 的语义。
3. 验证刷新后配置仍在、失败迁移不丢数据。
4. 更新本文件“持久化契约”和 README 的用户可见说明。

### 修改 PR cache

1. 先修改 `CACHE_SCHEMA_VERSION` 与 entry validation 测试；不在旧 schema 上猜测兼容。
2. 缓存只接受 full 数据，禁止为了命中率放宽 partial/failed 门禁。
3. 不把 `user_key`、`matched`、`display_name` 或 Token 写入 entry。
4. IndexedDB mutation 必须等待 transaction complete；用 close → reopen 测试持久性。
5. 同步 README 的协议边界与隐私披露。

### 修改 UI

1. 保持“配置 / 用户信息 / 统计数据”三个一级页签。
2. 保持单文件与无外部静态资源。
3. 桌面优先，同时检查窄屏布局、键盘操作、空态和错误态。
4. 若修改 DOM 标识或文案，同步 `test.html` 的生产 DOM 断言。

## 8. 测试与质量门禁

测试页通过隐藏 iframe 加载真实 `index.html`，不是复制生产函数。

启动本地静态服务器：

```bash
python3 -m http.server 8901 --bind 127.0.0.1
```

然后访问：

```text
http://127.0.0.1:8901/test.html
```

F003 开发分支基线为 `212 passed, 0 failed`；新增行为时测试总数应增加，不应通过删除断言维持全绿。

静态检查：

```bash
sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' | node --check -
sed -n '/^<script>$/,/^<\/script>$/p' test.html | sed '1d;$d' | node --check -
git diff --check
```

行为变更采用红绿流程：

1. 在 `test.html` 增加能复现问题或表达新契约的失败断言。
2. 确认旧生产代码失败。
3. 修改 `index.html`。
4. 用同一断言确认转绿，再跑全部测试。

浏览器手工验收至少覆盖：

- 成功测试 Token 后自动保存；失败测试不覆盖旧 Token。
- 用户和仓库 CRUD，刷新页面后仍可恢复。
- 日期、平台、用户、仓库、用户范围组合筛选。
- 开发 PR / Comments / 审核 PR 三个详情标签及键盘操作。
- 汇总和明细 CSV 与当前页面一致，且不含 Token。
- 单仓库失败、部分统计、取消查询时其他状态不被伪装成完整。
- 浏览器控制台无新增错误。

## 9. Review 与发布

任何代码或测试行为变更都需要非作者 Review，且 Review 必须覆盖准备发布的最终 SHA。

建议流程：

1. 建立功能分支或隔离 checkout。
2. 红测试 → 实现 → 全量测试 → 静态检查。
3. 由非作者复核原始需求、完整 diff、测试和浏览器行为。
4. Review 明确 APPROVE 后再合入或推送 `main`。
5. `main` 更新会触发 `.github/workflows/deploy.yml`。
6. 等 `Deploy to GitHub Pages` 成功后，再检查线上页面包含本次变更。

查看部署：

```bash
gh run list --commit "$(git rev-parse HEAD)" --limit 5
```

线上地址：

```text
https://mindfn.github.io/code-statistics/
```

部署成功不等于功能验收完成；至少做一次线上 smoke test。若 Actions 失败，线上仍是上一次成功版本。

## 10. 文档同步清单

每次迭代完成前逐项检查：

- 用户操作、权限、口径或限制变化 → 更新 `README.md`。
- 模块边界、数据契约、扩展方式或门禁变化 → 更新本文件。
- 生产行为变化 → 更新 `test.html`。
- API 行为变化 → 更新 Provider、失败传播测试及已知限制。
- 发布后 → 记录最终 SHA、测试结果和线上验收证据。

`docs/features/`、`docs/discussions/` 与 `feature-specs/` 已纳入版本控制；产品口径变更必须同步 feature truth，不能只改 README 或生产代码。
