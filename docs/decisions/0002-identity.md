# ADR-0002: How a person is identified on the chain

**Status:** proposed
**Issue:** #3
**Date:** 2026-09-28

## Context

The PRD says every weight names its author and keeps an append-only chain of
who updated it (stories 9, 11, 15), and lists "how a person is identified"
under decisions not made yet. [ROADMAP § Phase 0](../ROADMAP.md) says this
decision blocks ledger authorship and the stake book's keys. Constraints:

- Joining does not require permission from an operator (story 3), so there is
  no central account system. Accounts operated by a company are out of scope.
- "I want my identity to be the one recorded on the chain, so that someone else
  cannot take authorship of my update" (story 15). The identity must be
  something only its owner can produce, and anyone must be able to check it.
- The chain and the stake table are public (stories 11, 13, 18). Whatever the
  identity is, it will be visible to everyone, forever.
- The contributor is a phone or a browser tab with no install (stories 1, 2),
  so the identity must be creatable and usable there.

ADR-0001 (language and toolchain) is drafted in parallel. This ADR names types
and functions language-neutrally and does not assume a platform library.

## Options

### A. Self-generated Ed25519 keypair; the public key is the identity

A contributor's device generates an Ed25519 keypair
([RFC 8032](https://www.rfc-editor.org/rfc/rfc8032)). The 32-byte public key
is the identity. The private key never leaves the device. Every chain entry
carries the author's public key and an Ed25519 signature; the ledger verifies
the signature before appending and rejects an entry that fails.

Ed25519 is part of the
[W3C Web Cryptography API](https://www.w3.org/TR/WebCryptoAPI/) (`generateKey`,
`sign`, `verify` with `{ name: "Ed25519" }`), enabled by default in Safari 17.0
including iOS, Firefox 129, and Chrome/Edge 137, about 85% of global web usage
per [caniuse](https://caniuse.com/mdn-api_subtlecrypto_sign_ed25519); Node,
Deno, and Bun ship it too
([implementation status](https://github.com/WICG/webcrypto-secure-curves/issues/20)).
No native install, no bundled crypto library. Keys are 32 bytes, signatures
64 bytes, signing is deterministic (no randomness needed at sign time), and
verification is fast enough to check every entry on read.

Costs: a lost private key is a lost identity, and there is no "forgot
password". The key is pseudonymous, not anonymous: all entries signed by one
key are linkable, which is the point of story 9 but also a privacy fact to
state. Browsers without Ed25519 (Samsung Internet as of this writing) cannot
contribute until they ship it or a later issue names a fallback.

### B. DID-style identifier (`did:key`)

Same keypair as A, but the identity string is `did:key:z6Mk...`: the
multicodec prefix `0xed01` plus the raw key, base58-btc encoded, per the
[did:key method](https://w3c-ccg.github.io/did-key-spec/) and
[DID Core](https://www.w3.org/TR/did-core/).

What it adds over a bare key: a self-describing string that names its key type,
and interoperability with DID and Verifiable Credential tooling. Resolution is
offline and pure, so it adds no registry or network dependency.

What it costs: multicodec and base58-btc encoding (a dependency or hand-rolled
code), and DID Document machinery Telivi has no use for. It buys no property
that A lacks: the did:key spec itself states the method supports neither key
rotation nor deactivation, because the identifier is the key. It is a display
format for A, not a different identity.

### C. Opaque string assigned by the coordinator

The coordinator hands each new contributor a string and records it as the
author on entries it accepts.

Rejected. It fails story 3: the coordinator becomes the operator whose
permission is needed to join, the account system the PRD puts out of scope. It
fails story 15: anything the coordinator can assign it can also reassign or
forge, so nothing about the identity is something only its owner can produce.
It also makes the coordinator, which the PRD says does not own the weights, the
sole authority on who wrote them. A test double may look like this option, but
only behind the interface below and only in tests.

## Recommendation

Option A. An identity is a self-generated Ed25519 public key; every chain
entry is signed by its author's private key and the ledger verifies the
signature before appending. This is the only option that satisfies stories 3
and 15 at once, runs in a browser tab and on a phone with no install, and
adds no dependency where the platform ships Ed25519. B is a later, optional
display encoding of A, not a competing choice.

What the ledger stores per chain entry:

- `position`: the entry's index in that weight's chain, starting at 0.
- `author`: the 32-byte public key.
- `update`: the update bytes. Their meaning is not decided here.
- `signature`: 64 bytes, over the canonical bytes of
  (domain tag, weight id, `position`, `update`). Binding the weight id and
  position means a valid entry cannot be replayed onto another weight or
  another slot. The domain tag keeps a ledger signature from being valid
  anywhere else. The exact serialization is fixed by the first ledger issue.

An entry whose signature fails to verify is not an update by anyone and is not
appended. Story 14 (a bad update stays visible) is about the content of a
genuinely authored update, not a forged author.

What Phase 1 needs, language-neutrally:

- An `Identity` value: the public key bytes, compared by value, with a
  canonical string form for display and table keys chosen by the ledger issue.
- A `Signer` holding a private key: `identity(signer) -> Identity`,
  `sign(signer, bytes) -> Signature`.
- `verify(identity, bytes, signature) -> bool`.
- Two implementations behind that interface. A stand-in scheme where an
  `Identity` is any label (for example `"alice"`), `sign` returns a value
  derived from the label, and `verify` checks it trivially, so ledger and
  stake-book tests need no real crypto. And the real Ed25519 scheme on the
  platform's built-in crypto (WebCrypto in browsers, Node, Deno, Bun; the
  equivalent in what ADR-0001 picks), tested against RFC 8032 § 7.1 vectors.

The stake book keys its table by `Identity`. Stake belongs to a key. Two keys
are two rows; merging them would be a transfer, which the PRD does not allow.

Recovery and rotation are deferred to a later ADR. Any rule needs either a
second key held elsewhere, a social scheme, or a coordinator override; the last
fails story 15 and the others belong with governance in Phase 3. Until then: a
lost key is a lost identity. Entries and stake already recorded under it stay
valid forever, because verification uses the key stored in the entry (story 8).
The ledger must not assume one key per person, so a future signed
"successor key" statement stays possible without rewriting history.

Privacy: a key is a pseudonym. The ledger and stake book store no names,
emails, phone numbers, or device identifiers, and no issue may add them. Any
mapping from key to a display name is off-chain, optional, and out of scope.

## Consequences

- Phase 1 ledger and stake book issues can start; both take an `Identity`
  through the interface above and are tested with the stand-in scheme.
- Next issues: the identity module (interface plus stand-in); the Ed25519
  scheme on the platform's crypto; in Phase 2, generating and storing the key
  in the browser tab (decided there, not here); a later ADR for rotation and
  recovery.
- Harder: no account recovery; browsers without Ed25519 cannot contribute
  yet; the ledger issue must fix a string form for `Identity`, and `did:key`
  is a candidate if interoperability is ever wanted.
- Not decided here: how a device proves it did the work (ADR-0003), how the
  coordinator reaches a device, the model, or stake transfer.
