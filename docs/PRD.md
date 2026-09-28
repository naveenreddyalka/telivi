# Telivi PRD

**తెలివి** is Telugu for intelligence.

Telivi is an open model trained by people on their own devices. Phones and browsers lend the compute. The people who lend it hold a stake in the model. Each weight names its author and keeps a chain of who updated it.

## Problem Statement

Training a model takes a large amount of compute, and that compute is held by a few labs. The weights, the record of who changed them, and the ownership of the result stay with those labs.

A person with a phone or a browser cannot join that work. When distributed training does exist, the person who lent the device does not own any part of the model they helped train, and a weight does not say who wrote it or who changed it later.

## Solution

Telivi is one shared model, trained in the open, on devices people choose to lend.

A person opens Telivi on a phone or in a browser and lends compute. That work trains the model. In return they hold a stake: a share of the model that grows as they contribute verified training work. The stake is ownership of the model, not a separate coin.

Every weight is a node with an author. When someone updates it, the update is appended to a chain on that weight: who changed it, and in what order. The chain is not rewritten. Anyone can read a weight, its current author, and its history.

No company grants permission to join. The training rules are public, so a person can see that stake follows work that was actually done.

## User Stories

1. As a person with a phone, I want to lend its compute to training, so that I hold a stake in the model I helped build.
2. As a person with a browser, I want to contribute from a tab, so that I can join without installing an app.
3. As a new person, I want to join without asking a company for permission, so that the model stays open.
4. As a contributor, I want the project to give my device a specific piece of training to do, so that many devices train one model.
5. As a contributor, I want my finished step to land in the shared model, so that the work changes the weights everyone shares.
6. As a contributor, I want my stake to grow when I lend more verified compute, so that ownership follows contribution.
7. As a contributor, I want proof that my work was counted, so that my share cannot be dropped quietly.
8. As a contributor, I want to stop lending, so that I keep the stake I already earned and the device is mine again.
9. As a contributor, I want the weights I trained to name me as author, so that my work stays attached to me.
10. As a later contributor, I want to update a weight someone else started, so that the model can keep learning.
11. As anyone, I want the chain of who updated a weight, in order, so that I can see its history.
12. As anyone, I want a rewrite of that chain to fail, so that history stays intact.
13. As anyone, I want to read a weight, its current author, and its history, so that the model is open to inspect.
14. As anyone, I want a bad update to remain visible on the chain, so that authorship includes the whole history.
15. As a contributor, I want my identity to be the one recorded on the chain, so that someone else cannot take authorship of my update.
16. As anyone, I want a contribution accepted only when the device did the training work, so that stake cannot be claimed for nothing.
17. As a contributor, I want to see my own stake, so that I know what I hold.
18. As anyone, I want to see how much of the model each person holds, so that ownership is public.
19. As anyone, I want to use the model, so that the result of the shared training is not locked to the people who trained it.
20. As anyone, I want the training rules in the open, so that I can check that stake follows contributed work.

## Implementation Decisions

Telivi is four modules. Each one hides a hard problem behind a small interface.

- **Contributor runtime.** Runs on a phone or in a browser. Accepts a piece of training work, does it on the device, and returns a result that can be checked. The phone and the browser are the same kind of contributor.
- **Coordinator.** Hands out training work so many devices train one model, and accepts a result back into that model. It does not own the weights.
- **Weight ledger.** Stores each weight as a node. The node has a current author and an append-only chain of updates. A new update appends. A rewrite of an earlier entry fails. Anyone can read the node.
- **Stake book.** Turns verified training work into a share of the model for the person who did it. Unverified work does not change a stake. Stopping lending does not erase stake already earned. The book is readable by anyone.

Decisions already made:

- The model is open. Joining does not require permission from an operator.
- Ownership is both attribution and a stake. Attribution is the author and the update chain on each weight. A stake is a share of the model, earned by verified compute.
- A stake is not a coin, and this PRD does not make it transferable.
- Training runs on devices people lend, including phones and browsers.
- The record of who updated a weight is append-only.
- Telivi is written in TypeScript end to end: the contributor runtime as browser code, the coordinator, weight ledger, and stake book on Node. Toolchain, test runner, repo layout, and CI are set by [ADR-0001](decisions/0001-language-and-toolchain.md).

Decisions not made yet, and not to be invented in code until a later issue chooses them:

- How a person is identified.
- How a device proves it did the training work.
- The model architecture, size, and training objective.
- How the coordinator reaches a phone or a browser.
- Whether a stake can later be transferred, split, or used for governance.

## Testing Decisions

Test the behavior a contributor or a reader can observe. Do not test the internal structure of a module.

- **Stake book.** Verified work increases the person's stake. Unverified work leaves it unchanged. Stopping lending leaves the earned stake in place. A reader can see each person's stake.
- **Weight ledger.** A first write names the author. A later update appends to the chain and the current author becomes the person who made it. An attempt to rewrite an earlier entry fails. A reader can list the chain in order.
- **Coordinator and runtime.** Use a stand-in device that returns a known training result. A completed result changes the shared model. A result that fails verification does not.

There is no existing suite. These are the first tests, and they land with the module they cover.

## Out of Scope

- Training a real model in the first implementation issues.
- Choosing the architecture or publishing weights.
- A coin, a payout, or a marketplace for stake.
- Transferring or selling a stake.
- Governance votes.
- App-store release and accounts operated by a company.
- Desktop-only training. Phone and browser are the first devices.

## Further Notes

The name is the Telugu word తెలివి (telivi), intelligence. It names the model people hold in common.

The mining analogy is about the work, not a currency. People lend a device, the device does training, and the reward is a share of the model plus their name on the weights they change.
