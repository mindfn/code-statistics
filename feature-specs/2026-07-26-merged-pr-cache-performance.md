# 已合入 PR 缓存与查询加速 Implementation Plan

**Feature:** F003 — `docs/features/F003-merged-pr-cache-performance.md`

**Goal:** 重复查询已合入 PR 时复用完整本地缓存，用真实请求级 8/16 并发预算加速首次查询，并让进度与用户归属无需重发网络即可收敛。

**Acceptance Criteria:** AC-A1~A3、AC-B1~B3、AC-C1~C3（逐条见 F003 spec）

**Architecture cell:** `standalone-single-file-app`（`MAINTAINING.md` §2）

**Map delta:** none

**Map delta why:** 仅在单文件内新增 CacheStore / RequestBudget，并复用既有 Provider、StatsEngine、Query Controller。

**Architecture:** merged PR 列表仍由 Provider 发现；每个 PR 先按稳定 key 读取 CacheStore，只有 miss 才抓详情与协作来源。缓存保存 raw activity，用户归属永远在内存中重新投影；所有真实网络请求统一经过平台请求预算。

**Tech Stack:** 原生 HTML/CSS/JavaScript、IndexedDB、Memory Map、现有 `test.html` 浏览器测试
**前端验证:** Yes — Chrome / Playwright 实测进度终态、缓存命中和用户重映射

---

## Finish Line

用户在 HTTP/Pages 上首次查询后，重叠查询只为仓库发现发请求，命中 PR 不再请求详情或协作来源；GitHub/GitCode 实际并发分别不超过 8/16；进度结束后进入唯一终态；用户资料变化只做本地重映射。

不构建：TTL、merge 后评论刷新、手工刷新按钮、localStorage 大对象缓存、`file://` 跨重开持久化、服务端或第三方依赖。

## 终态数据结构

```js
var CACHE_SCHEMA_VERSION = 1;

// Cache key
function prCacheKey(platform, repository, prNumber) {
  return platform + ':' + repository.toLowerCase() + '#' + prNumber;
}

// Persisted value; user_key / matched / display_name are forbidden.
{
  schemaVersion: 1,
  cachedAt: '2026-07-26T00:00:00.000Z',
  platform: 'github',
  repository: 'owner/repo',
  prNumber: 42,
  core: { /* normalized PR */ },
  activities: [ /* raw comments + approvals */ ],
  completeness: 'full',
  partialReason: ''
}
```

## 真相源矩阵

| 数据 | 真相源（写） | 消费方（读） | 派生关系 | 级联规则 |
|------|-------------|-------------|---------|---------|
| Token / repositories | `Storage.CONFIG_KEY` | Query Controller / Provider | 直接读取 | 配置变化只影响后续查询 |
| users | `Storage.USERS_KEY` | `mapActivitiesToUsers` / filters | raw activity 的本地投影 | CRUD/toggle/import → 立即 remap + render |
| merged PR 列表 | Provider 远端发现 | Query Controller | 日期与默认分支过滤结果 | 每次查询重新发现 |
| PR core + raw collaboration | CacheStore 或 Provider | Query Controller | cache hit 或 miss 抓取 | 仅 full 写入；schema/partial/failed → miss |
| 当前活动 | `StatsEngine.currentActivities` | aggregate / details / CSV | cache/provider raw activity + users 投影 | 查询或 users 变化后重建映射 |
| 进度 | 当前 query 的内存 state | progress card | task settle 事件投影 | hit/success/partial/failed/cancelled 单向终结 |

## 生命周期对象普查

### 1. PR Cache Entry

**唯一 owner:** `CacheStore`

| 当前状态 | 事件 | 下一状态 | 行为 |
|---------|------|---------|------|
| absent | get | miss | 远端抓取 |
| absent | put(full) | valid | 原子写入 |
| absent | put(partial/failed) | absent | 拒绝写入 |
| valid | get(schema match) | valid | 返回深拷贝 hit |
| valid | get(schema mismatch/corrupt) | stale | 删除并返回 miss |
| stale | refetch full | valid | 新值替换 |
| any | file reload | absent | Memory 后端按契约失效 |
| valid | HTTP reload | valid | IndexedDB 后端恢复 |

旁路禁止：Provider、StatsEngine 不直接访问 IndexedDB；用户映射字段不得进入 entry；不允许 partial/failed 通过 generic put。

### 2. Request Budget

**唯一 owner:** `ProviderRequestBudget`

| 当前状态 | 事件 | 下一状态 | 行为 |
|---------|------|---------|------|
| idle/active<N | schedule | active+1 | 立即执行 |
| active=N | schedule | queued | FIFO 等待 |
| active | resolve/reject | active-1 | 唤醒下一项 |
| queued | query abort | queued→settled | fn 读取 AbortSignal 后拒绝，不占新业务状态 |

旁路禁止：Provider 内不得另起高于平台预算的网络调用；分页、重试也必须经 `_fetch` 预算入口。

### 3. Query Progress

**唯一 owner:** `runQuery` 内的 progress state/helper

| 当前状态 | 事件 | 下一状态 | 行为 |
|---------|------|---------|------|
| pending | cache hit | done | completed+1, cacheHits+1 |
| pending | network start | running | 不增加 completed |
| running | full success | done | completed+1, network+1 |
| running | partial/error | partial | completed+1, network+1, 保存原因 |
| pending/running | abort | cancelled | 禁止后续写 success |
| all terminal | finalize | query terminal | 标题显示完成/取消/部分/失败 |

### 4. User Attribution Projection

**唯一 owner:** `StatsEngine.mapActivitiesToUsers`

不单独持久化状态机；它是 `raw activities × users` 的纯投影。任何用户变化都对 `currentActivities` 原地重算后渲染，不维护第二份“已映射缓存”。

## 核心不变量

- **INV-1:** cache hit 必须同时满足 schema match、结构有效、`completeness === "full"`。
- **INV-2:** 缓存 entry 不含 Token 与 `user_key/matched/display_name` 用户投影字段。
- **INV-3:** 同一平台所有真实 `fetch` 的峰值并发不超过 GitHub 8 / GitCode 16。
- **INV-4:** `completed` 只在 hit 或网络任务 settle 后增加，永远不超过 total。
- **INV-5:** query finalize 后 UI 不含“正在获取”；每个仓库都处于 done/partial/failed/cancelled。
- **INV-6:** 用户 CRUD/toggle/import 后 Provider/fetch 调用数为 0，当前活动归属与 users 真相源一致。
- **INV-7:** partial/failed 不会因缓存而在下一次查询被当成完整结果。
- **INV-8:** F001/F002 页面、详情与 CSV 继续消费同一个可见活动投影。

## 对抗场景

| 场景 | 期望 | 测试 |
|------|------|------|
| IndexedDB 写入中页面关闭 | 旧完整值或无值，不出现半条 JSON | transaction completion + reopen |
| 两个相同 PR 同时请求 | 单 query 的 PR 发现去重；最多一次业务 fetch | request count fixture |
| schema 升级 | 旧条目 miss 并替换 | stale schema test |
| partial 网络结果 | 当前显示 partial；下次再次抓取 | partial rejection test |
| 用户在统计结果存在时被禁用 | 活动变未匹配且 0 fetch | remap DOM test |
| 取消时队列仍有任务 | 不显示完成，不写晚到缓存 | abort progress/cache test |

## 既有正确行为保护点

| 现有功能 | 当前正确行为 | 保护方式 |
|---------|------------|---------|
| merged PR 统计口径 | 默认分支 + merged_at 半开区间 | 既有 Provider tests + 全量回归 |
| A/B/C 活动归属 | development/comment/approval 分属不同用户 | 既有固定夹具 |
| partial 传播 | 来源失败不静默变 0 | 既有错误注入 + 新 partial cache test |
| 可见投影与 CSV | matched/all/unmatched 筛选一致 | 既有筛选/导出 tests |
| Token 安全 | 不进 URL、CSV、日志 | 既有 DOM/CSV tests + diff review |
| 三页与详情 tabs | UI 结构和键盘行为不变 | 既有 DOM tests + Chrome smoke |

### Task 1: CacheStore 契约与红测试

**Files:**
- Modify: `test.html`
- Modify: `index.html`（仅在绿阶段）

1. 在 `test.html` 添加 `prCacheKey`、entry validation、Memory backend hit/miss/schema/partial/deep-copy 测试。
2. 添加 HTTP IndexedDB backend 的 put → close → reopen → get 测试；测试库使用独立名称并在结束时删除。
3. 运行全量测试，确认新增断言因 `CacheStore` 不存在而红。
4. 在 `index.html` 的 Storage 后新增 `CacheStore`，实现 `createMemoryBackend/openIndexedDb/get/put/delete`。
5. 只接受 full + schema entry；读写均 clone，任何异常返回 miss 而不影响查询。
6. 运行测试转绿并提交：`feat(F003): add merged PR cache store`。

### Task 2: Provider 请求级 8/16 预算

**Files:**
- Modify: `test.html`
- Modify: `index.html`

1. 添加并行 mock：多 PR 内部同时发 3/2 路请求，记录 `_fetch` active/maxActive。
2. 断言 GitHub `maxActive <= 8 && maxActive > 3`、GitCode `maxActive <= 16 && maxActive > 3`；旧代码应红。
3. 新增 `ProviderRequestBudget`，并让两个 Provider `_fetch` 的每次实际 `fetch`（含 retry）经过对应预算。
4. 删除 detail/collaboration 上重复承重的内部 limiter；分页 delay 保留在业务层。
5. 跑并发夹具与全量测试转绿，提交：`perf(F003): enforce provider request budgets`。

### Task 3: 查询编排接入 PR cache

**Files:**
- Modify: `test.html`
- Modify: `index.html`

1. 用 fake CacheStore + Provider 计数写红测试：第一次 miss 写 full；第二次 hit 时 PR detail/collaboration 为 0；partial 下次仍 fetch。
2. 将 Provider PR 发现拆为“列表/搜索候选 → 对每个 PR 尝试 cache → miss 获取 core”。
3. Query Controller 对 cache hit 直接恢复 core + raw collaboration；miss 获取完整数据后再 `put`。
4. 仓库与 PR key 去重，防止同 query 双写同一 PR。
5. 跑 cache request-count 与全量测试转绿，提交：`perf(F003): reuse cached merged PR activities`。

### Task 4: 进度状态机与终态

**Files:**
- Modify: `test.html`
- Modify: `index.html`

1. 添加 deferred Promise 红测试，证明任务启动时 completed 仍为 0、settle 后才递增。
2. 添加 DOM 红测试：full/partial/failed/cancelled finalize 后标题无“正在”，仓库行包含 total/cache/network/reason。
3. 提取纯 `createQueryProgress` 与 render helper；所有 hit/network settle 通过同一入口。
4. `finally` 恢复按钮状态；abort 后 ignore 晚到 success。
5. 跑进度测试与全量测试转绿，提交：`fix(F003): make query progress terminal and accurate`。

### Task 5: 用户变化本地重映射

**Files:**
- Modify: `test.html`
- Modify: `index.html`

1. 添加 `fetch` 计数为 0 的红测试，依次覆盖 save/edit、toggle、delete、confirmImport。
2. 新增 `remapCurrentResultsAfterUsersChanged(users)`：无活动 no-op；有活动则 remap、刷新 filters、`renderResults(collectFilters())`。
3. 所有用户 mutation 成功保存后调用同一 helper；不复制 render 逻辑。
4. 验证禁用/启用、账号改名、删除后 DOM 立即变化。
5. 跑测试转绿，提交：`fix(F003): remap current results after user changes`。

### Task 6: 文档、质量门禁与浏览器验收

**Files:**
- Modify: `README.md`
- Modify: `MAINTAINING.md`
- Modify: `docs/features/F003-merged-pr-cache-performance.md`

1. README 说明 Pages 持久缓存、`file://` 页面内存缓存、已合入 PR 稳定口径与缓存清理方式。
2. MAINTAINING 增加 CacheStore schema、请求预算、进度状态机、迁移规则与测试方法。
3. 运行：
   - `node --check` 两段生产/测试脚本
   - Chrome `test.html` 全量测试
   - `git diff --check`
4. Chrome 验收首次/重复查询、进度终态、用户 remap；保存无 Token 截图与请求计数证据。
5. 更新 F003 AC 与需求 checklist；进入 `quality-gate`。
