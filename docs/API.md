# OFPAY API contract

**Status:** agreed P0 and P1 decisions (see `PRD.md` §§20–21). This document replaces the tentative proposals in PRD §11 where they differ. No implementation code exists yet.

## 1. Conventions

- HTTPS only; JSON UTF-8 bodies; timestamps are RFC 3339 UTC in the envelope, `int64` Unix seconds inside signed payloads.
- All money is integer pesewas (`GHS`): `1 GH₵ = 100 pesewas`. No floating point anywhere.
- Every request may carry `Idempotency-Key: <uuid-v4>`; mutating requests should.
- Authenticated endpoints use `Authorization: Bearer <opaque session token>`.
- `server_time` in every response is the phone's only trusted clock source.
- QR payloads are raw binary CBOR in byte mode with an `OFPAY1` header (see §5); base64url is used only for logs/tests.

### Envelope

```json
// success
{ "data": { }, "meta": { "request_id": "uuid", "server_time": "2026-11-27T12:00:00Z" } }

// error
{ "error": { "code": "COUNTER_REPLAY", "message": "…", "details": { "tx_id": "…" } },
  "meta": { "request_id": "uuid", "server_time": "2026-11-27T12:00:00Z" } }
```

Clients branch on `error.code`, never on `message`. Idempotent replays return the stored domain data with a **fresh** `meta.server_time`.

## 2. Error codes

| Code | HTTP | Class | Meaning |
| --- | --- | --- | --- |
| `AUTH_INVALID`, `AUTH_EXPIRED` | 401 | re-login | Missing/expired session |
| `DEVICE_NOT_REGISTERED`, `DEVICE_KEY_REVOKED` | 403 | terminal until re-enrolment | Device/key not active for user |
| `PAYLOAD_MALFORMED`, `SCHEMA_UNSUPPORTED` | 422 | terminal | Engine parse/version failure |
| `SIGNATURE_INVALID`, `ALLOWANCE_SIGNATURE_INVALID` | 422 | terminal | Ed25519 verification failure |
| `TX_ID_MISMATCH` | 422 | terminal | Recomputed hash ≠ submitted `tx_id` |
| `AMOUNT_INVALID`, `ALLOWANCE_MISMATCH`, `RECIPIENT_UNKNOWN` | 422 | terminal | Field/binding validation |
| `EXPIRED` | 422 | terminal | Allowance or payment expired (strict, no grace) |
| `DUPLICATE_TX_DIFFERENT_PAYLOAD`, `COUNTER_REPLAY`, `ALLOWANCE_EXHAUSTED` | 409 | conflict → `CONFLICT`/`REVIEW_REQUIRED` | Unique/CAS constraint violation |
| `IDEMPOTENCY_CONFLICT` | 409 | client bug | Same key, different request hash |
| `DUPLICATE_TX` | 200 | success — prior status returned | Idempotent replay of identical `tx_id` |
| `BLOCKCHAIN_UNAVAILABLE` | 503 | transient — chain query only | Outbox retries; sync still succeeds |
| `RATE_LIMITED`, `INTERNAL` | 429 / 500 | transient | Retry with backoff |

`POST /transactions/verify` reports reasons using the same codes with `data.valid = false`.

## 3. Endpoints

| Method | Endpoint | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/auth/register` | none | Create user + simulated wallet; returns 24h opaque token |
| POST | `/auth/login` | none | Issue new opaque token |
| GET | `/devices/challenge` | token | One-time nonce for proof of possession |
| POST | `/devices` | token | Register device public key with signed proof |
| POST | `/devices/{device_id}/keys` | token | Rotate to a new key (signed challenge proof) |
| POST | `/devices/{device_id}/revoke` | app admin | Immediate revocation (compromise/dispute) |
| GET | `/config/keys` | none | Current backend allowance issuer public keys (also app-pinned) |
| GET | `/wallets/me` | token | Wallet id, available/reserved pesewas, version |
| GET | `/wallets/balance` | token | Available/reserved pesewas, `as_of` |
| POST | `/allowances` | token + idempotency | Request signed allowance; reserves full cap |
| GET | `/allowances/active` | token | Active signed allowances for account/device |
| POST | `/transactions/verify` | token | **Read-only** preflight; no DB writes |
| POST | `/transactions/sync` | token + device signatures | Authoritative batch ingest |
| GET | `/transactions` | token | Paginated history (`?status=&cursor=&limit=`) |
| GET | `/transactions/{id}` | token | Full record, state and reason (`{id}` = `tx_id`) |
| GET | `/blockchain/transaction/{id}` | token | Chain status (`{id}` = canonical `tx_id`) |
| POST | `/reviews/{id}/decisions` | app admin | 2-of-3 decision on a `REVIEW_REQUIRED` case |
| GET/POST/DELETE | `/admins` | app admin | List/add/remove application admins (changeable list) |

### 3.1 Authentication and enrolment

```json
POST /auth/register   { "phone_e164": "+233…", "password": "…", "display_name": "…" }
→ { "user_id": "uuid", "wallet_id": "uuid", "token": "opaque", "expires_at": "…Z" }

POST /auth/login      { "phone_e164": "+233…", "password": "…" }
→ { "token": "opaque", "expires_at": "…Z" }

GET  /devices/challenge → { "challenge": "random nonce", "expires_at": "…Z" }

POST /devices          { "device_id": "client uuid v4", "key_id": "8-byte hex",
                         "algorithm": "Ed25519", "public_key_b64": "…",
                         "proof_b64": "sign(domain‖user_id‖challenge‖device_id‖public_key)" }
→ { "device_id": "…", "key_id": "…", "status": "active" }
```

Sessions are opaque random tokens; only their hash is stored (`sessions`). Token expiry (24h) is enforced by server time — no refresh token in MVP. The device's private key never leaves the device. `POST /transactions/sync` requires **both** a valid token (who) and a valid device-key signature (which key authorised the spend). `key_id` is the 8-byte truncated SHA-256 of the public key (P1-3), keeping signed payloads compact.

### 3.2 Allowances

```json
POST /allowances      { "device_id": "uuid", "amount_minor": 3000 }
→ { "allowance_id": "uuid", "nonce": "…", "max_amount_minor": 3000,
    "mode": "multi_spend", "not_before": 123, "expires_at": 456,
    "signature_b64": "…", "issuer_key_id": "…" }
```

Server caps the requested amount against available balance and configured maximum. It reserves the **full cap** immediately (`available → reserved`), signs the allowance with the backend issuer key, and generates id/nonce/expiry. Multiple active allowances per device are allowed. Unused reserved funds are auto-released at `expires_at + contest_window` unless the allowance is under review.

### 3.3 Transaction sync

```json
POST /transactions/sync
{ "batch_id": "uuid",
  "items": [ { "tx_id": "hex sha256", "payload": { }, "signature_b64": "…",
               "allowance": { "signed allowance bytes/payload", "signature_b64": "…" } } ] }

→ 200 { "data": { "batch_id": "uuid", "items": [
        { "tx_id": "…", "status": "BLOCKCHAIN_SUBMISSION", "code": null, "chain_status": "held" },
        { "tx_id": "…", "status": "REVIEW_REQUIRED", "code": "COUNTER_REPLAY" } ] }, "meta": {…} }
```

Rules:

- One HTTP `200` for the batch; envelope-level failures (auth, malformed JSON, batch too large → `413`) use the error shape.
- Each item commits in its **own DB transaction**. One bad item never blocks the others.
- Ordering inside a batch: `(server_received_at, tx_id)`. Device-supplied `created_at_device` is advisory only.
- Validate: recompute `tx_id` from canonical bytes → verify device and issuer signatures → check allowance state, `mode`, cap, strict expiry → CAS `devices.last_counter` (`counter > last_seen`) → unique constraints.
- Chain submission is **held** for the configurable contest window (default 5 minutes in the demo). A conflicting claim during that window moves both items to `REVIEW_REQUIRED` and freezes their unused reserve.
- Mobile retry rule: retry only on no response / `5xx` / `429`. Never retry `422` or `409`. Once an item has any accepted status, stop resending it.

`chain_status` values: `held`, `submitted`, `confirming`, `confirmed`, `failed`. Internal attempt counts and errors are never exposed.

### 3.4 Idempotency (P1-1)

Three layers, all active:

1. `Idempotency-Key` header — scope `(user_id, device_id, route, key)`; stores request hash and response; mismatched hash → `IDEMPOTENCY_CONFLICT`.
2. `batch_id` in the body — deduplicates a logical batch on retry.
3. `tx_id` — natural per-item key (`UNIQUE(tx_id)`); identical replay returns prior status; same `tx_id` with different bytes → `DUPLICATE_TX_DIFFERENT_PAYLOAD`.

Locked P1 behaviour:

- **Retention: 7 days.** A periodic cleanup deletes expired rows; `reconciliation_events` remain append-only and untouched. Domain-level uniqueness still prevents double effects after the window — retention controls response replay convenience, not money safety.
- **Replay body:** the stored hash detects a replay; `/transactions/sync`, `/transactions/{id}` and `/blockchain/transaction/{id}` regenerate the domain payload from current DB state (fresh `meta.server_time`) so a delayed client never sees a stale status. Non-status mutations such as `POST /allowances` return the stored response verbatim — re-running issuance would reserve funds twice.
- **Hash input:** canonicalised JSON body only (sorted keys, no `meta`); headers other than the key itself are excluded. Deterministic across client retries.

### 3.5 Review decisions and app admins (P1)

```json
POST /reviews/{id}/decisions  { "decision": "CONFIRM_FIRST | AWARD_OTHER | REJECT_BOTH" }
```

Requires two distinct app-admin approvals; an admin who submitted either claim is recused. When the second approval arrives, the outcome executes in one DB transaction and appends decision events. This workflow can also resolve chain-retry exhaustion, using the same decision codes.

Initial application admins: **`Tazorx7`, `asarepok`, `Neequaye279`** (all three are GitHub write collaborators; application admin status itself is stored in the app-level `admins` table, independent of repository roles). The admin list is **changeable later** via `/admins` (add/remove), with every change appended as an event; a change takes effect for new reviews only.

### 3.6 Key rotation and revocation (P1-3)

- User-initiated `POST /devices/{device_id}/keys`: server issues a new challenge, client signs it with the **new** key, server verifies and updates `key_id` with bounded validity windows.
- Admin-initiated `POST /devices/{device_id}/revoke`: immediate revocation for compromise/dispute; emits an event and invalidates new payments.
- **Bounded overlap:** the old key remains verifiable for its *existing* allowances until `min(24h, allowance.expires_at)`, but cannot authorise new payments. Short allowance lifetimes keep the exposure small; admin revocation overrides the overlap immediately.
- The device-wide counter **continues** across a same-device rotation; resetting it would look like `COUNTER_REPLAY`. A new `device_id` starts a fresh counter lineage (the P0-06 recovery path).
- No automatic time-based rotation in MVP — rotation is event-driven.

## 4. Amounts, caps and time

Locked values (P1 confirmations; see PRD §21):

| Parameter | Rule |
| --- | --- |
| `amount_minor` | `> 0`; `u64` in payload; server caps and stores as `BIGINT CHECK >= 0` |
| Allowance cap | **10,000 pesewas** per request (server config) |
| Initial simulated wallet | **500,000 pesewas** (seed config) |
| Contest window | 5 minutes (configurable) |
| Arithmetic | `checked_add` everywhere; overflow → `AMOUNT_INVALID` (422) |
| `expires_at` | Strict: `server_now >= expires_at` → `EXPIRED`; no grace |
| `not_before` | Issued as `server_now - 60s` so allowance is immediately valid |
| Reserve release | `expires_at + contest_window`, skipped for claims under review |

## 5. QR payload and on-chain contract (P1-2 / P1-5)

- QR encodes **raw binary CBOR** in byte mode with a small `OFPAY1` magic/version header; base64url text is only for logs and golden vectors.
- Budget: **600 bytes target / 800 bytes hard cap** at ECC level **M**; the golden-vector CI check fails above target. Compact 8-byte `key_id` is the first mitigation. No animated/multi-frame QR in MVP.
- The Solidity contract stores **`bytes32 txId` only** (the P0-03 SHA-256 hash), emits `TransactionRegistered(txId indexed, submitter, timestamp)`, reverts with `AlreadyRegistered` on duplicates (backend maps a matching-hash revert to success), is callable only by an allowlisted backend signer, and has **no pause/admin function** in MVP. No amounts, wallet IDs or identities on-chain.
