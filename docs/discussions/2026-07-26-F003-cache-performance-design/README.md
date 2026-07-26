---
feature_ids: [F003]
related_features: [F001, F002]
topics: [product-design, cache, concurrency, progress, indexeddb]
doc_kind: discussion
created: 2026-07-26
---

# F003 缓存与查询性能 Design Gate

## Operator Experience

> “pr已经合入的情况下；没必要再考虑comments之类的变化了；这个基本上很少遇到；没必要为了这个少数场景浪费兼容”

此前已授权对大查询做适当优化，并提出提高并行度与按 PR 持久缓存。本轮进一步确认：缓存无需拆分“稳定代码量”和“可能变化的协作快照”，整颗已合入 PR 作为稳定单元即可。

## 现场证据

| 现状 | 代码证据 | 影响 |
|------|---------|------|
| 仓库串行 | `runQuery()` 对 `repoStatuses` 使用 `for + await` | 多仓库无法共享等待时间 |
| 名义并发失真 | `collabLimiter(3)` 包裹 PR，但 GitHub PR 内部 `Promise.all(3 sources)` | 实际最多 9 个请求；改成 8/16 后可能放大为 24/48 |
| 进度提前完成 | `processedCollab++` 位于 `fetchCollaboration()` 前 | 显示的是“已开始”而非“已完成” |
| 进度无终态 | `Promise.all(collabPromises)` 后直接聚合渲染 | 结果出现后仍显示“正在获取” |
| 用户变化不重映射 | CRUD / toggle / import 只保存 users | 必须重新查询远端才能看到归属变化 |
| `file://` IDB 不可靠 | 2026-07-26 Chrome 探针 5 秒超时 | 不能承诺双击运行下持久化 |

## 设计收敛

```text
仓库级发现（始终走远端）
  → PR key
    → CacheStore.get
      ├─ full + schema match → core + raw activities
      └─ miss / partial / stale → Provider 请求 → full 才写 CacheStore
  → mapActivitiesToUsers
  → 页面 / 详情 / CSV
```

请求调度：

```text
GitHubProvider._fetch ─┐
                      ├─ githubRequestBudget(max=8)
GitHub pagination ─────┘

GitCodeProvider._fetch ─┐
                       ├─ gitcodeRequestBudget(max=16)
GitCode pagination ─────┘
```

缓存坐标：

| 页面来源 | CacheStore 后端 | 跨重开 |
|---------|-----------------|--------|
| `https:` / `http:` | IndexedDB | 是 |
| `file:` | Memory Map | 否 |

## UI Design in Context

- 不新增页签、模态框、筛选器或刷新按钮。
- 保留现有 progress card，只修正文案与状态：

```text
查询完成 · 120 个 PR（缓存 104 · 网络 16）· 2 个仓库
✅ github: org/repo — 80/80 PR（缓存 72 · 网络 8）
⚠️ gitcode: org/repo — 40/40 PR（缓存 32 · 网络 8）· 部分：Comments 获取失败
```

- 用户 CRUD 继续留在“用户信息”，变化后统计页已有结果自动重映射。
- `file://` 与 Pages 的缓存差异放在现有本地数据说明和 README，不制造新设置项。

## 元审美自检

这是坐标变换：把“PR 任务并发”改为“真实网络请求预算”，把“用户绑定的结果”改为“raw activity + 用户纯投影”。它消除了嵌套并发放大与缓存失效分支；没有增加 TTL、强制刷新、压缩 localStorage 或双层 core/collaboration 缓存。

## Architecture

```text
Architecture cell: standalone-single-file-app（MAINTAINING.md §2）
Map delta: none
Why: CacheStore 与 request budget 是既有 Storage / Provider / Query Controller 的内部增量，不改变单文件应用边界。
```

## Design Gate

- **Status**: approved from operator direction on 2026-07-26
- **Product decision**: 已合入 PR 整体稳定；不兼容少数 merge 后评论/审批变化
- **Surface decision**: 维持现有 UI 结构，只修进度终态与说明
- **Implementation owner**: sol
- **Next**: worktree + TDD；非作者 fresh-context review
