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

## Landscape

Related work, one line each, with the part of Telivi it informs. Scanned
2026-09-29. None of it decides anything in the PRD; it is input for the ADRs
and the Phase 2 stand-in replacements.

- **Verification of contributed work (ADR-0003).** Gensyn's
  [Verde](https://arxiv.org/abs/2502.19405) verifies training on untrusted
  nodes by bisecting a dispute down to one operator, and needs bitwise
  reproducible kernels (RepOps) to do it. Prime Intellect's
  [TOPLOC](https://proceedings.mlr.press/v267/ong25a.html) hashes
  intermediate activations to verify inference cheaply, but does not extend to
  training. Both papers describe Proof-of-Learning-style heuristics as
  spoofable. Phase 2's real verifier has to pick a point on that
  cost-versus-guarantee line.
- **Permissionless training over the internet (coordinator).** Prime
  Intellect's [INTELLECT-2](https://arxiv.org/abs/2505.07291) ran 32B RL
  training over a permissionless swarm, with untrusted workers doing rollouts
  and trusted nodes doing the weight updates. Nous Research's
  [Psyche](https://nousresearch.com/nous-psyche) coordinates a 40B
  pretraining run through a chain and compresses gradients with DisTrO.
  Pluralis's [Agora](https://pluralis.ai/docs/) splits a 13B model into
  pipeline stages so no contributor holds the whole model. All three assume
  a GPU per contributor; none targets a phone or a browser tab.
- **Who owns the result (stake book).** Bittensor's Templar subnet trained
  [Covenant-72B](https://docs.tplr.ai/validators/weight-setting/) with 70+
  contributors and pays them in a token from validator scores of gradient
  quality. Macrocosmos's
  [pretraining subnet](https://www.macrocosmos.ai/research/pretraining_whitepaper.pdf)
  ties each uploaded model to one miner UID on chain. Reward there is a coin,
  and ownership is per model, not per weight. Telivi's stake is a share of the
  model and its attribution is per weight, which is the gap this project fills.
- **Browser and phone as the runtime (Phase 2).**
  [quectoGPT](https://github.com/minusxai/quectoGPT) trains a small GPT
  across browser tabs over WebGPU with a coordination server averaging weight
  deltas; [EdgeTrain](https://github.com/v-code01/edgetrain) trains in the
  browser with WGSL shaders and a CPU fallback.
  [Flower-CEC-WebBrowser](https://github.com/fcrlab-unime/Flower-CEC-WebBrowser)
  wraps a browser as a Flower federated-learning client with nothing installed.
  WebGPU is native in Chrome and Edge and behind flags elsewhere, so the
  browser runtime needs a CPU path on day one.
