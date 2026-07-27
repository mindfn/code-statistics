---
feature_ids: [F002]
related_features: [F001]
topics: [github, gitcode, pull-request, comments, approvals, statistics, csv, local-first]
doc_kind: spec
created: 2026-07-25
---

# F002: GitHub / GitCode PR 协作活动统计

> **Status**: done | **Owner**: kimi | **Priority**: P1 | **Reviewed by**: sol (APPROVE @ `b4b57a5`)

## Why

> operator experience（2026-07-25）：
> “PR1 是用户 A 提交的，用户 B 补充了 comments，用户 C 进行了 approve；PR1 在 t1 时刻合入，那么选择的时间包含 t1、目标用户包含 A/B/C 时，应该看到 A 提交了一个 PR、B 补充了 n 个 comments、C 有一个 approve。”

F001 只能衡量 PR 作者的代码变更，不能展示评论与审核贡献；同时当前汇总导出绕过“用户范围”筛选，选择“所有匹配的用户”时仍会导出未匹配作者。F002 把统计单位从“PR 作者代码量”提升为“围绕已合并 PR 的协作活动”，并保证界面结果与导出结果严格一致。

## Current State / 现状基线

- 当前公开版本：`3fc6571`，单文件 Apple 深空灰界面，三个一级页签保持不变。
- 当前汇总渲染先按 `stat-user-type` 得到 `visibleSummary`，但导出仍读取筛选前的 `StatsEngine.summaryRows`，因此默认“所有匹配的用户”会把未匹配作者写入 CSV。
- 当前 `StatsEngine` 只维护 PR 作者记录；评论者或审核者若没有自己提交 PR，不会形成汇总行。
- 当前点击汇总行只显示一个 PR 明细表，没有开发、评论、审核三个详情视图。
- F001 的 Token、仓库、用户均保存在浏览器 `localStorage`；统计结果只在当前页面内存中，F002 不改变这一边界。

## What

### 1. 固定统计口径

所选时间范围仍以 PR 的 `merged_at` 为唯一归属时间：

- 只有 PR 的合并时间落入 `[开始日期, 结束日期次日)`，该 PR 的开发、评论与审核活动才进入本次统计。
- 评论或审核本身发生在合并前还是合并后，不改变该活动随 PR 归入哪个查询区间；详情仍展示实际活动时间。
- 只统计合入仓库默认分支的 PR，不统计未合并 PR、直接 push 或 commit 级贡献。

三类活动归属：

| 活动 | 归属用户 | 计数口径 |
|------|----------|----------|
| 开发 PR | PR 作者 | 每个已合并 PR 计 1；代码量与文件数只归作者 |
| Comments | 评论作者 | 每条唯一评论计 1 |
| Approve | 审核用户 | 同一 PR、同一用户最多计 1 个当前有效 approve |

#### Comments 定义

- GitHub：
  1. PR 普通对话评论（Issue comments）；
  2. 行内 review comments；
  3. review 中非空的总结正文。
- GitCode：PR comments 接口返回的评论。
- 统一使用 `platform + source_type + source_id` 去重。
- GitHub review 同时包含非空正文且状态为 `APPROVED` 时，可同时贡献 1 个 comment 和 1 个 approve；这是两类不同活动，不是重复计数。

#### Approve 定义

- GitHub：按接口返回的时间顺序，以同一用户在该 PR 上最后一个决定性 review 状态为准；`APPROVED` 有效，后续 `CHANGES_REQUESTED` 或 `DISMISSED` 会使 approve 失效，`COMMENTED` 不覆盖之前的决定性状态。
- GitCode：对 PR 详情中的 `approval_reviewers` 与 `assignees` 合并去重，只统计 `accept === true` 的用户；不统计 `testers`。
- GitCode 公开 API 没有可验证的完整审批历史，故界面和 CSV 明确称为“当前有效 Approve”，不是历史审批动作次数。

### 2. 汇总表

每一行的粒度固定为：

```text
平台 + 已配置用户 + 仓库
```

列顺序：

```text
平台 / 用户 / 显示名 / 仓库 /
开发 PR / Comments / Approve /
新增 / 删除 / 变更 / 净增 / 文件 / 完整性
```

- 只有评论或审核、没有开发 PR 的用户也必须出现，代码量列为 0。
- “所有匹配的用户”只显示已配置且启用的匹配用户；未匹配账号不出现。
- “全部（含未匹配）”追加未匹配账号；“仅未匹配”只显示未匹配账号。
- 用户、平台、仓库和用户范围筛选同时作用于页面汇总、详情和两种 CSV。

### 3. 三类详情

点击汇总行，沿用现有表格下方的内联详情区，不新增模态框或第四个一级页签：

```text
[开发 PR] [Comments] [审核 PR]
```

- 开发 PR：PR 编号、标题、合并时间、新增、删除、文件数、完整性、链接。
- Comments：PR 编号、评论类型、活动时间、内容摘要、评论链接、完整性。
- 审核 PR：PR 编号、审核时间、当前状态、审核链接、完整性。
- 标签展示该行对应数量；切换标签不重新请求 API。
- 再次点击当前汇总行收起详情；点击其他行切换详情对象。

### 4. 查询与数据流

查询阶段对时间范围内的 PR 一次性抓取协作活动，详情使用本次查询的内存缓存：

```text
仓库 → 默认分支 + merged PR
     → 每个入选 PR：代码量 + comments + reviews/approvers
     → 规范化 ActivityRecord
     → 用户匹配与聚合
     → 当前可见行
     → 页面 / CSV（同一数据源）
```

不采用“打开详情时才请求”的方案，因为汇总中的 Comments/Approve 数量本身已经需要这些数据；延迟请求不会减少请求量，只会造成汇总与详情口径不一致。

规范化记录：

```text
ActivityRecord {
  activity_type: development | comment | approval,
  platform, repository, pr_number, pr_title, pr_url, merged_at,
  actor_login, user_key, matched,
  event_at, source_type, source_id, activity_url,
  additions, deletions, changed_files,
  completeness, partial_reason
}
```

同一查询中每个 PR 的每类来源只请求一次。Provider 实现分页、有限并发和阶段进度展示；任何评论/审核来源失败都必须传播为“部分/失败”，不能静默当成 0。

### 5. CSV

- “导出汇总 CSV”只使用当前页面可见的汇总行，列与汇总表一致。
- “导出明细 CSV”导出当前筛选后的规范化活动记录，增加 `activity_type`、活动时间、活动链接及类型相关字段。
- 默认“所有匹配的用户”下，未匹配作者、评论者和审核者均不得进入任一 CSV。
- 保留 UTF-8 BOM、RFC 4180 转义、公式注入防护和 Token 永不导出的约束。
- 受当前平台/仓库查询影响的 partial/failed 诊断行仍可导出；不得因为用户范围过滤而把真实不完整性隐藏。

## User Journey

### Primary Journey: 查看某时间段内围绕已合并 PR 的协作贡献

- **Scope unit**: workspace
- **Actor**: 本地单用户
- **Entry**: 打开“统计数据”
- **Flow**:
  1. 用户选择时间、平台、用户范围、用户和仓库，点击“查询”。
  2. 页面按仓库抓取已合并 PR，再抓取相关评论与审核；进度明确区分 PR、评论、审核阶段。
  3. A 提交了 PR，B 留下 n 条评论，C 当前有效 approve；汇总分别显示 A 的开发 PR、B 的 Comments、C 的 Approve。
  4. 用户点击任一行，在开发 PR / Comments / 审核 PR 三个标签间切换查看该用户在该仓库的活动。
  5. 用户导出汇总或明细 CSV，CSV 与当前筛选和用户范围完全一致。
- **Success evidence**: A/B/C 固定夹具自动测试、真实平台抽样核对、最新 Chrome 页面截图及 CSV 内容对照。
- **Non-goals**: 按评论/审核发生时间独立筛选、未合并 PR、commit 统计、跨平台镜像去重、GitCode 完整审批历史、云端存储或多用户权限。

### Supporting Journeys

| ID | Scope unit | Actor | Flow | Evidence |
|----|------------|-------|------|----------|
| S1 | workspace | 本地单用户 | 选“所有匹配的用户” → 查询 → 导出 → 未匹配活动不在 CSV | 浏览器下载内容断言 |
| S2 | repository | 本地单用户 | 某 PR 评论接口失败 → 代码量仍可显示，但该仓库明确标记部分 | Provider mock 与截图 |
| S3 | user-repository | 本地单用户 | 用户只评论/approve、没有开发 PR → 仍出现汇总行 → 点击对应详情 | A/B/C 夹具测试 |

## Text Wireframe / UI Design Gate

保持现有 Apple 视觉、筛选栏、汇总表和内联展开行为：

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ 统计数据                                                                     │
│ [开始] [结束] [平台] [用户范围] [用户] [仓库] [查询]                        │
│ 查询进度：PR 12/12 · Comments 10/12 · Reviews 8/12                         │
│                                              [导出汇总] [导出明细]           │
├──────┬──────┬────────┬──────────┬──────┬──────────┬─────────┬──────┬──────┤
│ 平台 │ 用户 │ 显示名 │ 仓库     │开发PR│ Comments │ Approve │ 新增 │ …    │
├──────┼──────┼────────┼──────────┼──────┼──────────┼─────────┼──────┼──────┤
│ GH   │ A    │ Alice  │ org/repo │  1   │    0     │    0    │ 320  │ 完整 │ ←
│ GH   │ B    │ Bob    │ org/repo │  0   │    3     │    0    │  0   │ 完整 │
│ GH   │ C    │ Carol  │ org/repo │  0   │    0     │    1    │  0   │ 完整 │
└──────┴──────┴────────┴──────────┴──────┴──────────┴─────────┴──────┴──────┘

点击 B 行后：
┌──────────────────────────────────────────────────────────────────────────────┐
│ Bob · org/repo          [开发 PR 0] [Comments 3] [审核 PR 0]                │
├──────────────────────────────────────────────────────────────────────────────┤
│ PR #1 · 普通评论 · 2026-07-20 10:31 · “建议这里补一个边界测试…” · 查看 ↗  │
│ PR #1 · 行内评论 · 2026-07-20 10:35 · “这个变量可能为空…”       · 查看 ↗  │
│ PR #1 · Review 总结 · 2026-07-20 10:40 · “整体可以，再改两处…”   · 查看 ↗  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Design-in-context Checklist

| 检查项 | 现状证据 | F002 决策 |
|--------|----------|-----------|
| 视觉语言 | `index.html` @ `3fc6571`：Apple 深空灰、系统蓝、克制圆角 | 原样延续，不进行第三次视觉重构 |
| 页面结构 | 三个一级页签；统计页已有筛选栏、汇总表、内联详情 | 不新增页签；只扩列和详情内部标签 |
| 交互模式 | 点击汇总行展开/收起 PR 详情 | 保留，详情升级为 3 tabs |
| 响应式 | 当前表格在窄屏横向滚动 | 新增列后继续横向滚动，首列不做 sticky，避免遮挡 |
| 空态 | 当前无结果时显示提示文案 | 三个详情标签分别显示零活动空态 |
| 部分失败 | 当前有完整/部分/失败徽标 | 协作来源失败沿用同一徽标并补原因 |
| 可访问性 | 原生 button/select/table；键盘可聚焦 | tabs 使用 button + `aria-selected`，汇总行展开同时提供键盘按钮 |

**Design Gate 状态**：operator 于 2026-07-25 确认，交由 kimi 实现。

## 需求点 Checklist

| ID | 需求点 | AC 编号 | 验证方式 | 状态 |
|----|--------|---------|----------|------|
| R1 | 汇总导出按当前筛选，不导出未匹配用户 | AC-A1, AC-C1 | 筛选矩阵与下载内容断言 | [x] |
| R2 | 统计 Comments 和 Approve | AC-A2, AC-B1, AC-B2 | A/B/C 固定夹具、真实 API 对照 | [x] |
| R3 | 汇总表展示仓库、开发 PR、Comments、Approve、代码量 | AC-B3 | DOM 表头与聚合断言、截图 | [x] |
| R4 | 点击行查看三类详情 tabs | AC-B4 | 浏览器交互测试、截图 | [x] |
| R5 | 时间选择包含 PR 合并时间时，将该 PR 的三类活动归入结果 | AC-A2 | 边界时间夹具 | [x] |
| R6 | 明细与汇总都遵循用户/平台/仓库筛选 | AC-C1 | 组合筛选测试 | [x] |

### 覆盖检查

- [x] 每个需求点都映射到至少一个 AC。
- [x] 每个 AC 都有可执行验证方式。
- [x] 前端需求已准备需求→证据映射与 Design Gate 线框。

## Acceptance Criteria

### Phase A: 活动模型与筛选一致性

- [x] AC-A1: 页面渲染、汇总 CSV、明细 CSV 共同从“应用完日期/平台/仓库/用户/用户范围后的可见数据集”派生；默认已匹配范围中任何未匹配活动不得导出。以 matched/all/unmatched × user × repo 筛选矩阵自动测试复核。
- [x] AC-A2: 同一已合并 PR 的作者、评论者、审核者分别归属 development/comment/approval；只评论或只审核的已配置用户也生成汇总行。以 A/B/C 夹具和合并时间边界夹具复核。
- [x] AC-A3: 现有用户、Token、仓库 `localStorage` 结构兼容；不清空或迁移丢失既有配置。

### Phase B: 双平台 Provider 与 UI

- [x] AC-B1: GitHub 完整分页读取普通 PR 对话、行内评论和 reviews，按 `source_type + id` 去重；同一用户/PR 的 approve 仅在最后决定性 review 为 `APPROVED` 时计 1。以 Provider mock 行为测试和真实 PR 抽样复核。
- [x] AC-B2: GitCode 完整分页读取 PR comments；对 `approval_reviewers + assignees` 去重并统计 `accept === true`，文案明确为当前有效 Approve。以 Provider mock 与真实响应 schema 抽样复核。
- [x] AC-B3: 汇总行粒度为平台+用户+仓库，显示开发 PR、Comments、Approve 及原代码量指标；代码量只属于 PR 作者。以 DOM、聚合恒等式和 A/B/C 截图复核。
- [x] AC-B4: 点击汇总行显示开发 PR / Comments / 审核 PR 三个可键盘操作的标签；数量、空态、链接和内容均对应当前行，切换不发起新请求。
- [x] AC-B5: 每个入选 PR 的每类来源只请求一次；分页、限流、字段缺失或请求失败传播到仓库与活动完整性，不得静默按 0 计。以请求计数与错误注入测试复核。

### Phase C: 导出与交付

- [x] AC-C1: 汇总 CSV 与明细 CSV 都只包含当前筛选结果；明细以 `activity_type` 区分三类记录，保持 UTF-8 BOM、RFC 4180、公式注入防护和 Token 排除。以浏览器下载内容断言和 Numbers/Excel 抽查复核。
- [x] AC-C2: `test.html` 增加筛选导出回归、A/B/C 活动归属、评论去重、approve 状态折叠、GitCode `accept` 去重、部分失败传播、三标签交互测试，全部直接测试生产代码。
- [x] AC-C3: 最新 Chrome 中完成全流程浏览器验收；README 更新 Comments/Approve 口径、GitCode 当前状态限制、API 请求量与本地存储说明。

## Dependencies

- **Evolved from**: [F001 PR 代码变更统计](F001-pr-code-statistics.md)
- **Blocked by**: 无；UI Design Gate 已确认
- **Architecture cell**: 复用 `index.html` 现有 Provider / StatsEngine / CSV 边界；所有改动限定在单文件应用与生产代码测试，无 ownership map 变化

## Risk

| 风险 | 缓解 |
|------|------|
| 每个 PR 新增多类请求，较 F001 更容易触发限流 | 只处理已进入时间范围的 PR；每来源一次、分页、有限并发、阶段进度；403 明确提示 |
| GitHub review body 与行内评论可能被重复理解 | 三类来源使用不同 `source_type + id`；review body 与 review comments 本就是不同内容单元 |
| GitCode 没有完整审批历史接口 | 只称“当前有效 Approve”，不声称是历史次数 |
| 评论/审核接口失败导致看起来为 0 | 完整性状态与原因全链路传播，受影响行标记部分，导出保留诊断 |
| 汇总表列数增多 | 保持桌面表格横向滚动；不挤压为难读的多行卡片 |

## Source Audit / API 可行性证据

| Claim | 来源 | 结论 | 适用性 |
|-------|------|------|--------|
| GitHub PR 普通对话来自 Issue comments，PR 本身也是 issue | GitHub 官方 REST 文档 | 可用 | `GET /issues/{number}/comments`；Pull requests read 即可 |
| GitHub 行内评论来自 Pull request review comments | GitHub 官方 REST 文档 | 可用 | Pull requests read |
| GitHub reviews 按时间顺序返回并包含 `state/body/submitted_at/user` | GitHub 官方 REST 文档 | 可用 | 支持当前有效 approve 折叠 |
| GitCode PR comments 可列举评论及用户、时间、类型 | GitCode 官方 REST 文档 + 2026-07-25 只读真实响应抽样 | 可用 | 实现需分页并防字段缺失 |
| GitCode PR 详情含 `approval_reviewers/assignees[].accept` | GitCode 官方 REST 文档 + 2026-07-25 只读真实响应抽样 | 有条件使用 | 仅代表查询时当前状态，不代表完整历史 |

一手来源：

- <https://docs.github.com/en/rest/issues/comments>
- <https://docs.github.com/en/rest/pulls/comments>
- <https://docs.github.com/en/rest/pulls/reviews>
- <https://docs.gitcode.com/docs/apis/get-api-v-5-repos-owner-repo-pulls-number-comments/>
- <https://docs.gitcode.com/en/docs/apis/get-api-v-5-repos-owner-repo-pulls-number/>

## Open Questions

无待实现者自行猜测的产品口径；Text Wireframe / UI Design Gate 已由 operator 确认。

## Key Decisions

| # | 决策 | 理由 | 日期 |
|---|------|------|------|
| KD-1 | 三类活动都随 PR 的 `merged_at` 归入查询时间 | 与 operator A/B/C 示例一致，避免同一 PR 的协作被拆到不同周期 | 2026-07-25 |
| KD-2 | Comments 包含普通对话、行内评论、非空 review 总结 | 覆盖 PR 上实际可见的文字反馈，同时可用来源 ID 去重 | 2026-07-25 |
| KD-3 | Approve 按用户/PR 最多 1 个当前有效状态 | “审核了几个 PR”比“点了几次按钮”更稳定且可解释 | 2026-07-25 |
| KD-4 | 查询时一次性抓取活动，详情使用内存缓存 | 汇总已经依赖活动数据，延迟详情请求不会节省 API 调用 | 2026-07-25 |
| KD-5 | 页面与导出只共享一个可见数据投影 | 从根因上消除“页面已过滤、CSV 未过滤”分叉 | 2026-07-25 |

## Timeline

| 日期 | 事件 |
|------|------|
| 2026-07-25 | operator 提出筛选导出修复、Comments/Approve 与三类详情需求 |
| 2026-07-25 | GitHub 官方 API 核验、GitCode 官方文档与真实响应 schema 抽样完成 |
| 2026-07-25 | F002 spec 与 UI Design Gate 建立 |
| 2026-07-25 | operator 确认 Design Gate，指定 kimi 开发 |
| 2026-07-25 | F002 实现与回归修复完成，`b4b57a5` 经 sol review APPROVE（122 passed）并推送 `main` |
| 2026-07-25 | GitHub Pages 部署及线上页面验收通过，`94b316c` 将 BACKLOG 状态同步为 done |
| 2026-07-25 | `876bec3` 完成最终 UI、导出诊断、Token 保存与隐私说明收尾，经 kimi review APPROVE（137 passed） |

## Review Gate

- UI Design Gate 已确认；实现必须保持既定 Apple 视觉、三个一级页签与内联详情结构。
- 开发者完成 quality-gate 后，由非作者猫做 fresh-context review。
- 浏览器验收必须覆盖 A/B/C 用户示例、三个详情标签、筛选后两种 CSV 和部分失败展示。

## Links

| 类型 | 路径 | 说明 |
|------|------|------|
| **Thread** | `thread_mrylklzujj698s1f` | 原始需求、F001 实现与 F002 需求 |
| **Discussion** | `../discussions/2026-07-25-F002-design/README.md` | 需求推导与 UI Design Gate |
| **Maintenance** | `../../MAINTAINING.md` | 当前架构、数据契约、扩展落点、测试与发布流程 |

## Tips Contribution（F244）

`tips_exempt: 独立本地工具项目，不接入 Clowder AI 产品 tips 系统。`
