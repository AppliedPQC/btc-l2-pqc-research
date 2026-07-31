# btc-l2-pqc-research

A research report on **post-quantum migration for Bitcoin layer 2s**, produced
for [Applied Post-Quantum Cryptography](https://appliedpqc.io/) and
[awesome-pqc](https://github.com/AppliedPQC/awesome-pqc).

**[Read the report →](bitcoin-l2-pq-migration.md)**

A Bitcoin L2 settles to a base layer it cannot change, borrows consensus from a
third ecosystem, and runs a bridge whose trust root is cryptography of its own
choosing. The goal is never "make the L2 post-quantum safe" — it is to harden
what the L2 owns and bound the residual risk on what it does not.

| Part | Contents |
| --- | --- |
| I | The general problem: why an L2 differs, and a surface taxonomy keyed on who can fix each row |
| II | The base layers: Bitcoin, Ethereum, Cosmos |
| III | The evidence: the GOAT stack — `goat`, `goat-geth`, `bitvm2-node`, `bitvm2-gc` |
| IV | Ordering the work, and the honest limit |
| V | Method, recurring traps, verification log, open items |

The general part and the case study are one document on purpose: the general
claims were derived from reading one stack, and separating them would imply
independent grounding they do not have.

## Method

Claims come from **primary artifacts** — BIP text, the SDK changelog, issue
trackers, repository trees — not from secondary coverage. Part V logs what was
checked and, where a conclusion changed, what it changed from.

Six findings would have been wrong without that discipline:

- Cosmos ML-DSA-65 landed in SDK **v0.55**, not v0.54 as widely reported.
- BIP-360 is **not** a post-quantum signature proposal; its own text says so,
  and BIP-361 requires a PQ signature BIP that does not exist.
- BitVM's Winternitz signatures are invisible to GitHub code search, which does
  not index forks. The files are plainly in the tree.
- Ziren's STARK is **not** post-quantum: its multiset memory check is
  hash-to-curve and rests on DLOG (Ziren #276).
- GOAT runs **no** `cosmos-sdk` fork — the `replace` is commented out — so the
  consensus upgrade is far cheaper than first assessed.
- The relayer's BLS12-381 **vote key** is a third key type, and the hardest
  item in the report, because its value is aggregation.

## Status

A dated snapshot, not a maintained document. Standards, repositories and
roadmaps here move quickly; check the date at the top of the report.
