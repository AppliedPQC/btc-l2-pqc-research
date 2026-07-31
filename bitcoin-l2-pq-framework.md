# The Bitcoin L2 post-quantum problem

*Framework note, 2026-07-31. Derived from primary-source analysis of one L2
stack in depth (see [the GOAT case study](case-study-goat.md)) and of the three
base layers such a stack sits on (see
[blockchain-pqc-migration.md](blockchain-pqc-migration.md)). Section 5 states
plainly which parts are general and which are a hypothesis awaiting a second
case study.*

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
| 3 | Bridge proof system | Groth16, PLONK (pairing-based) or STARK (hash-based) | **depends** | **the L2** |
| 4 | Bridge data commitments | Merkle / Winternitz / Lamport | **no — hash-based** | already safe |
| 5 | L2 consensus keys | ed25519, BLS12-381 | yes | **the L2**, via its consensus stack |
| 6 | L2 execution accounts | secp256k1 ECDSA | yes | the L2, at tooling cost |

Two rows carry most of the signal.

**Row 3 is where severity is highest.** A broken *signature* scheme lets an
attacker steal specific funds. A broken *proof system* lets an attacker forge
the bridge's notion of truth — minting without deposit, or withdrawing without
right. Pairing-based systems (Groth16, PLONK over BN254/BLS12-381) fall to Shor
exactly as ECDSA does. Hash-based systems (STARKs) do not. **Whether an L2's
bridge is quantum-sound is therefore decided by a proof-system choice usually
made for performance reasons, years before anyone asked the question.**

**Row 4 is the good news, and it is frequently missed.** Bridge constructions
that commit data via Merkle trees, or that carry values through Bitcoin script
using Winternitz/Lamport one-time signatures, are already post-quantum on the
most conservative assumption available — the same assumption behind SLH-DSA. In
the one stack examined closely, this turned out to be true of the entire
Bitcoin-side commitment layer, which reframed the migration from a rewrite into
a contained swap.

## 3. Ordering the work

Ownership and severity, not novelty, should set the order:

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

## 4. Recurring traps

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
- **Interoperability makes migration a coordination problem.** Where light
  clients verify a counterparty's signatures with their own compiled-in crypto,
  enabling a new key type before counterparties can verify it breaks
  connectivity — and the failure is silent at upgrade time.

## 5. What is general and what is not

Sections 1 and 2 follow from the *structure* of a Bitcoin L2 — a chain that
settles to a base layer it does not control, runs borrowed consensus, and
operates its own bridge. That structure is shared across the category, so the
surface taxonomy and the ownership argument should transfer.

Sections 3 and 4 are grounded in **one** stack examined in depth plus the
base-layer survey. The ordering in section 3 reflects a judgement about value
and severity that I believe generalises, but it has not been tested against a
second L2. The traps in section 4 are observations, not a survey.

**The open work is a second and third case study** — a rollup with a different
bridge design, and an L2 with a non-Cosmos consensus stack — to find out which
of these findings are properties of the category and which are properties of one
implementation. Until then, treat sections 3 and 4 as a well-grounded hypothesis
rather than an established result.
