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

## 7. Recommendation summary

- **Implement signed and secure broadcast in ARC:** Enable configurable auth (Bearer/API key), optionally add request signing, and serve over TLS. Use the existing OpenAPI security schemes and example middleware.
- **Use Teranode as the node when needed:** Connect ARC to Teranode via RPC (and later optionally Propagation) with Teranode's existing auth and TLS.
- **Do not duplicate:** Avoid building a full ARC-style status/callback and auth layer inside Teranode; keep Teranode as the chain/node and ARC as the broadcast/status API.

This gives one clear place (ARC) for "signed and secure" broadcast while reusing both codebases according to their roles.
