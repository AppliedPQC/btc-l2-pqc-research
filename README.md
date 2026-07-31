# pqc-research

Verified research notes on post-quantum migration, produced for
[Applied Post-Quantum Cryptography](https://appliedpqc.io/) and
[awesome-pqc](https://github.com/AppliedPQC/awesome-pqc).

Each note follows the same discipline: claims are sourced from a **primary
artifact** — the BIP text, the SDK changelog, the repository tree — rather than
from secondary coverage, and every note ends with an explicit list of what was
*not* verified.

| Note | Subject |
| --- | --- |
| [blockchain-pqc-migration.md](blockchain-pqc-migration.md) | How Bitcoin, Ethereum and Cosmos plan their post-quantum upgrades |
| [goat-stack-pq-analysis.md](goat-stack-pq-analysis.md) | Post-quantum migration analysis of the GOAT stack: `goat`, `goat-geth`, `bitvm2-node` |

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

Three of those four would have produced a wrong recommendation.

## Status

Notes are dated snapshots, not maintained documents. Standards, repositories and
roadmaps in this area move quickly; check the date at the top of each note
before relying on it.
