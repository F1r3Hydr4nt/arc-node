# Broadcast Capabilities & Signed/Secure Plan: ARC vs Teranode

## Executive summary

**Goal:** Determine which repo (ARC or Teranode) can best deliver **signed and secure** transaction broadcasting, and outline a plan to get there.

**Conclusion:** **ARC** is the right place to implement signed, secure broadcast. Teranode is the chain/node layer and already has auth on RPC and Propagation; ARC is the dedicated broadcast/status layer, has the right API surface and callback model, and currently leaves auth optional—so we should enforce auth (and optionally request signing) in ARC and keep using Teranode as the backend node when needed.

---

## 1. Repo roles (quick reference)

| Aspect | **ARC** (bitcoin-sv/arc) | **Teranode** (bsv-blockchain/teranode) |
|--------|--------------------------|----------------------------------------|
| **Role** | Transaction processor / broadcast & status service | BSV full node (microservices) |
| **Broadcast entry** | REST API → Metamorph (P2P/multicast) or BitcoinNode (RPC) | Propagation (HTTP/gRPC) or RPC `sendrawtransaction` |
| **Status & callbacks** | Full lifecycle (QUEUED → MINED), callbacks, BlockTx | Node/validator focus; no ARC-style status API |
| **Auth today** | OpenAPI schemes defined; **default: security cleared** (no auth) | RPC: HTTP Basic (admin/limited); Propagation: rate limit + HTTP(S) |

---

## 2. ARC – broadcast capabilities

### 2.1 How broadcast works

- **Primary path (Metamorph):**
  - REST: `POST /v1/tx`, `POST /v1/txs` (raw tx hex or JSON).
  - API → gRPC/NATS → **Metamorph** → **bcnet Mediator** → either:
    - **P2P** (`p2p.NetworkMessenger`) to Bitcoin node peers, or
    - **Multicast** (hybrid mode) via `mcast.Multicaster`.
  - Status flow: Metamorph tracks status (e.g. QUEUED → STORED → ANNOUNCED_TO_NETWORK → SENT_TO_NETWORK → SEEN_ON_NETWORK → MINED); **BlockTx** handles block inclusion and feeds back to Metamorph.

- **Alternative path (BitcoinNode / RPC):**
  - `transaction_handler.BitcoinNode` talks to a Bitcoin RPC (e.g. Teranode RPC) via `SendRawTransaction`.
  - Used when the handler is configured as a "real Bitcoin node" (e.g. `api.PeerRPC`); single RPC call, no Metamorph status pipeline.

- **Broadcaster CLI / client:**
  - `internal/broadcaster`: `APIBroadcaster` calls ARC's REST API (`POST /v1/tx`, `/v1/txs`) with optional `X-CallbackUrl`, `X-CallbackToken`, `X-WaitFor`, etc.
  - Supports batch, fee validation options, and callback auth (token sent as `Authorization` on callbacks).

Relevant code areas:

- `arc/internal/metamorph/processor.go` – process/announce to network.
- `arc/internal/metamorph/bcnet/mediator.go` – P2P vs multicast.
- `arc/internal/api/transaction_handler/bitcoin_node.go` – RPC `SendRawTransaction`.
- `arc/internal/broadcaster/api.go` – client to ARC REST.
- `arc/pkg/api/arc.yaml` / `arc.json` – OpenAPI (including security schemes).

### 2.2 Security & signing (current)

- **OpenAPI:** Defines `BearerAuth`, `Api-Key`, `Authorization` and lists them in `security` (see e.g. `arc.yaml` / `arc.json`).
- **Default production behaviour:** In `internal/api/handler/helpers.go`, `CheckSwagger()` does:
  - `swagger.Security = nil` → **no auth enforced** by the validator.
  - Only schema/request validation is applied.
- **Custom auth:** Examples (`examples/custom/main.go`, `examples/customhandler/main.go`) show how to use `OapiRequestValidatorWithOptions` with `AuthenticationFunc` (e.g. Bearer token or API key). So the **mechanism** for auth exists; it's just not enabled by default.
- **Callback security:** `X-CallbackToken` is sent by ARC as `Authorization: Bearer <token>` when calling the client's callback URL – so callbacks can be authenticated.
- **No request signing:** There is no HMAC or other request-body signing in the codebase; security is limited to optional Bearer/API key and callback token.

### 2.3 ARC does **not** validate transactions against the UTXO set

**Conclusion:** ARC does **not** check that transaction inputs reference unspent outputs (no UTXO-set validation). Double-spend and “already spent” are detected only when the **Bitcoin node** rejects the transaction (e.g. via ZMQ or P2P), not at ARC’s API or validator.

**Evidence:**

1. **ARC’s own documentation** (`arc/doc/README.md`):
   - *"Obviously, double spending (for example) cannot be checked without an updated utxo set, and it is assumed that the API does not have this data."*
   - *"Therefore, the 'validator' sub-function within the API filters as much as it can to reduce spurious transactions passing through the pipeline but leaves others for the bitcoin nodes themselves."*
   - *"The actual utxos are left to be checked by the Bitcoin node itself."*

2. **What the API validator does:**
   - **extendTx** (defaultvalidator/helpers.go): Fetches **parent transactions** via `TxFinder.GetRawTxs()` (TransactionHandler, Node, or WoC) to fill in prevout script and value for each input. That only provides the *content* of the referenced output from the parent tx; it does **not** check that the output is still unspent.
   - **CommonValidateTransaction** (validator/common_validation.go): Structure/syntax only (size, empty inputs/outputs, coinbase check, sigops, push-only, etc.). No UTXO lookup.
   - **Fee / script validation:** Use the prevout data from the extended tx (sum of inputs vs outputs, script execution). Script verification runs against the *claimed* prevout from the parent tx; it does not verify that the referenced output still exists in the UTXO set.
   - So: **no** “is this outpoint still unspent?” check; double-spend is **not** detected at validation time.

3. **Metamorph:** Defines a `BitcoinNode` interface with `GetTxOut(txHex, vout, includeMempool)` (server.go), but the Processor does **not** take or use a BitcoinNode; `GetTxOut` is never called in the broadcast/validation path. Double-spend status (e.g. `DOUBLE_SPEND_ATTEMPTED`) comes from **ZMQ** messages from the node (`invalidtx` / discarded from mempool), i.e. after the node has rejected the tx.

4. **Implication for “signed and secure” broadcast:** If you need pre-broadcast assurance that inputs are unspent, that logic would have to be added (e.g. call a node’s `gettxout` or equivalent, or integrate with Teranode’s validator/UTXO store). Today, ARC intentionally leaves UTXO checks to the node.

---

## 3. Teranode – broadcast capabilities

### 3.1 How broadcast works

- **Propagation Service (main ingress):**
  - **HTTP:** `POST /tx` (single), `POST /txs` (batch, up to 1024 txs, 32 MB).
  - **gRPC:** `ProcessTransaction`, `ProcessTransactionBatch`.
  - Flow: Propagation → sanity checks → blob store → Validator (Kafka or HTTP). No ARC-style status API or callbacks.

- **RPC Service (Bitcoin-compat):**
  - `sendrawtransaction` (JSON-RPC) – submits a raw transaction; doc says it uses the validator for synchronous validation.
  - Standard Bitcoin RPC interface; used by ARC's `BitcoinNode` when pointing at Teranode.

- **Transaction lifecycle:** Ingress (Propagation) → Validator → Block Assembly → mining/block validation → Blockchain Service. No built-in "broadcast status" or callback API like ARC.

Refs: [Propagation Server Reference](https://bsv-blockchain.github.io/teranode/references/services/propagation_reference/), [Transaction Lifecycle](https://bsv-blockchain.github.io/teranode/topics/transactionLifecycle/), [RPC Service Reference](https://bsv-blockchain.github.io/teranode/references/services/rpc_reference/).

### 3.2 Security & signing (current)

- **RPC:** Two-tier HTTP Basic Auth:
  - `authsha` – admin (e.g. `generate`, `invalidateblock`, `stop`).
  - `limitauthsha` – limited (includes `sendrawtransaction`, `submitminingsolution`, `freeze`, etc.).
  - Implemented in `checkAuth(r *http.Request, require bool)`.
- **Propagation:** Supports "various security levels for HTTP/HTTPS" and **HTTP rate limiting** (`HTTPRateLimit`). No documented request-signing or Bearer/API-key in the fetched docs.
- **Operational:** Miner docs recommend strong auth for externally exposed services and firewall rules (e.g. [Security Best Practices](https://bsv-blockchain.github.io/teranode/howto/miners/docker/minersSecurityBestPractices/)).

So: Teranode already has **authenticated** broadcast (RPC Basic Auth; Propagation rate limit + HTTPS). It does not provide an ARC-style status/callback API; that lives in ARC.

---

## 4. Comparison: which repo for "signed and secure" broadcast?

| Criterion | ARC | Teranode |
|-----------|-----|----------|
| **Broadcast API** | REST + rich options (wait-for status, callbacks, batch) | Propagation (HTTP/gRPC) + RPC; no status/callback API |
| **Status & callbacks** | Yes (Metamorph + BlockTx + X-CallbackToken) | No |
| **Auth out of the box** | No (security cleared) | Yes (RPC Basic; Propagation rate limit + HTTPS) |
| **Auth extensibility** | OpenAPI + middleware; examples show Bearer/API key | RPC fixed to Basic; Propagation configurable |
| **Request signing** | Not present | Not documented |
| **Best suited to** | Public or internal "broadcast + status" API | Node/miner ingress and Bitcoin RPC compat |

To achieve **signed and secure** broadcast in one place:

- **Teranode** already secures its own ingress (RPC + Propagation). Adding full "ARC-style" secure broadcast (status, callbacks, optional request signing) there would mean re-implementing or re-exporting ARC-like semantics on top of Propagation/RPC.
- **ARC** already has the right broadcast and status model; it only needs **enforcement** of auth (and optionally request signing). That fits "signed and secure broadcast" with minimal duplication.

So the plan should focus on **ARC** as the place to implement and enforce signed, secure broadcast, and keep using **Teranode** as the node/backend (e.g. via RPC or future Propagation client) when needed.

---

## 5. Plan: signed and secure broadcast (ARC-centric)

### 5.1 Scope

- **In scope:** Authenticated and optionally request-signed access to ARC's broadcast API (POST /v1/tx, /v1/txs), with callbacks already secured by X-CallbackToken.
- **Out of scope (for this plan):** Changing Teranode's RPC or Propagation auth model; that stays as-is for node/miner use.

### 5.2 Phase 1 – Enforce authentication (ARC)

1. **Stop clearing security in production**
   - In `arc/internal/api/handler/helpers.go`, either:
     - Do **not** set `swagger.Security = nil` when auth is enabled, and wire a single authentication scheme (e.g. Bearer or Api-Key), or
     - Keep using `OapiRequestValidatorWithOptions` with a custom `AuthenticationFunc` (as in examples) so that every request is validated against a configured scheme.
   - Make this behaviour **configurable** (e.g. `api.auth.required` or `api.auth.mode: none | bearer | apikey` in config).

2. **Wire AuthenticationFunc in default server**
   - In `arc/cmd/services/api.go` (or equivalent), when auth is enabled:
     - Use the same pattern as `examples/custom/main.go` / `examples/customhandler/main.go`: `middleware.OapiRequestValidatorWithOptions(swagger, &middleware.Options{ Options: openapi3filter.Options{ AuthenticationFunc: ... } })`.
     - Resolve credentials from config (e.g. list of valid Bearer tokens or API keys, or integration with a secret store). Prefer hashed comparison and constant-time compare where applicable.

3. **Config**
   - Add settings under e.g. `api.auth`: `required`, `bearer_tokens` or `api_keys`, and/or path to external auth (future). Document env overrides (e.g. `ARC_API_AUTH_REQUIRED`).

4. **Backward compatibility**
   - Default can remain "auth optional" (current behaviour) until you flip the default; document that for production, auth should be enabled.

### 5.3 Phase 2 – Optional request signing (ARC)

1. **Design**
   - Define a small spec for request signing (e.g. `X-Signature`, `X-Timestamp`, `X-Nonce`):
     - Signature = HMAC-SHA256(secret, method + path + timestamp + nonce + body) or similar.
   - Reject requests with invalid or expired timestamp/nonce to limit replay.
   - Make signing **optional** per deployment (e.g. `api.auth.request_signing_required`).

2. **Implementation**
   - Add middleware that, when request signing is enabled:
     - Reads `X-Timestamp`, `X-Nonce`, `X-Signature` (or agreed header names).
     - Validates timestamp within a window (e.g. ±5 min) and optional nonce uniqueness (e.g. short-lived cache or store).
     - Computes expected signature and compares with constant-time compare.
   - Use the same config/secret store as Phase 1 for the signing key(s), or a separate key per client if needed.

3. **Docs and client**
   - Document the header contract and how to generate signatures.
   - Optionally extend `internal/broadcaster` or a small client lib to send signed requests for testing and reference.

### 5.4 Phase 3 – TLS and operational hardening (both)

- **ARC:** Serve API over TLS only in production; document recommended TLS versions and ciphers. Keep logging of auth failures (without logging secrets).
- **Teranode:** Already documented (firewall, strong auth). Ensure RPC and Propagation are not exposed without TLS when used as backend for ARC.
- **ARC ↔ Teranode:** When ARC uses Teranode as BitcoinNode (RPC), use TLS and the existing RPC Basic Auth (admin or limited) so that the broadcast path from API → node is authenticated and encrypted.

### 5.5 Callback security (already in place)

- ARC's **X-CallbackToken** is sent as `Authorization: Bearer <token>` on callbacks. Ensure:
  - Callback URLs are validated (ARC already has `RejectCallbackContaining` and URL checks).
  - Operators use strong, random tokens and HTTPS callback endpoints.
- No code change strictly required for "signed and secure" beyond ensuring this is documented and recommended.

---

## 6. Teranode's place in the plan

- **As node backend for ARC:** Use Teranode RPC (or in future Propagation client) as the "Bitcoin node" behind ARC when you want broadcast to go through Teranode. ARC's `BitcoinNode` already supports RPC with user/pass (and TLS if the client uses it). So:
  - **Signed and secure** is enforced at ARC's API.
  - Teranode is secured separately (RPC Basic Auth, firewall, TLS) as the node.
- **Standalone Teranode broadcast:** If someone submits directly to Teranode (Propagation or RPC), they already have auth (Propagation rate limit + HTTPS; RPC Basic). Adding request signing there would be a separate, Teranode-specific task and is not required to achieve "signed and secure broadcast" for the ARC-based flow.

---

## 7. Validate-only (simulate) flag & Teranode Validator for UTXO check

**Goal:** Add an ARC option to run **simulate-validation only** (no broadcast), so callers can check that a transaction would pass validation—including **UTXO spent status**—without submitting to the network. Teranode is intended to support direct API access with signed responses for secure, attestable validation.

### 7.1 Feasibility in ARC

**Adding a validate-only / simulate flag is straightforward.**

- **Current flow** (e.g. `POST /v1/tx`, `POST /v1/txs`):  
  `getTxDataFromHex` (decode + ARC validation) → `submitTransactions` (TransactionHandler → Metamorph or BitcoinNode).  
  So “validate only” means: run decode + ARC validation as today, then **skip** `submitTransactions` and return a structured “validation only” response (e.g. valid/invalid per tx, optional reason).

- **Where to hook the flag:**
  - Add a header (e.g. **`X-ValidateOnly`** or **`X-SimulateValidation`**) and map it into `global.TransactionOptions` (e.g. `ValidateOnly bool`), same pattern as `X-SkipTxValidation`, `X-ForceValidation`, etc. (see `getTransactionsOptions` in `internal/api/handler/default.go`).
  - In `processTransactions`, after `getTxDataFromHex` succeeds, if `options.ValidateOnly` is true: **do not call** `submitTransactions`; instead return a response that indicates “validation only” (e.g. success with a synthetic status like `VALIDATION_ONLY` or a dedicated response shape with `valid: true/false`, `reason` per tx).

- **ARC’s own validation** (structure, fee, script from parent tx data) already runs in `getTxDataFromHex`. So with only this change, “validate only” gives you **no broadcast** and **no UTXO check** (ARC still does not have UTXO set). To get **UTXO spent status** in the same call, ARC would need to call an external validator (e.g. Teranode) when the flag is set—see below.

### 7.2 Teranode Validator for UTXO check (simulate only, no broadcast)

Teranode’s **Validator** gRPC API is a good fit for “validate only, including UTXO” without broadcasting.

- **API** (from [Validator Proto](https://bsv-blockchain.github.io/teranode/references/protobuf_docs/validatorProto/)):
  - **`ValidateTransaction(ValidateTransactionRequest) → ValidateTransactionResponse`**  
    “Performs comprehensive validation including script verification and **UTXO checks**.”
  - **`ValidateTransactionBatch`** for multiple transactions.

- **Request fields relevant to “simulate only” (spend-simulation):**
  - **`add_tx_to_block_assembly`** (optional bool): Set to **false** so the validated tx is **not** added to block assembly → no broadcast, no mining path. This is the only flag needed for “validate only, no broadcast.”
  - **`skip_utxo_creation`** (optional bool): Do **not** set this to true for spend-simulation. “Skip UTXO creation” means skip creating/persisting new UTXOs from this tx’s outputs; in some implementations that could also skip or alter the **input** UTXO spent check. We need the validator to perform the full **UTXO spent status check** (inputs unspent) and return success/fail. So use **`add_tx_to_block_assembly: false`** only, and leave **`skip_utxo_creation`** false or omitted so the node still runs the full validation including UTXO spent checks.

- **Response:** `valid` (bool), `txid`, `reason` (rejection reason if invalid), `metadata`. That gives ARC (and the client) a clear “would this pass the node’s UTXO + script checks?” result.

**Integration with ARC:** When `X-ValidateOnly` is true and a Teranode Validator endpoint is configured, ARC could:

1. Run its own validation as today (`getTxDataFromHex`).
2. For each decoded tx, call Teranode’s `ValidateTransaction` (or batch) with **`add_tx_to_block_assembly=false`** only (do not set `skip_utxo_creation=true`, so the validator still performs the full UTXO spent check).
3. Return a combined response: ARC validation result + Teranode validation result (valid/reason), **without** calling `submitTransactions`.

That way the client gets a single “simulate only” response that includes UTXO spent status when Teranode is used.

### 7.3 Teranode security: direct API comms and signed responses

- **gRPC API key auth (documented):**  
  Teranode’s RPC reference describes **gRPC API key authentication** for certain operations: API key in gRPC metadata **`x-api-key`**; configurable via e.g. `grpc_admin_api_key`. So direct gRPC access to Teranode (including Validator) can be authenticated with an API key. Use TLS for the gRPC channel so the response is not visible on the wire.

- **“Signed responses”:**  
  The phrase “direct API comms with signed responses” could mean:
  - **TLS:** The entire gRPC response is integrity-protected (and confidential) over the channel; no separate payload signature.
  - **Application-level response signing:** The Validator (or another service) signs the response payload (e.g. `valid` + `txid` + `reason`) so the client can prove what the node said. That would require support in Teranode’s Validator (or gateway) and is not clearly documented in the references we have.  
  **Recommendation:** Confirm with Teranode docs or code whether Validator (or any gRPC service) returns a signature over the response body. If not, “signed” can be satisfied by TLS + API key for now; response-signing could be a follow-up.

- **Secure pattern for ARC → Teranode Validator:**  
  Use TLS for the gRPC connection and, if the Validator supports it, send `x-api-key` (or the configured auth) in metadata. That gives authenticated, confidential “validate only” calls. If Teranode adds response signing later, ARC can verify and optionally forward that to the client.

### 7.4 Summary

- **ARC:** Adding a flag (e.g. **`X-ValidateOnly`**) that skips `submitTransactions` and returns a “validation only” result is **feasible** and fits the existing options/flow.
- **UTXO spent status:** Use Teranode’s **Validator** gRPC with **`ValidateTransaction`** (or batch) and **`add_tx_to_block_assembly=false`** only. Do not set `skip_utxo_creation=true` (that could skip the input spent check); keep it false/omitted so the node returns a spend-simulation success/fail including UTXO spent checks, without broadcasting.
- **Security:** Teranode supports gRPC API key auth (`x-api-key`); run over TLS. Clarify with Teranode whether “signed responses” means payload signing; if not, TLS + auth is the baseline for secure validate-only calls.

---

## 8. Recommendation summary

- **Implement signed and secure broadcast in ARC:** Enable configurable auth (Bearer/API key), optionally add request signing, and serve over TLS. Use the existing OpenAPI security schemes and example middleware.
- **Use Teranode as the node when needed:** Connect ARC to Teranode via RPC (and later optionally Propagation) with Teranode's existing auth and TLS.
- **Validate-only for UTXO check:** Add **`X-ValidateOnly`** (or similar) in ARC; when set, skip broadcast and optionally call Teranode Validator with `add_tx_to_block_assembly=false` to return UTXO/spent-status in the response.
- **Do not duplicate:** Avoid building a full ARC-style status/callback and auth layer inside Teranode; keep Teranode as the chain/node and ARC as the broadcast/status API.

This gives one clear place (ARC) for "signed and secure" broadcast while reusing both codebases according to their roles.
