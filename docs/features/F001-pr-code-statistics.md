---
feature_ids: [F001]
related_features: [F002]
topics: [github, gitcode, pull-request, statistics, csv, local-first]
doc_kind: spec
created: 2026-07-24
---

# F001: GitHub / GitCode PR 代码变更统计

> **Status**: done | **Owner**: opus | **Priority**: P1 | **Reviewed by**: sol (APPROVE @ `6523cc0`)

## Why

> operator experience（2026-07-24）：
> “需要统计用户在 GitHub/GitCode 上指定时间范围内的代码数据……基于 PR 分析”，并希望任何用户只配置自己的 Token 就能使用。

给本地单用户提供一个无需安装服务、无需外部数据库的统计工具：维护人员与仓库映射后，可以按时间、用户和仓库准确查询已合并 PR 的代码变更量，并用 CSV 交付结果。

## Current State / 现状基线

- N/A（无既有业务代码）；仓库当前只有治理文档。
- 2026-07-24 已完成真实 Chrome `file://` 验证：
  - GitHub/GitCode PR API 均允许浏览器携带认证请求头并读取响应。
  - `localStorage` 跨浏览器重开可持久化。
  - CSV Blob 下载可用。
  - IndexedDB 在 `file://` 下打开超时，不采用。
- 2026-07-24 已用真实凭据完成只读 API 验证：
  - GitHub 可读取默认分支、已合并 PR、PR 作者、`merged_at`、`additions`、`deletions`、`changed_files`。
  - GitCode 可读取默认分支、已合并 PR、PR 作者、`merged_at` 和文件级增删行。
  - GitCode 默认分支过滤必须使用 `base=<default_branch>`；`target_branch` 查询参数实测无效。
- 临时浏览器探针已清理，不是项目交付物，也不是当前真相源。

## What

交付一个可直接双击打开的 `index.html`，所有 HTML、CSS 和 JavaScript 都内联，不使用 CDN、构建工具、运行时服务或第三方依赖。

### 三个页签

#### 1. 配置

- GitHub Token、GitCode Token 的录入、掩码显示、保存和清除。
- 分平台“测试连接”，显示成功账户、认证失败、限流或网络/代理错误。
- 代理只说明“沿用操作系统/浏览器代理”，并提示必须能访问：
  - `https://api.github.com`
  - `https://api.gitcode.com`
- Token 仅保存在浏览器 `localStorage`，不进入日志、CSV 或 JSON 导出。

#### 2. 用户信息

- 维护用户键、显示名、邮箱、GitHub 账号、GitCode 账号、仓库地址和启用状态。
- 一行数据表示一个“用户—仓库”关系；同一用户可关联多个仓库。
- 支持手工新增、编辑、删除。
- 支持下载 CSV 模板、CSV 导入预览、校验后确认导入。
- 仓库平台和 `owner/repo` 从 URL 解析；只接受 GitHub/GitCode 仓库 URL。

用户 CSV：

```csv
user_key,display_name,email,github_login,gitcode_login,repository_url,enabled
```

#### 3. 统计数据

- 筛选：开始日期、结束日期、平台、用户、仓库。
- 查询：按仓库拉取一次 PR 数据，再按 PR 作者账号映射用户；禁止按“用户 × 仓库”重复请求。
- 进度：显示当前仓库、请求进度、成功/失败/部分统计数量。
- 汇总表：平台、用户、仓库、合并 PR 数、新增行、删除行、变更行、净增行、变更文件数、完整性。
- PR 明细表：平台、仓库、PR 编号、作者、合并时间、目标分支、增删行、变更文件数、链接、完整性。
- 导出当前筛选后的“汇总 CSV”和“PR 明细 CSV”；UTF-8 BOM、RFC 4180 转义，并防止表格公式注入。

### 固定统计口径

- 只统计已合并 PR。
- 时间归属以 `merged_at` 为准；界面日期按用户本地时区解释，内部使用半开区间 `[开始, 结束次日)`。
- 用户归属按 PR 作者账号匹配；邮箱只作为人员资料，不参与归属。
- 只统计合入仓库默认分支的 PR。
- GitHub 与 GitCode 镜像仓库不自动去重。
- 指标含义：
  - 变更行数 = 新增行 + 删除行
  - 净增行数 = 新增行 − 删除行
- 不统计直接 push、未合并 PR、他人 PR 中的个人 commit。
- 产品文案统一称“PR 代码变更量”，不称“产能”或“有效代码量”。

### Provider 行为

#### GitHub

1. `GET /repos/{owner}/{repo}` 获取默认分支。
2. 查找默认分支、指定合并日期范围内的 merged PR。
3. `GET /repos/{owner}/{repo}/pulls/{number}` 获取作者、`merged_at` 和代码量。
4. 对最终结果再次按 `merged_at` 和默认分支过滤，不能只信搜索条件。
5. 处理分页、403 限流和搜索 1000 条上限；无法完整覆盖时必须显示“部分统计”，不能静默少算。

#### GitCode

1. 使用官方基址 `https://api.gitcode.com/api/v5`。
2. `GET /repos/{owner}/{repo}` 获取默认分支。
3. `GET /repos/{owner}/{repo}/pulls?state=merged&base={default_branch}&page=...&per_page=100` 分页读取。
4. 不以 `since` 作为精确合并时间口径；在浏览器中按 `merged_at` 做最终范围过滤。
5. 从 PR 返回字段及 `/pulls/{number}/files` 获取、校验代码量和变更文件数。
6. 遇到 `too_large`、文件列表不完整或字段不可解析时标记“部分统计”，不能按 0 计。

### 本地数据与模块边界

`localStorage` 只持久化版本化 JSON：

- `code-statistics.config.v1`
- `code-statistics.users.v1`

统计结果只保存在当前页面内存中，刷新后重新查询；CSV 是统计结果的持久交付物。

即使代码放在一个文件中，也保持下列逻辑边界：

```text
UI / State
  ├─ Storage
  ├─ CSV
  ├─ GitHubProvider
  ├─ GitCodeProvider
  └─ Statistics
```

## User Journey

### Primary Journey: 配置后查询并导出个人 PR 代码变更量

- **Scope unit**: workspace
- **Actor**: 本地单用户
- **Entry**: 双击 `index.html`
- **Flow**:
  1. 用户进入“配置”，填写 GitHub/GitCode Token 并测试连接。
  2. 用户进入“用户信息”，手工维护或用 CSV 导入用户、平台账号与仓库关系。
  3. 用户进入“统计数据”，选择日期范围，可选用户/仓库筛选并开始查询。
  4. 页面展示逐仓库进度、失败/部分统计提示、汇总和 PR 明细。
  5. 用户导出当前筛选结果的汇总 CSV 或明细 CSV。
- **Success evidence**: 最新 Chrome/Edge 的完整流程截图、下载 CSV、与两个平台已知 PR 原始数据逐项核对。
- **Non-goals**: 本地/远程后端服务、多人权限、定时任务、commit 统计、跨平台镜像去重、页面内自定义代理、云端同步。

### Supporting Journeys

| ID | Scope unit | Actor | Flow | Evidence |
|----|------------|-------|------|----------|
| S1 | workspace | 新用户 | 下载 CSV 模板 → 填写 → 导入预览 → 确认 → 用户/仓库关系可见 | 导入测试与截图 |
| S2 | repository | 本地单用户 | 某仓库请求失败 → 其他仓库继续 → 结果明确标记不完整 | 错误注入测试与截图 |
| S3 | workspace | 本地单用户 | 关闭浏览器 → 重开 `index.html` → 配置和用户仍在 | 浏览器重开截图 |

## Text Wireframe / Design Gate

operator 已明确要求“只要三个管理页签”，采用以下功能线框，不增加首页或额外导航：

```text
┌─────────────────────────────────────────────────────────────┐
│ PR 代码变更统计       [配置] [用户信息] [统计数据]           │
├─────────────────────────────────────────────────────────────┤
│ 配置：GitHub Token + 测试连接                               │
│       GitCode Token + 测试连接                              │
│       系统/浏览器代理说明                                   │
├─────────────────────────────────────────────────────────────┤
│ 用户：操作栏（新增 / CSV 导入 / 下载模板）                  │
│       用户—账号—仓库关系表                                  │
├─────────────────────────────────────────────────────────────┤
│ 统计：日期 / 平台 / 用户 / 仓库 / 查询 / 导出               │
│       进度与完整性提示                                      │
│       汇总表 / PR 明细表                                    │
└─────────────────────────────────────────────────────────────┘
```

视觉目标：本地管理工具风格、信息密度适中、桌面优先并兼容窄屏；不引入图表，避免图形误导代码量含义。

## 需求点 Checklist

| ID | 需求点（operator experience/转述） | AC 编号 | 验证方式 | 状态 |
|----|---------------------------|---------|----------|------|
| R1 | “录入用户的用户名/邮箱、GitHub/GitCode 账号和仓库地址” | AC-A1, AC-A2 | 浏览器流程截图、CSV 导入测试 | [x] |
| R2 | “看到用户提交的代码量，基于 PR 分析” | AC-B1, AC-B2, AC-B3 | 已知 PR 数据对照测试 | [x] |
| R3 | “按照 CSV 格式导出数据” | AC-C1 | CSV 内容自动测试、Excel/Numbers 打开检查 | [x] |
| R4 | “只有三个管理页签” | AC-A1 | 全页截图 | [x] |
| R5 | “按时间范围、用户、仓库筛选和查询” | AC-B2 | 筛选组合测试 | [x] |
| R6 | “只配置 Token 就能使用，尽量少外部依赖” | AC-A3 | 离线资产检查、双击运行测试 | [x] |
| R7 | “使用系统/浏览器代理即可” | AC-A4 | 代理说明截图、连接失败分类测试 | [x] |
| R8 | “已合并 PR + 合并时间 + PR 作者 + 默认分支 + 不跨平台去重” | AC-B1 | Provider 单元测试、真实 API 对照 | [x] |

### 覆盖检查

- [x] 每个需求点都映射到至少一个 AC。
- [x] 每个 AC 都有可执行验证方式。
- [x] 前端需求已准备需求→证据映射。

## Acceptance Criteria

### Phase A: 单文件应用与本地数据

- [x] AC-A1: 双击 `index.html` 后只出现“配置 / 用户信息 / 统计数据”三个一级页签，配置、用户 CRUD 和刷新后持久化流程可用；以 Chrome/Edge 截图和浏览器重开测试复核。
- [x] AC-A2: 用户 CSV 支持 BOM、引号、逗号和换行字段；导入前展示有效/无效行，确认后仅写入校验通过的数据；以自动化样例测试和截图复核。
- [x] AC-A3: 项目运行不依赖服务、Node、Docker、CDN 或第三方脚本；断开网络后页面静态资源仍完整加载，仅 API 查询不可用。
- [x] AC-A4: 配置页不提供自定义代理输入，只说明系统/浏览器代理，并把网络错误、401、403/限流区分展示。

### Phase B: 双平台统计

- [x] AC-B1: GitHub 与 GitCode 均只返回默认分支、所选半开时间区间内的 merged PR，并按 PR 作者账号映射用户；以两平台已知 PR 与原始 API 逐项核对。
- [x] AC-B2: 日期、平台、用户和仓库筛选会同时作用于汇总及明细；聚合恒满足 `变更行=新增+删除`、`净增=新增-删除`。
- [x] AC-B3: 分页、限流、网络失败、`too_large` 或不可解析字段不会静默变成 0；其他仓库继续执行，受影响结果明确标记“部分/失败”。
- [x] AC-B4: 同一查询中每个仓库只执行一次发现流程，再按作者分组；以请求计数测试复核无“用户 × 仓库”重复抓取。

### Phase C: 导出与交付验证

- [x] AC-C1: 汇总和明细 CSV 仅导出当前筛选结果，使用 UTF-8 BOM 与 RFC 4180 转义，不包含 Token，并对公式型文本做安全处理。
- [x] AC-C2: 在最新 Chrome 和 Edge 中从干净浏览器配置完成 Primary Journey；保存截图、测试日志及一个不含 Token 的示例 CSV。
- [x] AC-C3: README 说明打开方式、Token 权限建议、系统代理要求、统计口径、CSV 格式和已知限制。

## Dependencies

- **Evolved from**: 无（本项目首个 Feature）
- **Blocked by**: 无；浏览器 CORS、持久化与真实 API 已完成探针验证
- **Related**: [F002 PR 协作活动统计](F002-pr-collaboration-statistics.md) 在本能力上增加评论与审批归属、三类详情及严格按筛选导出

## Risk

| 风险 | 缓解 |
|------|------|
| GitCode 未把 CORS 明确写成稳定契约 | 启动连接测试给出明确诊断；README 记录未来可选本地代理边界，但首版不实现 |
| GitHub 搜索最多返回 1000 条 | 检测上限并提示缩小时间范围或拆分窗口；绝不静默截断 |
| GitCode 大 PR 文件数据不完整 | 检测 `too_large` 和字段完整性，标记“部分统计” |
| Token 存于 `localStorage` | 清楚提示仅适合可信本机；提供一键清除；Token 不进入任何导出 |
| API 限流与多仓库 N+1 请求 | 分页、并发上限、进度反馈；同一仓库只抓取一次 |
| `file://` 浏览器兼容差异 | 首版明确支持最新版 Chrome/Edge，并用真实浏览器验收 |

## Open Questions

无阻塞性产品问题。视觉细节和并发上限属于可逆实现决策，由开发者在不改变上述口径与三页结构的前提下自决。

## Key Decisions

| # | 决策 | 理由 | 日期 |
|---|------|------|------|
| KD-1 | 单个 `index.html`，不使用本地服务 | 浏览器 CORS 与持久化已实测可行，符合零安装目标 | 2026-07-24 |
| KD-2 | 使用 `localStorage`，不使用 IndexedDB | IndexedDB 在真实 `file://` Chrome 测试中超时 | 2026-07-24 |
| KD-3 | 统计口径固定为 merged + `merged_at` + PR 作者 + 默认分支 | operator 明确确认，且避免重复与归属歧义 | 2026-07-24 |
| KD-4 | 不做跨平台镜像去重 | 缺少可靠的跨平台 PR 共同标识，operator 已接受 | 2026-07-24 |
| KD-5 | GitCode 默认分支使用 `base` 参数并本地复核 | `target_branch` 实测被忽略；`base` 实测生效 | 2026-07-24 |

## Timeline

| 日期 | 事件 |
|------|------|
| 2026-07-24 | 需求讨论、CORS/存储/CSV 探针和真实 API 验证完成 |
| 2026-07-24 | F001 立项并交由 opus 开发 |
| 2026-07-24 | `6523cc0` 经 sol review APPROVE（65 passed），F001 标记 done |
| 2026-07-25 | F002 在 F001 上增加协作活动统计；`876bec3` 完成最终 UI、导出诊断、Token 保存与隐私说明收尾（137 passed） |

## Review Gate

- 开发者完成 quality-gate 后，由非作者猫做 fresh-context review。
- 浏览器验收必须覆盖 Primary Journey、两平台真实 API 对照和无 Token 残留检查。

## Links

| 类型 | 路径 | 说明 |
|------|------|------|
| **Thread** | `thread_mrylklzujj698s1f` | 原始需求、口径确认与实测记录 |
| **Maintenance** | `../../MAINTAINING.md` | 当前架构、数据契约、扩展落点、测试与发布流程 |
| **GitHub API** | `https://docs.github.com/en/rest/pulls/pulls` | PR 详情与代码量 |
| **GitCode API** | `https://docs.gitcode.com/en/docs/apis/get-api-v-5-repos-owner-repo-pulls/` | PR 列表与 `base` 参数 |

## Tips Contribution（F244）

`tips_exempt: 独立本地工具项目，不接入 Clowder AI 产品 tips 系统。`
