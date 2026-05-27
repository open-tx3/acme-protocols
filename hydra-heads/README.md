# hydra-heads

Tx3 models of the Cardano L1 transactions that drive a [Hydra Head](https://github.com/cardano-scaling/hydra)
through its lifecycle.

Tx3 only describes the *off-chain construction* of these transactions; the
on-chain validators (`µHead`, `µInitial`, `µCommit`, `µDeposit`, and the
`HeadPolicy` minting policy) live in [`hydra-plutus`](https://github.com/cardano-scaling/hydra/tree/master/hydra-plutus)
and are referenced from `main.tx3` via reference scripts.

## Head lifecycle

```
Initial ──init──▶ Initial ──commit─▶ Initial ──collectCom─▶ Open ──close─▶ Closed ──fanout─▶ Final
                          (×N)                                   │  ▲                ▲
                                                                 │  └─contest─┘      │
                                                                 │                   │
                                                              deposit/                │
                                                              recover/                │
                                                              increment/              │
                                                              decrement               │
                          └─────────────────abort────────────────────────────────────▶│
```

## Transactions

### Core lifecycle

| Tx | Phase | Purpose |
|---|---|---|
| **`init`** | → Initial | Any participant announces head parameters (participants, contestation period). Consumes a seed UTxO; HeadPolicy mints the **State Thread token (ST)** and one **Participation Token (PT)** per party. |
| **`commit`** | Initial | A participant moves their PT from `µInitial` to `µCommit`, attaching the L1 UTxOs they want inside the head. |
| **`collectCom`** | Initial → Open | Aggregates every `µCommit` UTxO into the head state UTxO. The head is now open. |
| **`abort`** | Initial → Final | Cancels a head before opening: burns ST + all PTs, refunds any pending commits. |
| **`close`** | Open → Closed | Posts a confirmed L2 snapshot on-chain and starts the contestation period (`deadline = now + cp`). |
| **`contest`** | Closed | Any participant posts a more recent snapshot before `deadline`. Validator enforces newer-snapshot semantics and appends the contester. |
| **`fanout`** | Closed → Final | After `deadline`, distributes the final L2 UTxO set back to L1 and burns ST + all PTs. |

### Incremental commit / decommit

| Tx | Purpose |
|---|---|
| **`deposit`** | User locks new funds at `µDeposit` with a `deadline`, awaiting inclusion in an L2 snapshot. |
| **`increment`** | After L2 consensus, the head claims the deposit and bumps `snapshot_number` + `utxo_hash`. |
| **`recover`** | If a deposit isn't claimed by `deadline`, the depositor reclaims it. |
| **`decrement`** | Produces the L1 output for funds the head agreed to release, without closing it. |

## Identifying tokens

- **ST (State Thread token)** — present on `init`, `collectCom`, `increment`, `decrement`, `close`, `contest`, `fanout` outputs; uniquely identifies a head.
- **PT (Participation Token)** — one per participant, present on `commit`; burned on `abort` / `fanout`.

## Project layout

- `main.tx3` — the eleven transactions above, plus all shared types, parties and env vars.
- `trix.toml` — project manifest.
- `devnet.toml` — devnet wallets / endpoints.
- `tests/` — invocation tests.

## Modelling notes / known simplifications

These are places where the Tx3 model deviates from the real Hydra
on-chain protocol, called out so future work can refine them:

- **PT bundles instead of per-participant outputs.** Real Hydra emits
  one `µInitial` UTxO per participant (each holding its own PT); we
  model the aggregate via `pt_count`.
- **One `µCommit` input per tx.** `collectCom` / `abort` are written
  for a single committed UTxO. For N participants, replicate the
  `commit_in` block N times.
- **Single fanout recipient.** `fanout` produces one consolidated
  output to `fanout_recipient`. The real fanout produces one output
  per UTxO in the final snapshot.
- **`empty_contesters`.** Tx3 has no empty-list literal in this
  position, so `close` takes `initial_contesters: List<Bytes>` as a
  parameter — pass an empty list at call time.
- **Validity ↔ datum-deadline.** `recover` and `fanout` need
  `validity.since_slot ≥ deadline`, but Tx3 can't yet read the datum
  into the validity block, so the caller passes the slot.

## References

- Hydra docs — [Protocol overview](https://hydra.family/head-protocol/docs/protocol-overview)
- Hydra docs — [Architecture](https://hydra.family/head-protocol/docs/dev/architecture)
- Hydra docs — [Incremental commits / decommits](https://hydra.family/head-protocol/docs/dev/protocol)
- Source — [cardano-scaling/hydra](https://github.com/cardano-scaling/hydra)
- Paper — [Hydra: Fast Isomorphic State Channels (IACR ePrint 2020/299)](https://eprint.iacr.org/2020/299.pdf)
- Tx3 language spec — [tx3-lang/tx3 v1beta0](https://github.com/tx3-lang/tx3/tree/main/specs/v1beta0)
