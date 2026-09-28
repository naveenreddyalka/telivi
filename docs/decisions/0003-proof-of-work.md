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
- Devices are phones and browsers people own. No trusted hardware, no
  attestation, no control over the numeric libraries a device runs.
- The coordinator does not own the weights and is not meant to own the
  compute, so a check that makes it redo most of the training defeats the
  project. The rules must be public (story 20), so a check a reader cannot
  follow is a weak check.

The threat is free-riding, not just noise: a device that returns the model it
was given plus small noise passes naive outlier filters and would collect
stake ([Fraboni et al. 2021](https://arxiv.org/abs/2006.11901)). Proofs of
training do not fit a phone: zero-knowledge proofs cost the prover minutes per
round even for tiny models ([Falafel, 2026](https://eprint.iacr.org/2026/1335)),
and non-cryptographic proof-of-learning has known forgeries
([Jia et al. 2021](https://arxiv.org/abs/2103.05633),
[Zhang et al. 2022](https://arxiv.org/abs/2208.03567)).

[ROADMAP Phase 1](../ROADMAP.md#phase-1--thin-slice-with-stand-ins) uses an
injected verifier. This ADR fixes its shape so the real one drops in during
Phase 2 without touching the coordinator, ledger, or stake book. It does not
decide identity (ADR-0002), the model, the transport, or stake transfer.

## Options

### A. Redundant computation

The same work unit goes to k devices; accept when a quorum agrees, as
volunteer computing has done for two decades
([BOINC: handling completed jobs](https://github.com/BOINC/boinc/wiki/Handling-completed-jobs)).
Floating-point results differ across phones, browsers, and GPU drivers, so
bit-equality fails between honest devices. Either compare within a tolerance
derived from measured honest-vs-honest disagreement, or restrict a unit's
replicas to one numeric class and demand equality
([BOINC homogeneous redundancy](https://github.com/BOINC/boinc/wiki/Homogeneous-Redundancy)).
Deterministic kernels shrink the tolerance but are not available on every GPU
([PyTorch reproducibility notes](https://pytorch.org/docs/stable/notes/randomness.html)).

Cost to contributors: with k=2, half of all lent compute is verification; k=3
wastes two thirds. Coordinator cost: comparing two vectors, O(d). What a
cheater still gets away with: nothing alone, if replicas are assigned at
random and results are held until quorum (no copying). Two identities under
one person's control on the same unit can accept each other, with probability
about 1/N per unit for N active devices, bounded by how costly ADR-0002 makes
a second identity. A loose tolerance lets a device run fewer steps than asked.

### B. Spot-check recomputation by the coordinator

The coordinator redoes a random fraction p of accepted units and rejects on
mismatch. Cost to contributors: none. Coordinator cost: p of all training
compute, so p must stay in low single-digit percent or the coordinator becomes
the thing the PRD says it is not. What a cheater gets away with: each unit
with probability 1-p; at p=0.02 a device expects about fifty free units before
the first catch. Weak alone, and it rests on the coordinator's own arithmetic.

### C. Gradient plausibility checks

Cheap tests on the returned update: finite values, expected shape, norm inside
a bound, loss goes down on a held-out batch, cosine similarity to peers or to
a trusted reference update ([FLTrust](https://arxiv.org/abs/2012.13995)).
Byzantine-robust aggregation (Krum,
[Blanchard et al. 2017](https://arxiv.org/abs/1703.02757); trimmed mean,
[Yin et al. 2018](https://arxiv.org/abs/1803.01498)) belongs here too: it
limits a bad update's damage rather than detecting it. Cost to contributors:
none. Coordinator cost: O(d), plus one forward pass for the loss check. What a
cheater gets away with: the disguised free-rider from Context, whose noise has
a plausible norm and direction and does not raise loss. These checks stop
damage; they do not prove work.

### D. Combination

Plausibility as a gate on every result, redundancy as the proof of work, and
coordinator recomputation as tie-break and audit. Redundancy plus a robust
rule is the shape [DETOX](https://arxiv.org/abs/1907.12205) uses; the
adaptive rate is [BOINC adaptive replication](https://github.com/BOINC/boinc/wiki/Adaptive-Replication),
which reports overhead falling from at least 50% to roughly 5–10%.

Cost to contributors: 100% overhead for a new device (every unit replicated
once), falling toward a floor after a run of agreeing results. Coordinator
cost: O(d) per result plus a small audit fraction. What a cheater gets away
with: a device that earned low replication can cheat in the unreplicated share
until a replica or audit catches it, after which its rate returns to 100%; an
identity pair on the same unit, bounded as in A. Collusion resistance rests on
random assignment, withholding results until quorum, and ADR-0002's cost.

## Recommendation

Option D. A result earns stake only after it passes a plausibility gate and
agrees, within a tolerance, with an independent recomputation of the same
work unit: another device at an adaptive replication rate, or the coordinator
as random spot-check and tie-break. A alone wastes half the compute forever on
phones with small batteries; B alone makes the coordinator the trainer and
cannot be checked by a reader; C alone accepts the disguised free-rider.

Rules a reader can follow: a new device's units are replicated (k=2);
agreement within tolerance accepts both; disagreement sends a third replica
and majority wins; no majority sends the unit to the coordinator; after c
consecutive agreements a device's replication probability falls toward a
floor, never zero; a small random fraction of accepted units is recomputed by
the coordinator regardless. Tolerance, c, the floor, and the audit fraction
are measured in Phase 2, not fixed here. The verifier is one injected
function, described language-neutrally:

```
verify(workUnit, result, evidence) -> Accepted | Rejected(reason) | Pending
```

- `workUnit` carries a work-unit id and a hash of its inputs (weights slice,
  data reference, step count, seed).
- `result` carries the work-unit id, an opaque contributor reference (shape
  fixed by ADR-0002, not here), the update itself, and a hash of the update.
- `evidence` is an opaque blob the runtime returns alongside the result; the
  coordinator stores and forwards it unchanged. Phase 2 puts the device's
  numeric class, intermediate state hashes, and timing there; Phase 1: empty.
- Only `Accepted` changes the model, the ledger, or a stake. `Rejected` and
  `Pending` change nothing. The real verifier returns `Pending` while a unit
  waits for its replica; the Phase 1 stand-in never does.
- The verifier may keep its own state (replica sets, per-device streaks); the
  coordinator only reads the return value.

Phase 1 tests inject a stand-in that accepts or rejects by a known rule (say,
rejects when `evidence` says so or the update hash does not match), which is
what the PRD's Testing Decisions call for.

## Consequences

- Phase 1's coordinator handles three outcomes from day one, so Phase 2 swaps
  the function without touching the coordinator, ledger, or stake book.
- Work assignment must be random and results withheld until quorum; the Phase
  1 hand-out step must not leak who else holds a unit.
- Early work is half verification; the public rules
  ([ROADMAP Phase 1, step 4](../ROADMAP.md)) must say so. Sybil resistance is
  bounded by ADR-0002's identity cost; that ADR should state it.
- Next issues: define `verify` and its three outcomes in the coordinator's
  Phase 1 API; measure honest-vs-honest disagreement on browser and phone in
  Phase 2 to set the tolerance; decide what happens to a caught device beyond
  rejection (a later `decision` issue; this ADR does not touch earned stake).
