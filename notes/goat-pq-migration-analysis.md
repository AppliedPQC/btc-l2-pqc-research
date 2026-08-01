# GOAT Network: a post-quantum migration analysis

*Analysis of [`GOATNetwork/goat`](https://github.com/GOATNetwork/goat) — "GOAT
Decentralized Sequencer" — as of 2026-07-31, against commit state at
`pushed_at 2026-07-28`. Every architectural claim below is read from the
repository, not inferred from marketing.*

## 1. What GOAT actually is, cryptographically

GOAT is a Bitcoin L2 sequencer that stacks three ecosystems, and therefore
inherits three separate quantum exposures:

| Layer | Stack | Signature scheme | Who controls the fix |
| --- | --- | --- | --- |
| Consensus | CometBFT v0.38.25 + Cosmos SDK **v0.53.8** (forked as `goat-cosmos-sdk`) | ed25519 validator keys | **GOAT** |
| Bridge attestation | `x/relayer` | secp256k1 **or** Schnorr (protobuf `oneof`) | **GOAT** |
| Execution | `go-ethereum` v1.16.7 (forked as `goat-geth` v0.4.1) | secp256k1 ECDSA accounts | GOAT, but diverging from EVM tooling has costs |
| Bitcoin peg | `btcd` / `btcec` | secp256k1, Schnorr on Bitcoin L1 | **Bitcoin, not GOAT** |

Modules: `bitcoin`, `consensusfork`, `goat`, `locking`, `relayer`.
`gnark-crypto` is present (BLS12-381 / pairings) as an indirect dependency.

The single most important structural fact: **three of these four surfaces are
GOAT's to fix, and the fourth is not.** Framing the goal as "make GOAT
post-quantum safe" obscures that. The honest framing is *harden what you own,
and treat the L1 peg as residual risk with a contingency plan*.

## 2. The highest-value change: relayer attestation keys

`x/relayer/types/pubkey.go` defines the bridge trust root as a protobuf `oneof`:

```go
const (
    secp256k1Type byte = iota
    schnoorType
)
```

with `PublicKey_Secp256K1` and `PublicKey_Schnorr` variants. This is the set of
keys that attests to Bitcoin-side events; compromising a threshold of it is
equivalent to compromising the peg. It is the most valuable target on the system
and it is entirely under GOAT's control.

**Why this is the best first move.** The `oneof` is already an extensibility
point. Adding a third variant — `PublicKey_MlDsa65` — is an extension of an
existing pattern rather than a redesign, and it can be rolled out with dual
attestation (classical *and* PQ signatures required during transition), which
gives a safe rollback at every step.

**The concrete blocker.** `VerifySign` currently gates on a fixed length:

```go
// note: msg is 32 bytes, sig is 64 bytes
if len(msg) != sha256.Size || len(sig) != schnorr.SignatureSize {
    return false
}
```

An ML-DSA-65 signature is **3309 bytes** and its public key **1952 bytes**
(both confirmed against our own ACVP-verified FIPS 204 implementation). That
length check must become per-variant before any PQ key type can verify. It is a
small change, but it is load-bearing and easy to miss: today the function is
structurally incapable of accepting a post-quantum signature.

Expect the encoded size of an attestation to grow by roughly 50x. Whether that
matters depends on attestation frequency and on how relayer signatures are
stored and gossiped — worth measuring before committing to a threshold size.

## 3. Consensus keys: the path exists, but the fork is in the way

Cosmos SDK **v0.55** registers ML-DSA-65 (FIPS 204) as a validator consensus key
type: the `cosmos.crypto.mldsa65` proto package, Amino routes, an `x/auth`
`SigVerifyCostMlDsa65` parameter, and `--consensus-key-algo ml_dsa_65` all ship
enabled, with the change itself opt-in behind
`genesis.consensus_params.validator.pub_key_types`.

GOAT is on **v0.53.8**, and — the complication — on a *fork*,
`GOATNetwork/goat-cosmos-sdk`, via a `replace` directive. So this is not a
version bump; it is a fork rebase across two minor versions, and the PQ work
cannot start until that rebase is done. **That makes the SDK rebase the critical
path for consensus-layer PQ, and it should be scheduled as such rather than
discovered later.**

Once rebased, the migration is unusually gentle by blockchain standards: add
`ml_dsa_65` to `pub_key_types` by consensus-params update, then have validators
rotate keys (rotation ships enabled in the same SDK release). No new genesis, no
flag day.

The cost lands on the consensus hot path: every validator signs every block, and
keys go 32 → 1952 bytes with signatures 64 → 3309. CometBFT's `MaxSignatureSize`
and per-validator `MaxCommitSigBytes` were raised upstream to accommodate this;
GOAT would need to revisit `consensus_params.block.max_bytes` and gossip framing
limits with its own validator count.

**One risk that does *not* apply here.** The Cosmos changelog's sharpest warning
is that enabling a new consensus key type breaks IBC light clients on
counterparty chains that cannot verify it — packet flow stops and the client
expires. `ibc-go` does not appear in GOAT's `go.mod`, so this hazard appears not
to apply. That is worth confirming against deployment reality rather than
`go.mod` alone, but if GOAT is genuinely non-IBC, it avoids the single most
awkward coordination problem in Cosmos PQ migration.

## 4. Execution layer: inherits Ethereum's unsolved problem

The `goat-geth` fork means accounts are secp256k1 ECDSA, exposed on first spend
like any EVM account. Ethereum's own answer — account abstraction as the
migration vehicle, with a custom signature slot — has not shipped as a
post-quantum mechanism, and its consensus-layer plan (hash-based signatures
aggregated in a zkVM) is prototype-stage.

Because GOAT controls its geth fork, it *could* move ahead of upstream here.
Whether it *should* is a product decision, not a cryptographic one: diverging
account semantics from mainline EVM breaks wallet and tooling compatibility,
which for an L2 is often the whole value proposition. My recommendation is to
follow rather than lead on this surface, and spend the migration budget on
layers 2 and 3 where GOAT's control is uncontested.

## 5. The Bitcoin peg: the limit of what GOAT can do

This is the honest boundary. Any BTC secured by a Bitcoin L1 output is protected
by Bitcoin's cryptography on Bitcoin's timeline. As of now, **Bitcoin has no
post-quantum signature scheme specified at all**: BIP-360 (Draft) removes
Taproot's key-path spend so no bare public key is exposed, and states in its own
text that it "does not include the introduction of post-quantum signature
schemes"; BIP-361 (Draft, Informational) proposes a legacy-signature sunset and
formally lists `Requires: TBD Post Quantum Signature BIP` — a document that does
not exist.

Consequences for GOAT:

- No amount of L2 work makes pegged BTC post-quantum safe while it sits in a
  classical Bitcoin output.
- What GOAT *can* do is minimise long exposure on the Bitcoin side: avoid
  address reuse, prefer output types that do not commit a bare public key, and
  avoid long-lived outputs with revealed keys. BIP-360's P2MR, if activated, is
  the eventual destination for peg custody.
- Custody design should be written so the script/key policy can migrate when
  Bitcoin gains PQ signatures, rather than assuming today's scheme is permanent.

## 6. Recommended sequencing

| Phase | Action | Rationale |
| --- | --- | --- |
| 0 | Inventory every signature verification path; add a test asserting no fixed signature-length assumptions remain | The `VerifySign` 64-byte gate shows these assumptions are already load-bearing and invisible |
| 1 | Extend the relayer `PublicKey` `oneof` with ML-DSA-65; make length checks per-variant; roll out with dual attestation | Highest value per unit of control — this is the peg's trust root, and the `oneof` already supports it |
| 2 | Rebase `goat-cosmos-sdk` onto SDK ≥ v0.55; opt into `ml_dsa_65`; rotate validators; re-tune `block.max_bytes` and gossip limits | The mechanism is upstream and gentle, but the fork rebase is the critical path |
| 3 | Track Ethereum account abstraction for execution accounts | Following costs less than diverging from EVM tooling |
| — | Treat the Bitcoin peg as residual risk; minimise key exposure; keep custody policy migratable | Blocked on Bitcoin, not on GOAT |

## 7. What I did not verify

Stated so the analysis is not read as more thorough than it is:

- I read `x/relayer/types/pubkey.go` but did not audit `x/locking`, `x/bitcoin`,
  or `x/consensusfork` for further signature surfaces.
- I did not inspect the `goat-cosmos-sdk` or `goat-geth` forks to see how far
  they diverge, which directly determines the phase-2 rebase cost — that is the
  single biggest unknown in this plan.
- The BLS12-381 usage (2 files, via `gnark-crypto`) was not traced; if it is
  load-bearing for aggregation, it is a fifth exposed surface, since pairing
  curves fall to Shor exactly as ECDSA does.
- The IBC conclusion rests on `go.mod` alone.

Each of these is a bounded follow-up, and the third is the one I would look at
first.
