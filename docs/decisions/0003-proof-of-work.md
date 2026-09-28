# ADR-0003: How a device proves it did the training work

**Status:** proposed
**Issue:** #4
**Date:** 2026-09-28

## Context

[PRD story 16](../PRD.md#user-stories): a contribution is accepted only when
the device did the training work, so that stake cannot be claimed for nothing.
The PRD fixes the constraints and leaves the mechanism open:

- Unverified work does not change a stake (Stake book). A result that fails
  verification does not change the shared model (Testing Decisions).
- Devices are phones and browsers people own: no trusted hardware, no
  attestation, no control over the numeric libraries a device runs.
- The coordinator owns neither the weights nor the compute, so a check that
  redoes most of the training defeats the project. Rules are public (story 20).
- Identities are free (ADR-0002): one person may hold any number of keys.

The threat is free-riding, not just noise: returning the given model plus
small noise passes naive outlier filters and would collect stake
([Fraboni et al. 2021](https://arxiv.org/abs/2006.11901)). Proofs of training
do not fit a phone: zero-knowledge proofs cost the prover minutes per round
for tiny models ([Falafel, 2026](https://eprint.iacr.org/2026/1335)), and
proof-of-learning has known forgeries ([Jia et al. 2021](https://arxiv.org/abs/2103.05633)).

[ROADMAP Phase 1](../ROADMAP.md#phase-1--thin-slice-with-stand-ins) uses an
injected verifier; this ADR fixes its shape so the real one drops in during
Phase 2 without touching the coordinator, ledger, or stake book.

## Options

### A. Redundant computation

The same work unit goes to k devices; accept when a quorum agrees
([BOINC: handling completed jobs](https://github.com/BOINC/boinc/wiki/Handling-completed-jobs)).
Floating-point results differ across phones, browsers, and GPU drivers, so
bit-equality fails between honest devices: compare within a tolerance derived
from measured honest-vs-honest disagreement, or restrict a unit's replicas to
one numeric class and demand equality
([BOINC homogeneous redundancy](https://github.com/BOINC/boinc/wiki/Homogeneous-Redundancy)).

Cost to contributors: with k=2, half of all lent compute is verification; k=3
wastes two thirds. Coordinator cost: comparing two vectors, O(d). What a
cheater still gets away with: nothing alone, if replicas are assigned at
random and results are held until quorum (no copying). m identities under one
person's control out of N active devices land both replicas of that person's
unit on that person with probability about (m-1)/(N-1) per unit and accept
each other; identities being free, only N bounds this. A loose tolerance lets
a device run fewer steps than asked.

### B. Spot-check recomputation by the coordinator

The coordinator redoes a random fraction p of accepted units and rejects on
mismatch. Cost to contributors: none. Coordinator cost: p of all training
compute, so p must stay in low single-digit percent or the coordinator becomes
the thing the PRD says it is not. What a cheater gets away with: each unit
with probability 1-p. Weak alone, and it rests on the coordinator's arithmetic.

### C. Gradient plausibility checks

Cheap tests on the returned update: finite values, expected shape, norm inside
a bound, loss goes down on a held-out batch, cosine similarity to peers or to
a trusted reference update ([FLTrust](https://arxiv.org/abs/2012.13995)).
Byzantine-robust aggregation (Krum, [Blanchard et al. 2017](https://arxiv.org/abs/1703.02757);
trimmed mean, [Yin et al. 2018](https://arxiv.org/abs/1803.01498)) belongs
here too: it limits a bad update's damage rather than detecting it. Cost to
contributors: none. Coordinator cost: O(d) plus one forward pass. What a
cheater gets away with: the disguised free-rider from Context, whose noise has
a plausible norm and direction and does not raise loss. Stops damage; does not
prove work.

### D. Combination

Plausibility as a gate on every result, redundancy as the proof of work, and
coordinator recomputation as tie-break and audit: redundancy plus a robust
rule is the shape of [DETOX](https://arxiv.org/abs/1907.12205); the adaptive
rate is [BOINC adaptive replication](https://github.com/BOINC/boinc/wiki/Adaptive-Replication),
which reports overhead falling from at least 50% to roughly 5–10%.

Cost to contributors: 100% overhead for a new identity (every unit replicated
once), falling toward a floor after a run of agreeing results. Coordinator
cost: O(d) per result plus a small audit fraction. What a cheater gets away
with: an identity that earned low replication can cheat in the unreplicated
share until a replica or audit catches it and its rate returns to 100%; an
identity pair on the same unit, as in A. Collusion resistance rests on random
assignment, results withheld until quorum, 100% replication for every new
identity, and the audit fraction; not on identity cost.

## Recommendation

Option D. A result earns stake only after it passes a plausibility gate and
agrees, within a tolerance, with an independent recomputation of the same
work unit: another device at an adaptive replication rate, or the coordinator
as random spot-check and tie-break. A alone wastes half the compute forever; B
alone makes the coordinator the trainer; C alone accepts the free-rider.

Rules a reader can follow: a new identity's units are replicated (k=2);
agreement within tolerance accepts both; disagreement sends a third replica
and majority wins; no majority sends the unit to the coordinator; after c
consecutive agreements an identity's replication probability falls toward a
floor, never zero; a small random fraction of accepted units is recomputed by
the coordinator regardless. Tolerance, c, floor, and audit fraction are
measured in Phase 2, not fixed here.

What one unit puts on the ledger and the model: every `Accepted` result is one
chain entry on each weight it touches, signed by its own author over its own
bytes and the work-unit id (ADR-0002), so both devices of an agreeing pair are
named on the weight and both earn stake. The coordinator writes no entry and
holds no stake; its recomputation is evidence, not authorship. The model
applies each unit once, as a public, deterministic function of that unit's
accepted entries (Phase 1: the one accepted update; Phase 2: the mean of the
agreeing replicas, any robust rule from C choosing among signed entries), so
nothing unsigned lands on a chain and a reader can recompute the model from it.

The verifier is one injected function, described language-neutrally:

```
verify(workUnit, result, evidence) -> Accepted | Rejected(reason) | Pending
```

- `workUnit` carries a work-unit id, issued once and never reused (ADR-0002
  signs over it), and a hash of its inputs (weights slice, data, steps, seed).
- `result` carries the work-unit id, the author's identity and signature per
  ADR-0002 (over weight id, work-unit id, and update, so no chain position is
  needed at sign time), the update itself, and a hash of the update.
- `evidence`: an opaque blob from the runtime, stored and forwarded unchanged;
  Phase 2 puts numeric class, state hashes, and timing there; Phase 1: empty.
- Only `Accepted` changes the model, the ledger, or a stake. The real verifier
  returns `Pending` while a unit waits for its replica; the Phase 1 stand-in
  never does, and accepts or rejects by a known rule (say, the hash mismatches).
- The verifier may keep its own state (replica sets, per-identity streaks);
  the coordinator only reads the return value.

## Consequences

- Phase 1's coordinator handles three outcomes from day one, so Phase 2 swaps
  the function without touching coordinator, ledger, or stake book. Hand-out
  is random, results wait for quorum, and nobody learns who else holds a unit.
- Early work is half verification, and collusion is bounded by network size,
  not identity cost: the (m-1)/(N-1) attack in A is not small while N is small
  in Phase 2. `docs/RULES.md` must state both in plain language.
- A caught identity loses only its streak and can be discarded for free; the
  later `decision` on caught devices must not assume it persists. Earned
  stake is untouched here.
- Next issues: define `verify`, its three outcomes, and the once-per-unit
  entry rule in the coordinator's Phase 1 API (#7); measure honest-vs-honest
  disagreement on browser and phone in Phase 2 to set the tolerance.
