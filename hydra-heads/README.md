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

### All transactions

| Parameter | Source |
|---|---|
| `head_id: Bytes` | HeadPolicy currency symbol for this head. Computed off-chain by applying the seed `OutputRef` to the policy template; identical for every tx in the head's lifecycle. |
| `pt_count: Int` | Number of participation tokens accompanying the ST (must equal participant count). |

### `init`

| Parameter | Source |
|---|---|
| `seed_ref: UtxoRef` | A spendable UTxO from the caller's wallet, used to make the HeadPolicy currency symbol unique. |
| `head_policy_ref: UtxoRef` | CIP-31 reference script UTxO for the HeadPolicy minting policy. |
| `contestation_period: Int` | Head contestation period in milliseconds; baked into the head parameters. |
| `parties: List<Bytes>` | Hydra signing keys (vkey-style) for the L2 multi-sig. Passed as an array. |
| `participants: List<Bytes>` | Cardano vkey hashes of the L1 participants. Passed as an array. |
| `head_utxo_lovelace: Int` | Min-UTxO lovelace pinned on the `µHead` state UTxO. Pre-computed from protocol params. |
| `initial_utxo_lovelace: Int` | Min-UTxO lovelace pinned on the `µInitial` PT bucket UTxO. |

### `commit`

| Parameter | Source |
|---|---|
| `party: Bytes` | The Hydra signing key of the participant making the commit. |
| `committed_cbor: Bytes` | Canonical CBOR encoding of the L1 UTxOs being committed (constructed externally). |
| `committed_lovelace: Int` | Total ada value of the bundle being committed. |
| `commit_utxo_lovelace: Int` | Min-UTxO lovelace required on the produced `µCommit` output. |
| `initial_pt_ref: UtxoRef` | The `µInitial` UTxO carrying this participant's PT. |
| `new_commit_ref_tx`, `new_commit_ref_idx` | Self-reference for the produced `µCommit` UTxO, baked into the `ViaCommit` redeemer. |

### `abort`

| Parameter | Source |
|---|---|
| `head_policy_ref: UtxoRef` | CIP-31 reference script UTxO for the HeadPolicy. |
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Initial`) being consumed. |
| `commit_ref: UtxoRef` | A pending `µCommit` UTxO to refund. |
| `initial_ref: UtxoRef` | A `µInitial` PT UTxO whose token will be burned. |
| `refund_lovelace: Int` | Lovelace refunded back to the committing party. |
| `refund_recipient: Bytes` | Address (raw bytes) receiving the refunded committed funds. |
| `pt_burn_count: Int` | Number of PTs to burn alongside the ST. |

### `collect_com`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Initial`) being consumed. |
| `commit_ref: UtxoRef` | A `µCommit` UTxO being absorbed into the head. |
| `params: HeadParameters` | Head parameters carried over from the `Initial` datum. |
| `initial_utxo_hash: Bytes` | Merkle root of the initial committed UTxO set. |
| `head_out_lovelace: Int` | Lovelace pinned on the new `µHead` `Open` output. |

### `deposit`

| Parameter | Source |
|---|---|
| `deadline: Int` | POSIX-ms deadline after which the depositor may `recover` the funds. |
| `deposit_cbor: Bytes` | Canonical CBOR encoding of the L1 UTxOs being deposited. |
| `deposit_lovelace: Int` | Ada value of the bundle being deposited. |
| `deposit_utxo_lovelace: Int` | Min-UTxO lovelace on the produced `µDeposit` output. |

### `increment`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Open`) being consumed. |
| `deposit_ref: UtxoRef` | The `µDeposit` UTxO being absorbed into the head. |
| `params: HeadParameters` | Head parameters from the previous `Open` datum. |
| `new_snapshot_number: Int` | New snapshot number after absorbing the deposit. |
| `new_utxo_hash: Bytes` | New Merkle root of the committed UTxO set. |
| `signature: Bytes` | Multi-party signature over the new snapshot. |
| `head_out_lovelace: Int` | Lovelace pinned on the new `µHead` `Open` output, excluding the pulled amount. |
| `pulled_lovelace: Int` | Lovelace pulled in from the deposit. |

### `recover`

| Parameter | Source |
|---|---|
| `deposit_ref: UtxoRef` | The `µDeposit` UTxO being reclaimed. |
| `refund_lovelace: Int` | Lovelace value carried by the deposit; refunded to the depositor. The caller must also ensure the tx's validity lower bound is ≥ the datum's deadline. |

### `decrement`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Open`) being consumed. |
| `params: HeadParameters` | Head parameters from the previous `Open` datum. |
| `new_snapshot_number: Int` | New snapshot number after the decommit. |
| `new_utxo_hash: Bytes` | New Merkle root of the committed UTxO set. |
| `signature: Bytes` | Multi-party signature over the new snapshot. |
| `head_out_lovelace: Int` | Lovelace pinned on the new `µHead` `Open` output. |
| `released_lovelace: Int` | Lovelace being released back to L1. |
| `release_recipient: Bytes` | L1 address (raw bytes) receiving the released funds. |

### `close`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Open`) being consumed. |
| `params: HeadParameters` | Head parameters from the `Open` datum. |
| `snapshot_number: Int` | Snapshot number being posted on close. |
| `utxo_hash: Bytes` | Merkle root of the snapshot's UTxO set. |
| `signature: Bytes` | Multi-party signature over the snapshot. |
| `head_out_lovelace: Int` | Lovelace pinned on the new `µHead` `Closed` output. |
| `deadline: Int` | POSIX-ms contestation deadline (`now + contestation_period`). |
| `initial_contesters: List<Bytes>` | Empty list at call time (tx3 has no empty-list literal here). |

### `contest`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Closed`) being consumed. |
| `params: HeadParameters` | Head parameters from the `Closed` datum. |
| `snapshot_number: Int` | Newer snapshot number being asserted. |
| `utxo_hash: Bytes` | Merkle root of the newer snapshot's UTxO set. |
| `signature: Bytes` | Multi-party signature over the newer snapshot. |
| `contester: Bytes` | Vkey hash of the participant posting this contest. |
| `contesters: List<Bytes>` | Updated contester list (existing entries with `contester` appended). |
| `deadline: Int` | Contestation deadline carried over from the `Closed` datum (unchanged by contest). |
| `head_out_lovelace: Int` | Lovelace pinned on the new `µHead` `Closed` output. |

### `fanout`

| Parameter | Source |
|---|---|
| `head_ref: UtxoRef` | The `µHead` state UTxO (in `Closed`) being consumed. |
| `head_policy_ref: UtxoRef` | CIP-31 reference script UTxO for the HeadPolicy. |
| `fanout_lovelace: Int` | Lovelace materialised back onto L1 from the final snapshot. |
| `fanout_recipient: Bytes` | Address (raw bytes) receiving the consolidated fanout output. |
| `deadline_slot: Int` | Slot at or after which the validator accepts the fanout (≥ contestation deadline). |
| `final_snapshot_number: Int` | Final snapshot number being materialised. |
| `final_utxo_hash: Bytes` | Final Merkle root of the L2 UTxO set. |

## References

- **Smart contracts:** PlutusV3 — [cardano-scaling/hydra-plutus](https://github.com/cardano-scaling/hydra/tree/master/hydra-plutus)
- **Source:** [cardano-scaling/hydra](https://github.com/cardano-scaling/hydra)
- **Protocol overview:** [hydra.family/head-protocol](https://hydra.family/head-protocol/docs/protocol-overview)
- **Architecture:** [hydra.family/head-protocol/dev/architecture](https://hydra.family/head-protocol/docs/dev/architecture)
- **Incremental commits / decommits:** [hydra.family/head-protocol/dev/protocol](https://hydra.family/head-protocol/docs/dev/protocol)
- **Paper:** [Hydra: Fast Isomorphic State Channels (IACR ePrint 2020/299)](https://eprint.iacr.org/2020/299.pdf)
- **Reference-script deployment table:** [hydra-node/networks.json](https://github.com/cardano-scaling/hydra/blob/master/hydra-node/networks.json)
- **tx3 language spec:** [tx3-lang/tx3 v1beta0](https://github.com/tx3-lang/tx3/tree/main/specs/v1beta0)
