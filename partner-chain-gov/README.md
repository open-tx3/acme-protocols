# Partner Chain Governance

A [Cardano Partner Chain](https://iohk.io/en/blog/posts/2024/11/21/partner-chains-are-coming-to-cardano/) is a sovereign sidechain that anchors its governance and validator set to Cardano L1. This tx3 covers the **L1-side** transactions that drive a partner chain's governance lifecycle: bootstrapping the on-chain governance authority, managing the validator candidate set, configuring D-parameters, and operating the token reserve that funds rewards.

## Overview

State lives in UTxOs at a family of validators, each authenticated by a dedicated minting policy. A single multi-sig governance policy gates every privileged operation:

- **Version oracle** — the root of trust. Holds the governance policy and validator-script references; updating it rotates the partner chain's governance authority.
- **Governed map** — a generic key-value store under governance oversight (used by the partner chain to publish typed configuration).
- **D-parameter** — controls the maximum number of permissioned vs. registered validator candidates per epoch.
- **Committee candidate** — registry of validator candidates who self-register via stake-pool ownership.
- **Permissioned candidates** — governance-curated list of pre-approved validator candidates.
- **Reserve** — holds the partner chain's token reserve and the settings that control vesting and release.
- **Illiquid circulation supply (ICS)** — receives tokens released from the reserve, gating further circulation.

Every transaction that mutates one of these UTxOs spends the current governance UTxO (or a permissioned-candidates auth UTxO) as proof of authority and re-mints the matching auth token at the new state.

## Transactions

| Transaction | Description |
|---|---|
| `init_governance` | Bootstrap the version-oracle UTxO and mint the initial governance auth token |
| `update_governance` | Rotate the governance authority by spending and re-minting the version-oracle UTxO |
| `insert_key_value` / `update_key_value` / `remove_key_value` | Create, modify, or delete an entry in the governed map |
| `insert_d_parameter` / `update_d_parameter` | Create or update the D-parameter UTxO controlling candidate limits |
| `register_candidate` / `deregister_candidate` | SPO-side flow for a validator candidate to register or withdraw via the committee-candidate validator |
| `insert_permissioned_candidates` / `update_permissioned_candidates` | Governance-side flow to publish or modify the permissioned-candidate list |
| `create_reserve` | Initialize the reserve UTxO with its settings and initial token deposit |
| `deposit_to_reserve` | Add tokens to an existing reserve |
| `release_from_reserve` | Move tokens from the reserve into the illiquid circulation supply per the configured vesting rules |
| `update_reserve_settings` | Update the reserve's release schedule and parameters |
| `handover_reserve` | Hand the reserve over (e.g. to a successor governance authority) |

## Important considerations

- **L1-only scope.** This protocol covers the Cardano-side transactions only. Partner-chain block production, sidechain consensus, and cross-chain message passing are handled off this surface.
- **Governance UTxO is consumed on every privileged tx.** Almost every transaction here spends the current version-oracle UTxO as proof of authority and re-mints it at the updated state. Concurrent governance actions will conflict and must be serialised by the caller.
- **Multi-sig policy lives in `env`.** The active governance policy script, its hash, and its current UTxO reference are configured through the `env { ... }` block — rotating governance means updating the env (and running `update_governance`), not changing tx code.
- **Script references are passed by `UtxoRef` from env.** Validator and minting scripts are published as on-chain reference scripts. Their `UtxoRef`s live in env and are loaded as `reference` inputs per transaction; this protocol does **not** inline script bodies.
- **Candidate flow is two-sided.** `register_candidate` / `deregister_candidate` are signed by the SPO from `candidate_address`, while `insert_permissioned_candidates` / `update_permissioned_candidates` are signed by governance from `payment_address` and require the governance UTxO.
- **Reserve release is gated by settings, not callers.** `release_from_reserve` enforces the vesting/release schedule encoded in the reserve datum; the caller chooses the amount but the on-chain validator rejects releases that exceed what the settings allow at the current slot.

## Caller preparation

Most parameters are populated from the active partner-chain configuration in `env`. Per-transaction parameters callers typically need to supply:

| Concern | Source |
|---|---|
| **Governance UTxO reference** | `env.governance_utxoref` — updated after every governance-mutating tx; the caller must track the latest. |
| **Governance policy / script hashes** | `env.governance_policy_hash`, `env.governance_policy_script`, `env.governance_policy_utxoref` — set once at bootstrap, rotated by `update_governance`. |
| **Validator and policy script references** | All `*_validator_script` / `*_policy_script` / `*_validator_hash` / `*_policy_hash` fields in `env` — published once at bootstrap and read every tx. |
| **Asset names** | `env.*_asset_name` — partner-chain-specific token names for governance, governed map entries, D-parameter, permissioned candidates, reserve auth, and ICS authority tokens. |
| **Candidate registration data** | `CandidateRegistration` and `PermissionedCandidate` payloads (stake ownership signatures, sidechain keys) must be assembled by the SPO / governance caller from off-chain partner-chain state. |
| **Reserve settings** | `ReserveSettings` (token, vesting schedule, distribution parameters) computed off-chain from the partner chain's economic policy. |
| **Signing keys** | `env.payment_key` for governance-side transactions; `env.candidate_key` for SPO-side candidate registration. |

## References

- **Upstream tx3 source:** [`txpipe/metis`](https://github.com/txpipe/metis)
- **Partner Chains overview:** [IOG announcement](https://iohk.io/en/blog/posts/2024/11/21/partner-chains-are-coming-to-cardano/)
- **Partner Chains toolkit:** [`input-output-hk/partner-chains`](https://github.com/input-output-hk/partner-chains)
