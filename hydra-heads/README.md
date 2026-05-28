# Hydra Heads

Hydra is Cardano's layer-2 scaling solution: a family of isomorphic state-channel protocols that let a fixed set of participants run transactions off-chain at high throughput and low latency, while preserving Cardano's settlement guarantees on the main chain. A *head* is a single instance of the protocol — a private off-chain ledger shared by N participants, opened by locking funds on L1, run via direct peer-to-peer message passing, and finalised back to L1 when the participants are done.

Hydra is built and maintained by Input Output's [cardano-scaling](https://github.com/cardano-scaling) team. Typical applications include payment channels, micropayments, gaming and ticketing rails, and any workload where Cardano's ~20 s L1 settlement is too slow and the participant set is small enough to coordinate directly. The protocol is described in the IACR paper [*Hydra: Fast Isomorphic State Channels*](https://eprint.iacr.org/2020/299.pdf) and documented at [hydra.family](https://hydra.family/head-protocol/).

This tx3 models the **L1 transactions that drive a head through its lifecycle** — `init`, `commit`, `collectCom`, `abort`, `close`, `contest`, `fanout`, plus the incremental `deposit` / `increment` / `decrement` / `recover` family. The off-chain L2 message protocol and snapshot signing happen inside `hydra-node`; tx3 only describes the *off-chain construction* of the L1 transactions.

## Overview

A head moves through four phases — Initial, Open, Closed, Final — gated by a state-thread (ST) token minted by `HeadPolicy` and one participation token (PT) per participant. The state UTxO carrying the ST sits at the `µHead` validator; PTs sit at `µInitial` until commit time, at `µCommit` between commit and `collectCom`, and back on the head state UTxO once the head is open. Funds queued for incremental commits sit at `µDeposit`.

Lifecycle transitions:

- **Initial → Initial:** `init` opens the head; each `commit` moves one participant's PT from `µInitial` to `µCommit` (repeated up to N times, once per participant).
- **Initial → Open:** `collect_com` aggregates every `µCommit` UTxO into the head state UTxO.
- **Initial → Final:** `abort` cancels the head before opening, burning ST + all PTs and refunding any pending commits.
- **Open → Open:** `deposit` / `recover` manage incremental commit UTxOs at `µDeposit`; `increment` pulls a confirmed deposit into the head; `decrement` releases funds to L1 without closing.
- **Open → Closed:** `close` posts a confirmed L2 snapshot and starts the contestation period (`deadline = now + cp`).
- **Closed → Closed:** `contest` posts a newer snapshot before `deadline`, appending the contester.
- **Closed → Final:** `fanout` runs after `deadline`, distributing the final L2 UTxO set back to L1 and burning ST + all PTs.

The on-chain validators (`µHead`, `µInitial`, `µCommit`, `µDeposit`, and the `HeadPolicy` minting policy) live in [`hydra-plutus`](https://github.com/cardano-scaling/hydra/tree/master/hydra-plutus) and are referenced from `main.tx3` via CIP-31 reference scripts.

## Transactions

| Transaction | Phase | Description |
|---|---|---|
| `init` | → Initial | Any participant announces head parameters (participants, contestation period). Consumes a seed UTxO; HeadPolicy mints the ST and one PT per party. |
| `commit` | Initial | A participant moves their PT from `µInitial` to `µCommit`, attaching the L1 UTxOs they want inside the head. |
| `collect_com` | Initial → Open | Aggregates every `µCommit` UTxO into the head state UTxO. The head is now open. |
| `abort` | Initial → Final | Cancels a head before opening: burns ST + all PTs, refunds any pending commits. |
| `deposit` | Open | User locks new funds at `µDeposit` with a deadline, awaiting inclusion in an L2 snapshot. |
| `increment` | Open | After L2 consensus, the head claims a deposit and bumps `snapshot_number` + `utxo_hash`. |
| `recover` | Open | If a deposit isn't claimed by its deadline, the depositor reclaims it. |
| `decrement` | Open | Produces the L1 output for funds the head agreed to release, without closing it. |
| `close` | Open → Closed | Posts a confirmed L2 snapshot on-chain and starts the contestation period (`deadline = now + cp`). |
| `contest` | Closed | Any participant posts a more recent snapshot before `deadline`. Validator enforces newer-snapshot semantics and appends the contester. |
| `fanout` | Closed → Final | After `deadline`, distributes the final L2 UTxO set back to L1 and burns ST + all PTs. |

## Important considerations

- **Identifying tokens.** The **ST** (state-thread token) is present on `init`, `collect_com`, `increment`, `decrement`, `close`, `contest`, `fanout` outputs; it uniquely identifies a head. The **PT** (participation token) — one per participant — is present on `commit`; burned on `abort` / `fanout`.
- **HeadPolicy is parameterised per head.** Its currency symbol (== `head_id`) is unique per head because the policy is parameterised by a seed `OutputRef`. The caller computes `head_id` off-chain before invoking `init` and passes it as a parameter to every subsequent tx.
- **Reference scripts deployed once per network.** `head_script_ref`, `initial_script_ref`, `commit_script_ref`, `deposit_script_ref` live in the env (sourced from upstream [`hydra-node/networks.json`](https://github.com/cardano-scaling/hydra/blob/master/hydra-node/networks.json)).
- **PT bundles instead of per-participant outputs.** Real Hydra emits one `µInitial` UTxO per participant. tx3 models the aggregate via a single output and a `pt_count` parameter. Same simplification applies to `collect_com` (one `µCommit` input in tx3 vs. N on-chain) and `fanout` (one consolidated recipient vs. one output per snapshot UTxO).
- **Validity vs. datum deadlines.** `recover` and `fanout` need `validity.since_slot ≥ deadline`, but tx3 can't yet read the datum into a `validity` block, so the caller passes `deadline_slot` explicitly.
- **`initial_contesters` parameter.** `close` requires an empty contester list, but tx3 has no empty-list literal in this position, so the caller passes an empty `List<Bytes>` at invoke time.
- **ST token name is protocol-wide.** `st_token_name` lives in env and is the same on every network (`"HydraHeadV1"` / hex `4879647261486561645631`).

## Caller preparation

Off-chain work required before invoking any lifecycle transaction. Per-parameter documentation is rendered from `main.tx3` docstrings — this section only covers what the caller must produce, query, or compute outside the tx3 module itself.

### Computing `head_id`

The `HeadPolicy` minting policy is parameterised by the `seed_ref` consumed in `init`. Apply the seed `OutputRef` to the policy template and take the blake2b-224 hash — that's the head's currency symbol and the value reused as `head_id` in every subsequent transaction.

### Sourcing reference scripts

`head_policy_ref` (passed to `init`, `abort`, `fanout`) and the four validator reference scripts in env come from the per-network Hydra deployment table at [`hydra-node/networks.json`](https://github.com/cardano-scaling/hydra/blob/master/hydra-node/networks.json).

### Pre-computing min-UTxO lovelace

Every script output needs a lovelace amount satisfying the protocol's min-UTxO rule (`head_utxo_lovelace`, `initial_utxo_lovelace`, `commit_utxo_lovelace`, `deposit_utxo_lovelace`, `head_out_lovelace`). Compute against current protocol params.

### Querying on-chain UTxOs

Each lifecycle step needs the live UTxOs it consumes: the `µHead` state UTxO (`head_ref`), a specific participant's `µInitial` PT bucket (`initial_pt_ref`, `initial_ref`), pending `µCommit` UTxOs (`commit_ref`), pending `µDeposit` UTxOs (`deposit_ref`). Look them up by validator address and identifying token (ST for the head state, PT for the others).

### Canonical CBOR for `committed` / `deposit`

`commit.committed_cbor` and `deposit.deposit_cbor` are the canonical CBOR encodings of the L1 UTxO bundles being placed inside the head. The on-chain validator hashes these to verify inclusion in the snapshot at `collect_com` / `increment`.

### Snapshot artifacts from `hydra-node`

`close`, `contest`, `increment`, `decrement`, and `fanout` carry snapshot data produced by L2 consensus inside `hydra-node`: `snapshot_number`, `utxo_hash` (Merkle root of the L2 UTxO set), `signature` (multi-party sig), and on `collect_com` the initial `initial_utxo_hash`. These all come from outside this tx3.

### Self-reference in `commit`

The `ViaCommit` redeemer carries the `OutputRef` of the `µCommit` UTxO that the same transaction produces. Predict the new tx hash and output index (e.g. by building the body first and hashing it) before setting `new_commit_ref_tx` / `new_commit_ref_idx`.

### Deadline-driven validity bounds

`recover` and `fanout` need `validity.since_slot ≥ datum.deadline`, but tx3 can't read the datum into the validity block. Pass `deadline_slot` equal to the datum's deadline.

### List-literal workarounds

- `close.initial_contesters` — pass `[]` at invoke time (no empty-list literal in this position).
- `contest.contesters` — pass the prior contester list with the caller's vkey hash appended.

### `pt_count` must match the participant set

Real Hydra would derive PT count from the participant list, but tx3 has no `length()` builtin. Pass `pt_count` explicitly; it must equal `participants.length` on the mint side and the number of outstanding PTs on the burn side (`abort.pt_burn_count`, `fanout.pt_count`).

## References

- **Smart contracts:** PlutusV3 — [cardano-scaling/hydra-plutus](https://github.com/cardano-scaling/hydra/tree/master/hydra-plutus)
- **Source:** [cardano-scaling/hydra](https://github.com/cardano-scaling/hydra)
- **Protocol overview:** [hydra.family/head-protocol](https://hydra.family/head-protocol/docs/protocol-overview)
- **Architecture:** [hydra.family/head-protocol/dev/architecture](https://hydra.family/head-protocol/docs/dev/architecture)
- **Incremental commits / decommits:** [hydra.family/head-protocol/dev/protocol](https://hydra.family/head-protocol/docs/dev/protocol)
- **Paper:** [Hydra: Fast Isomorphic State Channels (IACR ePrint 2020/299)](https://eprint.iacr.org/2020/299.pdf)
- **Reference-script deployment table:** [hydra-node/networks.json](https://github.com/cardano-scaling/hydra/blob/master/hydra-node/networks.json)
- **tx3 language spec:** [tx3-lang/tx3 v1beta0](https://github.com/tx3-lang/tx3/tree/main/specs/v1beta0)
