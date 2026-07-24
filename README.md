# PR 代码变更统计

统计用户在 GitHub / GitCode 上指定时间范围内的已合并 PR 代码变更量，支持按 CSV 导出。

## 使用方式

双击 `index.html` 用浏览器打开即可（推荐最新版 Chrome 或 Edge）。

无需安装、无需服务端、无需外部依赖。

## 快速开始

1. **配置** — 填入 GitHub / GitCode 的 Token，点击"测试连接"确认可用。
2. **用户信息** — 添加需要统计的用户及其关联仓库（手工新增或 CSV 导入）。
3. **统计数据** — 选择时间范围，点击"查询"，等待完成后查看汇总和 PR 明细，可导出 CSV。

## Token 权限

| 平台 | 最小权限 |
|------|----------|
| GitHub (Classic PAT) | `repo`（私有仓库）或 `public_repo`（仅公开仓库） |
| GitHub (Fine-grained PAT) | 目标仓库的 **Pull requests** 只读 |
| GitCode | 私人令牌，需要仓库读取权限 |

Token 仅保存在浏览器 `localStorage`，不会进入导出的 CSV、日志或任何网络请求参数中（仅通过请求头传递）。

## 网络与代理

本工具沿用操作系统或浏览器的代理设置。请确保浏览器可以正常访问：

- `https://api.github.com`
- `https://api.gitcode.com`

如果公司网络需要代理才能访问 GitHub / GitCode，请先配置系统代理或浏览器代理。

## 统计口径

- **只统计已合并 PR**，以 `merged_at`（合并时间）落入所选时间范围为准。
- **用户归属**按 PR 作者的平台账号匹配，邮箱仅作人员资料，不参与归属判断。
- **只统计合入仓库默认分支**的 PR，避免 feature 分支 PR 和最终合并 PR 重复计算。
- **不统计**直接 push、未合并 PR、他人 PR 中的个人 commit。
- GitHub 与 GitCode 镜像仓库**不自动去重**；如两边同步，建议只启用一个统计源。
- 指标：
  - 变更行 = 新增行 + 删除行
  - 净增行 = 新增行 - 删除行
- 产品文案统一称"PR 代码变更量"，不称"产能"或"有效代码量"。

## CSV 格式

### 用户导入 CSV

```csv
user_key,display_name,email,github_login,gitcode_login,repository_url,enabled
```

- `user_key`：唯一标识（必填）
- `repository_url`：GitHub 或 GitCode 仓库 URL（必填）
- `enabled`：`true` / `false`，默认 `true`
- 一行代表一个"用户-仓库"关系，同一用户可关联多个仓库

可从页面下载模板。

### 导出 CSV

- **汇总 CSV**：按用户、仓库聚合的代码变更量统计。
- **明细 CSV**：每个 PR 的详细信息（编号、作者、合并时间、增删行、链接等）。
- 编码：UTF-8 with BOM，兼容 Excel 直接打开。
- 格式：RFC 4180，对公式型文本做安全处理（防止 Excel 公式注入）。
- 不包含 Token 或其他敏感信息。

## 已知限制

| 限制 | 说明 |
|------|------|
| GitHub 搜索上限 | 单次搜索最多返回 1000 个 PR。超出时页面标记"部分统计"，建议缩小时间范围 |
| GitCode 大文件 PR | 文件过大时 GitCode 返回 `too_large`，该 PR 标记"部分统计"，不会按 0 计算 |
| 浏览器兼容 | 推荐最新版 Chrome / Edge。Safari 和 Firefox 在 `file://` 下的 localStorage 行为可能不同 |
| 无应用内代理 | 浏览器 `fetch` 不支持为单次请求指定代理，只能沿用系统/浏览器代理 |
| IndexedDB | 在 `file://` 协议下不可用，因此使用 localStorage（有 ~5MB 容量限制，通常足够） |
| Token 安全 | 仅适合可信本机使用。建议用完后及时清除 Token |
| 统计结果不持久化 | 查询结果仅保存在页面内存中，刷新后需重新查询。CSV 是统计结果的持久交付物 |

## 技术说明

- 单个 `index.html`，所有 HTML / CSS / JavaScript 内联，无 CDN、无第三方依赖
- `localStorage` 持久化配置和用户数据（Key: `code-statistics.config.v1` / `code-statistics.users.v1`）
- GitHub API: Search Issues + Pull Request Detail（认证搜索限额 30 次/分钟）
- GitCode API v5: Pull Request List + Detail/Files（限额 400 次/分钟）
