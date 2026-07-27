# Code Statistics

> Analyze merged PR code changes across GitHub and GitCode repositories — zero install, runs entirely in the browser.

**[中文说明](README.zh-CN.md)**

**Live Demo:** https://mindfn.github.io/code-statistics/

## Features

- **Dual-platform support** — GitHub and GitCode, queried via official APIs
- **PR-based metrics** — Additions, deletions, changed files, comments, and approvals for merged PRs
- **Zero install** — Single `index.html` file, no server/build/dependencies required
- **Privacy-first** — All data stays in your browser (`localStorage` + `IndexedDB`); tokens are never uploaded
- **CSV export** — Summary and detail exports with UTF-8 BOM, RFC 4180 compliant
- **Smart caching** — Merged PR data cached in IndexedDB; repeat queries skip already-fetched PRs
- **Branch selection** — Configure target branch per repository with searchable dropdown
- **Completeness tracking** — Partial/failed PR data is clearly marked, never silently zeroed

## Quick Start

1. Open https://mindfn.github.io/code-statistics/ (or download `index.html` and double-click)
2. **Config** — Enter your GitHub/GitCode tokens and add repositories
3. **Users** — Add users with their platform accounts (manual or CSV import)
4. **Statistics** — Select date range, click "Query", then export CSV

## Token Permissions

| Platform | Minimum Permission |
|----------|-------------------|
| GitHub (Classic PAT) | `repo` (private) or `public_repo` (public only) |
| GitHub (Fine-grained PAT) | **Pull requests** read-only on target repos |
| GitCode | Personal access token with repository read access |

Tokens are stored in browser `localStorage` only. They are sent directly to GitHub/GitCode APIs via `Authorization` headers — never included in CSV exports, logs, or URLs.

## Data Storage & Security

- **No backend server** — GitHub Pages hosts static files only; no user data is received
- **PR cache on device** — Uses IndexedDB (`code-statistics.pr-cache.v1`) over `http(s)`; memory-only on `file://`
- **One-click cache clear** — Config page "Data Management" clears PR cache without affecting settings or users
- **No cross-device sync** — Data is local to each browser instance

## Statistics Methodology

- Only **merged PRs** are counted, based on `merged_at` falling within the selected date range
- User attribution is by **platform account** (PR author / comment author / reviewer)
- Only PRs merged into the **target branch** (defaults to repository default branch) are included
- Direct pushes, unmerged PRs, and individual commits within others' PRs are excluded
- GitHub and GitCode mirror repos are **not auto-deduplicated**
- Three activity types, all attributed to the PR's merge timestamp:
  - **Development PR** — PR author; code metrics attributed to author only
  - **Comments** — Comment author; deduplicated by `source_type + id`
  - **Approve** — Reviewer (GitHub: last decisive review state; GitCode: current effective state)
- Metrics: changes = additions + deletions; net = additions − deletions

## CSV Format

**User import CSV:**
```csv
user_key,display_name,email,github_login,gitcode_login
```

**Export CSV** — Summary and detail CSVs, UTF-8 BOM, RFC 4180, formula injection protection, no tokens.

## Known Limitations

| Limitation | Details |
|-----------|---------|
| GitHub search limit | Max 1000 PRs per search; excess marked as "partial" |
| GitCode listing optimization | Uses `since=` (14-day buffer) to reduce API calls; final filtering by `merged_at` |
| GitCode Approve | Reflects current effective approval status, not full approval history |
| GitCode large PRs | `too_large` means diff text truncated; line counts may still be parseable |
| Browser compatibility | Latest Chrome / Edge recommended |
| No in-app proxy | Uses system/browser proxy settings |
| `file://` cache | IndexedDB unavailable on `file://`; memory-only cache |

## Tech Stack

- Single `index.html` — all HTML, CSS, JavaScript inline, no CDN or dependencies
- `localStorage` for config (`code-statistics.config.v1`) and users (`code-statistics.users.v1`)
- Repository config supports `{url, branch}` format with automatic migration from legacy strings
- IndexedDB caches complete merged PRs by `platform + repository + PR number`
- GitHub API: PR Search + PR Detail + PR Comments + Review Comments + Reviews
- GitCode API v5: PR List + Detail/Files + Comments

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, testing, and contribution guidelines.

## License

[MIT](LICENSE)

## Maintainers

- [Maintenance & Development Guide](MAINTAINING.md) — Architecture, data contracts, extension guide, test gates, release flow
