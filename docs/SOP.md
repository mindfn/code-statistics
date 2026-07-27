---
topics: [sop, workflow]
doc_kind: note
created: 2026-07-24
---

# Standard Operating Procedure

## Workflow (6 steps)

| Step | What | Skill |
|------|------|-------|
| 1 | Create worktree | `worktree` |
| 2 | Self-check (spec compliance) | `quality-gate` |
| 3 | Peer review | `request-review` / `receive-review` |
| 4 | Merge gate | `merge-gate` |
| 5 | PR + cloud review | (merge-gate handles) |
| 6 | Merge + cleanup | (SOP steps) |

## Code Quality

- Production script syntax:
  `sed -n '/^<script>$/,/^<\/script>$/p' index.html | sed '1d;$d' | node --check -`
- Test script syntax:
  `sed -n '/^<script>$/,/^<\/script>$/p' test.html | sed '1d;$d' | node --check -`
- Browser tests: serve the repository on a non-reserved port and open `test.html`; pass count must increase when behavior is added.
- Whitespace and patch hygiene: `git diff --check`
- Browser acceptance: validate the final branch locally, then smoke-test the deployed GitHub Pages SHA.
