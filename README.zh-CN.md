# 代码变更统计

> 分析 GitHub 和 GitCode 仓库中已合并 PR 的代码变更量 — 零安装，完全在浏览器中运行。

**[English](README.md)**

**在线使用：** https://mindfn.github.io/code-statistics/

## 功能特性

- **双平台支持** — GitHub 和 GitCode，通过官方 API 查询
- **基于 PR 的指标** — 已合并 PR 的新增行、删除行、变更文件、评论和审批
- **零安装** — 单个 `index.html` 文件，无需服务端/构建工具/外部依赖
- **隐私优先** — 所有数据保存在浏览器本地（`localStorage` + `IndexedDB`）；Token 不会上传
- **CSV 导出** — 汇总和明细导出，UTF-8 BOM 编码，RFC 4180 格式
- **智能缓存** — 已合入 PR 数据缓存在 IndexedDB，重复查询跳过已获取的 PR
- **分支选择** — 每个仓库可配置目标分支，支持输入过滤的下拉选择
- **完整性追踪** — 部分/失败的 PR 数据明确标记，不会静默按 0 计算

## 快速开始

1. 打开 https://mindfn.github.io/code-statistics/（或下载 `index.html` 双击打开）
2. **配置** — 填入 GitHub/GitCode Token，添加仓库，可选择目标分支
3. **用户信息** — 添加用户及其平台账号（手工新增或 CSV 导入）
4. **统计数据** — 选择时间范围，点击"查询"，然后导出 CSV

## Token 权限

| 平台 | 最小权限 |
|------|----------|
| GitHub (Classic PAT) | `repo`（私有仓库）或 `public_repo`（仅公开仓库） |
| GitHub (Fine-grained PAT) | 目标仓库的 **Pull requests** 只读 |
| GitCode | 私人令牌，需要仓库读取权限 |

Token 仅保存在浏览器 `localStorage`，请求时只通过请求头直接发送给 GitHub / GitCode API，不会进入 CSV、日志或 URL。

## 数据存储与安全

- **无自建后端** — GitHub Pages 只托管静态文件，不接收任何用户数据
- **PR 缓存在本机** — `http(s)` 下使用 IndexedDB（`code-statistics.pr-cache.v1`）；`file://` 下仅使用页面内存
- **一键清除缓存** — 配置页"数据管理"可单独清除 PR 缓存，不影响配置和用户信息
- **多设备不同步** — 数据存储在本地浏览器，不同设备间需重新配置

## 统计口径

- 只统计**已合并 PR**，以 `merged_at` 落入所选时间范围为准
- 用户归属按 PR 作者 / 评论作者 / 审核用户的**平台账号**匹配
- 只统计合入**仓库目标分支**（默认为仓库默认分支）的 PR
- 不统计直接 push、未合并 PR、他人 PR 中的个人 commit
- GitHub 与 GitCode 镜像仓库**不自动去重**
- 三类活动都随 PR 的合并时间归入同一查询周期：
  - **开发 PR** — PR 作者，代码量只归作者
  - **Comments** — 评论作者，按 `source_type + id` 去重
  - **Approve** — 审核用户（GitHub 取最后决定性状态，GitCode 取当前有效状态）
- 指标：变更行 = 新增行 + 删除行，净增行 = 新增行 − 删除行

## CSV 格式

**用户导入 CSV**

```csv
user_key,display_name,email,github_login,gitcode_login
```

**导出 CSV** — 汇总 CSV 和明细 CSV，UTF-8 BOM，RFC 4180，防公式注入，不含 Token。

## 已知限制

| 限制 | 说明 |
|------|------|
| GitHub 搜索上限 | 单次搜索最多 1000 个 PR，超出标记"部分统计" |
| GitCode 列表优化 | 使用 `since=` 按更新时间过滤（缓冲 14 天），最终以 `merged_at` 精确过滤 |
| GitCode Approve | 仅表示查询时的当前有效 Approve，不代表完整审批历史 |
| GitCode 大文件 PR | `too_large` 表示 diff 截断；行数可解析则不影响完整性 |
| 浏览器兼容 | 推荐最新版 Chrome / Edge |
| 无应用内代理 | 沿用系统/浏览器代理 |
| `file://` 缓存 | `file://` 下 IndexedDB 不可用，仅页面内存缓存 |

## 技术说明

- 单个 `index.html`，所有 HTML / CSS / JavaScript 内联，无 CDN、无第三方依赖
- `localStorage` 持久化配置（`code-statistics.config.v1`）和用户（`code-statistics.users.v1`）
- 仓库配置支持 `{url, branch}` 格式，旧字符串格式自动迁移
- IndexedDB 按 `platform + repository + PR number` 持久化完整已合入 PR
- GitHub API: PR 搜索 + PR 详情 + PR 评论 + Review Comments + Reviews
- GitCode API v5: PR List + Detail/Files + Comments

## 贡献

详见 [CONTRIBUTING.md](CONTRIBUTING.md)，包含开发环境搭建、测试方法和提交规范。

## 许可证

[MIT](LICENSE)

## 维护者入口

- [维护与迭代指南](MAINTAINING.md) — 架构、数据契约、扩展方法、测试门禁、发布流程
- `test.html` — 浏览器自动化测试，加载真实 `index.html` 生产代码
