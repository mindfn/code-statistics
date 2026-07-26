---
feature_ids: [F002]
related_features: [F001]
topics: [product-design, pull-request, comments, approvals, csv, ui]
doc_kind: discussion
created: 2026-07-25
---

# F002 需求推导与 UI Design Gate

## Operator Experience

> “导出汇总数据时应该按照筛选条件来导出；当前全部都导出了。”
>
> “PR1 是用户 A 提交的，用户 B 补充了 comments，用户 C 进行了 approve……应该看到 A 用户提交了一个 PR、B 用户补充了 n 个 comments、C 用户有一个 approve。”
>
> “表格应该是仓库/开发 PR 数量/comments 数量/approve 数量/新增代码这些；点击对应行可以看到详情，详情有 3 个 tab：当前开发 PR/comments/审核 PR。”

## Problem Decomposition

这不是给现有表格加两列，而是统计模型从单主体扩展为三主体：

```text
PR 作者 ── development ──┐
评论者  ── comment ──────┼─→ 同一个 merged PR
审核者  ── approval ─────┘
```

当前导出 bug 与新需求共享同一根因：页面、聚合和 CSV 没有建立唯一的“当前可见活动投影”。F002 先规范化三类 ActivityRecord，再从同一可见投影派生汇总、详情与 CSV。

## Design Direction

- 保持 `3fc6571` Apple 深空灰视觉，不再次更换主题。
- 保持统计 / 用户 / 配置三个一级页签。
- 保持汇总行点击后在表格下方内联展开，不使用模态框或侧栏。
- 内联详情升级为开发 PR / Comments / 审核 PR 三个标签。
- 汇总新增开发 PR、Comments、Approve 三个协作指标，原代码量指标继续保留。

## Wireframe

```text
筛选栏
[时间] [平台] [用户范围] [用户] [仓库] [查询]       [汇总 CSV] [明细 CSV]

汇总
平台 | 用户 | 显示名 | 仓库 | 开发 PR | Comments | Approve | 新增 | 删除 | … | 完整性
GH   | A    | Alice  | repo |    1    |    0     |    0    | 320  |  40  | … | 完整
GH   | B    | Bob    | repo |    0    |    3     |    0    |   0  |   0  | … | 完整
GH   | C    | Carol  | repo |    0    |    0     |    1    |   0  |   0  | … | 完整

点击 B：
Bob · repo      [开发 PR 0] [Comments 3] [审核 PR 0]
-------------------------------------------------------
#1 | 行内评论 | 时间 | 摘要 | 查看
#1 | 普通评论 | 时间 | 摘要 | 查看
#1 | Review总结 | 时间 | 摘要 | 查看
```

## Product Semantics

1. 时间以 PR `merged_at` 为准，评论和审核跟随 PR 归入同一周期。
2. Comments 包含 GitHub 普通对话、行内评论、非空 review 总结；GitCode 使用 PR comments。
3. Approve 是“当前有效审核过的 PR 数”：同一用户、同一 PR 最多 1。
4. GitCode 只展示当前 `accept=true`，不冒充完整审批历史。
5. 查询时抓取协作数据，详情只读取内存缓存，切换标签不额外发请求。
6. 页面看到什么，汇总和明细 CSV 就导出什么；默认匹配范围不含任何未匹配活动。

## Design-in-context Evidence

| 现有元素 | 保留 | 改动 |
|----------|------|------|
| Apple 深空灰 CSS | 是 | 无主题改动 |
| 顶部三个页签 | 是 | 无新增一级页面 |
| 统计筛选条 | 是 | 筛选语义扩展到三类活动 |
| 汇总表 | 是 | 新增协作列 |
| 点击行内联详情 | 是 | 从单表升级为三 tabs |
| CSV 两按钮 | 是 | 汇总导出可见行；明细导出规范化活动 |

## API Feasibility

- GitHub：官方接口完整覆盖 issue comments、review comments、reviews；`Pull requests: read` 即可。
- GitCode：官方接口与真实只读抽样覆盖 comments；PR 详情覆盖当前 `accept` 审核状态，但没有已验证的审批历史读取接口。
- 请求量会增长，因此实现必须限定为入选 PR、每来源一次、分页、有限并发和可见进度。

## Design Gate

- **Status**: approved by operator on 2026-07-25
- **Gate**: 确认上述时间口径、三类活动定义、汇总列与内联三标签布局
- **Implementation owner**: kimi
- **Next**: kimi 实现；非作者猫完成 fresh-context review
