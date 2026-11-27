# OFPAY threat model (agreed P0 and P1 decisions)

**Status:** agreed P0 and P1 decisions (see `PRD.md` §§20–21). Design record only; no code yet. This document makes PRD §9's limitations explicit rather than hiding them.

## 1. Principle and trust boundaries

PRD §19: OFPAY does **not** perform offline consensus. Phones exchange and locally verify signed payments; the backend is the authority; the permissioned chain is a tamper-evident record of **reconciled** transactions only.

| Boundary | Trusted for | Not trusted for |
| --- | --- | --- |
| Device key store | Signing with the device key (StrongBox preferred; Keystore/software fallback documented as weaker) | Resistance to a rooted device or physical extraction |
| Mobile app / SQLite | Local verification, pending-queue display, counter persistence | Final settlement, authoritative balance, authoritative time |
| Axum backend | Balances, reservations, expiry, conflict decisions, chain submission | Nothing — it is the authority but is still audited via events |
| Besu/QBFT contract | Duplicate `tx_id` registration, event evidence | Validation, balances, privacy |
| QR channel | Transport of raw CBOR signed bytes (`OFPAY1`) | Confidentiality; display text that is not bound in the signature |

Assets: device private keys, backend allowance issuer key, signed allowances, pending transactions, simulated wallet balances, canonical payload format. No real money, bank or mobile-money integration (PRD §4).

## 2. Layered defences

Preventive → detective → corrective:

| Threat | Preventive | Detective | Corrective / residual |
| --- | --- | --- | --- |
| Replay of an old payment | Ed25519 signature, domain strings, `tx_id` hash, device-wide counter | `UNIQUE(tx_id)`; `counter > last_seen` CAS | `DUPLICATE_TX` returns prior status; replayed counter → `REVIEW_REQUIRED` |
| Duplicate submission | Idempotency key (`user_id, device_id, route, key`, canonicalised-body hash, 7-day retention) + `batch_id` + `tx_id` | Per-item status comparison; replays recompute current state | No second debit; prior status returned |
| Double spend via copied wallet / restored backup / cloned device | Reserved allowance bound to device `key_id`; monotonic counter; cumulative cap | Two claims with same allowance/counter or conflicting spends during the 5-min contest window | One claim settles, loser frozen → `REVIEW_REQUIRED`; **cannot be prevented offline in software** |
| Allowance reuse / overspend | `allowance_hash` bound in payment; signed cap; full allowance in QR | `consumed_minor + amount <= max` with `checked_add`; `ALLOWANCE_EXHAUSTED` | Conflict freeze; unused reserve held for review |
| Tampered QR payload | Signature over canonical CBOR; recipient wallet ID bound; `chain_context`; `OFPAY1` header | Backend re-verifies with registered keys and recomputes `tx_id` | `SIGNATURE_INVALID` / `TX_ID_MISMATCH` → `REJECTED`; UI shows only verified fields |
| Expired allowance or payment | Signed `expires_at`; short-lived validity | **Strict server-time check, no grace** | `EXPIRED` (422); unused reserve auto-released at expiry + contest window unless under review |
| Clock skew / backdated claim | Device timestamps signed but advisory only | Server time authoritative; batch ordering by `server_received_at` | A backdated `created_at_device` cannot win "first" status |
| Key extraction / lost key | StrongBox preferred, software fallback labelled weaker; opaque key handles; zeroized seeds; private key never in Dart | User rotation with new-key proof; admin revocation via `key_id` validity windows | Bounded overlap for old-key allowances (≤24h, ≤expiry); counter continues on same device; recovery = new `device_id`/key; rooted devices out of scope |
| Canonicalisation drift / cross-protocol reuse | One Rust engine owns canonical CBOR; domain strings inside signed bytes; fixed field set; immutable `v1/` vectors | Golden vectors in engine/backend/mobile CI as a required check once DevOps lands workflows | `SCHEMA_UNSUPPORTED` / signature failure; no silent field changes; schema bumps get `v2/` |
| QR size / scan failure | Raw binary CBOR + compact 8-byte `key_id`; 600 B target / 800 B hard cap at ECC M | Golden-vector size fixture fails CI above target | No multi-frame QR in MVP; a real oversize requires review, not silent field changes |
| Chain failure / stuck submission | Outbox written in same DB commit as validation; held during contest window | Bounded backoff with attempt counter | Retry exhaustion → `REVIEW_REQUIRED`; never `RECONCILED` without receipt |
| On-chain privacy leak | Contract stores `txId` only + minimal event (`txId`, submitter, time); allowlisted signer | No amount, wallet ID or identity fields in call/event | No pause/admin key in MVP; disputes stay off-chain |
| Malicious or monopolised dispute decision | App-level `admins` table (`Tazorx7`, `asarepok`, `Neequaye279`, changeable via `/admins`); recusal of involved admins | 2-of-3 distinct approvals recorded in append-only `review_decisions`; admin changes append events | Owner alone cannot resolve; late conflicts after chain submission are not reversed |

## 3. Honest limitations to demonstrate

1. **Offline double spending is not prevented.** Two signed payments against the same allowance can both be locally verified. The backend detects the conflict at sync. Demonstrate this with PRD §15's double-spending simulation: two `LOCALLY_VERIFIED` payments → one `RECONCILED`, one `REVIEW_REQUIRED`.
2. **Rollback/cloning is indistinguishable.** A restored backup with an old counter is rejected by server-side CAS, but the original device and a clone sharing the same key look identical. Recovery re-enrols with a new key; evidence of the old conflict is retained.
3. **Late conflicts cannot undo chain records.** QBFT finality plus the 5-minute contest window reduce but do not eliminate a conflicting claim arriving after submission. Such claims stay as off-chain evidence; no on-chain reversal exists in MVP.
4. **Allowances can expire while offline.** Users may hold unusable credentials; UI must show advisory local expiry and authoritative server status after sync.
5. **Software key storage is extractable on rooted devices.** StrongBox is preferred where supported; the software fallback is documented as weaker.
6. **Old-key overlap is a real exposure window.** After user rotation, a compromised old key can still authorise against its existing allowances until `min(24h, allowance.expires_at)`. Short allowance lifetimes keep the window small; admin revocation overrides it immediately.
7. **QR size is a measured metric with a fixed budget (PRD §15).** The QR carries the full signed allowance so the recipient can verify offline; raw CBOR with 8-byte `key_id` targets 600 B, hard cap 800 B at ECC M, verified by golden-vector CI.
8. **Besu/QBFT gate reaffirmed.** Complexity is accepted for the MVP; DevOps adds initial infrastructure later.

## 4. Test scenarios from PRD §15

| Scenario | Expected result under these decisions |
| --- | --- |
| Offline payment | Two devices exchange QR; recipient verifies both signatures locally; state `PENDING_SYNC` |
| Synchronisation | Batch accepted per item; outbox held for contest window; status moves to `BLOCKCHAIN_SUBMISSION` |
| Invalid signature | `SIGNATURE_INVALID` 422 → `REJECTED`; golden vector covers tampered bytes |
| Replay attempt | Same `tx_id` returns prior status; different bytes → `DUPLICATE_TX_DIFFERENT_PAYLOAD`; old counter → `COUNTER_REPLAY` |
| Double-spending simulation | One claim settles, one `REVIEW_REQUIRED`; 2-of-3 admin review can award either before chain submission |
| Interrupted sync | Repeated batch/key replays produce identical per-item statuses and no second debit (7-day idempotency window) |
| Expired allowance | Strict server-time rejection with no grace; reserve released after expiry + contest window |
| Chain failure | Retry backoff; after 5 attempts `REVIEW_REQUIRED`; no `RECONCILED` without receipt |
| Key rotation | Old key accepted inside overlap, rejected after; continued counter accepted; revoked key rejected |
| Golden vectors | Engine/backend/mobile decode the same `v1/` fixtures; CI fails on drift or QR oversize |

## 5. Operational notes

- Admin decisions, admin-list changes, auto-release, key rotation/revocation and conflict outcomes are append-only events; no silent rewrites.
- Only accepted claims go on-chain; the contract stores `tx_id` (the SHA-256 hash) only — no network field, PII, private keys or sensitive payloads.
- All server configuration (caps, contest window, backoff, release delay, issuer keys) is versioned and reviewed like code.
- Status checks remain unset until DevOps lands CI; the golden-vector job then becomes a required check on `main` and `master`.
