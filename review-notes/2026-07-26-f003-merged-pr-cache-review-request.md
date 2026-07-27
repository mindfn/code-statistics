---
feature_ids: [F003]
related_features: [F001, F002]
topics: [review-request, cache, indexeddb, concurrency, progress, local-remap]
doc_kind: review
created: 2026-07-26
---

# Review Request: F003 已合入 PR 缓存与查询加速

Review-Target-ID: `f003`

Branch: `feat/f003-merged-pr-cache-performance`

## What

- 按 `platform + repository + PR number` 持久缓存完整已合入 PR 的 core 与 raw collaboration activities。
- `http(s)` 使用 IndexedDB；`file://` 使用页面内存，不叠加 localStorage 大对象方案。
- GitHub/GitCode 所有实际 `_fetch` 使用共享请求级预算 8/16，仓库发现并行。
- 查询进度按 settle 计数并进入明确终态，展示缓存/网络/部分/失败原因。
- 用户新增、编辑、启禁用、删除和 CSV 导入后，本地重映射当前活动，零网络请求。

## Why

大时间范围查询会重复拉取每个已合入 PR 的详情、Comments 和 Approve；原进度在任务开始时就计数，结果渲染后仍可能显示“正在获取”。用户资料变化也只能靠整次重查才能更新当前结果。

## Original Requirements

> “批量拉取后之后拉取详情的时候可以考虑用8个或者16个并行拉取。”
>
> “将拉取后的数据按照pr的纬度保存；因为pr已经合入了不会再修改。”
>
> “pr已经合入的情况下；没必要再考虑comments之类的变化了……没必要为了这个少数场景浪费兼容。”

- 来源：`docs/discussions/2026-07-26-F003-cache-performance-design/README.md`
- 请对照摘录判断交付物是否解决了 operator 的实际等待与重复拉取问题。

## Tradeoff

- 已合入 PR 整体按稳定单元永久缓存，不设置 TTL、不提供手工刷新、不兼容合入后极少见的评论/审批变化。
- 只允许当前 schema、结构完整、`completeness === "full"` 的条目命中；命中率让位于统计正确性。
- `file://` 不承诺跨重开持久化；选择一个 IndexedDB 持久坐标加一个 Memory 降级坐标，避免容量与 fallback 层堆叠。

## Architecture Ownership

Architecture cell: `standalone-single-file-app`（`MAINTAINING.md` §2）

Map delta: `none`

Why: CacheStore 与请求预算是既有 Storage / Provider / StatsEngine / Query Controller 边界内的组件；没有改变 owner、单文件应用边界、extension point 或 canonical anchor。

请 reviewer 重点确认新增 `CacheStore` 仍属于该 cell 的内部增量，没有形成需要独立 ownership cell 的并行持久层。

## Invariant Matrix

| 不变量 | 断言描述 | 验证方式 |
|---|---|---|
| INV-1 | cache hit 同时满足 schema、identity、branch/time 结构与 full 门禁 | cache lifecycle / missing branch / identity tests |
| INV-2 | 缓存不含 Token 与 `user_key/display_name/matched` 投影字段 | sanitize assertions + diff review |
| INV-3 | GitHub/GitCode 实际 fetch 峰值不超过 8/16 | direct budget + nested GitHub fan-out instrumentation |
| INV-4 | 进度 completed 只在任务 settle 后递增且不超过 total | progress state machine tests |
| INV-5 | query finalize 后无“正在获取”，partial/failed 原因可见 | terminal DOM tests + dogfood |
| INV-6 | 用户变更只做 raw activities × users 投影，fetch=0 | remap DOM + mutation wiring + request spy |
| INV-7 | partial/failed/损坏缓存不会成为完整命中 | rejection + corruption + core diagnostic tests |
| INV-8 | `cacheHits + networkPRs === selected PR total` | GitHub filtered network count + GitCode integration |

## E2E User Path Evidence

- Worktree: `/Users/lang/workspace/github/code-statistics-f003-merged-pr-cache`
- URL: `http://127.0.0.1:8902/index.html`
- First query: 1 PR（缓存 0 · 网络 1）；PR detail 与三类协作来源各请求 1 次。
- Repeat query: 1 PR（缓存 1 · 网络 0）；仓库发现继续执行，PR detail 与三类协作来源计数不增加。
- User edit: Alice → Alice Renamed；当前汇总立即更新，网络计数不变。
- Evidence:
  - `/tmp/cat-cafe-evidence/code-statistics-f003/01-first-query.png`
  - `/tmp/cat-cafe-evidence/code-statistics-f003/02-cache-hit.png`
  - `/tmp/cat-cafe-evidence/code-statistics-f003/03-local-remap.png`
  - `/tmp/cat-cafe-evidence/code-statistics-f003/04-dogfood-15s.mp4`

## Open Questions

### 技术 OQ

1. CacheStore 的结构/identity/full 门禁是否足以保证永久缓存不会冻结损坏或 partial 数据？
2. ProviderRequestBudget 是否覆盖分页、重试和 collaboration 内部 fan-out，且没有预算旁路？
3. 取消语义是否清晰：UI 不写成功；已经完整获取并通过 full 门禁的稳定 PR 允许写本地缓存。
4. `Map delta: none` 与新增 CacheStore 是否语义一致？

请逐条验证 Invariant Matrix。

### 价值 OQ

无。永久缓存、不追踪合入后评论变化的价值取舍已由 operator 明确确认。

## Fresh-Context Findings

Agent: `[宪宪/claude-opus-4-6🐾]`

SHA scanned: `2923d67`

Total findings: 10（0 high，0 medium，10 P3）

| Finding | Author 处置 | 状态 |
|---|---|---|
| FC-1 GitHub 网络计数包含被过滤 PR | fixed (`8766a99`) | ✅ |
| FC-2 空 baseBranch 缓存可通过结构门禁 | fixed (`8766a99`) | ✅ |
| FC-3 abort/cache write 时间窗 | clarified：UI 取消与稳定 full cache 分离，并加测试 (`8766a99`) | ✅ |
| FC-4 用户变更重复刷新筛选项 | fixed (`8766a99`) | ✅ |
| FC-5 Map delta 与 CacheStore | dismissed：单文件 cell 的内部组件，边界/owner 未变化 | ✅ |
| FC-6 网络计数缺测试 | fixed (`8766a99`) | ✅ |
| FC-7 仓库并行缺测试 | fixed (`8766a99`) | ✅ |
| FC-8 嵌套 fan-out 缺预算测试 | fixed (`8766a99`) | ✅ |
| FC-9 `_cacheHit` 不二次验证 | dismissed：private provenance 只由已验证的 CacheStore clone 设置 | ✅ |
| FC-10 cache entry factory 硬编码 full | fixed (`8766a99`) | ✅ |

Reviewer delta tracking：请在正式 finding 中标注 `[FC:covered]`、`[FC:new]` 或 `[FC:N/A]`。

## Next Action

请在最终 review SHA 上独立复跑验证并给出 `APPROVE` 或 `REQUEST-CHANGES` verdict；若有 finding，请给出文件、行号、严重度和 invariant 影响。

## Review Sandbox

- Path: `/tmp/cat-cafe-review/f003/kimi`
- State: detached HEAD / read-only review use
- Start Command: `python3 -m http.server 8903 --bind 127.0.0.1`
- Ports: `web=8903`, `api=N/A`
- Test URL: `http://127.0.0.1:8903/test.html`

## 自检证据

### Spec 合规

- Quality Gate: `docs/quality-gates/2026-07-26-F003.md`
- 全部 9 个 AC 已有 commit/test/browser 证据。
- Release Acceptance Gate 保持独立硬门禁：最终 main SHA 必须在 GitHub Pages 完成同一 journey，F003 才能标记 done。

### 测试结果

```bash
sed -n '/<script>/,/<\/script>/p' index.html | sed '1d;$d' | node --check -
# exit 0

sed -n '/<script>/,/<\/script>/p' test.html | sed '1d;$d' | node --check -
# exit 0

git diff --check
# exit 0

python3 -m http.server 8903 --bind 127.0.0.1
# open test.html → 222 passed, 0 failed
```

Fresh Chrome application load: HTTP 200，0 console/page errors。

Root media/design artifact gate: worktree 与 `origin/main...HEAD` 均无命中。

### 相关文档

- Feature: `docs/features/F003-merged-pr-cache-performance.md`
- Discussion: `docs/discussions/2026-07-26-F003-cache-performance-design/README.md`
- Plan: `feature-specs/2026-07-26-merged-pr-cache-performance.md`
- Maintenance: `MAINTAINING.md`

[砚砚/gpt-5.6-sol🐾]
