# OFPAY architecture (agreed P0 and P1 decisions)

**Status:** agreed P0 and P1 decisions (see `PRD.md` §§20–21). This is a design record only — no application code exists yet. It refines PRD §§6, 8, 12 and 14. The §5 blockchain gate is **reaffirmed**: Besu/QBFT stays in the MVP; DevOps will add initial infrastructure later.

## 1. Components and boundaries

| Path | Owns | Depends on |
| --- | --- | --- |
| `apps/mobile/` (Flutter) | UI, QR, SQLite, key storage, local states `CREATED → LOCALLY_VERIFIED → PENDING_SYNC` | `payment-engine` via FFI; API over HTTP |
| `crates/payment-engine/` (Rust) | Canonical CBOR, domain separation, Ed25519 sign/verify, SHA-256, allowance/payment structural validation, counter-check primitives, golden-vector generator | Crypto/serde crates only — no HTTP, SQL, clock or chain |
| `services/api/` (Rust/Axum) | Auth, wallets, devices, key rotation/revocation, allowances, sync validation, conflict/review, admin management, chain outbox, status APIs | `payment-engine`, Postgres, contract ABI |
| `database/` | SQL migrations and seeds | Postgres |
| `blockchain/` | Besu/QBFT config, Solidity contract, deploy scripts | Accepted claims only |
| `infrastructure/` | Podman compose, CI (added later by DevOps) | All of the above |
| `tests/vectors/v1/` | Golden vectors: canonical bytes, signatures, hashes, negative cases, QR-size fixture | Engine generator; shared by backend/mobile |

**Rule:** canonical encoding and validation primitives exist **only** in `payment-engine`. Dart and Axum call them; they never reimplement them. Anchor: PRD §19 — no offline consensus; chain is a notary, not a validator.

## 2. Engine contract (P0 #3–#4, P1 #2–#4)

- Canonical CBOR with a fixed field set, `schema_version`, and domain strings inside signed bytes: `ofpay/allowance/v1`, `ofpay/payment/v1`, `ofpay/wallet-id/v1`, `ofpay/challenge/v1`.
- `tx_id = hex(SHA256(domain ‖ canonical_payment_bytes))`; those same bytes are signed by the device key. `tx_id` is not a signed field.
- IDs are 16-byte UUIDs; timestamps `int64` Unix seconds; amounts `u64` pesewas. Debug renderer shows UUID text/UTC.
- **`key_id` is an 8-byte truncated SHA-256 of the public key** (P1-3) — compact for QR, globally unique, no version counter.
- Allowance signed fields: version, domain, `issuer_key_id`, `allowance_id`, `nonce`, `wallet_id`, `device_id`, `key_id`, `max_amount_minor`, `currency`, `not_before`, `expires_at`, `mode`.
- Payment signed fields: version, domain, `allowance_id`, `allowance_nonce`, `allowance_hash`, `sender_wallet_id`, `sender_device_id`, `sender_key_id`, `recipient_wallet_id`, `amount_minor`, `currency`, `counter`, `created_at_device`, `expires_at`, `chain_context`.
- Payment QR envelope carries the **full signed allowance** plus the signed payment; the payment binds `allowance_hash` so substitution is detected offline. QR encoding is raw binary CBOR with an `OFPAY1` header, ECC M, 600 B target / 800 B hard cap (P1-2).
- Recipient-ID QR is signed with the recipient's device key; binds wallet ID (not device ID) so device rotation does not invalidate it.
- Mobile obtains issuer public keys from `GET /config/keys` with a pinned copy in the app build.
- **Golden vectors (P1-4):** full set for allowance, payment, wallet-ID QR and enrolment challenge, plus negative cases (tampering, wrong domain, unknown version, expiry, replayed counter) and a representative GH₵30 QR-size fixture. An engine generator writes checked-in files under `tests/vectors/v1/`; CI fails on drift. Dedicated **test-only** Ed25519 keys plus RFC 8032 primitives. `v1/` is immutable; a schema bump creates `v2/` so outstanding old-format allowances remain regression-tested. When DevOps lands CI, the vectors job becomes a **required** status check on `main` and `master`.

## 3. Data model

All money `BIGINT` pesewas with `CHECK >= 0`; all server timestamps `timestamptz` UTC. Locked caps: allowance **10,000 pesewas**, initial wallet **500,000 pesewas**, contest window **5 minutes**.

| Table | Key columns / constraints |
| --- | --- |
| `users` | `id uuid PK`, `phone_e164 UNIQUE`, `password_hash`, `created_at` |
| `wallets` | `id uuid PK`, `user_id UNIQUE FK`, `available_minor`, `reserved_minor`, `version` |
| `sessions` | `token_hash UNIQUE`, `user_id FK`, `expires_at`, `revoked_at` |
| `devices` | `device_id uuid PK`, `user_id FK`, `status`, `last_counter bigint` (CAS target) |
| `device_public_keys` | `key_id bytea(8) PK`, `device_id FK`, `public_key bytea UNIQUE`, `valid_from`, `valid_to`, `status`; partial `UNIQUE(device_id) WHERE status='active'`; rotation keeps old row valid for bounded overlap |
| `offline_allowances` | `allowance_id UNIQUE`, `nonce UNIQUE`, `wallet_id/device_id/key_id FK`, `max_amount_minor`, `consumed_minor`, `mode`, `expires_at`, `signature bytea`, `status`; `CHECK consumed <= max` |
| `transactions` | `tx_id UNIQUE`, `payload_hash UNIQUE`, `allowance_id FK`, `sender/recipient wallet FK`, `amount_minor`, `counter`, `created_at_device`, `expires_at`, `state`, `failure_code`, `server_received_at`; `UNIQUE(device_id, counter)`, `UNIQUE(allowance_id, counter)` |
| `blockchain_outbox` | `tx_id UNIQUE FK`, `status`, `held_until`, `locked_at`, `attempts`, `next_attempt_at`, `decision` |
| `blockchain_records` | `tx_id UNIQUE FK`, `chain_id`, `blockchain_tx_hash UNIQUE`, `block_number`, `status`, `confirmed_at` |
| `reconciliation_events` | Append-only `(tx_id, event_type, from_state, to_state, actor, detail jsonb, created_at)` |
| `idempotency_keys` | `UNIQUE(user_id, device_id, scope, key)`, `request_hash`, response body, `expires_at` (**7-day retention**, P1-1) |
| `admins` | `user_id`, `added_by`, `added_at`, `status`; initial rows `Tazorx7`, `asarepok`, `Neequaye279`; **changeable later** via `/admins` with append-only events |
| `review_decisions` | One row per admin approval with decision + recusal check |
| `config` | Server-side caps, contest window, backoff schedule, release delay, issuer keys |

## 4. Flows

### 4.1 Enrolment (P0 #2)

Register/login → opaque token → `GET /devices/challenge` → device generates Ed25519 key, signs `domain‖user_id‖challenge‖device_id‖public_key` → `POST /devices` → active key registered. Only public keys reach the server. Key storage prefers Android Keystore/StrongBox with a documented software fallback for devices/emulators without hardware support (P1-3).

### 4.2 Allowance issuance (P0 #5/#8)

`POST /allowances` with requested pesewas → server caps against balance/config (max 10,000 pesewas; wallet seeded at 500,000) → in one transaction: `available -= max`, `reserved += max`, insert allowance with `not_before = now − 60s`, server-chosen expiry, unique nonce/id → sign with issuer key → return to mobile. Multiple active allowances per device; multi-spend cumulative mode.

### 4.3 Offline payment (PRD §7, P0 #4)

Sender checks allowance locally (signature, advisory expiry, amount ≤ remaining), increments device-wide counter, signs canonical payment, QR (raw CBOR + `OFPAY1`) carries payment + signed allowance. Recipient verifies both signatures, binding fields, cap and advisory expiry locally; stores `PENDING_SYNC`. UI must not imply settlement.

### 4.4 Sync pipeline (P0 #1/#6/#7/#8, P1 #1)

1. Authenticate token; check device key active; enforce batch cap, idempotency key (`user_id, device_id, route, key`, canonicalised-body hash) and `batch_id`.
2. Sort items by `(server_received_at, tx_id)`.
3. Per item, own DB transaction: recompute `tx_id` → verify both signatures and domain versions → check allowance state, cap, strict server expiry → CAS `devices.last_counter` with `counter > last_seen` → unique constraints → update `consumed_minor` with `checked_add` → append events → insert outbox row with `held_until = now + contest_window` and `status='held'`.
4. Exceptions map to P0 #1 codes: exact duplicate → prior status; different payload / counter replay / exhaustion → conflict; expiry/bad signature → terminal rejection. Idempotency rows live 7 days; replays of status endpoints recompute current state.
5. Conflicting claims freeze both outbox rows and set `REVIEW_REQUIRED`; unused reserve stays frozen.

### 4.5 Review and settlement (P0 #7, P1 #5)

A review requires **2 of 3 designated app admins** (initially `Tazorx7`, `asarepok`, `Neequaye279`; list changeable via `/admins`); an admin party to either claim is recused. Outcomes: `CONFIRM_FIRST`, `AWARD_OTHER` (cancel first outbox, release its reserve, submit chosen claim), or `REJECT_BOTH` (release reserves, no chain record). Every decision appends an event. Late conflicts after chain submission are documented, not reversed. Admin-management changes append events and affect new reviews only.

### 4.6 Key rotation and revocation (P1 #3)

`POST /devices/{device_id}/keys` (user, new-key challenge proof) rotates the active key with bounded overlap: the old key still verifies its existing allowances until `min(24h, allowance.expires_at)` but cannot sign new payments. `POST /devices/{device_id}/revoke` (app admin) immediately invalidates the key. The device-wide counter continues across a same-device rotation; a new `device_id` starts a new lineage. Test cases cover old-key acceptance inside overlap, rejection after, continued-counter acceptance, and revoked-key rejection.

### 4.7 Chain worker (P0 #9, P1 #5)

Claim ready outbox rows with `FOR UPDATE SKIP LOCKED`; stale `locked_at` reclaimed after 2 minutes. Submit `tx_id` + network context to the Besu/QBFT contract; the contract stores `bytes32 txId` only, emits `TransactionRegistered(txId indexed, submitter, timestamp)`, reverts `AlreadyRegistered` on duplicates (backend treats a matching-hash revert as success), is restricted to an allowlisted backend signer, and has no pause function. `RECONCILED` only after receipt success + inclusion in one QBFT block (immediate finality). Failure → backoff 10s/1m/5m/15m/1h, 5 attempts; exhaustion → `REVIEW_REQUIRED`. Chain failures never roll back DB validation. No PII or private keys on-chain.

### 4.8 Release job

At `expires_at + contest_window`, return unused reserved pesewas to `available`, skipping allowances under review. Append a release event.

## 5. State machine

```
CREATED → LOCALLY_VERIFIED → PENDING_SYNC → SERVER_VALIDATION
    SERVER_VALIDATION → BLOCKCHAIN_SUBMISSION(held) → submitted → confirming → RECONCILED
    SERVER_VALIDATION → REJECTED | EXPIRED | CONFLICT | REVIEW_REQUIRED
    BLOCKCHAIN_SUBMISSION → REVIEW_REQUIRED (chain retry exhaustion)
```

Mobile sees the PRD states plus coarse `chain_status` (`held`, `submitted`, `confirming`, `confirmed`, `failed`). Terminal states are immutable except an explicit admin decision out of `REVIEW_REQUIRED`.

## 6. Consistency guarantees

- **Exactly-once DB effects** via unique keys, CAS and per-item transactions.
- **At-least-once chain submission** via the outbox; contract dedupes by `tx_id`.
- No two-phase commit across Postgres and Besu; never mark `RECONCILED` without a receipt.
- Reservation prevents issuing overlapping allowances against the same simulated balance.
- Key rotation never rewrites history: old-key validity windows and append-only events preserve the audit trail.

## 7. Gates and open items

- **PRD §5 gate: RESOLVED.** Besu/QBFT stays in the MVP; DevOps adds initial infrastructure later.
- App admins named: `Tazorx7`, `asarepok`, `Neequaye279` (all write collaborators); the list is changeable via `/admins`.
- Caps locked: 10,000 pesewas allowance, 500,000 pesewas wallet, 5-minute contest window.
- P1 decisions all recorded in PRD §21. Status checks remain unset until DevOps lands CI; the golden-vector job becomes required then.
