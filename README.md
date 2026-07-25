# PR 代码变更统计

统计用户在 GitHub / GitCode 上指定时间范围内已合并 PR 的代码变更量、Comments 和 Approve，支持按 CSV 导出。

维护和二次开发请从 [MAINTAINING.md](MAINTAINING.md) 开始；其中记录了架构、数据契约、扩展落点、测试门禁和发布流程。

## 使用方式

双击 `index.html` 用浏览器打开即可（推荐最新版 Chrome 或 Edge）。

无需安装、无需服务端、无需外部依赖。

## 快速开始

1. **配置** — 填入 GitHub / GitCode 的 Token，点击“测试并保存”；验证成功后 Token 会自动保存到当前浏览器。在“仓库管理”中添加需要统计的仓库。
2. **用户** — 添加需要统计的用户及其平台账号（手工新增或 CSV 导入），启用/禁用通过开关控制。
3. **统计数据** — 选择时间范围和用户范围（已匹配/全部/仅未匹配），点击"查询"；汇总表展示开发 PR 数、Comments 数、Approve 数及代码量；点击汇总行可在"开发 PR / Comments / 审核 PR"三个标签间切换查看详情，可导出 CSV。

## Token 权限

| 平台 | 最小权限 |
|------|----------|
| GitHub (Classic PAT) | `repo`（私有仓库）或 `public_repo`（仅公开仓库） |
| GitHub (Fine-grained PAT) | 目标仓库的 **Pull requests** 只读 |
| GitCode | 私人令牌，需要仓库读取权限 |

Token 仅保存在浏览器 `localStorage`，不会进入导出的 CSV、日志或 URL 参数中；请求平台数据时只通过请求头直接发送给 GitHub / GitCode API。

## 数据存储与安全

**所有配置和用户数据均保存在浏览器的 `localStorage` 中**，不会上传到本工具自建的服务器。查询时，Token 会通过认证请求头直接发送给 GitHub / GitCode API。

- **无自建后端上传路径** — GitHub Pages 只托管静态文件，不接收 Token、用户信息、仓库配置或统计结果。仍应只在可信设备和可信浏览器环境中使用。
- **清除浏览器数据会同时清除已导入的配置** — 如果你清除了浏览器的站点数据（localStorage），之前导入或手动添加的用户和仓库配置将一同被清除。当前版本没有一键导出用户配置，建议保留原始导入 CSV；“下载模板”只包含表头和示例行，不是备份。
- **多设备不同步** — 由于数据存储在本地浏览器，不同设备或不同浏览器之间的数据不会自动同步。可在另一设备重新导入保留的用户 CSV；Token 和仓库列表需要重新配置。

## 网络与代理

本工具沿用操作系统或浏览器的代理设置。请确保浏览器可以正常访问：

- `https://api.github.com`
- `https://api.gitcode.com`

如果公司网络需要代理才能访问 GitHub / GitCode，请先配置系统代理或浏览器代理。

## 统计口径

- **只统计已合并 PR**，以 `merged_at`（合并时间）落入所选时间范围为准。
- **用户归属**按 PR 作者、评论作者、审核用户的平台账号匹配，邮箱仅作人员资料，不参与归属判断。
- **只统计合入仓库默认分支**的 PR，避免 feature 分支 PR 和最终合并 PR 重复计算。
- **不统计**直接 push、未合并 PR、他人 PR 中的个人 commit。
- GitHub 与 GitCode 镜像仓库**不自动去重**；如两边同步，建议只启用一个统计源。
- 三类活动都随 PR 的合并时间归入同一查询周期：
  - **开发 PR**：PR 作者，每个已合并 PR 计 1，代码量与文件数只归作者。
  - **Comments**：评论作者。GitHub 包含普通对话评论、行内 review comments、非空 review 总结；GitCode 包含 PR comments。按 `source_type + id` 去重。
  - **Approve**：审核用户。GitHub 取同一用户在该 PR 上最后一个决定性 review 状态，`APPROVED` 计 1；GitCode 取 PR 详情中 `approval_reviewers + assignees` 且 `accept === true` 的用户，去重后计 1。
- 指标：
  - 变更行 = 新增行 + 删除行
  - 净增行 = 新增行 - 删除行
- 产品文案统一称"PR 代码变更量"，不称"产能"或"有效代码量"。
- GitCode Approve 仅表示查询时的"当前有效 Approve"，不代表完整审批历史。

## CSV 格式

### 用户导入 CSV

```csv
user_key,display_name,email,github_login,gitcode_login
```

- `user_key`：唯一标识（必填）
- `display_name`：显示名（可选）
- `email`：邮箱（可选，仅作人员资料）
- `github_login` / `gitcode_login`：平台账号（至少填一个）
- 仓库在"配置"页签统一管理，不再绑定到用户

可从页面下载模板。

### 导出 CSV

- **汇总 CSV**：按用户、仓库聚合的协作活动统计，包含开发 PR 数、Comments 数、Approve 数及代码量；只导出当前筛选和用户范围下的可见行。
- **明细 CSV**：每个规范化活动记录（开发 PR / Comment / Approve），包含活动类型、来源、活动时间、链接等；与当前筛选条件一致。
- 编码：UTF-8 with BOM，兼容 Excel 直接打开。
- 格式：RFC 4180，对公式型文本做安全处理（防止 Excel 公式注入）。
- 不包含 Token 或其他敏感信息。

## 已知限制

| 限制 | 说明 |
|------|------|
| GitHub 搜索上限 | 单次搜索最多返回 1000 个 PR。超出时页面标记"部分统计"，建议缩小时间范围 |
| 请求量增长 | F002 后每个入选 PR 额外请求 Comments / Reviews，较 F001 更容易触发限流；工具已实现分页、有限并发和错误传播 |
| GitCode 当前有效 Approve | GitCode 公开 API 没有完整审批历史，仅按 PR 详情中当前 `accept=true` 统计，不表示历史审批动作次数 |
| GitCode 大文件 PR | 文件过大时 GitCode 返回 `too_large`，该 PR 标记"部分统计"，不会按 0 计算 |
| 浏览器兼容 | 推荐最新版 Chrome / Edge。Safari 和 Firefox 在 `file://` 下的 localStorage 行为可能不同 |
| 无应用内代理 | 浏览器 `fetch` 不支持为单次请求指定代理，只能沿用系统/浏览器代理 |
| IndexedDB | 在 `file://` 协议下不可用，因此使用 localStorage（有 ~5MB 容量限制，通常足够） |
| Token 安全 | 仅适合可信本机使用。建议用完后及时清除 Token |
| 配置备份 | 当前没有一键导出用户和仓库配置；请保留原始用户 CSV，并在迁移设备前手工记录仓库列表 |
| 统计结果不持久化 | 查询结果仅保存在页面内存中，刷新后需重新查询。CSV 是统计结果的持久交付物 |

## 技术说明

- 单个 `index.html`，所有 HTML / CSS / JavaScript 内联，无 CDN、无第三方依赖
- `localStorage` 持久化配置和用户数据（Key: `code-statistics.config.v1` / `code-statistics.users.v1`）
- GitHub API: Search Issues + Pull Request Detail + Issue Comments + Review Comments + Reviews；代码包含分页、节流、限流等待和部分结果标记
- GitCode API v5: Pull Request List + Detail/Files + Comments；代码包含分页、有限并发、限流报错和字段完整性检查

## 维护者入口

- [维护与迭代指南](MAINTAINING.md)：模块地图、Provider 契约、统计不变量、常见扩展方法、安全边界和发布流程。
- `test.html`：直接加载 `index.html` 生产代码的浏览器测试；本地通过 `python3 -m http.server 8901 --bind 127.0.0.1` 运行。
- `.github/workflows/deploy.yml`：`main` 更新后的 GitHub Pages 自动部署配置。
