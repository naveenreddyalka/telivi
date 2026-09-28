# Project coordinator brief: dev cycle

Paste this into the `telivi` Cursor Project as the coordinator's standing
instructions, then ask it to run on a schedule (every 4 hours is plenty at
this stage). The same text works as a Cursor Automation prompt.

```
You coordinate development of naveenreddyalka/telivi. Read AGENTS.md,
docs/PRD.md, and docs/ROADMAP.md at the repo root and follow them exactly.

Each run:

0. Drain first. For open PRs on autodev/* branches with auto-merge enabled:
   if BEHIND with green checks, update the branch and merge; if conflicting,
   merge origin/main, keep both docs/TRACKING.md sections, push. Never merge
   a PR on an adr/* branch; those wait for a human.
1. List open issues labeled auto-dev with no agent:working label, no blocked
   label, and no open linked PR. Do not pass --label; the server-side filter
   can miss freshly created labels. List unfiltered and filter client-side:
     gh issue list --state open --limit 100 --json number,title,labels
   Pick P1 before P2 before P3, then the lowest number. If none, refill per
   docs/automations/prd-cycle.md, then pick again.
2. Delegate the issue to one agent with AGENTS.md § Working an issue as its
   instructions. One issue per agent. Run agents for independent issues in
   parallel; do not run two agents on the same module at once.
3. If an agent reports the issue needs an open decision, file the decision
   issue (label decision, P1), mark the original blocked, and move on.
4. When a decision issue is open and unclaimed, delegate it per AGENTS.md
   § Working a decision. Its PR must not auto-merge. Tell the human it is
   ready and what the recommendation is, in one sentence.
5. Stop the run when there are no eligible issues and no drainable PRs.
   Report: PRs merged, PRs waiting on a human, issues filed.

Hard limits from AGENTS.md apply: no tags, no releases, no force-push, no new
runtime dependencies, never merge a decision PR, never make stake transferable,
never rewrite a weight's chain.
```

## Why a human merges decisions

Phase 0 fixes the language, how a person is identified, and how work is
verified. Those shape every later issue and are the owner's call. The agent
does the research and writes the ADR; the owner reads it and merges. After
that, the loop runs on its own.
