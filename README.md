# btc-l2-pqc-research

Verified research notes on **post-quantum migration for Bitcoin layer 2s**,
produced for
[Applied Post-Quantum Cryptography](https://appliedpqc.io/) and
[awesome-pqc](https://github.com/AppliedPQC/awesome-pqc).

Each note follows the same discipline: claims are sourced from a **primary
artifact** — the BIP text, the SDK changelog, the repository tree — rather than
from secondary coverage, and every note ends with an explicit list of what was
*not* verified.

A Bitcoin L2 has a post-quantum problem neither Bitcoin nor Ethereum has alone:
it settles to a base layer it cannot change, borrows consensus from a third
ecosystem, and operates a bridge whose trust root is cryptography of its own
choosing. The goal is never "make the L2 post-quantum safe" — it is *harden what
the L2 owns, and bound the residual risk on what it does not*.

| Note | Subject |
| --- | --- |
| [bitcoin-l2-pq-framework.md](bitcoin-l2-pq-framework.md) | The general problem: surface taxonomy, ownership, ordering, recurring traps |
| [blockchain-pqc-migration.md](blockchain-pqc-migration.md) | The base layers an L2 inherits from: Bitcoin, Ethereum, Cosmos |
| [case-study-goat.md](case-study-goat.md) | Case study 1 — the GOAT stack: `goat`, `goat-geth`, `bitvm2-node` |

GOAT is the first worked case study, not the subject. The framework note is
explicit about which findings follow from the *structure* of a Bitcoin L2 and
which are so far grounded in one implementation.

## Why primary sources

The method is not ceremony. Checking against source documents changed the
conclusions four times in these two notes:

- Cosmos ML-DSA-65 landed in SDK **v0.55**, not v0.54 as widely reported.
- BIP-360 is **not** a post-quantum signature proposal; its own text says it
  "does not include the introduction of post-quantum signature schemes".
  BIP-361 formally requires a PQ signature BIP that does not exist.
- A GitHub code search suggests BitVM uses no Winternitz signatures. It does —
  code search does not index forks, and the relevant repository is a fork.
  Listing the git tree shows the files.
- Cosmos's stated ML-DSA-65 sizes (1952-byte keys, 3309-byte signatures) were
  reproduced independently from an ACVP-verified FIPS 204 implementation.
- BitVM2's Groth16 proof is a *wrapper* over a Ziren FRI/STARK proof, not the
  proving system. That narrows the migration from "replace the proof system" to
  "drop the wrapper", and it is only visible from the dependency graph.

Four of those five would have produced a wrong or badly-scoped recommendation.

## Status

Notes are dated snapshots, not maintained documents. Standards, repositories and
roadmaps in this area move quickly; check the date at the top of each note
before relying on it.
