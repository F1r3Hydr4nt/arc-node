# arc-node

Planning and integration repo for **signed, secure BSV transaction broadcasting** and **validate-only (spend-simulation)** using [Arcade](https://github.com/bsv-blockchain/arcade), [ARC](https://github.com/bitcoin-sv/arc), and [Teranode](https://github.com/bsv-blockchain/teranode).

## Repository layout

| Directory / File | Description |
|------------------|-------------|
| **arcade/** | [Arcade](https://github.com/bsv-blockchain/arcade) source — next-gen transaction broadcaster for Teranode (replaces ARC). Single binary, SQLite, P2P-first. |
| **arc/** | [ARC](https://github.com/bitcoin-sv/arc) source — original broadcast & status API (Metamorph, BlockTx). Being superseded by Arcade. |
| **teranode/** | [Teranode](https://github.com/bsv-blockchain/teranode) source — BSV Blockchain node with Validator gRPC for UTXO validation. |
| [ARCADE_IMPLEMENTATION_PLAN.md](./ARCADE_IMPLEMENTATION_PLAN.md) | **Active plan** — Arcade + Teranode: validate-only, auth, TLS, merkle path refresh. |
| [IMPLEMENTATION_PLAN.md](./IMPLEMENTATION_PLAN.md) | ARC + Teranode plan (superseded by Arcade plan). |
| [BROADCAST_RESEARCH_AND_PLAN.md](./BROADCAST_RESEARCH_AND_PLAN.md) | Original research on broadcast capabilities across ARC and Teranode. |

## Goals

1. **Validate-only (spend-simulation)** — Send `X-ValidateOnly: true` with a transaction; get validation result (including UTXO spent status from Teranode Validator) without broadcast. If the tx is already mined, receive the current merkle path for BEEF/BUMP refresh after reorgs.
2. **Signed and secure broadcast** — Multi-token/API-key authentication, TLS, optional request signing.
3. **Secure Arcade-to-Teranode** — Teranode Validator gRPC with `x-api-key` auth over TLS.

## Quick links

- [Arcade implementation plan (active)](./ARCADE_IMPLEMENTATION_PLAN.md)
- [ARC implementation plan (superseded)](./IMPLEMENTATION_PLAN.md)
- [Broadcast research & plan](./BROADCAST_RESEARCH_AND_PLAN.md)
- [Arcade repo](https://github.com/bsv-blockchain/arcade)
- [ARC repo](https://github.com/bitcoin-sv/arc)
- [Teranode repo](https://github.com/bsv-blockchain/teranode)
