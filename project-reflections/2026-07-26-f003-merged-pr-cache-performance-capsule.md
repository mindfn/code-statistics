---
capsule_id: 2026-07-26-f003-merged-pr-cache-performance
feature_ids: [F003]
related_features: [F001, F002]
topics: [cache, concurrency, progress, user-remap, feature-lifecycle]
doc_kind: reflection
created: 2026-07-26
---

# F003 已合入 PR 缓存与查询加速反思胶囊

## Context

F001/F002 已能统计代码量与协作活动，但大范围重复查询会重新拉取每个
已合入 PR，进度在结果出现后仍可能停留在“正在获取”，用户映射变化也会
迫使用户重新查询。operator 明确要求把已合入 PR 当作稳定单元，并采用
GitHub 8 / GitCode 16 的并发预算与按 PR 持久缓存。

## What Worked

- 先把“整颗已合入 PR 永久缓存”的产品取舍写入 F003，再实现缓存，避免
  把罕见的 merge 后评论变化演化成 TTL、分层 freshness 与刷新入口。
- 把并发限制放在 Provider `_fetch` 请求层，统一约束 PR 内部的三路
  collaboration fan-out；测试既证明没有突破 8/16，也证明并行度超过 3。
- 缓存只保存 raw activity，用户归属保持纯投影，使用户 CRUD、启禁用与
  CSV 导入能够零网络即时重映射。
- 红绿处置 fresh-context findings 后，测试从 216 pass / 6 fail 收敛到
  222 pass / 0 fail；两个非作者 reviewer 对同一 SHA 独立 APPROVE。
- GitHub Pages 上的生产 journey 证明首查网络 1、复查缓存 1 / 网络 0，
  用户编辑后请求计数不变，控制台无错误。

## What Failed

- 初版 GitHub `networkPRs` 在日期/分支过滤前计数，使终态可能出现
  “缓存 + 网络 > 选中 PR 总数”；fresh-context review 才暴露。
- 初版用户 mutation 同时经 `renderUsers()` 与 remap 重建筛选器，产生
  重复 UI 工作。
- 首次尝试云端 Codex review 连续两次在协议窗口内未接单，最终按
  merge-gate 降级到跨 provider 的完整非作者 review。
- `file://` IndexedDB 在当前 Chrome 环境超时，不能承担跨重开持久化；
  设计收敛为 HTTP(S) IndexedDB、`file://` 页面内存。

## Trigger Missed

- 写 GitHub 计数器时没有立即用不通过日期/分支过滤的搜索结果构造 invariant
  测试；应在状态计数器落地时同步断言
  `cacheHits + networkPRs === selectedPRTotal`。
- 将仓库发现改为并行时，首轮测试只覆盖了请求 limiter，未同步覆盖多仓库
  并行与嵌套 fan-out 的端到端组合；fresh-context review 后补齐。
- feature close 前才发现当前独立项目没有 reflection 模板；模板应在项目
  第一次采用 feat lifecycle 时建立。

## Doc Links

- `docs/features/F003-merged-pr-cache-performance.md`
- `docs/discussions/2026-07-26-F003-cache-performance-design/README.md`
- `feature-specs/2026-07-26-merged-pr-cache-performance.md`
- `docs/quality-gates/2026-07-26-F003.md`
- `MAINTAINING.md`
- GitHub PR #1

## Rule Update Target

无需修改家规或全局 harness。项目内保留两条验证惯例：

1. 进度终态必须断言分项计数之和等于最终选中总数。
2. 请求预算测试必须同时覆盖多仓库并行与单 PR 内部 fan-out。
