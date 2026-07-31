# Post-Quantum Migration in Bitcoin, Ethereum and Cosmos: three chains, three different problems

*Survey compiled 2026-07-31. Every claim below is sourced from a primary
document — the BIP text, the SDK changelog, the project repository — rather
than from secondary coverage. Where a widely repeated secondary claim proved
wrong, that is recorded.*

## Abstract

Bitcoin, Ethereum and Cosmos are usually discussed as though they face one
shared post-quantum problem and differ only in how fast they are moving. The
primary sources do not support that reading. The three are solving structurally
*different* problems, and their relative progress inverts the usual assumption:
the chain with the largest exposure has specified no post-quantum signature
scheme at all, while the least-discussed of the three has already shipped one.
Bitcoin's work is about *key exposure* and the governance of a sunset; Ethereum's
is about *aggregation* at validator scale; Cosmos's is about *key-type
negotiation* across an interoperable ecosystem. Read as one race, the picture is
misleading; read as three problems, each roadmap makes sense.

## 1. Research questions

- **RQ1.** What mechanism does each chain actually propose — which scheme, at
  which layer — and how far has each moved from discussion to specification to
  shipped code?
- **RQ2.** What constraints drive the designs, and why do the three diverge?
- **RQ3.** What remains unresolved?

## 2. Method

Five search perspectives (per-chain roadmaps; cross-chain risk analysis;
standards bodies; implementation repositories; critical commentary). Every
candidate was then verified against a primary artifact: BIP text from
`bitcoin/bips`, `UPGRADING.md` from `cosmos/cosmos-sdk`, repository metadata
from the GitHub API. Secondary coverage was used only to *locate* sources,
never as evidence.

Two corrections came out of that discipline, both cases where secondary
coverage was confidently wrong:

| Claim in secondary coverage | Primary source says |
| --- | --- |
| Cosmos ML-DSA-65 landed in SDK **v0.54** | `UPGRADING.md`: **v0.55** |
| BIP-360 is Bitcoin's post-quantum *signature* proposal | BIP-360: "this proposal **does not include** the introduction of post-quantum signature schemes" |

A third check ran in the other direction. The Cosmos changelog states ML-DSA-65
sizes as 1952-byte public keys and 3309-byte signatures; recomputing those from
our own ACVP-verified FIPS 204 implementation gives exactly 1952 and 3309. The
documentation and an independent implementation agree.

## 3. Taxonomy: what problem is each chain actually solving?

| | **Bitcoin** | **Ethereum** | **Cosmos** |
| --- | --- | --- | --- |
| Core problem | public-key exposure on an immutable ledger | signature size at validator scale | key-type negotiation across chains |
| Layer attacked first | output type / script | consensus signatures | validator consensus keys |
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
exist. The exposure is not hypothetical: BIP-361 states that as of 1 March 2026
over 34% of all bitcoin have revealed a public key on chain.

## 5. Ethereum: an aggregation problem

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

## 7. Synthesis

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

## 8. Open problems

1. **Bitcoin's missing signature BIP.** BIP-361 formally requires it and it does
   not exist. Until it does, Bitcoin's sunset has no destination.
2. **The freeze dispute.** A pre-announced sunset renders unmigrated coins —
   including provably unmigratable P2PK — unspendable. Genuinely unresolved.
3. **Short-exposure attacks.** BIP-360 explicitly does not address the window
   between broadcast and confirmation.
4. **IBC coordination at ecosystem scale.** No published mechanism sequences a
   PQ key-type rollout across a large IBC graph without breaking connectivity.
5. **Nothing is on mainnet by default.** Bitcoin's BIPs are Draft, Ethereum's
   code is prototype and self-described as unaudited, and Cosmos's key type is
   opt-in and off by default.

## 9. Answers to the research questions

**RQ1.** Bitcoin: a Draft output type that removes key-path exposure, plus a
Draft informational sunset — and *no* PQ signature scheme, by the BIP's own
words. Ethereum: hash-based signatures aggregated in a purpose-built zkVM,
public prototypes, nothing on mainnet. Cosmos: ML-DSA-65 shipped in SDK v0.55 as
an opt-in consensus key type, off by default.

**RQ2.** They diverge because their binding constraints differ — governance for
Bitcoin, validator-scale signature size for Ethereum, cross-chain heterogeneity
for Cosmos. Each design is a reasonable response to its own constraint, which is
why comparing them on a single "who is ahead" axis misleads.

**RQ3.** Unresolved: Bitcoin's absent signature BIP and the freeze dispute;
short-exposure attacks; IBC rollout sequencing; and the plain fact that no chain
here has post-quantum signatures securing mainnet funds by default.

## References

All verified live on 2026-07-31.

- BIP-360, *Pay-to-Merkle-Root (P2MR)* — <https://github.com/bitcoin/bips/blob/master/bip-0360.mediawiki>
- BIP-361, *Post Quantum Migration and Legacy Signature Sunset* — <https://github.com/bitcoin/bips/blob/master/bip-0361.mediawiki>
- Bitcoin Optech, *Quantum resistance* — <https://bitcoinops.org/en/topics/quantum-resistance/>
- Cosmos SDK, `UPGRADING.md` — <https://github.com/cosmos/cosmos-sdk/blob/main/UPGRADING.md>
- Cosmos SDK PR #26436, ML-DSA-65 consensus keys — <https://github.com/cosmos/cosmos-sdk/pull/26436>
- Cosmos docs, *Post-quantum keys* — <https://docs.cosmos.network/sdk/latest/keys/post-quantum-keys>
- Cosmos docs, *Migrate a validator to ML-DSA* — <https://docs.cosmos.network/sdk/latest/keys/migrate-validator-ml-dsa>
- Post-Quantum Ethereum — <https://pq.ethereum.org/>
- `leanEthereum/leanVM` — <https://github.com/leanEthereum/leanVM>
- `leanEthereum/leanSig` — <https://github.com/leanEthereum/leanSig>
- `b-wagn/hash-sig`, eprint 2025/055 — <https://github.com/b-wagn/hash-sig>
