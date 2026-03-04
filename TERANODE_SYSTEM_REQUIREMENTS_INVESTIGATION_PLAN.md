# Teranode System Requirements Investigation Plan

## Goal

Determine what is needed to run Teranode (cloned in this repo at `teranode/`) on **testnet** and **mainnet**, in **full**, **pruned**, and (if applicable) **UTXO-subset** modes, and to clarify the relationship between pruned mode and the UTXO set.

## Key distinction (from codebase review)

**Pruned mode in Teranode does keep the full live UTXO set.** It is not a "subset" of UTXOs:

- **Full node** (per [teranode/util/storage_mode.go](teranode/util/storage_mode.go)): Block Persister is running and within the retention window (default 288 blocks). Recent block/transaction data is retained.
- **Pruned node**: Either Block Persister is not running, or it lags beyond the retention window, or the pruner uses the `OnBlockMined` trigger. Old **blob/transaction data** is deleted, and **fully spent parent UTXO records** are removed (Delete-At-Height). The **current unspent output set** (live UTXOs) is still fully maintained in Aerospike.

So: **pruned = full live UTXO set + deletion of old transaction blobs and spent parent records.** There is no separate "UTXO-subset" mode in the current codebase (e.g. keeping only certain addresses' UTXOs). The investigation should confirm this and document it clearly.

---

## 1. Document official system requirements (testnet vs mainnet)

**Source:** [Teranode System Requirements](https://bsv-blockchain.github.io/teranode/howto/miners/systemRequirements/) (already fetched).

**Findings to capture:**

- **Docker Compose**
  - **Mainnet:** Min 8 CPU / 128 GB RAM / 1 TB NVMe SSD; recommended 16 CPU / 256 GB RAM / 2 TB NVMe. HDD not supported (Aerospike/blob IOPS).
  - **Testnet:** Min 4 CPU / 16 GB RAM / 64 GB SSD; recommended 8 CPU / 32 GB RAM / 128 GB (NVMe preferred).
- **Kubernetes**
  - **Mainnet:** Per-pod CPU/memory for each service (e.g. blockValidator 1 CPU / 8Gi, subtreeValidator 1 / 16Gi, legacy 4 / 32Gi); external Aerospike 4 cores / 32 GB / 400 GB NVMe, PostgreSQL 2 / 4 GB / 50 GB, Kafka 2 / 4 GB / 50 GB; shared RWX storage 1 TB.
  - **Testnet:** Baseline 100m CPU / 512Mi per service; Aerospike 2 cores / 8 GB / 50 GB NVMe, PostgreSQL 1 / 2 GB / 20 GB, Kafka 1 / 2 GB / 20 GB; shared RWX 50 GB.
- **Storage breakdown (mainnet, seeded + pruned, default retention):** Aerospike ~400 GB (UTXO set ~340M records), Blob ~600 GB, PostgreSQL < 1 GB, Prometheus ~1 GB → ~1 TB total.
- **Caveats:** Requirements assume a **seeded, pruned node** with default retention. Full sync from genesis or higher retention increases needs (see below).

**Action:** Produce a short requirements matrix (testnet vs mainnet, Docker vs K8s, min vs recommended) and note assumptions (seeded, pruned, default retention).

---

## 2. Full vs pruned mode (requirements and behavior)

**Sources:** [teranode/util/storage_mode.go](teranode/util/storage_mode.go), [teranode/docs/topics/services/pruner.md](teranode/docs/topics/services/pruner.md), [teranode/settings/interface.go](teranode/settings/interface.go) (`GlobalBlockHeightRetention`), pruner settings.

**Behavior (already summarized):**

- **Full:** Block Persister running and within `GlobalBlockHeightRetention` (default 288 blocks). Serves as "full" for recent history within that window.
- **Pruned:** Persister not running, or lag > retention, or pruner trigger = `OnBlockMined` (prunes before persistence).

**Investigation steps:**

1. **Requirements impact:** Compare resource implications of:
   - Running with Block Persister + Pruner (full) vs Pruner only / no persister (pruned). Official numbers are for "pruned"; document whether full mode is explicitly documented as needing more (e.g. blob/store).
   - Check [teranode/docs/howto/miners/minersManagingDiskSpace.md](https://bsv-blockchain.github.io/teranode/howto/miners/minersManagingDiskSpace/) and any retention/pruning docs for "full" vs "pruned" storage guidance.
2. **Config summary:** Document the main levers: `global_blockHeightRetention` (default 288), Block Persister on/off, `pruner_force_ignore_block_persister_height` / Block trigger (`OnBlockPersisted` vs `OnBlockMined`), and how they map to full vs pruned in `DetermineStorageMode`.
3. **Optional diagram:** One small flowchart or table: "Block Persister + retention window → full; otherwise or OnBlockMined → pruned."

**Deliverable:** A "Full vs pruned" subsection: definition, config levers, and any stated or inferred requirement differences (CPU/RAM/disk).

---

## 3. Pruned mode and the UTXO set (clarify "live set")

**Goal:** State clearly that pruned mode **retains the full live UTXO set** and only removes old data and spent parent records.

**Sources:** [teranode/docs/topics/services/pruner.md](teranode/docs/topics/services/pruner.md) (two-phase process), [teranode/docs/utxo-safe-deletion.md](teranode/docs/utxo-safe-deletion.md), [teranode/stores/utxo/aerospike/pruner/README.md](teranode/stores/utxo/aerospike/pruner/README.md), system requirements (Aerospike ~400 GB = UTXO set).

**Investigation steps:**

1. **Document pruner behavior:** Phase 1 = preserve parents of old unmined txs; Phase 2 = delete records where `deleteAtHeight <= safeHeight` (and blob cleanup). Emphasize: deleted records are **fully spent parent transactions**, not unspent outputs.
2. **State explicitly:** In pruned mode the node still holds the **complete set of unspent outputs** (live UTXO set) in Aerospike; pruning reduces blob storage and removes spent parent TX records to free space.
3. **Reference:** Add a one-sentence reference to `DetermineStorageMode` and the system requirements note that specs assume "seeded, pruned node" (i.e. pruned with full UTXO set).

**Deliverable:** Short "Pruned mode and UTXO set" note: pruned = full live UTXO set + pruning of old blobs and spent parent records; no reduction of the unspent set.

---

## 4. UTXO-subset mode (existence and definition)

**Goal:** Determine whether Teranode supports (or plans) a "UTXO-subset" mode (e.g. only a subset of the global UTXO set, such as for specific addresses or scripts).

**Investigation steps:**

1. **Code/search:** Search the local `teranode/` repo for concepts such as "UTXO subset", "partial UTXO", "filtered UTXO", "watch-only", or config that limits which UTXOs are stored. (Initial grep did not find a UTXO-subset run mode; "partial" in code refers to partial transactions/batches, not a subset of the chain's UTXO set.)
2. **Docs:** Check [teranode/docs/topics/commands/seeder.md](teranode/docs/topics/commands/seeder.md), UTXO Store and Persister docs, and any roadmap or design docs for "subset" or "light" modes.
3. **Conclusion:** If no such mode exists, state: "Teranode does not currently support a UTXO-subset mode. Pruned mode is not a subset of UTXOs; it keeps the full live set and prunes old transaction data and spent parent records." If there is a planned or experimental mode, describe it and how it differs from full and pruned.

**Deliverable:** One subsection: "UTXO-subset mode" — present/planned/not present, and how it differs from pruned (full live UTXO set).

---

## 5. Full sync from genesis vs seeded bootstrap

**Sources:** [Teranode System Requirements](https://bsv-blockchain.github.io/teranode/howto/miners/systemRequirements/) ("Full blockchain sync from genesis requires additional temporary storage and significantly more time"), [teranode/docs/howto/miners/docker/minersHowToSyncTheNode.md](teranode/docs/howto/miners/docker/minersHowToSyncTheNode.md), [teranode/test/tstn-seed/README.md](teranode/test/tstn-seed/README.md).

**Findings to capture:**

- **Seeded:** Import headers + full UTXO set from seed files (e.g. from UTXO Persister export); avoids replay from genesis. System requirements assume this.
- **Full sync (P2P from genesis):** Sync guide states ~14 days and **minimum 4TB** with default prune (288 blocks); more if retention is increased.
- **Third-party:** [references/thirdPartySoftwareRequirements.md](teranode/docs/references/thirdPartySoftwareRequirements.md) if it mentions resource requirements.

**Investigation steps:**

1. Extract from the sync guide: time estimates, minimum storage for full sync, and any mainnet vs testnet differences.
2. Compare to the 1 TB (mainnet) and 64–128 GB (testnet) in the system requirements doc and state explicitly: "Official table = seeded + pruned; full sync from genesis needs more (e.g. 4TB+ and days)."
3. Optionally list sync methods (P2P, legacy SV node seeding, Teranode data seeding) and which one the requirement table assumes.

**Deliverable:** A "Sync method and requirements" subsection: seeded vs full sync, impact on storage/time, and how that affects the published requirement numbers.

---

## 6. Optional: network and OS requirements

**Source:** System requirements page (already fetched): Ubuntu 24.04 LTS recommended; UDP buffer sizes for QUIC (e.g. `net.core.rmem_max` / `wmem_max` 7500000).

**Action:** If the final deliverable is a single "runbook" or requirements doc, add a short note on OS and network (QUIC buffers) so testnet/mainnet and full/pruned are not the only variables.

---

## 7. Deliverable format

Produce a single **investigation report** (e.g. `TERANODE_SYSTEM_REQUIREMENTS.md` in the repo root or under `docs/`) that includes:

1. **Summary:** Testnet vs mainnet, full vs pruned, and "pruned = full live UTXO set; no UTXO-subset mode found."
2. **Requirements matrix:** Tables for Docker and K8s (testnet/mainnet, min/recommended), with assumptions (seeded, pruned, default retention).
3. **Full vs pruned:** Definition, config levers, and any requirement differences.
4. **Pruned mode and UTXO set:** Short clarification that pruned keeps the full live UTXO set.
5. **UTXO-subset mode:** Result of search (exists / planned / not present) and relation to pruned.
6. **Sync method:** Seeded vs full sync from genesis and impact on storage/time.
7. **References:** Links to Teranode system requirements, sync guide, `storage_mode.go`, pruner docs, and (if used) third-party software requirements.

No code or config changes in this phase—investigation and documentation only.
