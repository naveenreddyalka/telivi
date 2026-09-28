# Telivi roadmap

The order of work. Each phase ends with something a person can run and
observe. Issues carry the detail; this file only says what comes before what.

## Phase 0 — Decide

The PRD left five decisions open. Three block the first code and are filed as
`decision` issues. The other two are deferred to Phase 2, where a stand-in is
replaced by the real thing.

| ADR | Decision | Blocks |
|-----|----------|--------|
| 0001 | Implementation language, toolchain, test runner, CI | everything in Phase 1 |
| 0002 | How a person is identified on the chain | ledger authorship, stake book keys |
| 0003 | How a device proves it did the training work | Phase 2 only; Phase 1 uses an injected verifier |

Not decided in Phase 0, by design: model architecture and training objective;
how the coordinator reaches a phone or a browser; stake transfer or
governance. The PRD keeps them under "Decisions not made yet".

## Phase 1 — Thin slice with stand-ins

The four modules exist as small, tested libraries. The device is a stand-in
that returns a known training result. Verification is an injected function.
The "model" is any array of weights. Nothing is trained for real.

Order, from the PRD's Testing Decisions:

1. **Weight ledger.** First write names the author. Update appends and changes
   the current author. Rewrite of an earlier entry fails. Chain is readable in
   order.
2. **Stake book.** Verified work increases stake. Unverified work leaves it
   unchanged. Stopping lending keeps earned stake. Anyone can read every stake.
3. **Coordinator + stand-in runtime.** Hands out one unit of work. A completed
   result changes the shared model and writes to the ledger and stake book. A
   result that fails verification changes nothing.
4. **Public rules.** `docs/RULES.md` states the training and stake rules in
   plain language, so a reader can check that stake follows verified work.

Exit: one command runs a stand-in device through a full step and prints the
weight it changed, its author chain, and the contributor's stake.

## Phase 2 — Replace the stand-ins

One at a time, each behind the interface Phase 1 fixed:

- A browser tab as the contributor runtime (real compute, toy model).
- Real verification per ADR-0003, replacing the injected function.
- A phone as the contributor runtime, same interface as the browser.
- A real, small model and training objective (a new `decision` issue first).

Exit: a person opens a browser tab, lends compute to a toy model, and sees
their name on a weight and their stake grow.

## Phase 3 — Many devices, one model

Coordinator handles concurrent contributors; ledger and stake book survive
restarts; anyone can read the model, its history, and the stake table from
outside the project. Only then: the questions the PRD parked (transfer,
governance, app-store release).
