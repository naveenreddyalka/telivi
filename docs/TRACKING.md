# Tracking

Status log for merged work. One `###` section per issue, newest at the bottom
of the list, above `## How to update`. The loop appends here; humans read it.

## Phase 0 — Decide

_No ADRs merged yet._

## Phase 1 — Thin slice

### docs/RULES.md: the training and stake rules in the open ([#8](https://github.com/naveenreddyalka/telivi/issues/8))

<!-- tracking:#8 -->

**Status:** merged 2026-09-28. Added `docs/RULES.md` (ten rules, each linked
to its PRD line) and a link from `README.md`. Documentation only; no test file
covers it. The rules link to the ledger and stake book tests once #5 and #6
land.

<!-- tracking-append: add the next ### section above ## How to update; on conflict keep both -->

## How to update

1. Add a `### <title> ([#N](https://github.com/naveenreddyalka/telivi/issues/N))`
   section with a `<!-- tracking:#N -->` marker and a **Status** line: what
   merged, the date, and the test file that covers it.
2. Do not mark anything done without a merged PR.
3. If two PRs conflict here, keep both sections.
