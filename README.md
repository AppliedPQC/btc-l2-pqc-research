# btc-l2-pqc-research

A research report on **post-quantum migration for Bitcoin layer 2s**, produced
for [Applied Post-Quantum Cryptography](https://appliedpqc.io/) and
[awesome-pqc](https://github.com/AppliedPQC/awesome-pqc).

**[Read the report →](bitcoin-l2-pq-migration.md)**

A Bitcoin L2 settles to a base layer it cannot change, borrows consensus from a
third ecosystem, and runs a bridge whose trust root is cryptography of its own
choosing. The goal is never "make the L2 post-quantum safe" — it is to harden
what the L2 owns and bound the residual risk on what it does not.

The general part and the case study are one document on purpose: the general
claims were derived from reading one stack, and separating them would imply
independent grounding they do not have.

## Contents

**[Part I — The general problem](bitcoin-l2-pq-migration.md#part-i--the-general-problem)**

1. [Why a Bitcoin L2 is a distinct problem](bitcoin-l2-pq-migration.md#1-why-a-bitcoin-l2-is-a-distinct-problem) — it cannot fix its base layer, borrows consensus, and owns a bridge
2. [The surface taxonomy](bitcoin-l2-pq-migration.md#2-the-surface-taxonomy) — seven surfaces, keyed on who holds the fix

**[Part II — The base layers an L2 inherits from](bitcoin-l2-pq-migration.md#part-ii--the-base-layers-an-l2-inherits-from)**

3. [Taxonomy: what problem is each chain actually solving?](bitcoin-l2-pq-migration.md#3-taxonomy-what-problem-is-each-chain-actually-solving)
4. [Bitcoin: an exposure problem, with the signature question deferred](bitcoin-l2-pq-migration.md#4-bitcoin-an-exposure-problem-with-the-signature-question-deferred) — BIP-360, BIP-361, and the sunset's rescue protocol
5. [Ethereum: an aggregation problem at consensus, an account problem above it](bitcoin-l2-pq-migration.md#5-ethereum-an-aggregation-problem-at-consensus-an-account-problem-above-it) — `leanVM`, `leanSig`, EIP-7702/8141/7885
6. [Cosmos: a negotiation problem, and the one that shipped](bitcoin-l2-pq-migration.md#6-cosmos-a-negotiation-problem-and-the-one-that-shipped) — ML-DSA-65 in SDK v0.55, the IBC hazard, and why commits scale linearly

**[Part III — The evidence: the GOAT stack](bitcoin-l2-pq-migration.md#part-iii--the-evidence-the-goat-stack)**

7. [Summary of findings](bitcoin-l2-pq-migration.md#7-summary-of-findings) — six of seven surfaces carry quantum-exposed cryptography
8. [`bitvm2-node`: the plumbing is hash-based, the content is not](bitcoin-l2-pq-migration.md#8-bitvm2-node-the-plumbing-is-hash-based-the-content-is-not) — Winternitz commitments carrying a Groth16 proof, and Ziren's DLOG memory check
9. [The on-chain Groth16 verifier: script, garbling, witness encryption](bitcoin-l2-pq-migration.md#9-the-on-chain-groth16-verifier-script-garbling-witness-encryption) — four designs, one unchanged BN254 assumption, stated in BABE's own security theorem
10. [The peg's weakest quantum link is the Taproot key path](bitcoin-l2-pq-migration.md#10-the-pegs-weakest-quantum-link-is-the-taproot-key-path) — and the NUMS mitigation available today
11. [`goat`: relayer and consensus](bitcoin-l2-pq-migration.md#11-goat-relayer-and-consensus) — the 64-byte gate in `VerifySign`, and the SDK upgrade path
12. [The relayer's BLS vote key](bitcoin-l2-pq-migration.md#12-the-relayers-bls-vote-key) — the hardest item, because its value is aggregation
13. [`goat-geth`: the divergence, measured](bitcoin-l2-pq-migration.md#13-goat-geth-the-divergence-measured) — 377 commits behind, and the precompiles nobody can migrate

**[Part IV — What to do](bitcoin-l2-pq-migration.md#part-iv--what-to-do)**

14. [Ordering the work](bitcoin-l2-pq-migration.md#14-ordering-the-work) — by ownership and severity, with a seven-phase plan for GOAT

**[Part V — Traps and open items](bitcoin-l2-pq-migration.md#part-v--traps-and-open-items)**

15. [Recurring traps](bitcoin-l2-pq-migration.md#15-recurring-traps) — fixed-length checks, forks hiding blockers, and the wrapper trap in both directions

Also: [Abstract](bitcoin-l2-pq-migration.md#abstract) ·
[Remaining open items](bitcoin-l2-pq-migration.md#remaining-open-items) ·
[References](bitcoin-l2-pq-migration.md#references)

## Method

Claims come from **primary artifacts** — BIP text, the SDK changelog, issue
trackers, repository trees, and the design documents of the schemes themselves —
not from secondary coverage.

These findings would have been wrong without that discipline:

- Cosmos ML-DSA-65 landed in SDK **v0.55**, not v0.54 as widely reported.
- BIP-360 is **not** a post-quantum signature proposal; its own text says so,
  and BIP-361 requires a PQ signature BIP that does not exist.
- BitVM's Winternitz signatures are invisible to GitHub code search, which does
  not index forks. The files are plainly in the tree.
- Ziren's STARK is **not** post-quantum today: its multiset memory check is
  hash-to-curve and rests on DLOG (Ziren #276) — with a working LtHash
  prototype that makes it the tractable layer of the three.
- GOAT runs **no** `cosmos-sdk` fork — the `replace` is commented out — so the
  consensus upgrade is far cheaper than first assessed.
- The relayer's BLS12-381 **vote key** is a third key type, and the hardest
  item in the report, because its value is aggregation — confirmed in use at a
  call site, not merely available as an API.
- **`leanVM` cannot rescue BLS.** It holds no pairing code at all, and proving a
  BLS verification inside a hash-based zkVM would prove a true statement about a
  broken primitive. It removes the *need* for BLS rather than fixing it.
- **Witness encryption does not make the bridge post-quantum.** BABE moves the
  Groth16 verifier off chain and leaves an on-chain surface that is entirely
  hash-based — and GOAT's own security analysis still reduces it to discrete log
  on BN254. The pairing survives in the relation being encrypted against.

## Status

A dated snapshot, not a maintained document. Standards, repositories and
roadmaps here move quickly; check the date at the top of the report.

## Layout

```text
bitcoin-l2-pq-migration.md   the report
```
