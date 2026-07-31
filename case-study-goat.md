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
| 3 | Bridge proof system | `bitvm2-node` | Ziren STARK, **wrapped in Groth16/BN254** | wrapper broken by Shor | **GOAT** |
| 3b | Bridge bit commitments | `bitvm2-node` → BitVM | **Winternitz OTS** | **already PQ-safe** | — |
| 4 | Peg custody | `bitvm2-node` | MuSig2 over a **Taproot key path** | broken by Shor, **and exposed from output creation** | **GOAT** |
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

**Groth16 is a wrapper, not the proving system.** `bitvm2-node` depends on
Ziren (`zkm-prover`, `zkm-verifier`, `zkm-sdk`, `zkm-zkvm`, and four more
crates), and Ziren is a FRI/STARK zkVM — its whitepaper mentions FRI 31 times
and STARK 19, over Poseidon and a small prime field. Occurrence counts in
`bitvm2-node` are consistent with the standard pattern: `zkm` 93, `wrap` 87,
`stark` 22, `groth16` 24. The types confirm what the Groth16 proof is *for*:

```rust
pub type Groth16Proof = ark_groth16::Proof<ark_bn254::Bn254>;
pub type OperatorWotsSignatures = (GuestPubinSignatures, Groth16ProofSignatures);
```

The Groth16 proof elements are committed into Bitcoin script through Winternitz
signatures. So the pipeline is: **Ziren produces a hash-based STARK proof, that
proof is wrapped into a constant-size Groth16 proof over BN254, and the Groth16
verifier is what BitVM2 runs in Bitcoin script.**

**Consequence, and it is a good one.** The quantum-vulnerable component is the
*final wrapper*, not the proving stack. Ziren's STARK core is already
post-quantum, and the Winternitz commitment layer is already post-quantum. The
wrapper exists solely to compress the proof enough to be affordable on Bitcoin.

The migration is therefore narrower than "replace the proof system": it is
*remove the Groth16 wrap and verify the STARK directly in Bitcoin script*. The
cost is script weight — a FRI proof is far larger than Groth16's constant few
hundred bytes. BitVM2's architecture is unusually well suited to absorbing that,
because the verifier only executes during a dispute, never on the happy path.
Whether the resulting challenge transaction is economically viable is the real
question, and it is measurable today with the existing stack.

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


## 3a. The `gc-v2` branch and BitVM3 garbling: orthogonal to post-quantum

`bitvm2-node`'s `gc-v2` branch (head `73cdb49`, 2026-07-22) and the companion
repository [`GOATNetwork/bitvm2-gc`](https://github.com/GOATNetwork/bitvm2-gc)
move BitVM2 toward a garbled-circuit design: rather than compiling the verifier
into raw Bitcoin script, the verifier circuit is *garbled*, and the dispute
protocol evaluates the garbling. `bitvm2-gc`'s workspace is built around a crate
named exactly for this — `garbled-snark-verifier` — alongside Bristol-format
circuits.

Because garbled circuits are built from symmetric primitives, it is tempting to
read this as a post-quantum improvement. It is not, and the reason is the
distinction this note keeps returning to: **garbling changes how the verifier is
executed, not what it asserts.**

`garbled-snark-verifier/Cargo.toml` depends on `ark-groth16`, `ark-bn254`,
`ark-ec` and `ark-relations` *and* on `aes` and `blake3`. The arkworks
dependencies are the circuit being garbled; the AES and BLAKE3 dependencies are
the garbling machinery. The circuit tree confirms the shape — `circuits/groth16.rs`,
`circuits/bn254/`, `circuits/dv_snark.rs`, `dv_bn254/` (designated-verifier
variants), and `circuits/sect233k1/`.

Three consequences.

**The garbled statement is still elliptic-curve.** Whether the garbled circuit
verifies Groth16 over BN254 or a designated-verifier SNARK, soundness rests on
pairing or discrete-log assumptions that Shor breaks. `gc-v2` retains
`ark-groth16` and `ark-bn254` (22 files reference `groth16`), so the conclusion
of section 2 is unchanged by the garbling work.

**`sect233k1` is a curve too, and a smaller one.** The `sect233k1` circuit family
is a binary-field Koblitz curve, presumably chosen because binary-field
arithmetic is XOR-heavy and therefore cheap under free-XOR garbling — a sensible
engineering choice. It is still an elliptic curve, so Shor applies, and at a
233-bit binary field it sits around 112-bit classical security, below
secp256k1. Garbling efficiency and quantum resistance are pulling in different
directions here, and it is worth being explicit that the choice optimised the
first.

**Garbling adds a security parameter that did not previously exist.**
`LABEL_SIZE = 16` and the PRF is `Aes128` (`garbled-snark-verifier/src/core/utils.rs`,
`verifiable-circuit-babe/src/gc/utils.rs`). 128-bit labels under AES-128 put the
garbling layer at NIST post-quantum security category 1 — the floor of the
approved range, since Grover search over a 128-bit space is the reference for
that category. That is a defensible choice and not a vulnerability, but it is a
*decision*, and for a bridge holding long-lived commitments it deserves to be
made explicitly rather than inherited from a default. Moving labels and PRF to
256-bit restores category-5 margin at roughly double the garbling cost.

**Net effect on the post-quantum position: none.** `gc-v2` still uses MuSig2
(8 files) and Taproot (9 files), so section 4a's key-path exposure is unchanged,
and it still wraps in Groth16, so section 2's conclusion is unchanged. The
garbling work is valuable for other reasons; it should simply not be counted as
progress toward quantum resistance. The one thing it does add to this analysis is
the label-size decision above.

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


## 4a. The peg's weakest quantum link is the Taproot key path

`crates/bitvm2-ga/src/committee/api.rs` shows the peg custody model directly:

```rust
pub type CommitteeSignatures = CommitteeMusig2Data<TaprootSignature>;
...
let n_of_n_taproot_public_key = ...;
let connector_0 = Connector0::new(network, &n_of_n_taproot_public_key);
```

The committee aggregates MuSig2 partial signatures into a Taproot signature over
an **n-of-n aggregated Taproot public key**, used by the BitVM2 connector
outputs. MuSig2 producing a `TaprootSignature` over an aggregate key is the
canonical **key-path** spend pattern; a script-path spend would not need
aggregation. Script paths are present too — `TaprootBuilder::new().add_leaf(...)`
with an internal key appears in the light-client crate — so the design uses
both.

**Why this is the sharpest exposure in the stack.** A Taproot output commits its
output key in the `scriptPubKey` at the moment the output is *created*, not when
it is spent. So the peg's aggregated public key is on chain, in the clear, for
the entire lifetime of the UTXO. An adversary with a cryptographically relevant
quantum computer recovers the corresponding secret from that public key and
spends via the key path — **without forging a proof, without compromising any
committee member, and without waiting for a spend to reveal anything.** It is
strictly easier than attacking the Groth16 wrapper, and it is the textbook
long-exposure case that BIP-360 exists to remove.

**There is a mitigation available today, with no post-quantum scheme required.**
Taproot outputs may commit a provably unspendable NUMS point as the internal
key, which disables the key path and leaves only script paths. That is precisely
the construction BIP-360 proposes to standardise as P2MR, and it can be adopted
unilaterally now. The cost is real and must be weighed: losing the key path
means losing the cheap, private cooperative spend, so every peg movement becomes
a script-path spend with its larger witness. But it converts the peg from
long-exposure vulnerable to long-exposure safe, which no other change in this
document achieves without waiting on Bitcoin.

This should be evaluated before the proof-system work. It is cheaper, it is
available immediately, and it closes the easier of the two attacks.

## 5. Recommended sequencing

| Phase | Action | Why here |
| --- | --- | --- |
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | The `VerifySign` 64-byte gate shows these assumptions are load-bearing and invisible |
| 1 | Relayer: add ML-DSA-65 to the `PublicKey` `oneof`, make length checks per-variant, roll out with dual attestation | Highest value per unit of control; the `oneof` already supports it; dual signing gives rollback at every step |
| 2 | Peg custody: evaluate NUMS-internal-key (script-path-only) Taproot outputs | Cheapest real gain, available today, no PQ scheme needed; closes the easiest attack |
| 3 | BitVM2: drop the Groth16 wrap and verify the Ziren STARK directly in script; measure script weight and challenge cost | Ziren's core and the Winternitz layer are already PQ-safe; only the wrapper is not. Fixes *soundness*, not just theft |
| 4 | Rebase `goat-cosmos-sdk` onto SDK ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | Mechanism is upstream and gentle, but gated on the rebase |
| 5 | Reduce `goat-geth`'s 377-commit lag; track Ethereum account abstraction | Following costs less than diverging; the lag is the real blocker |
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
  phase-4 rebase cost and is the largest remaining unknown.
- The BLS12-381 usage in `goat` (2 files, via `gnark-crypto`) was not traced. If
  it is load-bearing for aggregation it is another Shor-exposed surface.
- The 37 custom `goat-geth` commits were not read individually, so whether any
  touch cryptography is unknown.
- In `bitvm2-gc`, the `verifiable-circuit*` crates and the Bristol circuits were
  not read; only the workspace manifests, the garbling core and the circuit tree
  were. Whether the designated-verifier variants change the trust model is not
  assessed here.
- The IBC conclusion rests on `go.mod` alone.
