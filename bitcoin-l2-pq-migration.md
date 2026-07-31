# Post-quantum migration for Bitcoin layer 2s

**A research report.** Compiled 2026-07-31 from primary sources: BIP text from
`bitcoin/bips`, `UPGRADING.md` from `cosmos/cosmos-sdk`, the Ziren issue
tracker, and the GOAT repositories and their dependency trees read through the
GitHub API and local clones. Where a widely repeated secondary claim proved
wrong, or where a maintainer correction changed a conclusion, that is recorded
rather than quietly patched — see the verification log in Part V.

## Abstract

A Bitcoin layer 2 has a post-quantum problem that neither Bitcoin nor Ethereum
has alone. It settles to a base layer it cannot change, borrows consensus from a
third ecosystem, and operates a bridge whose trust root is cryptography of its
own choosing. The honest goal is therefore never "make the L2 post-quantum
safe"; it is to harden the surfaces the L2 owns and bound the residual risk on
the ones it does not.

Part I sets out that structure and a seven-row surface taxonomy keyed on *who
can actually fix each row*. Part II surveys the three base layers an L2
inherits from, where the central finding is that progress inverts exposure:
Bitcoin, with the sharpest exposure, has specified no post-quantum signature
scheme at all, while Cosmos has one in shipped code. Part III reads one stack —
GOAT — in depth as the evidence the framework was derived from. Part IV gives
an ordering. Part V records the method, the traps, and what still rests on a
single case study.

The single most useful finding for practitioners: **symmetric and hash-based
layers protect only what they carry.** In the stack examined, the Bitcoin-side
commitment layer, the garbling layer and the FRI commitment are all
post-quantum, and the bridge is still not, because every one of them is
wrapping or carrying an elliptic-curve assertion.

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

**Row 3 fails differently, not more severely.** An earlier draft called this the
highest-severity row. That was sloppy, and it contradicted this report's own
case study. Rows 1, 2 and 3 all terminate in the same place — loss of pegged
funds — so ranking them by severity is not informative. Three axes are:

| | Is the failure silent? | Attacker effort | Tractability of the fix |
| --- | --- | --- | --- |
| Row 1, L1 outputs | no, theft is visible | key recovery | not fixable by the L2 |
| Row 2, operator keys | no | compromise a threshold | key-type swap, unless aggregated |
| Row 3, proof system | **yes** | forge a proof | **varies enormously by sub-layer** |

Row 3's distinction is that failure is **silent**: the verifier behaves exactly
as specified while accepting falsehoods. Nothing looks wrong. That is a reason
to treat it seriously, but it is not a reason to put it first — by attacker
effort, an exposed Taproot key path is strictly easier than forging a proof, and
its fix is cheaper, which is why the ordering in Part IV puts it ahead.

**Tractability within row 3 varies more than between rows.** Pairing-based
systems (Groth16, PLONK over BN254/BLS12-381) fall to Shor exactly as ECDSA
does. But a proof system is not one thing, and its sub-layers have very
different fix costs — from a drop-in primitive swap to a full rebuild. The
worked example in Part III has all three kinds at once.

**And "STARK" is not by itself an answer.** A zkVM's headline
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

Two Draft BIPs sit in the canonical `bitcoin/bips` repository. Neither is
activated.

**BIP-360, "Pay-to-Merkle-Root" (P2MR)**, `Layer: Consensus (soft fork)`,
`Status: Draft`, v0.12.0, `Requires: 340, 341, 342`. It proposes an output type
that is Taproot with the key-path spend removed, so no bare public key is ever
committed on chain. The BIP is explicit about the limits of that: protection
"does not depend on the activation of post-quantum signatures", it defends
against *long* exposure only, and "P2MR does not, by itself, protect against
short exposure quantum attacks". Most decisively for RQ1, the text states:
"While this proposal does not include the introduction of post-quantum
signature schemes" — the authors are "currently researching options".

**BIP-361, "Post Quantum Migration and Legacy Signature Sunset"**,
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

**The sunset is not a blanket freeze, and an earlier draft of this report
implied it was.** BIP-361's Phase A (160,000 blocks, roughly three years after
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

The concrete artifacts are named and public. `leanEthereum/leanVM` describes
itself as a "minimal hash-based zkVM, for a Post-Quantum Ethereum" and exists to
do recursive aggregation. `leanEthereum/leanSig` is a Rust prototype of the
proposed signature scheme, built on tweakable hash functions and incomparable
encodings, and grew out of the research implementation `b-wagn/hash-sig`
(eprint 2025/055). `leanSig`'s README is explicit that the code is unaudited and
not for production. `pq.ethereum.org` is the coordination hub.

**That is only the consensus front.** An earlier draft of this report described
Ethereum's plan as an aggregation problem and stopped there, which omitted the
half that concerns user funds. Accounts have their own two-track story:

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

Nothing post-quantum has shipped to Ethereum mainnet on either front.

## 6. Cosmos: a negotiation problem, and the one that shipped

Cosmos SDK **v0.55** registers ML-DSA-65 (FIPS 204) as a supported validator
consensus key type. This is shipped code, not a proposal: the
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

## 7. One technique both chains converged on

The two base layers face different problems and have chosen different
mechanisms, but on the question of *rescuing funds whose key is already
exposed* they arrived independently at the same answer: **prove knowledge of a
secret the quantum attacker cannot derive, and verify that proof with a STARK.**

Bitcoin's BIP-361 rescue protocol proves knowledge of a BIP-32 parent extended
private key, which an attacker who recovered only the child key would not have.
Ethereum's emergency hard fork proves knowledge of the hash preimage behind an
address, which was never published. The asymmetry differs — a derivation path in
one case, a preimage in the other — but the shape is identical, and in both
cases the verifier is hash-based rather than a signature scheme.

This is worth noting for two reasons. It is the strongest evidence in this
report that hash-based proof systems, not post-quantum signatures alone, are the
migration primitive that matters. And it is a caution against reading the
"different problems" thesis too strongly: the constraints diverge, and some
techniques still generalise across them.

## 8. Synthesis

Three observations survive cross-comparison.

**Progress inverts exposure.** Bitcoin has the sharpest exposure — an immutable
ledger, over a third of supply with revealed keys, no administrator — and has
specified no post-quantum signature scheme. Cosmos, with far less public
attention on its quantum posture, has ML-DSA-65 in shipped code. The ordering is
explained by governance surface, not by risk: Cosmos can ship an opt-in key type
because each chain decides for itself, whereas Bitcoin must reach rough
consensus across an ecosystem that treats a forced migration as confiscation.

**Each chain's hardest constraint is different, and each roadmap fits its own
constraint.** Bitcoin's is governance, which is why BIP-361 is framed in terms
of incentives rather than mechanism. Ethereum's is signature size at validator
scale, which is why its flagship artifact is a zkVM rather than a signature
library. Cosmos's is heterogeneity across connected chains, which is why its
mechanism is a negotiable key type rather than a flag day.

**Size is the shared physical fact.** ML-DSA-65 at 1952/3309 bytes and
hash-based signatures at roughly 3 KB are the same order of magnitude, and both
are one to two orders larger than what they replace. Every roadmap here is
downstream of that number: Bitcoin defers it, Ethereum aggregates it away,
Cosmos raises its limits and warns operators.

---

# Part III — The evidence: the GOAT stack

## 9. Summary of findings

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

## 10. bitvm2-node: the plumbing is hash-based, the content is not

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
forges the memory check and with it the execution proof. The replacement is a lattice-based multiset hash (LtHash), and it is **further
along than a proposal**: Ziren carries a working prototype branch,
[`feat/lthash`](https://github.com/ProjectZKM/Ziren/tree/feat/lthash) — six
commits over 39 files, last touched 2026-02-18, whose commits read *"replace
ecmh by lthash multiset hash"* and which reaches into the core machine and the
recursion circuits. The approach follows Zisk's
[lattice-based multiset hashing](https://zisk.technology/secure-challenge-derivation-in-zisk/).

This matters for how the layer should be ranked. Of the three quantum-exposed
layers in this proof pipeline, **this is the tractable one**: it is a primitive
swap inside an existing argument, not an architectural change, and someone has
already built it. It remains upstream work rather than something GOAT can fix in
`bitvm2-node`, and it still gates everything above it — a post-quantum wrapper
over a DLOG-dependent execution proof buys nothing — but it should be read as a
dependency to track and support, not as an open research problem.

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
| Ziren multiset memory check | **hash-to-curve (ECMH), DLOG** | **broken** | upstream, Ziren #276 — **prototype exists** (`feat/lthash`) |
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

## 11. The `gc-v2` branch and BitVM3 garbling: orthogonal to post-quantum

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

## 12. The peg's weakest quantum link is the Taproot key path

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

## 13. goat: relayer and consensus

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

## 14. goat-geth: the divergence, measured

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

# Part IV — What to do

## 15. Ordering the work

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
| 3 | Track and support Ziren #276 / `feat/lthash` through to merge | Gates everything above it, and is the tractable layer: a primitive swap with a working 39-file prototype. Influence and test rather than implement |
| 4 | Re-target the proof pipeline and the `bitvm2-gc` garbling stack away from Groth16/BN254 | Largest item in this plan. `bitvm2-gc` is Groth16-verifier-oriented by construction, so this is a rebuild, not a wrapper swap |
| 5 | Upgrade `cosmos-sdk` v0.53.8 → ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | No SDK fork exists, so this is a dependency upgrade, not a rebase. Cheaper than previously assessed |
| 6 | Reduce `goat-geth`'s 377-commit lag; inventory which deployed contracts call `0x06`–`0x08` and `0x0a` | The lag is the delivery channel for upstream precompile work. The inventory matters because immutable contracts calling a broken pairing cannot be migrated later — only identified now |
| — | Peg: minimise Bitcoin-side key exposure; keep custody policy migratable | Blocked on Bitcoin, which by BIP-360's own text has no PQ signature scheme |

## 16. The honest limit

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

---

# Part V — Method, traps, and limits

## 17. Recurring traps

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
- **The wrapper trap runs forwards as well as backwards.** It is easy to spot
  in existing code — a hash-based commitment carrying a curve assertion — and
  just as easy to *introduce* while migrating, by wrapping a broken primitive in
  a post-quantum proof system and treating the composition as fixed. Proving
  that a broken signature verified is a true statement about a dead assumption.
  A migration step only helps if it changes what is asserted, not merely who
  attests to it.
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

## 18. What is general and what is not

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

## Verification log

Everything below was open at an earlier draft and has since been checked. Four
items changed a conclusion.

| Item | Result | Effect |
| --- | --- | --- |
| Is `cosmos-sdk` forked? | **No.** The `replace` to `../goat-cosmos-sdk` is commented out; `GOATNetwork/goat-cosmos-sdk` returns 404. Only `go-ethereum => GOATNetwork/goat-geth v0.4.1` is an active source replace | **Correction.** Consensus-key migration is a plain v0.53.8 → v0.55 dependency upgrade, not a fork rebase. Moves *up* the order |
| BLS12-381 usage in `goat` | **Load-bearing.** `pkg/crypto/blst.go` implements `BLS_SIG_BLS12381G2_XMD:SHA-256_SSWU_RO_POP_` with `AggregateVerify`; relayers hold a 96-byte G2 `VoteKey` distinct from their `TxKey` and their attestation `PublicKey` | **New surface, and the hardest one.** See below |
| `x/bitcoin` crypto surface | Bitcoin SPV/deposit verification: `witness` 214, `sha256` 33, `secp256k1` 32, `schnorr` 24, `taproot` 4 | Confirms L1 verification is secp256k1/Schnorr-bound; no new scheme |
| `x/locking` crypto surface | `pubkey` 133, `secp256k1` 18, `ecdsa` 1 — validator locking keys | No new scheme |
| `x/consensusfork` crypto surface | Zero crypto references | Not a surface |
| IBC beyond `go.mod` | No IBC wiring in `app/` either | **Confirmed.** The Cosmos IBC light-client hazard does not apply |
| Do `goat-geth`'s 37 commits touch crypto? | **No.** They concentrate in `core/types` (23 files), `eth/tracers`, `eth/catalyst`, `core/goat`; `params/config.go` is the only config-adjacent file. `core/vm/contracts.go` and `crypto/` are untouched | Precompiles are inherited from upstream unchanged, so the fix must come from upstream |
| Does the designated-verifier variant remove pairings? | **No.** `dv_bn254/dv_snark.rs` uses `ark_bn254::G1Projective` with a trapdoor | DV is not a post-quantum path |
| `sect233k1` field size | GF(2^233) confirmed — 233 secret input wires, 233 coefficient bits, 30-byte encodings | Binary Koblitz curve; still Shor-exposed, below secp256k1 classically |
| Is the garbling hash fixed to AES-128? | **No.** `verifiable-circuit` exposes `aes`, `blake3`, `sha2` and `poseidon2` feature flags | Softens the AES point: the PRF is a build-time choice. `LABEL_SIZE = 16` is the parameter that matters, and it is fixed |

### The relayer's BLS vote key is the hardest single item

This did not appear in earlier drafts and it changes the relayer recommendation.
The relayer carries **three** distinct key types, not one:

| Key | Scheme | Purpose |
| --- | --- | --- |
| `PublicKey` (`oneof`) | secp256k1 **or** Schnorr | attestation / proposals |
| `TxKey` | secp256k1 | transaction authorisation |
| `VoteKey` | **BLS12-381 G2, 96-byte compressed** | voting, verified via `AggregateVerify` |

Earlier drafts recommended "add ML-DSA-65 to the `PublicKey` `oneof`". That
remains correct and remains the cheapest first move — but it addresses only the
attestation key. The vote key is a different problem, and a much harder one,
because **its value is aggregation**. BLS lets N relayer votes verify as one
48-byte signature. No standardised post-quantum signature aggregates: ML-DSA and
SLH-DSA have no aggregation, so replacing BLS naively turns one signature into
N, at 3309 bytes each. For twenty relayers that is roughly 66 KB where there was
48 bytes.

**Aggregation is confirmed in use, not merely available.** An earlier draft
inferred this from the presence of an aggregate API, which is weaker evidence
than it sounds. The call site is `x/relayer/keeper/proposal.go:66`:

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
carrying a Groth16 proof and garbled circuits garbling a Groth16 verifier — the
third instance in this one stack, and the first that would be a *prospective*
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

### Follow-up checks, 2026-07-31

| Checked | Result |
| --- | --- |
| Is BLS aggregation actually *used*, or just available? | **Used.** `x/relayer/keeper/proposal.go:66` calls `AggregateVerify`, resolving to `FastAggregateVerify` over a bitmap-selected voter set against `relayer.Threshold()`. Upgraded from inference to a call site |
| Could `leanVM` aggregate BLS instead? | **No.** Zero `bls`/`pairing`/`bls12` files in its tree; it is KoalaBear, Poseidon, WHIR and XMSS with a `rec_aggregation` crate. And proving a BLS verification inside a hash-based zkVM proves a true statement about a broken primitive |

### Consistency re-check of the base-layer sections, 2026-07-31

The Bitcoin and Ethereum sections were written first and re-checked against
primary sources after the rest of the report was complete. Two corrections
followed, both understatements rather than errors of fact.

| Checked | Result |
| --- | --- |
| Does a post-quantum signature BIP exist yet? | **No.** The BIPs index still lists exactly one post-quantum entry, BIP-361 itself. The core finding holds |
| BIP-360 status and version | Unchanged: `Status: Draft`, v0.12.0, `Layer: Consensus (soft fork)` |
| Is the 34% figure really BIP-361's? | Yes, line 46: "As of March 1, 2026, over 34% of all bitcoin have revealed a public key on-chain" |
| BIP-361 phase timings | Phase A at 160,000 blocks (~3 years); Phase B 2 years after Phase A |
| Is Phase B a freeze? | **No — correction.** It encumbers legacy spends with a rescue protocol based on BIP-32 hardened-derivation knowledge, verified by ZK-STARK or commit/reveal. Only abandoned keys and P2PK, which has no known knowledge asymmetry, become unspendable |
| Is Ethereum's plan aggregation-only? | **No — correction.** The report omitted the account track: the emergency hard fork with STARK preimage recovery, EIP-7702 (Final, Pectra) and EIP-8141 (Draft, created 2026-01-29) |

The second correction also surfaced something the report had missed entirely,
now section 7: both chains independently chose the same rescue technique.

## Remaining open items

Honestly short, and none of them block the recommendations:

- The 37 `goat-geth` commits were classified by file area, not read line by
  line. Nothing in the changed areas is cryptographic, but a deliberate check of
  `core/types` changes against consensus-critical serialisation would close it
  fully.
- Which deployed L2 contracts call the `0x06`–`0x08` and `0x0a` precompiles is
  not enumerated. That inventory needs chain state, not source, and it is the
  one item on this list with a deadline: immutable callers can be identified now
  but never migrated later.
- Ziren's `feat/lthash` prototype has not been reviewed here for completeness or
  merged upstream; whether it covers every ECMH use site is unchecked.
- No second L2 has been examined, so Part I's taxonomy is structural reasoning
  supported by one case, not a survey.

## References

Primary sources, all verified live on 2026-07-31.

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
- `GOATNetwork/goat` — <https://github.com/GOATNetwork/goat>
- `GOATNetwork/goat-geth` — <https://github.com/GOATNetwork/goat-geth>
- `GOATNetwork/bitvm2-node` — <https://github.com/GOATNetwork/bitvm2-node>
- `GOATNetwork/bitvm2-gc` — <https://github.com/GOATNetwork/bitvm2-gc>
