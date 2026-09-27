# Transfer

The simplest Cardano payment: move ADA from one address to another. A sender picks an amount, the receiver gets exactly that amount, and the sender pays the network fee and keeps the change.

This tx3 is a reference protocol for consumer tooling and assistants that prepare transactions on a user's behalf. It carries no scripts, datums or tokens, so it exercises the full preparation path — argument encoding, input selection, fee calculation and change — without any on-chain deployment.

## Overview

`transfer` spends UTxOs from the sender's address that cover `quantity` plus the fee, pays `quantity` to the receiver, and returns the remainder to the sender. Both parties are ordinary payment addresses supplied by the caller for every transaction; nothing is bound per network.

## Transactions

| Transaction | Description |
|---|---|
| `transfer` | Send `quantity` lovelace from the sender to the receiver, returning change to the sender |

## Important considerations

- **Amounts are in lovelace.** `quantity` is an integer number of lovelace (1 ADA = 1,000,000 lovelace). The receiver output must also satisfy the ledger's minimum UTxO value (roughly 1 ADA for a plain ADA output).
- **ADA only.** Native tokens held by the sender stay in the change output; this protocol does not send them.
- **The sender pays the fee.** The resolver selects sender UTxOs covering `quantity + fees`; if the sender cannot cover both, resolution fails.
- **No deployment inputs.** There is no `env` block and no `.env.*` file. Every network profile is empty and works the same way; the network is chosen by where the transaction is resolved.
- **Unsigned result.** The transaction still has to be signed by the sender's key and submitted.

## Caller preparation

| Parameter | Source |
|---|---|
| `quantity: Int` | Amount to send in lovelace. Convert from ADA by multiplying by 1,000,000. |
| `sender` | The paying address, bech32 (`addr1…` on mainnet, `addr_test1…` on testnets), on the network being used. It must hold `quantity` plus the fee. |
| `receiver` | The destination address, bech32, on the same network as the sender. |

## References

- **Source shape:** the `trix init` project template ([`tx3-lang/trix`](https://github.com/tx3-lang/trix))
- **Language:** [Tx3 documentation](https://docs.txpipe.io/tx3)
