# The GOAT stack: a post-quantum migration analysis

*Analysis of [`GOATNetwork/goat`](https://github.com/GOATNetwork/goat),
[`goat-geth`](https://github.com/GOATNetwork/goat-geth) and
[`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node), read on
2026-07-31. Every architectural claim is read from the repositories via the
GitHub API, not inferred from documentation.*

## 1. Summary of findings

Four surfaces carry quantum-exposed cryptography, and they are not equally
GOAT's to fix.

| # | Surface | Repo | Scheme | Quantum status | Owner of the fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Bridge attestation | `goat` `x/relayer` | secp256k1 / Schnorr | broken by Shor | **GOAT** |
| 2 | Consensus keys | `goat` (CometBFT) | ed25519 | broken by Shor | **GOAT** |
| 3 | Bridge proof system | `bitvm2-node` | **Groth16 over BN254** | broken by Shor | **GOAT** |
| 3b | Bridge bit commitments | `bitvm2-node` → BitVM | **Winternitz OTS** | **already PQ-safe** | — |
| 4 | Peg-out signatures | `bitvm2-node` | MuSig2 / secp256k1 | broken by Shor | **GOAT** |
| 5 | EVM accounts | `goat-geth` | secp256k1 ECDSA | broken by Shor | GOAT, at tooling cost |
| 6 | Bitcoin L1 outputs | — | secp256k1 / Schnorr | broken by Shor | **Bitcoin, not GOAT** |

The headline is in row 3/3b. **BitVM2's Bitcoin-side plumbing is already
post-quantum; its cryptographic content is not.** That reframes the work from a
rewrite into two contained swaps.

## 2. bitvm2-node: the plumbing is hash-based, the content is not

`bitvm2-node` is a Rust ZK bridge: crates for `bitcoin-light-client-circuit`,
`header-chain`, `state-chain`, `commit-chain`, `bitvm2-ga`, with circuits for
header-chain, state-chain, commit-chain and operator proofs. Its crypto
dependencies are explicit in `Cargo.toml`:

```toml
ark-bn254   = "0.5.0"   # pairing-friendly curve
ark-groth16 = "0.5.0"   # pairing-based SNARK
musig2      = "0.1.0"
secp256k1   = "0.29.1"
sha2        = "0.10.9"
bitvm       = { git = "https://github.com/GOATNetwork/BitVM.git", branch = "GA" }
```

Occurrence counts in the repository: `groth16` 24, `bn254` 15, `musig2` 11.

**What is broken.** Groth16 over BN254 is a *pairing-based* proof system.
Pairing-friendly curves fall to Shor exactly as ECDSA does, so a
cryptographically relevant quantum computer could forge bridge proofs, not
merely steal keys. That is a more severe failure mode than key theft: it
compromises the bridge's soundness rather than an individual's funds. MuSig2
over secp256k1, used for peg-out authorisation, is broken in the ordinary way.

**What is already safe, and this is the useful part.** BitVM2 carries values
between Bitcoin script fragments using *bit commitments* implemented as
Winternitz one-time signatures — hash-based, and therefore post-quantum on the
same conservative assumption as SLH-DSA. The BitVM tree contains
`bitvm/src/signatures/winternitz.rs`, `winternitz_hash.rs`,
`signing_winternitz.rs` and `wots_api.rs`, and `bitvm2-node` references them
from `crates/bitvm2-ga/src/types.rs` and `crates/bitvm2-ga/src/operator/api.rs`.

> **Verification note.** A GitHub *code search* for `winternitz` in
> `GOATNetwork/BitVM` returns zero hits, which would suggest the opposite
> conclusion. That is an artifact: GitHub code search does not index forks, and
> `GOATNetwork/BitVM` is a fork of `BitVM/BitVM`. Listing the git tree directly
> shows the files. Any audit relying on code search over this stack will
> silently under-report.

**Consequence.** GOAT's BitVM2 migration is a *proof-system swap*, not a rewrite
of the Bitcoin-side machinery. Replacing Groth16/BN254 with a hash-based proof
system — a STARK, or the kind of hash-based zkVM Ethereum is building as
`leanVM` — leaves the Winternitz commitment layer intact, because that layer was
never the quantum-vulnerable part. The Bitcoin script verifier changes; the
mechanism for getting bits into it does not.

The cost is size and script weight. Hash-based proofs are substantially larger
than Groth16's constant-size proof, and BitVM2's economics depend on what fits
in Bitcoin script and what a challenge transaction costs. That trade is the real
engineering question, and it should be measured before it is decided.

## 3. goat-geth: the divergence, measured

The fork question flagged in the earlier pass now has an answer. Comparing
`GOATNetwork:goat-geth:dev` against `ethereum/go-ethereum:master`:

```
status = diverged
ahead_by  =  37     (GOAT's own commits)
behind_by = 377     (upstream commits not merged)
files changed = 78
```

Last push was 2026-03-31, roughly four months stale relative to the main `goat`
repository. Thirty-seven custom commits over seventy-eight files is a tractable
amount of divergence — this is a customised fork, not a rewrite. But 377 commits
behind is the number that matters for planning: any post-quantum work on the
execution layer inherits an upstream catch-up first, and that catch-up will only
grow.

Recommendation for this surface remains *follow, do not lead*. Ethereum's
account-abstraction route to post-quantum accounts has not shipped; diverging
account semantics ahead of it would break wallet and tooling compatibility,
which for an L2 is usually the product. The actionable item here is not
cryptographic: **reduce the 377-commit gap so the fork can absorb upstream PQ
work when it lands.**

## 4. goat: relayer and consensus

Unchanged from the first pass, and still where the best return is.

**Relayer attestation keys** (`x/relayer/types/pubkey.go`) are the bridge trust
root, held as a protobuf `oneof` over `Secp256K1 | Schnorr`. The `oneof` is an
extensibility point, so adding an ML-DSA-65 variant extends an existing pattern.
The blocker is a fixed-length gate in `VerifySign`:

```go
// note: msg is 32 bytes, sig is 64 bytes
if len(msg) != sha256.Size || len(sig) != schnorr.SignatureSize {
    return false
}
```

An ML-DSA-65 signature is 3309 bytes and its public key 1952 bytes — both
confirmed against an independent ACVP-verified FIPS 204 implementation. Until
that check is per-variant, the function is structurally incapable of accepting a
post-quantum signature.

**Consensus keys.** Cosmos SDK v0.55 registers ML-DSA-65 as a validator
consensus key type, opt-in behind
`genesis.consensus_params.validator.pub_key_types`, with validator key rotation
shipping in the same release. GOAT runs SDK **v0.53.8** via a fork
(`goat-cosmos-sdk`), so this is a fork rebase across two minors, not a version
bump — the rebase is the critical path.

`ibc-go` does not appear in `goat`'s `go.mod`, so the IBC light-client hazard
that dominates Cosmos post-quantum migration — enabling a key type counterparties
cannot verify stops packet flow and expires the client — appears not to apply.
Worth confirming against deployment reality rather than `go.mod` alone.

## 5. Recommended sequencing

| Phase | Action | Why here |
| --- | --- | --- |
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | The `VerifySign` 64-byte gate shows these assumptions are load-bearing and invisible |
| 1 | Relayer: add ML-DSA-65 to the `PublicKey` `oneof`, make length checks per-variant, roll out with dual attestation | Highest value per unit of control; the `oneof` already supports it; dual signing gives rollback at every step |
| 2 | BitVM2: prototype a hash-based proof system to replace Groth16/BN254, and measure script weight and challenge cost | The Winternitz layer already survives; this is the one change that fixes bridge *soundness* rather than key theft |
| 3 | Rebase `goat-cosmos-sdk` onto SDK ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | Mechanism is upstream and gentle, but gated on the rebase |
| 4 | Reduce `goat-geth`'s 377-commit lag; track Ethereum account abstraction | Following costs less than diverging; the lag is the real blocker |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

## 6. The honest limit

Bitcoin has specified **no post-quantum signature scheme**. BIP-360 (Draft)
removes Taproot's key-path spend and says in its own text that it "does not
include the introduction of post-quantum signature schemes"; BIP-361 (Draft,
Informational) lists `Requires: TBD Post Quantum Signature BIP`, a document that
does not exist. Any BTC secured by a classical Bitcoin L1 output is therefore on
Bitcoin's timeline, not GOAT's.

What GOAT can do is bound the exposure: avoid address reuse, prefer outputs that
do not commit a bare public key, avoid long-lived outputs with revealed keys,
and write custody policy so the script and key policy can migrate when Bitcoin
gains PQ signatures.

## 7. What was not verified

Stated so this is not read as more thorough than it is.

- `x/locking`, `x/bitcoin` and `x/consensusfork` were not audited for further
  signature surfaces.
- The `goat-cosmos-sdk` fork was not inspected; its divergence determines the
  phase-3 rebase cost and is the largest remaining unknown.
- The BLS12-381 usage in `goat` (2 files, via `gnark-crypto`) was not traced. If
  it is load-bearing for aggregation it is another Shor-exposed surface.
- The 37 custom `goat-geth` commits were not read individually, so whether any
  touch cryptography is unknown.
- The IBC conclusion rests on `go.mod` alone.
