OFPAY
Design and Implementation of a Cryptographically Secured Offline Peer-to-Peer Payment System Using Permissioned Blockchain Technology
Project Name: OFPAY
Project Type: Software Engineering Project
Platform: Android-first mobile application
Primary Objective: Enable peer-to-peer payments without an active internet connection, with subsequent backend verification and blockchain reconciliation.
Tagline: Pay anywhere. Reconcile everywhere.

Implementation note: Sections 20 and 21 record the agreed P0 and P1 decisions that refine this PRD. Detailed contracts live in docs/API.md, docs/architecture.md and docs/threat-model.md; where those documents differ from the tentative proposals below, they take precedence.


⸻


1. Project Overview
OFPAY is an Android-first offline peer-to-peer payment prototype that enables users to exchange payment information and verify transactions without requiring an active internet connection.
The system uses QR codes to facilitate communication between devices, cryptographic signatures to authenticate transactions, local storage to retain offline transaction records, and a permissioned blockchain network to maintain a tamper-evident record of successfully reconciled transactions.
When internet connectivity becomes available, the application synchronises pending transactions with the backend. The backend validates the transactions before submitting eligible records to the blockchain network.
The prototype will use simulated Ghanaian cedi (GHS) balances. It will not process actual money or integrate with banks or mobile money providers.
2. Problem Statement
Most conventional digital payment systems depend on continuous internet connectivity to authenticate users, verify balances and process transactions.
This creates difficulties in environments where connectivity is unreliable, expensive or unavailable.
OFPAY aims to demonstrate how offline payment exchanges can be implemented through QR-based communication and cryptographic verification, while maintaining transaction integrity through later synchronisation and permissioned blockchain reconciliation.
3. Project Objectives
The project aims to:
Develop an Android application capable of initiating and receiving offline payments.
Implement QR-based payment information exchange between devices.
Use public-key cryptography to authenticate transactions.
Store transaction records securely on devices while offline.
Implement an online synchronisation mechanism for pending transactions.
Use a permissioned blockchain to record successfully reconciled transactions.
Detect duplicate transactions, replay attempts and conflicting spending claims.
Evaluate the performance and limitations of the proposed architecture.
4. Project Scope
Included in the MVP
User registration and authentication.
Simulated digital wallets and balances.
Device-specific cryptographic identities.
Offline spending allowances.
QR-based recipient identification.
QR-based payment exchange.
Local transaction verification.
Offline transaction storage.
Backend synchronisation.
Transaction validation and reconciliation.
Permissioned blockchain integration.
Transaction history and status tracking.
Excluded from the MVP
Actual financial transactions.
Bank and mobile money integrations.
NFC and Bluetooth payment exchange.
Cross-border payments.
Merchant payment processing.
Multi-hop offline payments.
Production-grade financial infrastructure.
5. Proposed Technology Stack
Component Technology
Mobile application Flutter
Primary platform Android
Backend Rust + Axum
Cryptographic engine Rust
Digital signatures Ed25519
Hashing SHA-256
Online database PostgreSQL
Local database SQLite
QR generation/scanning Flutter QR libraries
Blockchain Hyperledger Besu
Consensus QBFT
Smart contracts Solidity
Development containers Podman
CI/CD GitHub Actions
The proposed blockchain architecture should be reviewed before implementation to confirm that the additional complexity is justified by the project’s objectives. Gate reaffirmed (§21): Besu/QBFT stays in the MVP; DevOps will add initial infrastructure code later.
6. System Architecture
OFPAY will consist of five major components.
6.1 Mobile Application
The Flutter application provides the user interface and handles:
Wallet creation and management.
QR generation and scanning.
Payment initiation.
Offline transaction verification.
Local transaction storage.
Synchronisation when connectivity returns.Transaction history and status display.
6.2 Cryptographic Payment Engine
A Rust-based component responsible for:
Generating and managing cryptographic identities.
Signing payment transactions.
Verifying digital signatures.
Validating transaction payloads.
Checking allowance limits.
Detecting replay attempts.
Producing transaction hashes.
6.3 Backend API
A Rust/Axum service responsible for:
Authentication.
Wallet management.
Provisioning offline spending allowances.
Receiving synchronised transactions.
Validating transaction claims.
Detecting conflicting transactions.
Communicating with the blockchain network.
6.4 Database
PostgreSQL will store:
User records.
Wallet information.
Device registrations.
Spending allowances.
Transaction records.
Reconciliation results.
Blockchain transaction references.
SQLite will store locally required wallet metadata, allowances and pending transaction records on mobile devices.
6.5 Permissioned Blockchain
Hyperledger Besu will provide a permissioned blockchain network using QBFT consensus.
A Solidity smart contract will record reconciled transaction identifiers and relevant hashes, prevent duplicate registrations and emit transaction events.
Personal information, private keys and sensitive transaction payloads must not be stored directly on-chain.
7. How OFPAY Works
Stage 1: Wallet Setup
A user registers an account.
The backend creates or registers a wallet.
A device-specific cryptographic key pair is generated.
The public key is registered with the backend.
The user receives an offline spending allowance.
Stage 2: Recipient Identification
The recipient opens the application.
The recipient displays a wallet-identification QR code.
The sender scans the QR code.
The sender’s application identifies the intended recipient.
Stage 3: Offline Payment
The sender enters the payment amount.
The application checks the available offline allowance.
A transaction payload is created.
The transaction is digitally signed.
A payment QR code is generated.
The recipient scans the payment QR code.
The recipient verifies the transaction locally.
The transaction is stored as pending reconciliation.
Stage 4: Synchronisation
When either device regains internet connectivity:
Pending transaction records are submitted to the backend.
The backend validates the signatures and spending allowances.
The backend checks expiry, transaction identifiers and spending conflicts.
Valid transactions proceed to reconciliation.
The blockchain records the reconciled transaction.
The application updates the transaction status.
8. Transaction Lifecycle
The system should maintain explicit transaction states.
State Description
CREATED Transaction has been initiated.
LOCALLY_VERIFIED Recipient has verified the payment locally.
PENDING_SYNC Transaction is awaiting synchronisation.
SERVER_VALIDATION Backend is validating the transaction.
BLOCKCHAIN_SUBMISSION Validated transaction is being submitted to the blockchain.
RECONCILED Transaction has been successfully reconciled.
REJECTED Transaction failed validation.
CONFLICT Conflicting spending claim detected.
EXPIRED Relevant allowance or transaction has expired.
REVIEW_REQUIRED Transaction requires further investigation.
Important: Local verification does not mean that a transaction has been finally settled. The user interface must distinguish pending reconciliation from confirmed reconciliation.
Agreed refinement (§20): blockchain submission is held for a configurable contest window (5 minutes in the demo). A conflicting claim freezes both transactions in REVIEW_REQUIRED; two of three designated application admins decide whether to confirm the first claim, award the other, or reject both. Only accepted claims are written on-chain; late conflicts are documented, not reversed.
9. Security and Cryptographic Requirements
Security is a central part of OFPAY.
The implementation should consider:
Ed25519 digital signatures.
SHA-256 transaction hashing.
Unique transaction identifiers.
Transaction counters or nonces.
Expiring offline spending allowances.
Device-specific cryptographic identities.
QR payload integrity.
Replay attack prevention.
Duplicate transaction detection.
Secure local key storage.
Server-side validation.
Idempotent synchronisation.
Agreed refinements (§20): canonical CBOR with versioned domain strings; tx_id is the SHA-256 of the canonical payment bytes; device-wide monotonic counters accepted when greater than the last seen value; strict server-time expiry with no grace. Synchronisation uses an idempotency key, batch_id and tx_id.
Offline Spending Allowances
An allowance represents a portion of a user’s available funds that has been reserved by the backend for offline spending.
The backend provisions and signs an allowance containing information such as:Wallet identifier.
Device identifier.
Maximum permitted spending amount.
Expiry timestamp.
Allowance identifier.
Unique allowance nonce.
All monetary values must use integer minor units rather than floating-point numbers.
For example:
GH₵30.00 = 3000 pesewas.
Agreed refinements (§20): allowances are multi-spend with cumulative consumption; the backend reserves the full cap at issuance and auto-releases unused funds at expiry plus the contest window; several active allowances per device are allowed; the client requests an amount and the server enforces caps. Allowances are signed by a backend issuer key, bound to a wallet and device key, and carry a spend-mode field.
Double-Spending Considerations
A major challenge in offline payment systems is double spending.
Digital signatures alone do not prevent a user from copying wallet data, restoring an old device state or attempting to spend the same allowance multiple times.
OFPAY should use reserved offline allowances, device-bound keys where supported, transaction counters and backend conflict detection.
However, a software-only prototype cannot guarantee complete prevention of device cloning or rollback attacks.
The project must clearly demonstrate this limitation rather than claim that offline double spending is impossible.
10. Proposed Functional Requirements
ID Requirement
FR-01 Users must be able to register and authenticate.
FR-02 Users must have a simulated wallet and balance.
FR-03 Users must be able to receive offline spending allowances.
FR-04 Users must be able to identify recipients through QR codes.
FR-05 Users must be able to initiate offline payments.
FR-06 Recipients must be able to verify payment QR codes locally.
FR-07 Pending transactions must be stored locally.
FR-08 Transactions must synchronise when connectivity returns.
FR-09 The backend must validate and reconcile transactions.
FR-10 Reconciled transactions must be recorded on the blockchain.
FR-11 Users must be able to view transaction history and status.
11. Proposed API Endpoints
The backend may expose the following endpoints:
Method Endpoint Purpose
POST /auth/register Register a user
POST /auth/login Authenticate a user
GET /wallets/me Retrieve wallet details
GET /wallets/balance Retrieve simulated balance
POST /allowances Request an offline allowance
GET /allowances/active Retrieve active allowances
POST /transactions/verify Validate a submitted transaction
POST /transactions/sync Synchronise pending transactions
GET /transactions Retrieve transaction history
GET /transactions/{id} Retrieve transaction details
GET /blockchain/transaction/{id} Retrieve reconciliation information
These endpoints are proposed interfaces and should be refined during backend and mobile integration. Agreed refinements (§20, see docs/API.md): add GET /devices/challenge, POST /devices and GET /config/keys; all responses use a {data,error,meta} envelope; POST /transactions/sync returns one HTTP 200 with per-item statuses; POST /transactions/verify is read-only; admin decisions use POST /reviews/{id}/decisions.
12. Database Entities
The proposed database should contain the following core entities:
Users
Wallets
Devices
Device public keys
Offline allowances
Transactions
Blockchain records
Reconciliation events
The team should agree on database schemas and API contracts before parallel implementation begins.
13. Suggested Development Workstreams
These are areas of responsibility to distribute among team members, not fixed individual assignments.
Workstream A: Mobile Application
Flutter project setup.
Wallet interface.
QR generation and scanning.
Offline payment interface.
SQLite integration.
Transaction history.
Synchronisation interface.
Workstream B: Backend and Database
Rust/Axum API.
Authentication.
PostgreSQL schema and migrations.
Wallet management.
Offline allowance provisioning.
Transaction synchronisation.
Backend validation.
Workstream C: Cryptographic Payment Engine
Key generation and management.
Transaction signing.
Signature verification.
Transaction hashing.
Canonical transaction encoding.
Nonce and replay protection.
Allowance validation.
Workstream D: Blockchain Infrastructure
Besu network configuration.
QBFT validator setup.
Solidity smart contract.
Transaction registration.
Duplicate registration prevention.
Backend-to-blockchain integration.
Workstream E: Integration, Testing and Documentation
API contract coordination.
End-to-end integration.
Offline payment testing.
Security testing.
Double-spending simulations.
Performance evaluation.
Technical documentation.
Demonstration preparation.
The team should decide ownership based on members’ experience and availability.
14. Proposed Repository Structure
ofpay/
├── apps/
│   └── mobile/
├── services/
│   └── api/
├── crates/
│   └── payment-engine/
├── blockchain/│   ├── contracts/
│   ├── scripts/
│   └── network/
├── database/
│   ├── migrations/
│   └── seeds/
├── docs/
│   ├── PRD.md
│   ├── architecture.md
│   ├── API.md
│   ├── threat-model.md
│   └── diagrams/
├── tests/
│   ├── integration/
│   ├── security/
│   └── performance/
├── infrastructure/
│   ├── compose.yaml
│   └── ci/
├── .github/
│   └── workflows/
├── .gitignore
├── LICENSE
└── README.md
15. Testing and Evaluation
The project should be evaluated using the following scenarios:
Offline payment: Two devices successfully exchange a payment QR code without internet access.
Transaction synchronisation: A pending transaction is synchronised after connectivity returns.
Invalid signature: A modified payment payload is rejected.
Replay attempt: A previously processed transaction cannot be registered again.
Double-spending simulation: Conflicting claims against the same allowance are detected.
Interrupted synchronisation: Repeated synchronisation does not create duplicate transactions.
Expired allowance: An expired allowance cannot be used to authorise a new payment.
Agreed additions (§20): contested sync (two conflicting claims during the 5-minute window produce one settlement and one REVIEW_REQUIRED with a 2-of-3 admin decision), counter rollback detection, interrupted batch replay, chain retry exhaustion, and canonical golden-vector checks. Blockchain confirmation latency includes the contest window.
Performance Metrics
Local transaction verification time.
QR payload size.
Offline transaction processing time.
Synchronisation latency.
Blockchain confirmation latency.
Number of transactions processed per second.
Local storage overhead.
16. Proposed Development Timeline
Period Target
Week 1 Requirements, architecture, repository, database and API contracts
Week 2 Mobile foundation, cryptographic engine, QR exchange and local storage
Week 3 Backend synchronisation, Besu infrastructure and smart contract
Week 4 Integration, security testing, performance evaluation and demonstration
The timeline should be adjusted to the actual project deadline and team capacity.
17. Expected Deliverables
At completion, the team should produce:
A functional Android application.
A Rust-based backend API.
A cryptographic payment engine.
A working permissioned blockchain network.
A Solidity smart contract.
PostgreSQL and SQLite database implementations.
Automated tests.
Technical documentation.
Architecture and transaction-flow diagrams.
A working demonstration of offline payment and subsequent reconciliation.
18. Definition of Success
OFPAY will be considered successful when:
Two Android devices can exchange a payment without internet connectivity.
The recipient can locally verify the signed payment.
The transaction is stored locally while offline.
The transaction can be synchronised after connectivity returns.
The backend validates the transaction.
A valid transaction is recorded on the permissioned blockchain.
Duplicate and conflicting transaction claims can be detected.
The team can demonstrate the system’s security model and limitations.
19. Important Architectural Principle
OFPAY does not use blockchain to perform offline consensus.
The mobile devices handle offline payment exchange and local verification. The backend performs authoritative validation when connectivity returns. The permissioned blockchain provides a shared, tamper-evident record of reconciled transactions.
This distinction is essential to the correctness of the architecture.

20. Agreed Implementation Decisions (P0)
The following decisions refine the proposals above and are binding for implementation. Full details are in docs/API.md, docs/architecture.md and docs/threat-model.md. These decisions are confirmed; the initial three application admins and final caps are named in §21. No application code is to be written until the Rust workstream is unblocked by group discussion.

ID Decision
P0-01 API: {data,error,meta} envelope; 422 terminal / 409 conflict / 200 duplicate; sync returns one 200 with per-item results; per-item DB transactions; idempotency via request key, batch_id and tx_id.
P0-02 Auth: opaque 24h session token (hash stored); separate device enrolment with a signed challenge proof; client-generated UUID v4 device_id; sync requires token plus device signature.
P0-03 Encoding: canonical CBOR; tx_id = SHA-256 of domain-prefixed canonical payment bytes; v1 domain strings inside signed bytes; fixed field set; issuer public keys from GET /config/keys plus app-pinned copy.
P0-04 Payloads: allowance carries spend mode; payment binds allowance_hash and QR carries the full signed allowance; recipient bound by wallet ID only; chain_context is a network ID; 16-byte UUIDs and int64 seconds; device timestamps signed but advisory.
P0-05 Allowances: multi-spend cumulative; reserve full cap at issuance; auto-release unused at expiry plus contest window; multiple active allowances per device; client requests amount, server caps.
P0-06 Counters: device-wide monotonic; accept counter greater than last seen (gaps allowed); devices.last_counter compare-and-set; replay leads to review; recovery by re-enrolment with a new key.
P0-07 Conflicts: provisional first acceptance with a configurable 5-minute contest window; losing claim frozen in REVIEW_REQUIRED; two of three designated app admins decide, with recusal of involved parties; ordering by server_received_at then tx_id; only accepted claims on-chain; late conflicts documented, not reversed.
P0-08 Time and money: server UTC only; strict expiry with no grace; not_before = server_now minus 60s; explicit server caps with checked arithmetic; release at expiry plus contest window.
P0-09 Blockchain: receipt plus one QBFT block is RECONCILED; backoff with five attempts; exhaustion leads to REVIEW_REQUIRED; duplicate registration with matching hash is success; mobile sees coarse chain_status.

Initial application admins: Tazorx7, asarepok and Neequaye279. All three are GitHub write collaborators; application admin status is stored in the app-level admins table, independent of repository roles, and the list must be changeable later through an admin-management endpoint with append-only events.

Confirmed caps: 10,000 pesewas maximum per allowance request, 500,000 pesewas initial simulated wallet balance, 5-minute configurable contest window. Section 5's blockchain-complexity gate is reaffirmed in §21.

21. Agreed Implementation Decisions (P1)
The following decisions complete the design contract. Full details are in docs/API.md, docs/architecture.md and docs/threat-model.md.

ID Decision
P1-01 Idempotency: records retained 7 days; replay detected by stored hash of the canonicalised body; status endpoints recompute current DB state; scope is (user_id, device_id, route, key); non-status mutations return the stored response.
P1-02 QR: raw binary CBOR in byte mode with an OFPAY1 header; 600-byte target / 800-byte hard cap at ECC level M; compact 8-byte key_id; golden-vector CI size check; no multi-frame QR in MVP.
P1-03 Keys: user-initiated rotation with new-key challenge proof plus admin revocation; bounded old-key overlap until min(24h, allowance.expires_at) with no new payments; device counter continues on same-device rotation; StrongBox preferred with a documented software fallback; key_id is an 8-byte hash of the public key.
P1-04 Vectors: full canonical encoding/signature/hash set for allowance, payment, wallet-ID QR and enrolment challenge plus negative cases (tampering, wrong domain, unknown version, expiry, replayed counter) and a QR-size fixture; engine-generated files checked into tests/vectors/v1/; dedicated test-only keys and RFC 8032 primitives; required CI check once DevOps lands workflows; v1 immutable with a new directory per schema bump.
P1-05 Chain: contract stores tx_id only (bytes32); event is TransactionRegistered(txId indexed, submitter, timestamp); duplicate registration reverts AlreadyRegistered and the backend maps a matching-hash revert to success; allowlisted backend signer only; no pause or emergency admin function in MVP.

Open items: none blocking the design contract. Status checks remain unset until DevOps adds CI; the golden-vector job becomes a required check at that point. Application code remains on hold until the group unblocks the Rust workstream.


⸻


Project: OFPAY
Core idea: Cryptographically secured offline payment exchange with online reconciliation through a permissioned blockchain.
Development principle: Modular implementation, clearly defined interfaces, independent testing and continuous integration.
