# Teranode System Requirements — Investigation Report

This document records the findings from investigating system requirements for running Teranode (cloned in this repo at `teranode/`) on **testnet** and **mainnet**, in **full**, **pruned**, and (where applicable) **UTXO-subset** modes. It also clarifies the relationship between pruned mode and the UTXO set.

**Investigation plan:** See [TERANODE_SYSTEM_REQUIREMENTS_INVESTIGATION_PLAN.md](TERANODE_SYSTEM_REQUIREMENTS_INVESTIGATION_PLAN.md).

---

## Summary

- **Testnet vs mainnet:** Official requirements differ by network and deployment (Docker Compose vs Kubernetes). Mainnet needs substantially more CPU, RAM, and storage than testnet.
- **Full vs pruned:** “Full” means Block Persister is running and within the retention window (default 288 blocks); “pruned” means persister is off or lagging beyond that window, or the pruner uses the `OnBlockMined` trigger. The **published requirement numbers assume a seeded, pruned node**; full mode with Block Persister uses more blob storage for the retention window.
- **Pruned mode and UTXO set:** Pruned mode **keeps the full live UTXO set**. Pruning removes old transaction/blob data and **fully spent parent** UTXO records (Delete-At-Height). It does **not** reduce the set of unspent outputs.
- **UTXO-subset mode:** Teranode does **not** currently support a “UTXO-subset” mode (e.g. storing only UTXOs for certain addresses). There is no such run mode in the codebase or docs.

---

## 1. Requirements matrix (testnet vs mainnet)

All figures below assume a **seeded, pruned node with default retention** unless stated otherwise. Full sync from genesis needs more storage and time (see Section 6).

### Docker Compose (single node)

| Resource     | Mainnet (min) | Mainnet (recommended) | Testnet (min) | Testnet (recommended) |
|-------------|----------------|------------------------|---------------|------------------------|
| CPU         | 8 cores        | 16 cores               | 4 cores       | 8 cores                |
| RAM         | 128 GB         | 256 GB                 | 16 GB         | 32 GB                  |
| Storage     | 1 TB           | 2 TB                   | 64 GB         | 128 GB                 |
| Storage type | NVMe SSD     | NVMe SSD               | SSD           | NVMe SSD               |

**Caveats:** NVMe SSD is strongly recommended for mainnet. HDD is not supported (Aerospike and blob storage IOPS requirements).

### Kubernetes (services + external dependencies)

**Mainnet — Teranode services (per pod):**

| Service           | CPU request | Memory request |
|-------------------|-------------|----------------|
| alertSystem       | 1           | 1Gi            |
| asset             | 1           | 1Gi            |
| blockAssembly     | 1           | 4Gi            |
| blockchain        | 1           | 1Gi            |
| blockValidator    | 1           | 8Gi            |
| legacy            | 4           | 32Gi           |
| peer              | 1           | 1Gi            |
| propagation       | 1           | 1Gi            |
| rpc               | 1           | 1Gi            |
| subtreeValidator  | 1           | 16Gi           |

Optional services (validator, blockPersister, utxoPersister, coinbase): baseline 1 CPU / 1Gi each.

**Mainnet — external dependencies:**

| Component   | CPU    | Memory | Storage    |
|------------|--------|--------|------------|
| Aerospike  | 4 cores| 32 GB  | 400 GB NVMe|
| PostgreSQL | 2 cores| 4 GB   | 50 GB SSD  |
| Kafka      | 2 cores| 4 GB   | 50 GB SSD  |

Shared storage (RWX): 1 TB.

**Testnet — Teranode services:** Baseline 100m CPU / 512Mi memory per service.

**Testnet — external dependencies:**

| Component   | CPU    | Memory | Storage    |
|------------|--------|--------|------------|
| Aerospike  | 2 cores| 8 GB   | 50 GB NVMe |
| PostgreSQL | 1 core | 2 GB   | 20 GB SSD  |
| Kafka      | 1 core | 2 GB   | 20 GB SSD  |

Shared storage (RWX): 50 GB.

### Storage breakdown (mainnet, seeded + pruned, default retention)

| Component   | Storage used | Description                    |
|------------|--------------|--------------------------------|
| Aerospike  | ~400 GB      | UTXO set (~340M records)      |
| Blob storage | ~600 GB    | Transactions and subtrees     |
| PostgreSQL | &lt; 1 GB     | Block headers and chain state |
| Prometheus | ~1 GB        | Metrics (varies with retention) |
| **Total**  | **~1 TB**    |                                |

- **PostgreSQL:** SSD recommended.
- **Blob storage:** SSD recommended (sequential read/write).
- **Aerospike:** NVMe SSD required (high random IOPS for UTXO lookups).

---

## 2. Full vs pruned mode (definition and config)

### Definition (from code)

Logic is in `teranode/util/storage_mode.go` (`DetermineStorageMode`):

- **Full node:** Block Persister is running and its height is within the retention window of the current best height. Default retention is 288 blocks (~2 days at 10‑minute blocks). While the persister stays within this window, recent block/transaction data is retained and the node is considered “full.”
- **Pruned node:** Any of:
  - Block Persister is not running (height 0), or
  - Block Persister lags beyond the retention window, or
  - Pruner is triggered by `OnBlockMined` (pruning happens before persistence, so the node cannot serve as “full”).

So “full” is about **recent history availability** (block persister + retention), not about whether the UTXO set is complete. Both full and pruned maintain the full live UTXO set (see Section 3).

### Config levers

| Setting / behaviour | Effect on full vs pruned |
|--------------------|---------------------------|
| `global_blockHeightRetention` | Default 288. Retention window used to decide if persister is “within range.” Higher = more blob storage, deeper reorg support. |
| Block Persister on/off | Off → always pruned. On and keeping up → full (if trigger is OnBlockPersisted). |
| `pruner_block_trigger` | `OnBlockPersisted` (default) = coordinate with Block Persister; can be full. `OnBlockMined` = prune on block validation; node is always pruned. |
| `pruner_force_ignore_block_persister_height` | If true, pruner uses Block notifications instead of Block Persister height; typically used when Block Persister is not deployed. |

### Requirements impact

- The **official system requirements** are for a **pruned** (seeded) node. Running in **full** mode (Block Persister on, default retention) increases **blob storage** use for the retention window; CPU/RAM in the tables remain the main baseline. Disabling pruning or increasing retention increases storage proportionally.
- Full mode is mainly a storage/retention trade-off: more disk for recent blocks and transaction data so the node can serve catchup and reorg scenarios within the window.

---

## 3. Pruned mode and the UTXO set

**Conclusion: Pruned mode retains the full live UTXO set.** It does not store only a subset of unspent outputs.

- The **UTXO store** (Aerospike on mainnet) holds:
  - The **complete set of unspent outputs** (live UTXO set), and
  - **Spent parent transaction records** that are still within the retention period (marked with Delete-At-Height, DAH).
- The **Pruner** (see `teranode/docs/topics/services/pruner.md`):
  - **Phase 1:** Preserves parents of old unmined transactions (so resubmitted txs can still validate).
  - **Phase 2 (DAH pruning):** Deletes **records** where `deleteAtHeight <= safeHeight` — i.e. **fully spent parent** transaction records — and cleans up associated blob files. It does **not** delete unspent outputs.
- So pruning **reduces blob storage** and **removes spent parent records** from the UTXO store. The **live set of unspent outputs** is unchanged and is still the full set (~340M records on mainnet, ~400 GB in the reference breakdown).

The official system requirements explicitly assume a “seeded, **pruned** node”; that configuration still includes the full UTXO set in Aerospike.

---

## 4. UTXO-subset mode

**Conclusion: Teranode does not support a “UTXO-subset” mode.**

- A search over the local `teranode/` repo for “UTXO subset,” “partial UTXO,” “filtered UTXO,” “watch-only,” and similar concepts found **no** run mode or config that limits which UTXOs are stored (e.g. by address or script).
- Uses of “subset” in the codebase refer to: subsets of outputs in tests, interface subsets (e.g. a subset of an API), RPC “subset of Bitcoin Core methods,” and similar — not to storing a subset of the chain’s UTXO set.
- There is no documented or implemented “light” or “UTXO-filtered” node mode.

**So:** “Pruned” is **not** “UTXO subset.” Pruned = full live UTXO set + pruning of old blobs and spent parent records. A true UTXO-subset mode (e.g. only certain addresses) would be a different, currently unavailable mode.

---

## 5. Sync method and impact on requirements

Teranode supports three synchronization approaches (see [Syncing the Blockchain (Docker Compose)](https://bsv-blockchain.github.io/teranode/howto/miners/docker/minersHowToSyncTheNode/)):

| Method                  | Use case              | Time (typical) | Notes |
|-------------------------|-----------------------|---------------|--------|
| Default network sync (P2P) | Fresh install, no data | **~5–14 days** | Minimum **4 TB** storage with default prune (288 blocks). Full validation from genesis. |
| Legacy SV Node seeding  | Existing BSV node     | ~1 hour       | Requires SV Node export; ~1 TB for export files (temporary), ~10 TB recommended for Teranode data in that guide. |
| Teranode data seeding   | Existing Teranode     | ~1 hour       | Fastest; requires access to existing Teranode data and version compatibility. |

- The **published requirement tables** (1 TB mainnet, 64–128 GB testnet) assume a **seeded** node, not full P2P sync from genesis.
- **Full blockchain sync from genesis** requires **additional temporary storage and significantly more time** (e.g. minimum 4 TB and around 14 days for P2P); requirements may be higher for testnet/mainnet with larger history or higher retention.

---

## 6. OS and network

- **OS:** Ubuntu 24.04 LTS is recommended and tested. Other Linux distributions may work for Docker/Kubernetes but are untested.
- **Network (QUIC):** Teranode’s P2P service uses QUIC (libp2p). Larger UDP buffers are required on all hosts running Teranode:

  ```text
  net.core.rmem_max=7500000
  net.core.wmem_max=7500000
  ```

  Apply on the **host** for Docker; for Kubernetes, apply on worker nodes or via a DaemonSet. Without this, QUIC can hit “failed to sufficiently increase receive buffer size” and packet drops.

---

## 7. References

- [Teranode System Requirements](https://bsv-blockchain.github.io/teranode/howto/miners/systemRequirements/) (official)
- [Syncing the Blockchain (Docker Compose)](https://bsv-blockchain.github.io/teranode/howto/miners/docker/minersHowToSyncTheNode/)
- [Managing Disk Space](https://bsv-blockchain.github.io/teranode/howto/miners/minersManagingDiskSpace/)
- Teranode in this repo: `teranode/util/storage_mode.go` (full vs pruned), `teranode/docs/topics/services/pruner.md`, `teranode/settings/interface.go` and `teranode/settings/pruner_settings.go` (retention and trigger), `teranode/docs/utxo-safe-deletion.md`, `teranode/test/tstn-seed/README.md` (seeding)
- [Third Party Software Requirements](https://bsv-blockchain.github.io/teranode/references/thirdPartySoftwareRequirements/) (Kafka, PostgreSQL, Aerospike, blob storage, etc.)
