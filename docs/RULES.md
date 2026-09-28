# Telivi rules

The training and stake rules, in plain language, so anyone can check that
stake follows work that was actually done
([PRD § Solution](PRD.md?plain=1#L21)).

Every rule below comes from one line of [the PRD](PRD.md) and links to it.
Once the tests that enforce a rule exist, the rule links to them too. No
tests exist yet; each rule says which issue they land with.

## Stake

### 1. Work becomes stake

You lend your phone or browser. It does a piece of training. When that work
is verified, your stake in the model grows. A stake is a share of the model,
not a separate coin.

- Source: [PRD § Solution, line 17](PRD.md?plain=1#L17);
  [PRD § Decisions already made, line 58](PRD.md?plain=1#L58)
- Test: none yet — lands with the stake book
  ([#6](https://github.com/naveenreddyalka/telivi/issues/6))

### 2. Unverified work earns nothing

Work that is not verified does not change your stake. Nobody can claim stake
for training that did not happen.

- Source: [PRD § Implementation Decisions, Stake book, line 53](PRD.md?plain=1#L53);
  [PRD § Testing Decisions, line 75](PRD.md?plain=1#L75)
- Test: none yet — lands with the stake book
  ([#6](https://github.com/naveenreddyalka/telivi/issues/6))

### 3. Stopping keeps your stake

You can stop lending at any time. The stake you already earned stays yours,
and the device is yours again.

- Source: [PRD § Implementation Decisions, Stake book, line 53](PRD.md?plain=1#L53);
  [PRD § Testing Decisions, line 75](PRD.md?plain=1#L75)
- Test: none yet — lands with the stake book
  ([#6](https://github.com/naveenreddyalka/telivi/issues/6))

### 4. A stake is not a coin, and these rules do not make it transferable

A stake is ownership of the model. It is not a coin, and the PRD does not make
it transferable. Whether a stake can later be transferred, split, or used for
governance is an open question, not a decision made here.

- Source: [PRD § Decisions already made, line 59](PRD.md?plain=1#L59);
  open question at [PRD § Decisions not made yet, line 69](PRD.md?plain=1#L69)
- Test: none yet — lands with the stake book
  ([#6](https://github.com/naveenreddyalka/telivi/issues/6))

### 5. Anyone can read every stake

You can see your own stake, and anyone can see how much of the model each
person holds.

- Source: [PRD § Implementation Decisions, Stake book, line 53](PRD.md?plain=1#L53);
  [PRD § Testing Decisions, line 75](PRD.md?plain=1#L75)
- Test: none yet — lands with the stake book
  ([#6](https://github.com/naveenreddyalka/telivi/issues/6))

## Weights

### 6. Every weight names its author

Each weight records who wrote it. The first person to write a weight is its
author. When someone else updates it, they become its current author, and the
earlier author stays in the weight's history.

- Source: [PRD § Solution, line 19](PRD.md?plain=1#L19);
  [PRD § Implementation Decisions, Weight ledger, line 52](PRD.md?plain=1#L52)
- Test: none yet — lands with the weight ledger
  ([#5](https://github.com/naveenreddyalka/telivi/issues/5))

### 7. Every weight keeps an append-only chain

Each update to a weight is added to the end of that weight's chain: who
changed it, and in what order. Nothing is removed from the chain. A bad
update stays visible, because the history is the whole history.

- Source: [PRD § Implementation Decisions, Weight ledger, line 52](PRD.md?plain=1#L52);
  [PRD § Decisions already made, line 61](PRD.md?plain=1#L61)
- Test: none yet — lands with the weight ledger
  ([#5](https://github.com/naveenreddyalka/telivi/issues/5))

### 8. A rewrite fails

An attempt to change or replace an earlier entry in a weight's chain fails.
The only thing you can do to a chain is add to it.

- Source: [PRD § Implementation Decisions, Weight ledger, line 52](PRD.md?plain=1#L52);
  [PRD § Testing Decisions, line 76](PRD.md?plain=1#L76)
- Test: none yet — lands with the weight ledger
  ([#5](https://github.com/naveenreddyalka/telivi/issues/5))

### 9. Anyone can read any weight

You can read a weight, its current author, and its chain in order, without
asking anyone.

- Source: [PRD § Solution, line 19](PRD.md?plain=1#L19);
  [PRD § Testing Decisions, line 76](PRD.md?plain=1#L76)
- Test: none yet — lands with the weight ledger
  ([#5](https://github.com/naveenreddyalka/telivi/issues/5))

## Joining

### 10. No one grants permission to join

The model is open. You do not ask a company or an operator for permission to
lend compute.

- Source: [PRD § Decisions already made, line 57](PRD.md?plain=1#L57)
- Test: none yet — lands with the coordinator
  ([#7](https://github.com/naveenreddyalka/telivi/issues/7))

## What these rules do not decide

These rules say that work must be verified before it counts, and that a weight
names its author. They do not say how work is verified or how a person is
identified. Those, along with the model architecture, how the coordinator
reaches a device, and whether a stake can be transferred, are listed in the PRD
as [decisions not made yet](PRD.md?plain=1#L63). Each will be settled by its
own decision record in [docs/decisions/](decisions/README.md) before any code
depends on it.
