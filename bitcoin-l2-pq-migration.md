# Post-quantum migration for Bitcoin layer 2s

*Compiled 2026-07-31 from primary sources: BIP text from `bitcoin/bips`,
`UPGRADING.md` from `cosmos/cosmos-sdk`, and the GOAT repositories and their
dependency trees read through the GitHub API. Corrections contributed by the
GOAT team are marked where they changed a conclusion. Section 9 lists what was
not verified.*

This note does two things at once, and deliberately keeps them together. Part I
states the general shape of the post-quantum problem for a Bitcoin L2. Part II
is the evidence it was derived from: a reading of the GOAT stack — `goat`,
`goat-geth`, `bitvm2-node`, `bitvm2-gc`. The general claims are not independent
of the case study, and separating them into two documents would have implied
otherwise.

---

# Part I — The general problem

## 1. Why a Bitcoin L2 is a distinct problem

Most post-quantum blockchain analysis treats a chain as a single cryptographic
system: one signature scheme securing funds, one consensus mechanism, one
governance process to change them. A Bitcoin L2 is not that. It is a composition
of systems that were designed separately, migrate on separate timelines, and are
owned by separate parties.

That composition produces a problem neither Bitcoin nor Ethereum has alone:

- The L2 **cannot fix its own base layer.** Bitcoin has specified no
  post-quantum signature scheme at all — BIP-360 (Draft) says in its own text
  that it "does not include the introduction of post-quantum signature schemes",
  and BIP-361 (Draft, Informational) lists `Requires: TBD Post Quantum Signature
  BIP`, a document that does not exist. Any BTC in a classical L1 output is on
  Bitcoin's timeline, not the L2's.
- The L2 **usually borrows a consensus stack from a third ecosystem**, and
  inherits that ecosystem's migration path, maturity and constraints.
- The L2 **owns a bridge**, whose trust root is cryptography of its own choosing
  — and which is the highest-value target in the system.

So the honest goal is never "make the L2 post-quantum safe". It is *harden the
surfaces the L2 owns, and bound the residual risk on the ones it does not*.

## 2. The surface taxonomy

Any Bitcoin L2 that settles to L1 and runs its own consensus has, at minimum,
these cryptographic surfaces. The taxonomy is the useful artifact: it makes the
ownership question explicit before the scheme-selection question.

| # | Surface | Typical scheme | Fails to Shor? | Who can fix it |
| --- | --- | --- | --- | --- |
| 1 | L1 settlement outputs | secp256k1 / Schnorr | yes | **base layer only** |
| 2 | Bridge attestation / operator set | secp256k1, Schnorr, MuSig2, BLS | yes | **the L2** |
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | **depends, and check inside** | **the L2**, often with upstream |
| 4 | Bridge data commitments | Merkle / Winternitz / Lamport | **no — hash-based** | already safe |
| 5 | L2 consensus keys | ed25519, BLS12-381 | yes | **the L2**, via its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | the L2, at tooling cost |
| 7 | **EVM crypto precompiles** | ecrecover, BN254 pairing, BLS12-381, KZG | yes | **upstream, and unfixable for deployed callers** |

Two rows carry most of the signal.

**Row 3 is where severity is highest.** A broken *signature* scheme lets an
attacker steal specific funds. A broken *proof system* lets an attacker forge
the bridge's notion of truth — minting without deposit, or withdrawing without
right. Pairing-based systems (Groth16, PLONK over BN254/BLS12-381) fall to Shor exactly
as ECDSA does. **"STARK" is not by itself an answer, though.** A zkVM's headline
commitment scheme can be hash-based while an auxiliary argument inside it is
not: in the stack examined here, the FRI commitment is hash-based but the
offline memory-consistency check commits memory values as elliptic-curve points
and its soundness rests explicitly on DLOG hardness. A quantum adversary attacks
the memory argument, not FRI. **The question to ask is never "is it a STARK"
but "which assumptions does every argument in this proof rely on" — and the
answer is often only visible in the project's own issue tracker.**

**Row 4 is the good news, and it is frequently missed.** Bridge constructions
that commit data via Merkle trees, or that carry values through Bitcoin script
using Winternitz/Lamport one-time signatures, are already post-quantum on the
most conservative assumption available — the same assumption behind SLH-DSA. In
the one stack examined closely, this turned out to be true of the entire
Bitcoin-side commitment layer, which reframed the migration from a rewrite into
a contained swap.

---

# Part II — The evidence: the GOAT stack

## 3. Summary of findings

Four surfaces carry quantum-exposed cryptography, and they are not equally
GOAT's to fix.

| # | Surface | Repo | Scheme | Quantum status | Owner of the fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Bridge attestation | `goat` `x/relayer` | secp256k1 / Schnorr | broken by Shor | **GOAT** |
| 2 | Consensus keys | `goat` (CometBFT) | ed25519 | broken by Shor | **GOAT** |
| 3 | Bridge proof system | `bitvm2-node` | Ziren STARK (**DLOG multiset memory check**) → **Groth16/BN254** wrap → garbled | broken at three layers | **GOAT + upstream Ziren** |
| 3b | Bridge bit commitments | `bitvm2-node` → BitVM | **Winternitz OTS** | **already PQ-safe** | — |
| 4 | Peg custody | `bitvm2-node` | MuSig2 over a **Taproot key path** | broken by Shor, **and exposed from output creation** | **GOAT** |
| 5 | EVM accounts | `goat-geth` | secp256k1 ECDSA | broken by Shor | GOAT, at tooling cost |
| 6 | Bitcoin L1 outputs | — | secp256k1 / Schnorr | broken by Shor | **Bitcoin, not GOAT** |

The headline is in row 3/3b. **BitVM2's Bitcoin-side plumbing is already
post-quantum; its cryptographic content is not.** That reframes the work from a
rewrite into two contained swaps.

## 4. bitvm2-node: the plumbing is hash-based, the content is not

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

**But the STARK core is not post-quantum either.** An earlier draft of this note
claimed it was, on the reasoning that FRI and Poseidon are hash-based. That is
wrong, and the counter-example is upstream and documented. Ziren issue
[#276](https://github.com/ProjectZKM/Ziren/issues/276), *"Replace hash-to-curve
in multiset hash by quantum safe primitives"*, open since 2025-08-14, states the
problem in its own words:

> In our multiset hash, we rely on hash-to-curve to calculate the hash of values
> of the memory addresses, and consider each hash as the x of a point, then
> commit the accumulation of all the points in the EC subgroup. **If DLOG
> hardness is preserved**, prover can not forge another set of scalars for all
> the points respectively, and hence can not forge the proof of the offline
> memory checking. […] To achieve quantum safe, we need to remove the DLOG
> hardness assumption.

So the offline memory-consistency argument — not the polynomial commitment —
rests on discrete log. A quantum adversary does not need to attack FRI; it
forges the memory check and with it the execution proof. The candidate
replacement is a lattice-based homomorphic multiset hash such as LtHash, which
would remove the DLOG assumption. **This is upstream work in Ziren, not
something GOAT can fix in `bitvm2-node`, and it gates everything built on top.**

**And the garbling stack is built around Groth16 specifically.** `bitvm2-gc` is
not a general circuit garbler that happens to be pointed at Groth16; its
headline crate is `garbled-snark-verifier` and its circuit tree is organised
around `groth16.rs`, `bn254/`, `dv_snark.rs` and `dv_bn254/`. Swapping the proof
system therefore does not mean deleting a wrapper — it means rebuilding the
garbled-circuit stack around a different verifier, which is the single largest
item in this document.

**Revised scope.** The proof path contains three independent Shor-vulnerable
layers, not one:

| Layer | Primitive | Quantum status | Where the fix lives |
| --- | --- | --- | --- |
| Ziren FRI / Poseidon commitment | hash-based | safe | — |
| Ziren multiset memory check | **hash-to-curve, DLOG** | **broken** | upstream, Ziren #276 |
| Groth16 wrap | **BN254 pairing** | **broken** | GOAT |
| `bitvm2-gc` garbled verifier | AES-128 garbling of a **Groth16 verifier** | garbling safe, **statement broken** | GOAT, and it is a rebuild |
| WOTS commitment into script | hash-based | safe | — |

The two hash-based layers at the ends are fine. Everything between them depends
on either DLOG or pairings. Any credible post-quantum plan for this bridge has
to address all three, in dependency order: Ziren's memory check first, since a
post-quantum wrapper over a DLOG-dependent execution proof buys nothing.

The cost is size and script weight. Hash-based proofs are substantially larger
than Groth16's constant-size proof, and BitVM2's economics depend on what fits
in Bitcoin script and what a challenge transaction costs. That trade is the real
engineering question, and it should be measured before it is decided.

## 5. The `gc-v2` branch and BitVM3 garbling: orthogonal to post-quantum

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
`ark-groth16` and `ark-bn254` (22 files reference `groth16`), so the proof-pipeline conclusion is unchanged by the garbling work.

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

**`bitvm2-gc` is Groth16-oriented by construction, and that is the structural
problem.** `bitvm2-node` depends on `bitvm2-gc`, whose entire purpose is
garbling a Groth16 verifier. So the proof-system choice is not a parameter of
this architecture; it is baked into the component that implements it. Any move
off pairings requires rebuilding that component, and that dependency — not the
wrapper — is the real obstacle.

**Net effect on the post-quantum position: none.** `gc-v2` still uses MuSig2
(8 files) and Taproot (9 files), so the Taproot key-path exposure is unchanged,
and it still wraps in Groth16, so the proof-pipeline conclusion is unchanged. The
garbling work is valuable for other reasons; it should simply not be counted as
progress toward quantum resistance. The one thing it does add to this analysis is
the label-size decision above.

## 6. The peg's weakest quantum link is the Taproot key path

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

## 7. goat: relayer and consensus

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

## 8. goat-geth: the divergence, measured

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

### The EVM's cryptographic substrate is the real exposure here

Treating this surface as "accounts use ECDSA, follow Ethereum" understates it
badly. The EVM exposes elliptic-curve and pairing operations as *consensus-level
precompiles*, and `core/vm/contracts.go` in `goat-geth` carries the full upstream
set:

| Address / type | Primitive | Quantum status |
| --- | --- | --- |
| `0x01` `ecrecover` | secp256k1 ECDSA | **broken** |
| `0x06`–`0x08` `bn256Add`, `bn256ScalarMul`, `bn256Pairing` | BN254 group ops and **pairing** | **broken** |
| `0x0a` `kzgPointEvaluation` | KZG over BLS12-381 | **broken** |
| `bls12381G1Add/G2Add/MultiExp/Pairing/MapG1/MapG2` | BLS12-381 (EIP-2537) | **broken** |
| `p256Verify` | secp256r1 ECDSA (RIP-7212) | **broken** |
| `0x02`, `0x03`, `0x04`, `0x05`, `0x09` | SHA-256, RIPEMD-160, identity, modexp, BLAKE2F | safe |

Three curve families and two pairing engines, all consensus-critical.

**This is a harder migration than account signatures, not an easier one.**
Account keys can migrate through account abstraction, because the *user* opts
in. A precompile cannot: it is a consensus rule, and the contracts that call it
are immutable. Every on-chain SNARK verifier deployed on the chain calls
`0x08`; that bytecode cannot be upgraded, and in general nobody has the
authority to upgrade it. Removing or changing the precompile breaks those
contracts permanently. There is no opt-in path for an already-deployed Groth16
verifier.

**And the severity is soundness, not theft.** If BN254 discrete log falls, a
forged proof satisfies `bn256Pairing`, and every on-chain verifier accepts it. For
a bridge that verifies withdrawal proofs on the L2 side, that is direct loss of
funds through an interface that is behaving exactly as specified.

**The same assumption is load-bearing twice.** BN254 appears on the Bitcoin side
through the BitVM2 Groth16 wrapper *and* on the EVM side through `bn256Pairing`.
One broken assumption compromises the bridge from both directions, so the two
should be tracked as a single dependency rather than two independent risks.

**GOAT's own changes do not touch this.** Of the 37 commits, the changed files
concentrate in `core/types` (23 files), `eth/tracers`, `eth/catalyst` and
`core/goat`; the only configuration-adjacent file is `params/config.go`. The
precompile set is inherited from upstream unchanged, which means the fix must
also come from upstream — and that is the concrete argument for closing the
377-commit gap. It is not hygiene; it is the delivery channel for any future
post-quantum precompile work.

Recommendation for *account semantics* remains follow-not-lead: diverging ahead
of Ethereum's account-abstraction route breaks wallet and tooling compatibility,
which for an L2 is usually the product. But the precompile exposure is not an
account-semantics question, and it should not inherit that low priority.

---

# Part III — What to do

## 9. Ordering the work

**The general principle.** Ownership and severity, not novelty, should set the
order:

1. **Bridge attestation keys** (row 2). Highest value per unit of control. The
   operator or relayer set is the peg's trust root; compromising a threshold is
   equivalent to compromising the peg. It is entirely the L2's to change, and it
   can usually be rolled out with dual signing, giving rollback at every step.
2. **Bridge proof system** (row 3). Highest severity. Fixes soundness rather
   than theft. Usually the largest engineering item, because proof size and
   verification cost drive the L2's economics.
3. **Consensus keys** (row 5). Mechanism often already exists upstream — Cosmos
   SDK v0.55 ships ML-DSA-65 as an opt-in validator key type, for instance — so
   the work is frequently a dependency upgrade rather than cryptographic design.
4. **Execution accounts** (row 6). Follow the upstream ecosystem. Diverging
   account semantics ahead of it breaks wallet and tooling compatibility, which
   for an L2 is often the product itself.
4b. **EVM precompiles** (row 7). Cannot be sequenced like the others, because
   the fix is upstream and the callers are immutable. What *can* be done now is
   an inventory: which deployed contracts call the pairing and KZG precompiles,
   and which of them are consensus- or bridge-critical. A contract that cannot
   be upgraded can at least be known about before it matters.
5. **L1 settlement** (row 1). Not fixable. Bound it instead: avoid address
   reuse, prefer outputs that do not commit a bare public key, avoid long-lived
   outputs with revealed keys, and write custody policy so the script and key
   policy can migrate when the base layer can.

**Applied to GOAT.**

| Phase | Action | Why here |
| --- | --- | --- |
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | The `VerifySign` 64-byte gate shows these assumptions are load-bearing and invisible |
| 1 | Relayer: add ML-DSA-65 to the `PublicKey` `oneof`, make length checks per-variant, roll out with dual attestation | Highest value per unit of control; the `oneof` already supports it; dual signing gives rollback at every step |
| 2 | Peg custody: evaluate NUMS-internal-key (script-path-only) Taproot outputs | Cheapest real gain, available today, no PQ scheme needed; closes the easiest attack |
| 3 | Track and support Ziren #276: replace the hash-to-curve multiset hash with a lattice-based homomorphic hash (LtHash) | Gates everything above it — a PQ wrapper over a DLOG-dependent execution proof buys nothing. Upstream, so influence rather than implement |
| 4 | Re-target the proof pipeline and the `bitvm2-gc` garbling stack away from Groth16/BN254 | Largest item in this plan. `bitvm2-gc` is Groth16-verifier-oriented by construction, so this is a rebuild, not a wrapper swap |
| 5 | Rebase `goat-cosmos-sdk` onto SDK ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | Mechanism is upstream and gentle, but gated on the rebase |
| 6 | Reduce `goat-geth`'s 377-commit lag; inventory which deployed contracts call `0x06`–`0x08` and `0x0a` | The lag is the delivery channel for upstream precompile work. The inventory matters because immutable contracts calling a broken pairing cannot be migrated later — only identified now |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

## 10. The honest limit

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

## 11. Recurring traps

Collected from the analysis so far; each cost real effort to notice.

- **Fixed-length signature checks.** Code that validates `len(sig) == 64` is
  structurally incapable of accepting a post-quantum signature. ML-DSA-65
  signatures are 3309 bytes and public keys 1952. These checks are small,
  load-bearing and invisible until something fails.
- **Size lands on the hot path.** Post-quantum signatures are one to two orders
  of magnitude larger than what they replace. On a consensus layer every
  validator signs every block, so block size limits, gossip framing and
  commit-signature budgets all need re-tuning, not just the key type.
- **Forks hide the real blocker.** An L2 running forked dependencies inherits an
  upstream catch-up before it can inherit upstream post-quantum work. Measure
  the divergence early; it is often the schedule driver, not the cryptography.
- **Code search under-reports on forks.** GitHub code search does not index
  forks. In the one stack examined, searching for `winternitz` in a forked
  dependency returned zero hits while the files plainly exist in the tree.
  An audit built on code search over a fork-heavy stack will silently miss
  things. Enumerate the git tree instead.
- **Precompiles are the exposure that cannot be migrated.** An account can
  adopt a new signature scheme; a deployed contract calling a pairing
  precompile cannot. On any EVM chain the elliptic-curve and pairing
  precompiles are consensus rules with immutable callers, so the exposure is
  fixed at deployment time and only an inventory — not a migration — is
  available afterwards.
- **A "hash-based" proof system can still depend on DLOG.** Check every
  sub-argument, not the headline commitment scheme: permutation arguments,
  lookup arguments and offline memory checks are all places where an
  elliptic-curve accumulator can hide inside an otherwise hash-based system.
- **Ask what a component is *oriented around*, not just what it depends on.** A
  garbling or proving stack purpose-built for one verifier makes the proof
  system a structural property rather than a configuration choice — swapping it
  is a rebuild of that component, and the dependency graph will not show this.
- **Symmetric-primitive layers are not automatically "post-quantum done".**
  Garbled circuits, hash commitments and MPC layers are built from symmetric
  primitives and survive Shor, but they only protect what they *carry*. If the
  statement being garbled or committed is an elliptic-curve or pairing
  assertion, the composition is only as quantum-safe as that assertion. Check
  the content, not the wrapper — and separately check the symmetric layer's own
  parameter, since 128-bit labels sit at the floor of the approved
  post-quantum range.
- **Interoperability makes migration a coordination problem.** Where light
  clients verify a counterparty's signatures with their own compiled-in crypto,
  enabling a new key type before counterparties can verify it breaks
  connectivity — and the failure is silent at upgrade time.

## 12. What is general and what is not

Part I follows from the *structure* of a Bitcoin L2 — a chain that
settles to a base layer it does not control, runs borrowed consensus, and
operates its own bridge. That structure is shared across the category, so the
surface taxonomy and the ownership argument should transfer.

The ordering and the traps are grounded in **one** stack examined in depth plus
the base-layer survey. The ordering reflects a judgement about value
and severity that I believe generalises, but it has not been tested against a
second L2. The traps are observations, not a survey.

**The open work is a second and third case study** — a rollup with a different
bridge design, and an L2 with a non-Cosmos consensus stack — to find out which
of these findings are properties of the category and which are properties of one
implementation. Until then, treat sections 3 and 4 as a well-grounded hypothesis
rather than an established result.

## 13. What was not verified

Stated so this is not read as more thorough than it is.

- `x/locking`, `x/bitcoin` and `x/consensusfork` were not audited for further
  signature surfaces.
- The `goat-cosmos-sdk` fork was not inspected; its divergence determines the
  phase-5 rebase cost and is the largest remaining unknown.
- The BLS12-381 usage in `goat` (2 files, via `gnark-crypto`) was not traced. If
  it is load-bearing for aggregation it is another Shor-exposed surface.
- The 37 custom `goat-geth` commits were not read individually, so whether any
  touch cryptography is unknown.
- In `bitvm2-gc`, the `verifiable-circuit*` crates and the Bristol circuits were
  not read; only the workspace manifests, the garbling core and the circuit tree
  were. Whether the designated-verifier variants change the trust model is not
  assessed here.
- The IBC conclusion rests on `go.mod` alone.

