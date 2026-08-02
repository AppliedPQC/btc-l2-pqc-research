# Post-quantum migration for Bitcoin layer 2s

**A research report.** Compiled 2026-07-31, revised 2026-08-01 and 2026-08-02,
from primary sources: BIP text from
[`bitcoin/bips`](https://github.com/bitcoin/bips),
[`UPGRADING.md`](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
from [`cosmos/cosmos-sdk`](https://github.com/cosmos/cosmos-sdk), the
[Ziren issue tracker](https://github.com/ProjectZKM/Ziren/issues), the
[GOAT repositories](https://github.com/GOATNetwork) and their dependency trees
read through the GitHub API and local clones, and — for section 9 — the BABE
and Argo MAC papers alongside GOAT's own
[Deferred Binding design doc](https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex). Claims are stated as verified; checks
still outstanding are listed under *Remaining open items*.

## Abstract

A Bitcoin layer 2 has a post-quantum problem that neither Bitcoin nor Ethereum
has alone. It settles to a base layer it cannot change, borrows consensus from a
third ecosystem, and operates a bridge whose trust root is cryptography of its
own choosing. The goal is to make both the L2 and the base layer post-quantum
safe — but only one of those is the L2's to schedule, so the work divides into
surfaces it can act on and surfaces where it can only bound exposure and press
upstream.

Part I sets out that structure and a seven-row surface taxonomy keyed on *who
can actually fix each row*. Part II surveys the three base layers an L2
inherits from, where the central finding is that progress inverts exposure:
Bitcoin, with the sharpest exposure, has specified no post-quantum signature
scheme at all, while Cosmos has one in shipped code. Part III reads one stack —
GOAT — in depth as the evidence the framework was derived from. Part IV gives
an ordering. Part V records the method and the recurring traps.

The single most useful finding for practitioners: **symmetric and hash-based
layers protect only what they carry.** In the stack examined, the Bitcoin-side
commitment layer, the garbling layer, the FRI commitment and the witness
encryption that replaces the on-chain verifier are all post-quantum, and the
bridge is still not, because every one of them is wrapping or carrying an
elliptic-curve assertion. Section 9 follows that pattern through four
generations of the same component and finds it stated outright in the security
proof of the newest one.

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
  post-quantum signature scheme at all —
  [BIP-360](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki)
  (Draft) says in its own text that it "does not include the introduction of
  post-quantum signature schemes", and
  [BIP-361](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki)
  (Draft, Informational) lists `Requires: TBD Post Quantum Signature
  BIP`, a document that does not exist. Any BTC in a classical L1 output is on
  Bitcoin's timeline, not the L2's.
- The L2 **usually borrows a consensus stack from a third ecosystem**, and
  inherits that ecosystem's migration path, maturity and constraints.
- The L2 **owns a bridge**, whose trust root is cryptography of its own choosing
  — and which is the highest-value target in the system.

So the goal is *make both the L2 and the base layer post-quantum safe* — while
recognising that the L2 can schedule only half of it. The rest of this report
separates the surfaces it owns from the ones where the honest move is to bound
exposure and track someone else's roadmap.

## 2. The surface taxonomy

Any Bitcoin L2 that settles to L1 and runs its own consensus has, at minimum,
these cryptographic surfaces. The taxonomy is the useful artifact: it makes the
ownership question explicit before the scheme-selection question.

| # | Surface | Typical scheme | Fails to Shor? | Who can fix it |
| --- | --- | --- | --- | --- |
| 1 | L1 settlement outputs | secp256k1 / Schnorr | yes | **base layer only** |
| 2 | Bridge attestation / operator set | secp256k1, Schnorr, MuSig2, BLS | yes | **the L2** |
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | wrap proof and offline memory check are EC-based | **ZKM**, with the L2 |
| 4 | Bridge data commitments | Merkle / Winternitz / Lamport | **no — hash-based** | already safe |
| 5 | L2 consensus keys | ed25519, BLS12-381 | yes | **the L2**, via its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | the L2, at tooling cost |
| 7 | **EVM crypto precompiles** | ecrecover, BN254 pairing, BLS12-381, KZG | yes | **upstream, and unfixable for deployed callers** |

The grouping, not the list, is the point: three different parties hold the fix,
and only the middle group is the L2's to schedule.

---

# Part II — The base layers an L2 inherits from

*Bitcoin, Ethereum and Cosmos are usually discussed as though they face one
shared problem and differ only in speed. They do not: they are solving
structurally different problems, and their relative progress inverts the usual
assumption.*

## 3. Taxonomy: what problem is each chain actually solving?

| | **Bitcoin** | **Ethereum** | **Cosmos** |
| --- | --- | --- | --- |
| Core problem | public-key exposure on an immutable ledger | signature size at validator scale | key-type negotiation across chains |
| Layer attacked first | output type / script | consensus signatures, and accounts separately | validator consensus keys |
| PQ scheme chosen | **none yet** | hash-based (XMSS/Winternitz family) | **ML-DSA-65 (FIPS 204)** |
| Furthest artifact | Draft BIPs | working prototypes | **shipped, opt-in** |
| Governance style | soft fork, rough consensus | coordinated upgrades | per-chain opt-in + genesis params |

## 4. Bitcoin: an exposure problem, with the signature question deferred

Two Draft BIPs sit in the canonical
[`bitcoin/bips`](https://github.com/bitcoin/bips) repository. Neither is
activated.

**[BIP-360, "Pay-to-Merkle-Root" (P2MR)](https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki)**, `Layer: Consensus (soft fork)`,
`Status: Draft`, v0.12.0, `Requires: 340, 341, 342`. It proposes an output type
that is Taproot with the key-path spend removed, so no bare public key is ever
committed on chain. The BIP is explicit about the limits of that: protection
"does not depend on the activation of post-quantum signatures", it defends
against *long* exposure only, and "P2MR does not, by itself, protect against
short exposure quantum attacks". Most decisively for RQ1, the text states:
"While this proposal does not include the introduction of post-quantum
signature schemes" — the authors are "currently researching options".

**[BIP-361, "Post Quantum Migration and Legacy Signature Sunset"](https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki)**,
`Status: Draft`, `Type: Informational`, assigned 2026-02-11. It proposes a
pre-announced sunset of legacy ECDSA/Schnorr, framing quantum security as "a
private incentive". Its `Requires` field reads: **"TBD Post Quantum Signature
BIP"**.

That dependency is the finding. Bitcoin has a hardening step and a deadline
plan, and the deadline plan formally depends on a document that does not yet
exist. Checked again on 2026-07-31: the BIPs index still contains exactly one
post-quantum entry, BIP-361 itself. The exposure is not hypothetical: BIP-361
states that as of 1 March 2026 over 34% of all bitcoin have revealed a public
key on chain.

**The sunset is not a blanket freeze.** BIP-361's Phase A (160,000 blocks, roughly three years after
activation) stops sends to quantum-vulnerable address types. Phase B, two years
later, does not simply invalidate legacy signatures: it *encumbers* ECDSA and
Schnorr spends with a **rescue protocol** designed "to rule out quantum
attackers, but to permit spends from the authentic coin-holders". The mechanism
is an asymmetry of knowledge — most wallets since 2012 derive keys through
BIP-32 hardened derivation, so a holder can prove knowledge of a parent extended
private key that a quantum attacker recovering only the child key would not
have. The BIP points at ZK-STARK-based rescue protocols and at commit/reveal
schemes for this.

Two qualifications, both from the BIP itself. Coverage is unresolved: "it
remains to be seen how much of the legacy Bitcoin supply can be theoretically
covered by such rescue protocols", and only if the majority is covered will the
restriction be "at most mildly confiscatory". And **P2PK has no rescue
protocol** — "it's not currently believed possible to construct a rescue
protocol for P2PK UTXOs, as no knowledge asymmetry is known" — which is why
BIP-361's authors separately support an Hourglass-style proposal for those
outputs. So the coins that genuinely become unspendable are the abandoned ones
and the ones where no secret asymmetry exists, not the legacy supply as a
whole.

## 5. Ethereum: an aggregation problem at consensus, an account problem above it

Ethereum's exposure is ordinary — accounts reveal a public key on first spend,
consensus uses BLS12-381 — but its *constraint* is not. Hash-based signatures
are the most conservative post-quantum option available, resting only on hash
security, but at roughly 3 KB each they cannot go on chain one per validator.
Ethereum therefore treats post-quantum migration substantially as an
aggregation-engineering problem.

The concrete artifacts are named and public.
[`leanEthereum/leanVM`](https://github.com/leanEthereum/leanVM) describes
itself as a "minimal hash-based zkVM, for a Post-Quantum Ethereum" and exists to
do recursive aggregation.
[`leanEthereum/leanSig`](https://github.com/leanEthereum/leanSig) is a Rust
prototype of the proposed signature scheme, built on tweakable hash functions
and incomparable encodings, and grew out of the research implementation
[`b-wagn/hash-sig`](https://github.com/b-wagn/hash-sig)
([eprint 2025/055](https://eprint.iacr.org/2025/055)). `leanSig`'s README is
explicit that the code is unaudited and not for production.
[`pq.ethereum.org`](https://pq.ethereum.org/) is the coordination hub.

**That is only the consensus front.** Accounts have their own two-track story,
and it is the half that concerns user funds:

- an **emergency track** — Buterin's proposal to hard-fork on evidence of
  large-scale theft, revert to the last pre-theft block, freeze legacy ECDSA
  accounts, and let users recover through a transaction carrying a **STARK proof
  of knowledge of the hash preimage** behind their address, authenticating by a
  secret that was never exposed on chain;
- a **structural track** — account abstraction as the migration vehicle.
  [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) ("Set Code for EOAs",
  **Final**, shipped in Pectra) lets an EOA delegate to contract code, though its
  authorisations are themselves secp256k1-signed, so it is a stepping stone
  rather than a post-quantum mechanism. The dedicated vehicle is
  [EIP-8141](https://eips.ethereum.org/EIPS/eip-8141) ("Frame Transaction",
  **Draft**, created 2026-01-29), which supports a slot for an arbitrary
  post-quantum signature scheme alongside the classical ones.

At the **execution layer** the mechanism is deliberately scheme-agnostic.
Rather than adding an ML-DSA or Falcon precompile — which would repeat the
`bn256Pairing` mistake of welding a precompile to one construction —
[EIP-7885](https://eips.ethereum.org/EIPS/eip-7885) ("Precompile for NTT
operations", Draft, created 2025-02-12) exposes the *shared primitive*. Its
motivation states the reasoning directly: "Choosing to integrate NTT and InvNTT
instead of a specific algorithm provides agility, as DILITHIUM or FALCON or any
equivalent can be implemented with a modest cost from those operators", and the
same operator "is also of interest to speed-up STARK verifiers", so one
precompile serves both scaling and the quantum transition. That is crypto-agility
designed in at the layer where Ethereum previously failed to have it.

Nothing post-quantum has shipped to Ethereum mainnet on any of these fronts;
EIP-7885, EIP-8141 and
[EIP-8151](https://eips.ethereum.org/EIPS/eip-8151) are all Draft.

## 6. Cosmos: a negotiation problem, and the one that shipped

Cosmos SDK **v0.55** registers
[ML-DSA-65 (FIPS 204)](https://docs.cosmos.network/sdk/latest/keys/post-quantum-keys)
as a supported validator consensus key type. This is shipped code, not a
proposal ([PR #26436](https://github.com/cosmos/cosmos-sdk/pull/26436)): the
`cosmos.crypto.mldsa65` proto package is in the tree, Amino routes and the
`hd.MlDsa65Type` constant are enabled by default, `x/auth` gained a
`SigVerifyCostMlDsa65` parameter, and `init`/`testnet` accept
`--consensus-key-algo ml_dsa_65`.

It is deliberately *opt-in*: `genesis.consensus_params.validator.pub_key_types`
still defaults to `["ed25519"]`, so nothing changes for an existing chain until
its governance says so. Existing chains can combine the new key type with
validator consensus key rotation, which ships enabled in the same release, to
move validators onto post-quantum keys without a new genesis.

The size consequences are stated plainly and match FIPS 204 exactly: public keys
grow from 32 to 1952 bytes and signatures from 64 to 3309 bytes, roughly 60x and
50x. CometBFT's `MaxSignatureSize` and per-validator `MaxCommitSigBytes` were
raised to accommodate them, and chains are told to revisit
`consensus_params.block.max_bytes` and gossip framing limits.

**The IBC hazard.** The most transferable finding in this survey is a warning in
the Cosmos changelog that has no analogue in the Bitcoin or Ethereum material.
IBC light clients on a counterparty chain verify your validator set's commit
signatures using *the counterparty's own compiled-in crypto*. If validators
holding sufficient voting power sign with a key type a counterparty cannot
verify, headers fail verification there, packet flow stops, and the light client
eventually expires. The counterparty needs the verification code, not the key
type. Post-quantum migration in an interoperable ecosystem is therefore a
*coordination* problem before it is a cryptographic one — and the failure mode
is silent until connectivity breaks.

**And the cost is linear, because CometBFT does not aggregate.** The word
*per-validator* in that parameter name is the whole story. A commit
([`types/block.go`](https://github.com/cometbft/cometbft/blob/main/types/block.go))
carries one signature for each validator that voted:

```go
type Commit struct {
    Signatures []CommitSig     // in validator-address order
}
type CommitSig struct {
    BlockIDFlag, ValidatorAddress, Timestamp, Signature
}
func MaxCommitBytes(valCount int) int64
```

There is no aggregate to grow; there is an array whose length is the validator
set. Signature bytes per commit therefore scale as N × 3309:

| Validators | ed25519 | ML-DSA-65 |
| --- | --- | --- |
| 50 | 3.2 KB | 165 KB |
| 100 | 6.4 KB | 331 KB |
| 150 | 9.6 KB | 496 KB |

CometBFT's response to that arithmetic is more deliberate than the upgrade note
suggests, and it is visible in two constants. `MaxSignatureSize` is not a fixed
number but a maximum taken over every supported key type:

```go
var MaxSignatureSize = cmtmath.MaxInt(
    cmtmath.MaxInt(ed25519.SignatureSize, bls12381.SignatureLength),
    mldsa65.SignatureSize,
)
```

so admitting ML-DSA-65 to the key-type set raises it from 96 to 3309 bytes
globally. `MaxCommitSigBytes` then adds the address, flag, timestamp and proto
framing on top of that, and
[`DefaultBlockParams`](https://github.com/cometbft/cometbft/blob/main/types/params.go)
budgets the worst-case commit *separately from application data*:

```go
// MaxBytes budgets 21MiB for data plus the worst-case commit for a
// maximum-size validator set (MaxVotesCount validators at
// MaxSignatureSize), so the default stays valid for any pub key type
// and validator count.
MaxBytes: 22020096 + MaxCommitBytes(MaxVotesCount), // ~53MiB
```

At `MaxVotesCount` of 10,000 that is 21 MiB of data plus roughly 32 MiB of
commit headroom. The design choice is to make the commit budget explicit and
size it for the worst case, rather than leave operators to discover the
interaction — and the proto-framing comment in the same file reasons directly
about "ML-DSA-65's 3309-byte sigs", so this was sized for post-quantum keys on
purpose.

It has a consequence worth stating, because it is easy to miss: the maximum is
over key types the build *supports*, not the ones a chain *enables*. Every chain
on a CometBFT carrying `mldsa65` inherits the larger default block size whether
or not its validators ever use a post-quantum key. The cost of the option is
paid by everyone; only the per-commit bytes are paid by adopters.

[`UPGRADING.md`](https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md)
does not mention aggregation anywhere, and the upstream work to
add it has not landed: CometBFT issue
[#3455](https://github.com/cometbft/cometbft/issues/3455) (BLS signature
aggregation) has been open since 2024, issue
[#1305](https://github.com/cometbft/cometbft/issues/1305) (halving commit size
with partial ed25519 signatures) since 2023, and two pull requests adding
aggregation to `crypto/bls12381`,
[#3632](https://github.com/cometbft/cometbft/pull/3632) and
[#4763](https://github.com/cometbft/cometbft/pull/4763), were both closed without
merging. BLS12-381 exists in CometBFT as a *key type*, not as aggregated commits.
Two abandoned attempts is evidence about difficulty, not only about priority.

**The IBC hazard.** The most transferable finding in this survey is a warning in
the Cosmos changelog that has no analogue in the Bitcoin or Ethereum material.
IBC light clients on a counterparty chain verify your validator set's commit
signatures using *the counterparty's own compiled-in crypto*. If validators
holding sufficient voting power sign with a key type a counterparty cannot
verify, headers fail verification there, packet flow stops, and the light client
eventually expires. The counterparty needs the verification code, not the key
type. Post-quantum migration in an interoperable ecosystem is therefore a
*coordination* problem before it is a cryptographic one — and the failure mode
is silent until connectivity breaks.

---

# Part III — The evidence: the GOAT stack

## 7. Summary of findings

Six of the seven surfaces below carry quantum-exposed cryptography, and they are
not equally GOAT's to fix. Row numbers refer to the taxonomy of section 2, so
that the case study and the framework can be read against each other.

| Row | Surface | Repo | Scheme | Quantum status | Owner of the fix |
| --- | --- | --- | --- | --- | --- |
| 1 | Bitcoin L1 outputs | — | secp256k1 / Schnorr | broken by Shor | **Bitcoin, not GOAT** |
| 1–2 | Peg custody | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | MuSig2 over a **Taproot key path** | broken by Shor, **and exposed from output creation** | **GOAT** |
| 2 | Bridge attestation | [`goat`](https://github.com/GOATNetwork/goat) `x/relayer` | secp256k1 / Schnorr | broken by Shor | **GOAT** |
| 3 | Bridge proof system | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) | [Ziren](https://github.com/ProjectZKM/Ziren) STARK (**DLOG multiset memory check**) → **Groth16/BN254** wrap → garbled | broken at three layers | **GOAT and ZKM** |
| 4 | Bridge bit commitments | [`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) → [BitVM](https://github.com/GOATNetwork/BitVM) | **Winternitz OTS** | **already PQ-safe** | — |
| 5 | Consensus keys | [`goat`](https://github.com/GOATNetwork/goat) (CometBFT) | **secp256k1** | broken by Shor | **GOAT** |
| 6 | EVM accounts | [`goat-geth`](https://github.com/GOATNetwork/goat-geth) | secp256k1 ECDSA | broken by Shor | GOAT, at tooling cost |

Peg custody spans two rows: the output is an L1 settlement output, but its
*construction* is GOAT's choice, which is what makes the mitigation in section 10
available without waiting on Bitcoin.

The headline is in rows 3 and 4. **BitVM2's Bitcoin-side plumbing is already
post-quantum; its cryptographic content is not.** That reframes the work from a
rewrite into two contained swaps.

## 8. bitvm2-node: the plumbing is hash-based, the content is not

[`bitvm2-node`](https://github.com/GOATNetwork/bitvm2-node) is a Rust ZK bridge: crates for `bitcoin-light-client-circuit`,
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
same conservative assumption as SLH-DSA. The
[BitVM](https://github.com/GOATNetwork/BitVM) tree contains
`bitvm/src/signatures/winternitz.rs`, `winternitz_hash.rs`,
`signing_winternitz.rs` and `wots_api.rs`, and `bitvm2-node` references them
from `crates/bitvm2-ga/src/types.rs` and `crates/bitvm2-ga/src/operator/api.rs`.

> **Verification note.** A GitHub *code search* for `winternitz` in
> [`GOATNetwork/BitVM`](https://github.com/GOATNetwork/BitVM) returns zero hits,
> which would suggest the opposite
> conclusion. That is an artifact: GitHub code search does not index forks, and
> `GOATNetwork/BitVM` is a fork of
> [`BitVM/BitVM`](https://github.com/BitVM/BitVM). Listing the git tree directly
> shows the files. Any audit relying on code search over this stack will
> silently under-report.

**Groth16 is a wrapper, not the proving system.** `bitvm2-node` depends on
[Ziren](https://github.com/ProjectZKM/Ziren) (`zkm-prover`, `zkm-verifier`,
`zkm-sdk`, `zkm-zkvm`, and four more
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

**But the STARK core is not post-quantum either.** FRI and Poseidon are
hash-based, so the core looks safe; the code says otherwise. Ziren issue
[#276](https://github.com/ProjectZKM/Ziren/issues/276), *"Replace hash-to-curve
in multiset hash by quantum safe primitives"*, open since 2025-08-14:

> In our multiset hash, we rely on hash-to-curve to calculate the hash of values
> of the memory addresses, and consider each hash as the x of a point, then
> commit the accumulation of all the points in the EC subgroup. **If DLOG
> hardness is preserved**, prover can not forge another set of scalars for all
> the points respectively, and hence can not forge the proof of the offline
> memory checking. […] To achieve quantum safe, we need to remove the DLOG
> hardness assumption.

So it is the offline memory-consistency argument, not the polynomial commitment,
that rests on discrete log: a quantum adversary forges the memory check rather
than attacking FRI, and the execution proof falls with it.

The fix is a lattice-based multiset hash (LtHash), and it is **past proposal**.
Ziren's [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)
branch — six commits over 39 files, last touched 2026-02-18, commits reading
*"replace ecmh by lthash multiset hash"* — reaches into the core machine and the
recursion circuits, following Zisk's
[lattice-based multiset hashing](https://zisk.technology/secure-challenge-derivation-in-zisk/).

This matters for how the layer should be ranked. Of the three quantum-exposed
layers in this proof pipeline, **this is the tractable one**: it is a primitive
swap inside an existing argument, not an architectural change, and someone has
already built it. It remains ZKM's work rather than something GOAT can fix in
`bitvm2-node`, and it still gates everything above it — a post-quantum wrapper
over a DLOG-dependent execution proof buys nothing — but it should be read as a
dependency to track and support, not as an open research problem.

**And the garbling stack is built around Groth16 specifically.** `bitvm2-gc` is
not a general circuit garbler that happens to be pointed at Groth16; its
headline crate is `garbled-snark-verifier` and its circuit tree is organised
around `groth16.rs`, `bn254/`, `dv_snark.rs` and `dv_bn254/`. Swapping the proof
system therefore does not mean deleting a wrapper — it means rebuilding the
garbled-circuit stack around a different verifier, which is the single largest
item in this report.

The proof path contains three independent Shor-vulnerable layers, not one:

| Layer | Primitive | Quantum status | Where the fix lives |
| --- | --- | --- | --- |
| Ziren FRI / Poseidon commitment | hash-based | safe | — |
| Ziren multiset memory check | **hash-to-curve (ECMH), DLOG** | **broken** | ZKM, [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276) — **prototype exists** ([`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)) |
| Groth16 wrap | **BN254 pairing** | **broken** | GOAT |
| `bitvm2-gc` garbled verifier | AES-128 garbling of a **Groth16 verifier** | garbling safe, **statement broken** | GOAT, and it is a rebuild |
| BABE / Deferred Binding | witness encryption **against the Groth16 relation** | on-chain surface safe, **relation broken** | GOAT — see section 9 |
| WOTS commitment into script | hash-based | safe | — |

The two hash-based layers at the ends are fine. Everything between them depends
on either DLOG or pairings, and the first break sits *inside* the component
usually described as the hash-based one. Any credible post-quantum plan for this
bridge has to address all of them, in dependency order: Ziren's memory check
first, since a post-quantum wrapper over a DLOG-dependent execution proof buys
nothing.

The last row is the least obvious, and it is treated on its own in section 9,
because the on-chain verifier has been through four designs and they are best
read as one topic.

The cost is size and script weight. Hash-based proofs are substantially larger
than Groth16's constant-size proof, and BitVM2's economics depend on what fits
in Bitcoin script and what a challenge transaction costs. That trade is the real
engineering question, and it should be measured before it is decided.

## 9. The on-chain Groth16 verifier: script, garbling, witness encryption

This is one topic, not three. Every design in this line answers the same
question — *how is a Bitcoin script convinced that a Groth16 proof verified* —
and each generation pushes the pairing arithmetic further off chain. Reading
them together is what makes the post-quantum content visible, because the thing
that moves is the *execution* and the thing that never moves is the *assertion*.

| Generation | Mechanism | Cost it attacks | Relation asserted |
| --- | --- | --- | --- |
| BitVM2 | Groth16 verifier compiled to Bitcoin script | on-chain: Disprove script of several hundred KB, **over \$14,000** in a recent unhappy-path experiment | Groth16 / BN254 |
| [BitVM3](https://eprint.iacr.org/2026/933) | verifier circuit *garbled*; dispute evaluates the garbling | on-chain cost collapses, but the garbled circuit is **42 GiB** | Groth16 / BN254 |
| [BABE](https://eprint.iacr.org/2026/065) | witness encryption for linear pairing relations, plus a garbled EC scalar-mul for the non-linear part | off-chain storage and setup, **~1000× lower** than BitVM3 | Groth16 / BN254 |
| [Deferred Binding](https://hackmd.io/@goatresearch/HkKp2g1Zfl) (GOAT) | BABE extended so part of the public input may be fixed *after* garbling | makes BABE usable when L2 state is only known at peg-out | Groth16 / BN254 |

The right-hand column is constant. That is the finding, and everything below is
the evidence for it.

### The lineage, and where GOAT sits in it

[BABE](https://eprint.iacr.org/2026/065) (Garg, Kolonelos, Sergeevitch, Sridhar
and Tse) states the motivation plainly: BitVM2 "suffers from very high on-chain
Bitcoin transaction fees in the unhappy path (over \$14,000 in a recent
experiment)", and BitVM3 fixes that "but each garbled circuit is 42 Gibytes in
size, so the off-chain storage and setup costs are huge". BABE keeps BitVM3's
on-chain savings and "reduces its off-chain storage and setup costs by three
orders of magnitude", using "a witness encryption scheme for linear pairing
relations to verify Groth16 proofs", augmented — because "Groth16 verification
involves non-linear pairings" — with a 2PC protocol built on "a very efficient
garbled circuit for scalar multiplication on elliptic curves". That garbled
circuit builds on [Argo MAC](https://eprint.iacr.org/2026/049) (Eagen and Lai),
which "enables over 1000× more efficient garbled SNARK verifiers". Both encodings
were then improved again by
[Duty-Free Bits](https://eprint.iacr.org/2026/476) (Khambhati, Bhattacharya and
Heath), which reports "we improve BABE's encoding size by 45×, and Argo MAC's by
20×".

GOAT's contribution sits on top of that stack. Vanilla BABE requires the full
public input to be fixed before garbling, which the bridge cannot do: the
dynamic part $x_D$ — L2 state, sequencer commitments, watchtower data — is only
known at peg-out, so by then the committed $vk_x$ is stale and decryption fails.
[**Deferred Binding**](https://hackmd.io/@goatresearch/HkKp2g1Zfl) — whose formal
write-up defines the primitive it rests on,
[partial-binding witness encryption](https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex) —
resolves this with a **Dual-Scalar Garbled Circuit** whose two outputs,
$r \cdot \pi_1$ and $r \cdot P_D + r \cdot B$, "provide prefix binding on $x_S$
and proactive cryptographic binding on $x_D$", with $B$ a verifier-chosen
blinding point hidden inside the circuit. It is implemented, not proposed: the
[`feat/goat-bitvm3`](https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3)
branch of `bitvm2-gc` carries a
[`verifiable-circuit-babe`](https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3/verifiable-circuit-babe)
crate and a
[`babe-programs`](https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3/babe-programs)
workspace alongside the older `garbled-snark-verifier`.

### What the scheme actually encrypts against

The design doc fixes the relation in its first paragraph: a Type-III pairing on
**BN254**, with the verifier accepting iff

$$e(\pi_1, \pi_2) = e(\alpha, \beta) \cdot e(vk_x, \gamma) \cdot e(\pi_3, \delta_2)$$

A ciphertext is $(gc\_ct, adaptor, ct_2, ct_3)$ with $ct_2 = r \cdot \delta_2$
and $ct_3 = \mathsf{msg} \oplus H(r \cdot Y)$, where
$Y = e(\alpha, \beta) \cdot e(vk_x, \gamma)$. Decryption recovers
$r \cdot Y \gets e(ct_1, \pi_2) / e(\pi_3, ct_2)$ — that step is *justified by
the Groth16 equation above* — and then unmasks $\mathsf{msg}$.

So the witness that unlocks the payout is a valid Groth16 proof over BN254, and
the correctness property of the scheme guarantees that anyone holding one can
decrypt. This is the design working as intended; it is also the entire
post-quantum problem.

**The on-chain side genuinely is hash-based.** The doc is explicit that Bitcoin
script is used only for hash commitments to $(ct_2, ct_3)$, gate-by-gate
garbled-circuit unlocking, and "a hashlock on $\mathsf{msg}$ gating the final
UTXO spend", concluding: "No pairings or target-group arithmetic are evaluated
on chain." Every on-chain primitive here — hashlocks, Lamport commitments, the
garbling — survives Shor. That is exactly what makes the composition
misleading.

### The audit: the security theorem names discrete log

The usual way to argue this section would be inference. It is not necessary
here, because GOAT's own security analysis states the reduction:

$$\mathsf{Adv}^{\mathrm{pbWE}}_{\mathcal{A}}(\lambda) \;\leq\; \mathsf{Adv}^{\mathrm{BABE}}(\lambda) \,+\, \mathsf{Adv}^{\mathrm{DLog}}_{\mathbb{G}_1}(\lambda)$$

with a companion lemma bounding leakage of the secret scalar $r$ by
$\mathsf{Adv}^{\mathrm{DLog}}_{\mathbb{G}_1} + \mathsf{Adv}^{\mathrm{coDLog}}$,
because "public exposures of $r$-scaled samples ... are multi-instance DLog
challenges with a shared scalar". The conclusion states the assumption set:
"The reduction stays within standard BABE assumptions plus GC-security and
Lamport one-wayness."

Read that against Shor. $\mathsf{Adv}^{\mathrm{DLog}}_{\mathbb{G}_1}$ on BN254
goes to 1 against a cryptographically relevant quantum computer, and
$\mathsf{Adv}^{\mathrm{BABE}}$ rests on the same curve. The bound does not
degrade gracefully — it becomes vacuous. Concretely, an adversary who can solve
discrete log on BN254 forges a Groth16 proof for a peg-out that never happened,
decrypts $\mathsf{msg}$ through the scheme's own correctness property, and
spends the hashlocked UTXO. No garbling is broken, no hash is inverted, and no
step of the protocol misbehaves.

**This is the wrapper trap for the fourth time in one stack**, and it is the
most seductive instance yet. The first three — Winternitz commitments carrying a
Groth16 proof, garbled circuits garbling a Groth16 verifier, and the prospective
error of proving BLS verification inside `leanVM` (section 12) — at least look
like wrappers. Here the vocabulary actively argues the other way: *witness
encryption*, *garbled circuit*, *hashlock*, *Lamport* are all post-quantum-safe
terms, three of the four on-chain primitives really are post-quantum, and the
security proof is a clean reduction. The pairing survives in the *relation being
encrypted against*, which is the one place the vocabulary never points.

### Ziren sits on the dispute path

Deferred Binding needs *soldering* — translating garbled labels across
cut-and-choose instances — and proves it in a zkVM. The
[`babe-programs`](https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3/babe-programs)
workspace has `soldering/guest` and `soldering/host` members, and the guest is a
Ziren program: it depends on `zkm-zkvm` from `ProjectZKM/Ziren`, reads a
`SolderedWiresInput`, and commits `SolderedLabelsData`. The workspace pulls
`zkm-build`, `zkm-sdk`, `zkm-zkvm` and `poseidon2` from ZKM. GOAT's note reports
the resulting STARK proof at **56.92 MB over 93,867,321 cycles**.

That has a consequence the note does not draw. Ziren's offline memory-consistency
argument rests on an elliptic-curve multiset hash — the ECMH construction of
[Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276), whose own text says
soundness holds only "if DLOG hardness is preserved" (section 8). So the
soldering proof, which is what makes the cut-and-choose argument and therefore
the 1-of-$n$ honest-verifier assumption meaningful, **inherits a second,
independent discrete-log dependency**. Before this work, Ziren's exposure sat in
the proving pipeline. It now also sits on the bridge's dispute path.

Two DLog dependencies in one protocol, reached by different routes, is worth
stating as a planning fact: [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276)
and its [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash)
prototype are now load-bearing for two surfaces, not one, which strengthens the
case for phase 3 of the ordering in section 14.

One version detail worth recording, since it affects which Ziren fixes are
inherited: GOAT's note cites ZKM/Ziren 1.2.5, but `Cargo.lock` on
`feat/goat-bitvm3` pins `zkm-sdk` and `zkm-zkvm` at **1.2.4**, commit
`d5b7577`.

### The garbling parameter, now on the critical path

`LABEL_SIZE = 16` and the PRF is `Aes128` — in both
`garbled-snark-verifier/src/core/utils.rs` and
`verifiable-circuit-babe/src/gc/utils.rs`. 128-bit labels under AES-128 put the
garbling layer at NIST post-quantum security category 1, the floor of the
approved range, since Grover search over a 128-bit space is the reference for
that category. That was a defensible default when garbling was an optimisation.
Under BABE it is the mechanism that gates decryption of the payout secret, so
the decision deserves to be made explicitly rather than inherited. Moving labels
and PRF to 256-bit restores category-5 margin at roughly double the garbling
cost.

The rest of `bitvm2-gc` is unchanged in character. `garbled-snark-verifier`
depends on `ark-groth16`, `ark-bn254`, `ark-ec` and `ark-relations` *and* on
`aes` and `blake3` — arkworks for the circuit being garbled, AES and BLAKE3 for
the garbling machinery — and `verifiable-circuit-babe` carries the same arkworks
set. `bitvm2-node`'s `gc-v2` branch (head `73cdb49`, 2026-07-22) still uses
MuSig2 (8 files) and Taproot (9 files), so the key-path exposure of section 10 is
untouched by any of this.

### Why this does not port to a Ziren STARK verifier

The natural next question is whether the same machinery can carry a hash-based
verifier — witness-encrypt against Ziren's STARK instead of Groth16, and the
BN254 dependency disappears. It does not follow, for a structural reason.

BABE is a witness encryption scheme **for linear pairing relations**. What makes
Groth16 expressible is that its verification equation is a pairing product, so
$Y$ can be formed at encryption time from $vk$ and the public input, and
recovered at decryption time by pairing the proof elements. A FRI/Poseidon
verifier has no pairing relation, no target group, and no short algebraic
acceptance predicate — it is a Merkle-and-hash argument with a large,
non-algebraic verification transcript. There is nothing for the ciphertext to be
keyed to. Constructing the analogue would mean a witness encryption scheme for
hash-based proof systems, which is not a parameter change but an open problem.

That produces the strategic tension this section exists to name. **BABE makes
Groth16 dramatically cheaper to verify on Bitcoin, which reduces the pressure to
stop using Groth16 — and stopping is the post-quantum requirement.** The cost
curve and the quantum curve point in opposite directions. Every increment of
engineering invested in the BABE path deepens the commitment to a pairing-based
statement, and section 8's conclusion still stands: `bitvm2-gc` is
Groth16-oriented by construction, so moving off pairings is a rebuild of that
component rather than a swap of a wrapper.

### Verdict

**Net effect on the post-quantum position: none.** It is not neutral either.
Three things are true of it: the pairing assumption lives in the security
theorem of the scheme that replaces the on-chain verifier, not merely in a
script the bridge could swap out; Ziren's discrete-log dependency is
load-bearing in two places, the proving pipeline and the dispute path; and the
AES-128 garbling parameter gates the payout secret rather than sitting as an
optimisation detail.

None of this is an argument against BABE. It is an excellent answer to the
question it was asked — on-chain cost — and the on-chain surface it produces is
genuinely hash-based. It is an argument against *counting* it as post-quantum
progress, and against the assumption that a future hash-based proof system can
be dropped into the same protocol.

## 10. The peg's weakest quantum link is the Taproot key path

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

## 11. goat: relayer and consensus

This is where the best return is.

**Relayer attestation keys**
([`x/relayer/types/pubkey.go`](https://github.com/GOATNetwork/goat/blob/main/x/relayer/types/pubkey.go))
are the bridge trust
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

GOAT's validators sign with **secp256k1**, not the Cosmos default of ed25519.
[`cmd/goatd/cmd/modgen/init.go`](https://github.com/GOATNetwork/goat/blob/main/cmd/goatd/cmd/modgen/init.go)
sets
`consensusParam.Validator.PubKeyTypes = []string{cmttypes.ABCIPubKeyTypeSecp256k1}`,
and the mainnet genesis agrees: `"pub_key_types": ["secp256k1"]`. This changes
nothing about the exposure, since both fall to Shor, but it does mean the
migration lever is already in use — the chain sets `pub_key_types` explicitly
rather than inheriting a default, so adopting ML-DSA-65 is an edit to a value the
genesis already carries.

Two version facts bound that work. `go.mod` pins CometBFT **v0.38.25**, which
predates both the ML-DSA-65 key type and the expanded signature budget described
in section 6; that release does not even contain `crypto/bls12381`. And the same
`init.go` sets `Block.MaxBytes = 50 * 1024 * 124`, which is 6,348,800 bytes,
about 6 MiB — the arithmetic reads like an intended 50 MiB with a transposed
digit. Whatever the intent, a chain moving to 3309-byte consensus signatures
should confirm this number deliberately: at 6 MiB the commit is a materially
larger fraction of the block than the CometBFT defaults assume.

`ibc-go` does not appear in `goat`'s `go.mod`, so the IBC light-client hazard
that dominates Cosmos post-quantum migration — enabling a key type counterparties
cannot verify stops packet flow and expires the client — appears not to apply.
Worth confirming against deployment reality rather than `go.mod` alone.

Consensus keys lose nothing structurally in that move, because CometBFT never
aggregated them: a commit is an array of one signature per validator (section 6),
so adopting ML-DSA-65 costs bytes in proportion to the validator set and nothing
else. The work is a dependency upgrade and a block-parameter re-tune.

**The relayer vote key is the opposite case, and the distinction is easy to
miss.** Its aggregation is not inherited from Cosmos — it is GOAT's own code,
[`pkg/crypto/blst.go`](https://github.com/GOATNetwork/goat/blob/main/pkg/crypto/blst.go)
called from
[`x/relayer/keeper/proposal.go:66`](https://github.com/GOATNetwork/goat/blob/main/x/relayer/keeper/proposal.go#L66).
Upstream has
no aggregation to extend, and the attempts to add it have not landed, so no
amount of tracking Cosmos releases produces an answer for this surface. It is
GOAT's alone, and it is the one place in this stack where the post-quantum move
costs a *property* rather than bytes.

`ibc-go` does not appear in `goat`'s `go.mod`, so the IBC light-client hazard
that dominates Cosmos post-quantum migration — enabling a key type counterparties
cannot verify stops packet flow and expires the client — appears not to apply.
Worth confirming against deployment reality rather than `go.mod` alone.

## 12. The relayer's BLS vote key

Section 11's recommendation addresses the attestation key. That is not the whole
picture, because the relayer carries **three** distinct key types, not one:

| Key | Scheme | Purpose |
| --- | --- | --- |
| `PublicKey` (`oneof`) | secp256k1 **or** Schnorr | attestation / proposals |
| `TxKey` | secp256k1 | transaction authorisation |
| `VoteKey` | **BLS12-381 G2, 96-byte compressed** | voting, verified via `AggregateVerify` |

Adding ML-DSA-65 to the `PublicKey` `oneof` remains correct and remains the
cheapest first move, but it addresses only the attestation key. The vote key is a different problem, and a much harder one,
because **its value is aggregation**. BLS lets N relayer votes verify as one
48-byte signature. No standardised post-quantum signature aggregates: ML-DSA and
SLH-DSA have no aggregation, so replacing BLS naively turns one signature into
N, at 3309 bytes each. For twenty relayers that is roughly 66 KB where there was
48 bytes.

**Aggregation is confirmed in use, not merely available.** The presence of an
aggregate API is weaker evidence than it appears; what settles it is the call
site,
[`x/relayer/keeper/proposal.go:66`](https://github.com/GOATNetwork/goat/blob/main/x/relayer/keeper/proposal.go#L66):

```go
sigdoc := types.VoteSignDoc(req.MethodName(), sdkctx.ChainID(),
                           relayer.Proposer, sequence, relayer.Epoch, req.VoteSigDoc())
if !goatcrypto.AggregateVerify(pubkeys, sigdoc, req.GetVote().GetSignature()) {
    return 0, errorsmod.Wrap(sdkerrors.ErrInvalidRequest, "verify aggregation signature failed")
}
```

which resolves to `signature.FastAggregateVerify(true, pubkeys, msg, blsMode)` in
`pkg/crypto/blst.go`. `FastAggregateVerify` is specifically the
many-signers-one-message case, and the surrounding code confirms the shape: a
participation bitmap selects voters, the proposer's `VoteKey` and each selected
voter's are collected, participation is checked against `relayer.Threshold()`,
and **one** aggregate signature is verified against **N** public keys over a
single `sigdoc`. Relayer consensus is therefore a threshold vote carried by one
48-byte signature regardless of signer count — and the bitmap design exists
precisely so that signer count can grow.

### `leanVM` is not a way to keep BLS

The obvious hope is to wrap the existing BLS verification in a post-quantum
zkVM: prove inside `leanVM` that the aggregate verified, and inherit the zkVM's
hash-based soundness. That does not work, in two distinct senses, and the second
is a restatement of this report's central trap.

**Mechanically, there is nothing to build on.** `leanEthereum/leanVM` contains no
BLS or pairing code at all — searching its tree for `bls`, `pairing` and `bls12`
returns zero files. What it is built from is KoalaBear (505 references),
Poseidon (75) and WHIR (48), with XMSS across 26 files and a dedicated
`crates/rec_aggregation`. It is purpose-built to recursively aggregate
*hash-based* signatures. A BLS verifier could be written as a guest program, but
emulating BLS12-381 pairing arithmetic over a 31-bit field is precisely the work
that EVM chains give a native precompile to avoid.

**And even if it were built, it would buy nothing.** Proving "this BLS aggregate
verified" inside a hash-based zkVM yields a post-quantum-sound *proof* of a
quantum-broken *statement*. An adversary who can forge BLS signatures forges one
and then honestly proves that it verified; the zkVM faithfully attests to a true
claim about a dead assumption. This is the same error as Winternitz commitments
carrying a Groth16 proof, garbled circuits garbling a Groth16 verifier, and
witness-encrypting against the Groth16 relation (section 9) — the fourth
instance in this one stack, and the only one that would be a *prospective*
mistake rather than an existing one.

**What `leanVM` actually offers is the removal of the need for BLS.** BLS was
chosen for one property, aggregation. Hash-based signatures are post-quantum but
do not aggregate. Recursive proving restores that property. The path is
therefore:

```
BLS                    hash-based signatures       + recursive aggregation
(aggregates, broken) →  (safe, N x 3309 bytes)  →  (aggregation restored)
```

not "keep BLS and add a zkVM". The ordering matters: the signature scheme is
replaced first, and recursion recovers what the replacement costs.

Concretely for GOAT, `AggregateVerify` at `proposal.go:66` is not wrapped, it is
replaced, with the threshold-and-bitmap voting logic rebuilt around a hash-based
scheme plus recursion. GOAT already operates a zkVM (Ziren) and a garbling
stack, so the ingredients are unusually close to hand — but this is an
architectural workstream, not a key-type change, and it should be planned
separately from the attestation-key work.

### Threshold signing is the other route, and it preserves the on-chain shape

Recursion is not the only way to recover what BLS provided. The requirement at
`proposal.go:66` is narrower than *aggregation* in the general sense: it is many
signers, **one** message, verified against a threshold. That is the shape a
**threshold signature** has natively — N parties jointly emit a single ordinary
signature — and unlike aggregation it leaves the verifier untouched.

Two 2026 results make this concrete for ML-DSA specifically, and they differ in
the dimension that matters here:

| | Parties | Communication | Verifier |
| --- | --- | --- | --- |
| [Quorus](https://eprint.iacr.org/2025/1163) (Bienstock et al., USENIX 2026) | **any number**, t-of-n | ~100 KB per party per rejection-sampling round | unmodified ML-DSA; signature and key sizes match |
| [Efficient Threshold ML-DSA](https://eprint.iacr.org/2026/013) (Celi et al.) | up to **6** | ≤ 1 MB per party | unmodified ML-DSA |

For a relayer set in the tens, only the first scales; the second is explicitly a
small-party construction. What either buys is that the chain keeps verifying one
3309-byte signature with a stock FIPS 204 verifier no matter how many relayers
voted — the constant-size property that made BLS attractive.

**The price is a change in signing shape, not in key type.** Today each relayer
signs independently and offline, and the bitmap merely records who did. Threshold
signing replaces that with a live multi-round protocol among the selected quorum,
which converts a liveness-tolerant design into one with a signing-time
availability requirement. That is a heavier change than it appears in a table.

**And none of it is standardised.** NIST's first call for multi-party threshold
schemes, [IR 8214C](https://csrc.nist.gov/projects/threshold-cryptography),
reached final only on 2026-01-20, with preview submissions due 2026-08-07. For a
peg trust root, depending on a construction with no FIPS number and no
validation programme is a governance decision as much as a technical one, and it
argues for sequencing the attestation key — which needs no aggregation at all —
ahead of the vote key.

**A note on the third possibility.** Post-hoc *signature* aggregation, which
would squash N existing ML-DSA signatures into one small object without changing
how anyone signs, is the option that sounds best and works worst.
[Boudgoust and Takahashi](https://eprint.iacr.org/2023/159) (ESORICS 2023) gave
the first Fiat–Shamir-with-aborts aggregate signature, applicable to Dilithium,
and report "quite small compression rates" in their own words; it also aggregates
*distinct* messages, where this use case has one. The strong result in this line,
[aggregating with LaBRADOR](https://eprint.iacr.org/2024/311) (Aardal et al.,
CRYPTO 2024), is for **Falcon**, not ML-DSA. That asymmetry is worth reading as a
design signal: if aggregability is a first-order requirement, it is an argument
about *which* post-quantum scheme to adopt, not a problem to be solved after
adopting ML-DSA.

## 13. goat-geth: the divergence, measured

Comparing
[`GOATNetwork:goat-geth:dev`](https://github.com/GOATNetwork/goat-geth) against
[`ethereum/go-ethereum:master`](https://github.com/ethereum/go-ethereum):

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
precompiles*, and
[`core/vm/contracts.go`](https://github.com/GOATNetwork/goat-geth/blob/dev/core/vm/contracts.go)
in `goat-geth` carries the full upstream set:

| Address / type | Primitive | Quantum status |
| --- | --- | --- |
| `0x01` `ecrecover` | secp256k1 ECDSA | **broken** |
| `0x06`–`0x08` `bn256Add`, `bn256ScalarMul`, `bn256Pairing` | BN254 group ops and **pairing** | **broken** |
| `0x0a` `kzgPointEvaluation` | KZG over BLS12-381 | **broken** |
| `bls12381G1Add/G2Add/MultiExp/Pairing/MapG1/MapG2` | BLS12-381 ([EIP-2537](https://eips.ethereum.org/EIPS/eip-2537)) | **broken** |
| `p256Verify` | secp256r1 ECDSA ([RIP-7212](https://github.com/ethereum/RIPs/blob/master/RIPS/rip-7212.md)) | **broken** |
| `0x02`, `0x03`, `0x04`, `0x05`, `0x09` | SHA-256, RIPEMD-160, identity, modexp, BLAKE2F | safe |

Three curve families and two pairing engines, all consensus-critical.

**This is a harder migration than account signatures, though not a hopeless
one.** Account keys can migrate through account abstraction, because the *user*
opts in. A precompile cannot be migrated that way: it is a consensus rule, and
the contracts calling it are immutable bytecode that in general nobody has the
authority to upgrade.

Ethereum's own work shows the one lever that does exist, and it is narrower than
"remove the precompile".
**[EIP-8151](https://eips.ethereum.org/EIPS/eip-8151)** ("Account Code
Restricted ecRecover",
Draft, created 2026-02-09) does not delete `ecRecover`; it *narrows* it, keyed on
state the protocol already controls. The attack it closes is instructive: an EOA
that migrates to post-quantum authorization via
[EIP-7702](https://eips.ethereum.org/EIPS/eip-7702) has its ECDSA
*transaction* authority disabled by
[EIP-3607](https://eips.ethereum.org/EIPS/eip-3607) (Final), but `ecRecover` ignores
account state — so a quantum attacker holding the derived ECDSA key could still
authorize transfers through **immutable contracts**, notably ERC-20 `permit`
implementations. EIP-8151 makes `ecRecover` return zero when the account has
non-empty code that is not a valid delegation indicator.

The generalisable rule is therefore sharper than "precompiles are unmigratable":

- a precompile **bound to an account identity** can be narrowed conditionally,
  because there is per-account state (has this account migrated?) to key the
  restriction on;
- a **stateless mathematical** precompile cannot. `bn256Pairing` answers a
  question about field elements; there is no account whose migration status
  could gate it, and every deployed verifier that calls it would break equally.

`ecRecover` is in the first category. The pairing, BLS12-381 and KZG precompiles
are in the second, and a search of the EIPs repository returns no proposal to
deprecate or replace any of them.

### The caller set, not the precompile, is the unit of analysis

Ethereum's own coverage is uneven in a way that turns out to be informative.
`ecRecover` has a plan. KZG has a stated plan: the roadmap acknowledges that
"KZG commitments rely on elliptic curve pairings, the same mathematical
structure that quantum computers can attack", and commits to "replace KZG with a
quantum-safe commitment scheme", with STARK-based and lattice-based commitments
named as candidates still under research. The BN254 and BLS12-381 group and
pairing precompiles are not addressed at all — the official quantum-resistance
page does not discuss precompiles as a category.

That asymmetry is not an oversight, and the reason generalises. What decides
whether an exposure can be migrated is not the precompile but **who calls it**:

| Precompile | Caller set | Migratable? |
| --- | --- | --- |
| `ecRecover` | every account, but gated by per-account state | yes — narrow it (EIP-8151) |
| KZG point evaluation | rollup contracts: bounded, known, actively maintained | plausibly — coordinate the set |
| BN254 / BLS12-381 pairing | every SNARK verifier ever deployed: unbounded, largely abandoned | no — nobody to coordinate with |

A bounded set of maintained callers can be coordinated through an upgrade. An
unbounded set of abandoned contracts cannot, by anyone, at any price. This is why
the plans exist where they exist.

### What that means in practice

For stateless precompiles with unbounded callers, the levers are all
unattractive and worth naming honestly:

- **Additive only** — ship post-quantum precompiles alongside the old ones, as
  EIP-7885 proposes. This is the actual plan. It solves the *forward* problem
  and leaves the legacy one untouched.
- **Gas repricing** as soft deprecation — make the call prohibitively expensive
  rather than removing it. This breaks callers economically instead of
  technically, which fails *silently* through out-of-gas rather than loudly, and
  is arguably worse than a clean break.
- **Flag-day removal** — the shape of Bitcoin's BIP-361 sunset applied to a
  precompile. There is no Ethereum precedent for this, and it breaks every
  caller at once.
- **Nothing** — the default. When BN254 falls, every deployed verifier accepts
  forged proofs while behaving exactly as specified.

**The actionable question for any given chain is therefore not "is the
precompile exposed" but "are my callers upgradeable".** A project whose
verifier contracts are its own and upgradeable sits in the tractable category
regardless of what upstream does; one whose verifiers are immutable has a
deadline it cannot move. That is a question about deployment artifacts, and it
can be answered today from chain state — which is why the inventory in the
ordering below is dated work rather than a formality.

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

# Part IV — What to do

## 14. Ordering the work

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
5. **L1 settlement** (row 1). Not fixable. Bound it instead: avoid address
   reuse, prefer outputs that do not commit a bare public key, avoid long-lived
   outputs with revealed keys, and write custody policy so the script and key
   policy can migrate when the base layer can.

**EVM precompiles (row 7) sit outside this sequence.** They cannot be ordered
against the rest, because the fix is upstream and the callers are immutable.
What can be done now is an inventory: which deployed contracts call the pairing
and KZG precompiles, and which of those are consensus- or bridge-critical. A
contract that cannot be upgraded can at least be known about before it matters.

**Applied to GOAT.**

| Phase | Action | Why here |
| --- | --- | --- |
| 0 | Inventory every signature and proof verification path; add tests asserting no fixed signature-length assumptions | The `VerifySign` 64-byte gate shows these assumptions are load-bearing and invisible |
| 1 | Relayer: add ML-DSA-65 to the `PublicKey` `oneof`, make length checks per-variant, roll out with dual attestation | Highest value per unit of control; the `oneof` already supports it; dual signing gives rollback at every step |
| 2 | Peg custody: evaluate NUMS-internal-key (script-path-only) Taproot outputs | Cheapest real gain, available today, no PQ scheme needed; closes the easiest attack |
| 3 | Track and support [Ziren #276](https://github.com/ProjectZKM/Ziren/issues/276) / [`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash) through to merge | Gates everything above it, and is the tractable layer: a primitive swap with a working 39-file prototype. Influence and test rather than implement. Now load-bearing twice, since BABE soldering also proves in Ziren (section 9) |
| 4 | Re-target the proof pipeline and the `bitvm2-gc` garbling stack away from Groth16/BN254 | Largest item in this plan. `bitvm2-gc` is Groth16-verifier-oriented by construction, so this is a rebuild, not a wrapper swap. Sequence against the BABE work deliberately: BABE lowers the cost of *keeping* Groth16, and its witness encryption does not port to a hash-based verifier |
| 5 | Upgrade `cosmos-sdk` v0.53.8 → ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | No SDK fork exists, so this is a dependency upgrade, not a rebase. A dependency upgrade, not a rebase |
| 6 | Reduce `goat-geth`'s 377-commit lag; inventory callers of `0x06`–`0x08` and `0x0a`, and record for each whether it is **upgradeable** | The lag is the delivery channel for EIP-7885 and EIP-8151 when they land. The inventory's key column is upgradeability, not existence: an upgradeable verifier is tractable whatever upstream does, an immutable one has a deadline that cannot move |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

---

# Part V — Traps and open items

## 15. Recurring traps

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
- **Prefer exposing a primitive to blessing a scheme.** A precompile welded to
  one construction (a specific curve, a specific pairing) becomes unremovable
  the moment contracts depend on it. Exposing the shared underlying operation
  instead — NTT rather than ML-DSA, as Ethereum's EIP-7885 proposes — keeps the
  scheme choice in contract code, where it can still be changed.
- **Precompiles are the exposure that is hardest to migrate.** An account can
  adopt a new signature scheme; a deployed contract calling a pairing
  precompile cannot. On any EVM chain the elliptic-curve and pairing
  precompiles are consensus rules with immutable callers. One lever exists:
  a precompile bound to an *account identity* can be narrowed conditionally on
  per-account state, as EIP-8151 does for `ecRecover`. A stateless mathematical
  precompile such as a pairing check has no such state to key on, and for those
  the exposure is fixed at deployment time with only an inventory available.
- **A "hash-based" proof system can still depend on DLOG.** Check every
  sub-argument, not the headline commitment scheme: permutation arguments,
  lookup arguments and offline memory checks are all places where an
  elliptic-curve accumulator can hide inside an otherwise hash-based system.
- **Ask what a component is *oriented around*, not just what it depends on.** A
  garbling or proving stack purpose-built for one verifier makes the proof
  system a structural property rather than a configuration choice — swapping it
  is a rebuild of that component, and the dependency graph will not show this.
- **The wrapper trap runs forwards as well as backwards.** It is easy to spot
  in existing code — a hash-based commitment carrying a curve assertion — and
  just as easy to *introduce* while migrating, by wrapping a broken primitive in
  a post-quantum proof system and treating the composition as fixed. Proving
  that a broken signature verified is a true statement about a dead assumption.
  A migration step only helps if it changes what is asserted, not merely who
  attests to it.
- **Read the security theorem, not the primitive list.** The sharpest version of
  the wrapper trap hides behind vocabulary that is entirely post-quantum-safe.
  A scheme built from witness encryption, garbled circuits, hashlocks and
  Lamport commitments can still reduce to elliptic-curve discrete log, because
  the pairing survives in the *relation being encrypted against* rather than in
  any named primitive. Section 9's case states it in the proof itself: the
  advantage bound carries an explicit `Adv^DLog` term. When a design claims to
  remove an on-chain verifier, ask what the ciphertext is keyed to.
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

## Remaining open items

Honestly short, and none of them block the recommendations:

- The 37 `goat-geth` commits were classified by file area, not read line by
  line. Nothing in the changed areas is cryptographic, but a deliberate check of
  `core/types` changes against consensus-critical serialisation would close it
  fully.
- Which deployed L2 contracts call the `0x06`–`0x08` and `0x0a` precompiles is
  not enumerated, and neither is the more important question of whether each
  caller is upgradeable. That needs chain state rather than source, and it is
  the one item on this list with a deadline: an immutable caller can be
  identified now but never migrated later.
- Ziren's `feat/lthash` prototype has not been reviewed here for completeness or
  merged upstream; whether it covers every ECMH use site is unchecked.
- The BABE work of section 9 lives on the `feat/goat-bitvm3` branch of
  `bitvm2-gc`, not on `main`, so the audit describes an in-flight design. The
  cut-and-choose and cost figures are taken from GOAT's note rather than
  reproduced; only the dependency structure, the Ziren pin and the garbling
  parameters were read from the tree.
- No second L2 has been examined, so Part I's taxonomy is structural reasoning
  supported by one case, not a survey.

## References

Primary sources. Verified live on 2026-07-31; the aggregation sources on
2026-08-01.

- BIP-360, *Pay-to-Merkle-Root (P2MR)* — <https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki>
- BIP-361, *Post Quantum Migration and Legacy Signature Sunset* — <https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki>
- Bitcoin Optech, *Quantum resistance* — <https://bitcoinops.org/en/topics/quantum-resistance/>
- Cosmos SDK, `UPGRADING.md` — <https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md>
- Cosmos SDK PR #26436, ML-DSA-65 consensus keys — <https://github.com/cosmos/cosmos-sdk/pull/26436>
- Cosmos docs, *Post-quantum keys* — <https://docs.cosmos.network/sdk/latest/keys/post-quantum-keys>
- Post-Quantum Ethereum — <https://pq.ethereum.org/>
- `leanEthereum/leanVM` — <https://github.com/leanEthereum/leanVM>
- `leanEthereum/leanSig` — <https://github.com/leanEthereum/leanSig>
- `b-wagn/hash-sig`, eprint 2025/055 — <https://github.com/b-wagn/hash-sig>
- Ziren issue #276, *Replace hash-to-curve in multiset hash by quantum safe primitives* — <https://github.com/ProjectZKM/Ziren/issues/276>
- CometBFT, `types/block.go` (`Commit`, `CommitSig`, `MaxCommitBytes`) — <https://github.com/cometbft/cometbft/blob/main/types/block.go>
- CometBFT, `types/params.go` (`DefaultBlockParams`) — <https://github.com/cometbft/cometbft/blob/main/types/params.go>
- CometBFT issue #3455, *BLS signature aggregation* — <https://github.com/cometbft/cometbft/issues/3455>
- A. Bienstock, L. de Castro, D. Escudero, A. Polychroniadou, A. Takahashi, *Quorus: Efficient, Scalable Threshold ML-DSA Signatures from MPC*, USENIX Security 2026 — <https://eprint.iacr.org/2025/1163>
- S. Celi, R. del Pino, T. Espitau, G. Niot, T. Prest, *Efficient Threshold ML-DSA* — <https://eprint.iacr.org/2026/013>
- K. Boudgoust, A. Takahashi, *Sequential Half-Aggregation of Lattice-Based Signatures*, ESORICS 2023 — <https://eprint.iacr.org/2023/159>
- M. Aardal, D. Aranha, K. Boudgoust, S. Kolby, A. Takahashi, *Aggregating Falcon Signatures with LaBRADOR*, CRYPTO 2024 — <https://eprint.iacr.org/2024/311>
- NIST, *Multi-Party Threshold Cryptography* (IR 8214C, final 2026-01-20) — <https://csrc.nist.gov/projects/threshold-cryptography>
- `GOATNetwork/goat` — <https://github.com/GOATNetwork/goat>
- `GOATNetwork/goat-geth` — <https://github.com/GOATNetwork/goat-geth>
- `GOATNetwork/bitvm2-node` — <https://github.com/GOATNetwork/bitvm2-node>
- `GOATNetwork/bitvm2-gc` — <https://github.com/GOATNetwork/bitvm2-gc>

On-chain proof verification (section 9). Verified 2026-08-02.

- S. Garg, D. Kolonelos, M. Sergeevitch, S. Sridhar, D. Tse, *BABE: Verifying Proofs on Bitcoin Made 1000x Cheaper* — <https://eprint.iacr.org/2026/065>
- L. Eagen, Y. T. Lai, *Argo MAC: Garbling with Elliptic Curve MACs* — <https://eprint.iacr.org/2026/049>
- N. Khambhati, A. Bhattacharya, D. Heath, *Duty-Free Bits: Projectivizing Garbling Schemes* — <https://eprint.iacr.org/2026/476>
- R. Linus et al., *BitVM3: Efficient Bitcoin Bridges via Garbled Circuits* — <https://eprint.iacr.org/2026/933>
- GOAT Research, *Deferred Binding: Extending BABE for Dynamic Public Inputs in GOAT BitVM3* — <https://hackmd.io/@goatresearch/HkKp2g1Zfl>
- GOAT, *Partial-Binding Witness Encryption over Groth16* — the formal write-up of the primitive Deferred Binding rests on (`docs/partial_binding_we.tex`) — <https://github.com/GOATNetwork/bitvm2-gc/blob/feat/goat-bitvm3/docs/partial_binding_we.tex>
- `bitvm2-gc`, branch `feat/goat-bitvm3` (`verifiable-circuit-babe`, `babe-programs/soldering`) — <https://github.com/GOATNetwork/bitvm2-gc/tree/feat/goat-bitvm3>
