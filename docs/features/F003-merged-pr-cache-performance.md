---
feature_ids: [F003]
related_features: [F001, F002]
topics: [github, gitcode, pull-request, cache, indexeddb, concurrency, progress, local-first]
doc_kind: spec
created: 2026-07-26
---

# F003: 已合入 PR 缓存与查询加速

> **Status**: in-progress | **Owner**: sol | **Priority**: P1

## Why

大时间范围查询会对每个已合入 PR 重复获取代码量、Comments 与 Approve；仓库多、PR 多时等待时间和平台 API 请求量都过高。用户维护账号映射后，当前结果也不会随之更新，只能重新发起整次网络查询。

> operator experience（2026-07-26）：
> “pr已经合入的情况下；没必要再考虑comments之类的变化了；这个基本上很少遇到；没必要为了这个少数场景浪费兼容”

目标是把已合入 PR 视为稳定统计单元：重复查询只做仓库级 PR 发现，命中的 PR 从本地缓存恢复全部活动；同时用真实请求级并发预算缩短首次查询，并让进度与用户映射在本地立即收敛。

## Current State / 现状基线

- `runQuery()` 按仓库串行执行发现；协作阶段只限制 3 个 PR 任务，但 GitHub 每个任务内部会同时请求 issue comments、review comments、reviews，实际并发可达名义值的 3 倍。
- `processedCollab` 与 `repoProgress` 在任务开始前递增；`Promise.all` 完成后没有写终态，因此结果已渲染时进度卡仍可能显示“正在获取”。
- 用户新增、编辑、启禁用、删除和 CSV 导入只更新 `localStorage` 与用户表，不重新映射 `StatsEngine.currentActivities`。
- F001 将统计结果限定为内存数据。2026-07-26 当前 Chrome 探针再次确认：`file://` 下 IndexedDB `open` 超过 5 秒仍未完成；不能把它当作双击运行的可靠持久层。
- `localStorage` 典型配额不足以可靠保存上万条带评论摘要与链接的 ActivityRecord，不采用压缩、分片与配额淘汰补丁来伪装成大容量缓存。

## What

### Phase A: 稳定 PR 缓存

缓存单位固定为：

```text
platform + repository + pr_number
```

终态条目：

```js
{
  schemaVersion,
  cachedAt,
  platform,
  repository,
  prNumber,
  core,              // PR 作者、merged_at、代码量、默认分支、链接
  activities,        // 未绑定本地用户的 comment / approval
  completeness,      // 只允许 full 条目成为 cache hit
  partialReason
}
```

- 已合入 PR 的代码量与协作活动整体按稳定数据处理，不设置时间 TTL，不追踪 merge 后评论/审批变化。
- schema 不匹配、字段损坏、`partial` 或 `failed` 条目一律视为 miss，并由本次查询重抓；不完整结果不得污染下一次统计。
- `http(s)`（GitHub Pages / 本地静态服务器）使用 IndexedDB 持久化；`file://` 使用相同接口的页面内存实现，关闭页面后失效。
- 缓存保存未绑定用户的原始活动；每次查询或用户资料变化后重新做本地用户投影，缓存不依赖 user_key。
- 仓库默认分支与 merged PR 列表仍从平台读取，保证日期范围与 PR 发现正确；命中后跳过该 PR 的详情、代码量和协作来源请求。

### Phase B: 共享请求预算与可信进度

- 所有 Provider `_fetch` 经过平台级共享请求限制器：GitHub 最多 8 个、GitCode 最多 16 个并发网络请求。
- 仓库发现可以并行；PR 详情与协作来源不再叠加各自的隐形 limiter，统一由请求预算承重。
- 进度只在任务 settle 后增加“完成”计数；分别展示缓存命中、网络获取、部分与失败。
- 查询完成后标题进入明确终态；每个仓库行显示已完成 PR 数、缓存命中数，以及 partial/failed 原因。
- 取消后不再把晚到的任务写成成功终态。

### Phase C: 用户变更本地重映射

用户新增、编辑、启禁用、删除与 CSV 确认导入后：

```text
Storage.users
  → mapActivitiesToUsers(currentActivities)
  → collectFilters
  → renderResults
```

该路径不调用 Provider、不读取缓存、不发送网络请求。没有当前结果时只刷新用户表与筛选项。

## User Journey

### Primary Journey: 首次查询后快速复用已合入 PR

- **Scope unit**: workspace
- **Actor**: 本地单用户
- **Entry**: 打开“统计数据”
- **Flow**:
  1. 用户首次查询一批仓库，页面在共享并发预算内获取 PR 与协作活动。
  2. 进度显示实际完成数、缓存命中数与部分原因，结束后明确显示“查询完成”。
  3. 用户再次查询重叠时间范围，已合入 PR 命中缓存，不再请求其详情、Comments 或 Approve。
  4. 用户编辑账号映射或启禁用用户，现有汇总和详情立即本地更新，无需再次查询。
- **Success evidence**: `test.html` 请求计数/并发/状态机断言；Chrome 首查与复查截图；Network 请求对照。
- **Non-goals**: 未合并 PR 缓存、merge 后评论刷新、时间 TTL、手工刷新按钮、跨浏览器缓存同步、`file://` 持久化、服务端缓存。

### Supporting Journeys

| ID | Scope unit | Actor | Flow | Evidence |
|----|------------|-------|------|----------|
| S1 | pull-request | 本地单用户 | 某 PR 协作来源部分失败 → 本次显示 partial → 下次仍重抓该 PR | partial cache test |
| S2 | workspace | 双击运行用户 | `file://` 查询 → 当前页面重复查询命中内存 → 关闭后不承诺保留 | Chrome file probe + README |
| S3 | user | 本地单用户 | 禁用已匹配用户 → 当前活动立即变为未匹配 → 启用后恢复 | DOM + zero-fetch test |

## 需求点 Checklist

| ID | 需求点（operator experience/转述） | AC 编号 | 验证方式 | 状态 |
|----|---------------------------|---------|----------|------|
| R1 | 已合入 PR 无需为少数 merge 后评论变化增加兼容 | AC-A1, AC-A2 | cache hit/miss/schema/partial tests | [x] |
| R2 | 按 PR 持久缓存，减少重复查询请求 | AC-A1, AC-A3 | IndexedDB reopen + request count tests | [x] |
| R3 | 采用 8/16 并行但不能放大成 24/48 个请求 | AC-B1 | instrumented max concurrency tests | [x] |
| R4 | 查询完成后进度必须真实结束并展示原因 | AC-B2, AC-B3 | progress state DOM tests | [x] |
| R5 | 用户资料变化后无需重拉远端 | AC-C1 | fetch=0 + remap DOM tests | [x] |

### 覆盖检查

- [x] 每个需求点都能映射到至少一个 AC。
- [x] 每个 AC 都有验证方式。
- [x] 前端需求已准备需求→证据映射表。

## Acceptance Criteria

### Phase A: 稳定 PR 缓存

- [x] AC-A1: 完整、同 schema 的 PR 条目在重复查询中命中，且该 PR 的详情、代码量、Comments、Reviews/Approve 请求数为 0；仓库级 PR 发现仍执行。
- [x] AC-A2: schema 不匹配、损坏、partial 或 failed 条目不得命中；重抓后的完整条目原子替换旧值，且用户映射字段不写入缓存。
- [x] AC-A3: `http(s)` 下 IndexedDB 关闭并重新打开后仍能读取完整条目；`file://` 明确使用内存缓存且 README/界面不声称跨重开持久化。

### Phase B: 并发与进度

- [x] AC-B1: 并行夹具中 GitHub 同时进行的网络请求不超过 8，GitCode 不超过 16；每个平台都能实际达到大于 3 的并行度，内部嵌套请求不会突破预算。
- [x] AC-B2: PR 总完成数仅在 cache hit 或网络任务 settle 后递增；查询结束、取消、部分与失败都有唯一终态，结果渲染后不再显示“正在获取”。
- [x] AC-B3: 仓库进度终态显示 PR 总数、缓存命中数和网络获取数；partial/failed 原因在仓库行可见并保留到导出诊断。

### Phase C: 本地重映射与回归

- [x] AC-C1: 用户新增、编辑、启禁用、删除和 CSV 导入都会对当前活动重新映射并重渲染，期间 Provider/fetch 调用数为 0；无当前活动时安全 no-op。
- [x] AC-C2: F001/F002 的 Provider、筛选、三类详情、CSV、安全与完整性测试全部保持通过；新增 cache/concurrency/progress/remap 覆盖后总测试数只增不减。
- [x] AC-C3: 最新 Chrome 在本地静态服务器完成首次查询、重复查询与用户重映射 journey，控制台无新增错误。

## Release Acceptance Gate

F003 只有在 `main` 发布完成、GitHub Pages 以最终提交运行同一 journey 且控制台无新增错误后才可标记 `done`。部署验收属于 merge 后的发布门禁，不作为 pre-review 实现 AC；验收结果必须回填本文件 Timeline 与 `MAINTAINING.md` 发布记录。

## Dependencies

- **Evolved from**: [F001 PR 代码变更统计](F001-pr-code-statistics.md)、[F002 PR 协作活动统计](F002-pr-collaboration-statistics.md)
- **Blocked by**: 无
- **Related**: [F003 设计讨论](../discussions/2026-07-26-F003-cache-performance-design/README.md)
- **Architecture cell**: `standalone-single-file-app`（`MAINTAINING.md` §2）
- **Map delta**: none — 只在既有 Storage / Provider / StatsEngine / Query Controller 边界中增加 CacheStore 与共享请求预算

## Risk

| 风险 | 缓解 |
|------|------|
| 缓存 schema 演进导致旧结果误读 | schemaVersion 硬门禁；不匹配即 miss |
| partial 被缓存后长期少算 | 只有完整条目可命中；不完整条目下次重抓 |
| 用户映射污染跨用户缓存 | 缓存只存 actor_login 等原始活动；映射永远是内存投影 |
| 并发增加触发平台限流 | 所有实际 `_fetch` 共享平台预算；现有限流诊断保持 |
| `file://` IndexedDB 不可用 | 明确采用内存坐标，不叠 localStorage 容量补丁 |

## Open Questions

无阻塞性产品问题。缓存不考虑 merge 后评论变化是 operator 已确认的价值取舍；具体 IndexedDB 事务与进度 helper 属可逆实现细节。

## Key Decisions

| # | 决策 | 理由 | 日期 |
|---|------|------|------|
| KD-1 | 已合入 PR 的 core + collaboration 整体永久缓存 | operator 明确不为极少的 merge 后变化增加复杂度 | 2026-07-26 |
| KD-2 | 仅 full + 当前 schema 条目可命中 | 防止网络失败或字段缺失被永久冻结 | 2026-07-26 |
| KD-3 | `http(s)` IndexedDB，`file://` 页面内存 | 当前 Chrome 复测 `file://` IDB 仍超时，localStorage 容量不足 | 2026-07-26 |
| KD-4 | GitHub 8 / GitCode 16 是请求级预算 | PR 任务级 limiter 无法约束内部 3 路请求 | 2026-07-26 |
| KD-5 | 用户归属是 raw activity 的纯投影 | 用户变化无需网络，也不会使缓存失效 | 2026-07-26 |

## Timeline

| 日期 | 事件 |
|------|------|
| 2026-07-26 | 根因诊断、`file://` IndexedDB 复测、缓存新鲜度取舍确认与 F003 立项 |
| 2026-07-26 | 实现完成：212/212 浏览器测试通过；本地 Chrome 首查、缓存复查与用户重映射 journey 通过 |

## Review Gate

- 保持三个一级页签与现有 Apple 视觉，不新增刷新入口。
- 生产行为变更完成 quality-gate 后，由非作者 fresh-context review。
- 浏览器验收覆盖 HTTP 持久缓存、file 内存边界、进度终态和用户本地重映射。

## Links

| 类型 | 路径 | 说明 |
|------|------|------|
| **Discussion** | `docs/discussions/2026-07-26-F003-cache-performance-design/README.md` | Design Gate 与数据流 |
| **Plan** | `feature-specs/2026-07-26-merged-pr-cache-performance.md` | TDD 实施计划 |
| **Maintenance** | `MAINTAINING.md` | 运行边界与发布流程 |

## Tips Contribution（F244）

`tips_exempt: 独立本地工具项目，不接入 Clowder AI 产品 tips 系统。`
