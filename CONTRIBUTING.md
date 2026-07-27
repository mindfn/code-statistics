# Contributing / 贡献指南

Thank you for your interest in contributing! This project is a single-file browser application — all production code lives in `index.html`.

感谢你对本项目的关注！这是一个单文件浏览器应用，所有生产代码都在 `index.html` 中。

## Development Setup / 开发环境

```bash
# Clone the repository
git clone https://github.com/mindfn/code-statistics.git
cd code-statistics

# Start a local server (required for tests — file:// doesn't support iframe cross-document access)
python3 -m http.server 8901 --bind 127.0.0.1
```

Open in browser:
- App: http://127.0.0.1:8901/index.html
- Tests: http://127.0.0.1:8901/test.html

## Testing / 测试

Tests run in the browser via `test.html`, which loads the real `index.html` through a hidden iframe — no test doubles for production functions.

```bash
# Syntax check
sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' | node --check -
sed -n '/^<script>$/,/^<\/script>$/p' test.html | sed '1d;$d' | node --check -

# Whitespace check
git diff --check
```

### Red-Green Flow / 红绿测试流程

1. Add a failing test in `test.html` that reproduces the issue or expresses the new behavior
2. Confirm the old code fails the test
3. Fix `index.html`
4. Confirm the test turns green, then run all tests

## Commit Messages / 提交规范

Use conventional-style messages in Chinese or English:

```
feat: 新功能描述
fix: 修复了什么问题
docs: 文档更新
```

If developed with AI assistance, include the co-author trailer:

```
Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

## Architecture / 架构概览

```
index.html          — All HTML, CSS, JavaScript (single file, no dependencies)
test.html           — Browser tests (loads index.html via iframe)
MAINTAINING.md      — Module map, data contracts, extension guide
docs/features/      — Feature specifications
docs/discussions/   — Design discussions
screenshots/        — UI screenshots for review evidence
```

Key modules inside `index.html` (find by section comments):

| Module | Responsibility |
|--------|---------------|
| `Storage` | Versioned `localStorage` read/write |
| `CacheStore` | Merged PR cache (IndexedDB / Memory) |
| `GitHubProvider` | GitHub API integration |
| `GitCodeProvider` | GitCode API integration |
| `StatsEngine` | User mapping, aggregation, filtering |
| `CSV Helper` | RFC 4180 parse/generate, BOM, formula protection |

## Guidelines / 开发原则

- **Single file** — Do not split `index.html` into multiple files or add external dependencies
- **No CDN** — All assets must be inline (CSS, JS, SVG favicon)
- **Privacy** — Tokens must never appear in CSV, logs, URLs, or error messages
- **Completeness** — API failures must propagate as "partial" or "failed", never silently zero
- **Evidence** — "Done" means tests pass + browser verification
- **Statistics invariants** — See [MAINTAINING.md](MAINTAINING.md) §5 before changing any counting logic

## Reporting Issues / 反馈问题

Open an issue with:
- Browser version and access method (`http(s)` or `file://`)
- Steps to reproduce
- Expected vs actual behavior
- Console errors if any (F12 → Console)
