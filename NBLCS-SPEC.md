## Problem

Bitcoin Lightning Network implementations are failing under attacks (pinning, jamming, force-close races) that AI has helped surface. I am a NANO (XNO) fan and believe Nano — zero-fee and fast — can serve as the backend under the tech so customers affected by Bitcoin LN's shutdown can keep operating.

Propose a Nano tech stack that makes Bitcoin LN work again by REPLACING its backend, with these hard constraints:
- The customer-facing Bitcoin interface stays exactly the same: accepting Bitcoin and sending Bitcoin are unchanged. Customers modify nothing. Only the backend changes and nobody feels the change.
- Do NOT run a custom Nano node — rely only on the live Nano network.
- The result must actually be secure and workable (not merely look like Bitcoin LN).


# Nano-Based Bitcoin Lightning Compatibility Stack (NBLCS)

This document presents the complete, buildable tech stack that replaces the Bitcoin Lightning Network backend with a Nano (XNO) settlement rail, preserves the customer-facing Bitcoin interface unchanged, runs only on the live Nano network, and structurally removes pinning, jamming, and force-close races from the customer backend.

It also specifies the external Bitcoin Lightning interoperability path: how a gateway customer can pay an arbitrary external BOLT11 invoice and how an external Lightning wallet can pay a gateway-generated invoice, without running a customer-facing Lightning backend inside NBLCS.

---

## 1. LN Backend Inventory

### 1.1 LN Backend Components Replaced

1. Channel open/fund: 2-of-2 multisig P2WSH funding transaction built from both parties' UTXOs; requires cross-signing of initial commitment transactions before broadcast.
2. Commitment transactions: each party holds a version spending the funding output with to_local (with revocation and CSV) and to_remote (P2WPKH) outputs, plus HTLC outputs with success/timeout paths.
3. State transitions: update protocol where both parties sign new commitments and exchange revocation secrets for the previous state, using a per-commitment secret hash chain.
4. Routing table: source-based onion routing (Sphinx) where sender constructs the path and layered payloads; each hop decrypts, extracts next hop/amount/fee, and forwards via HTLCs.
5. HTLC machinery: per-channel HTLC outputs with ADDED/FULFILLED/FAILED/EXPIRED states; lock condition via payment_hash, settlement via preimage, timeout refund.
6. Fee policy: funder proposes fee rate via update_fee; anchor outputs enable CPFP fee bumping; RBF for unconfirmed commitment; dust limiting trims low-value outputs.
7. Watchtowers: third-party services monitor for revoked commitment broadcasts, store encrypted penalty transactions, and enforce penalties or CPFP-bump force-closes.
8. Force-close arbitration: unilateral broadcast of latest commitment; to_local spend after CSV delay, to_remote immediate, HTLC resolution via CLTV/CSV, penalty for old states.
9. Channel rebalancing: mechanisms like splicing, submarine swaps, or dual-funded channels to adjust capacity and maintain liquidity without closing/reopening.

None of these exist in the NBLCS customer backend.

### 1.2 Customer Interface Surface Preserved

1. BOLT11 invoice: bech32-encoded string starting `lnbc`/`lntb` with payment hash, amount, timestamp, description, expiry, and recoverable signature from a node pubkey.
2. On-chain deposit address: native SegWit (bc1) derived from a customer-controlled key, matching prior derivation path and format.
3. Confirmation feedback: instant balance credit on payment receipt with history entry (amount, description, timestamp), treated as final and irreversible.
4. Outbound send flow: scan/paste invoice or address, display amount/description, confirm, atomic payment (full amount or no funds leave) with success/failure indication.
5. Balance display: real-time total in BTC/satoshis reflecting actual spendable amount, updated instantly on sends/receives.
6. Payment history: chronological list of all payments with invoice details, amounts, timestamps, and status (success/failure).
7. Invoice expiry: configurable (default 1 hour) with clear indication when expired and unpayable.
8. Amountless invoice handling: allow customer to enter satoshi amount at payment time; payment succeeds only if entered amount matches expected value.
9. Error handling: identical error messages and retry options for failures (expired invoice, insufficient balance, route failure).

---

## 2. Nano Capabilities and Hard Limits

### 2.1 Nano Usable Properties

1. Near-instant finality: representative quorum (67% online vote weight) confirms blocks in 0.3–1.2s typical, p95 3s, p99 8s; once cemented, irreversible.
2. Zero fees: transfers cost only PoW computation, no miner/validator fees, full amount transferred.
3. Account-based block lattice: each account has its own chain, send/receive semantics: send reduces balance, creates pending; receive claims it, increasing balance.
4. Dynamic PoW anti-spam: work difficulty adjusts based on network load; current minimum 8 leading zero bits for send/receive, 5 for change; typical PoW ~2s on CPU.
5. Live public network with public infrastructure: multiple public RPCs (rpc.nano.org, nanocrawler.cc, mynano.ninja, nanode.co), websocket for live confirmations, no custom node required; trust model: query multiple nodes, require majority agreement.
6. Cryptographic primitives: Ed25519 signatures, Blake2b-256 hashing, 256-bit keypair generation, address derivation with checksum.
7. Confirmation and cementing: block confirmed when >50% online weight votes; cemented when all predecessors confirmed; irreversible after cementing.
8. Fork resolution: first block to gain majority wins; double-spend rejected; representatives vote only once per height.
9. Async send/receive: sender can send without receiver being online; receiver claims later, enabling off-chain coordination.

### 2.2 Nano Hard Limits

1. No scripting or conditional execution: cannot implement HTLCs, timelocks, multisig, or any conditional payment logic in-band.
2. No timelocks: transactions are immediately valid and final; no time-based validity or expiration.
3. No state channels: no channel abstraction; any channel state must be tracked externally.
4. Single balance per account: no multiple outputs or UTXO model; each account has one balance and one representative.
5. No arbitrary data storage: only balance and representative; cannot commit to secrets or hashes on-chain.
6. Receive requires an existing account: the first receive must be an open block; subsequent receives require the account to already have an open block.
7. No reversible transactions: once confirmed, cannot be contested or reversed; no dispute window.
8. No native multisig: only single Ed25519 private key controls an account; multisig must be off-chain.

---

## 3. Attack Threats and Immunity Requirements

### 3.1 Exploited LN Properties

1. Mempool as a malleable staging area: transactions can be accepted as valid but not mined, and a first-seen rule lets a low-fee conflicting transaction block a correct higher-fee transaction via RBF-disabling or package pinning.
2. Time-critical HTLC claims: CLTV/CSV deadlines make settlement a race against block inclusion, so an attacker only needs to delay, not invalidate, the honest transaction.
3. Shared, lockable liquidity pools: channel capacity and HTLC slots are finite resources that an attacker can flood with long-timeout pending payments, denying service to legitimate users.
4. Multi-hop forwarding with per-hop timeouts: each hop adds a delay window during which funds are locked, enabling cascade jamming with minimal attacker cost.
5. Two-phase unilateral exit: a force-close is a provisional commitment followed by a required response (penalty/sweep) within a timelock window, creating race incentives and making settlement vulnerable to pinning, flooding, or eclipse.
6. Fee-market competition for block space: block inclusion is an auction, so an attacker can outbid or price out the honest party's settlement transaction.

### 3.2 Immunity Requirements

1. No mempool, no fee auction, no non-final waiting area: every valid transaction is either confirmed by network consensus or rejected outright; no conflicting transaction can be parked to block a later correct one.
2. Single-phase, non-interactive, unconditional finality: a settlement or payment transaction is itself the final state, with no timelocks, no second-level transactions, no challenge window, and no required response.
3. Atomic and direct value transfer with no intermediary-held escrow: payer sends directly to payee in one transaction; no multi-hop routes, no per-hop timeouts, no pending 'offered' state.
4. No shared, lockable liquidity pool: each party's balance is under its own exclusive control; no third party can place a hold, and there are no HTLC slots or channel capacities to flood.
5. Unique spendability of the current state: exactly one key can spend a channel's funds at any moment, and that key can only produce the current state; old states are structurally unspendable, so broadcasting them is impossible.
6. Deterministic conflict resolution by ledger rules: if claims conflict, the network's consensus (vote weight, block height) decides automatically, invalidating older or conflicting claims without fee competition or first-seen ordering.



##  **Federated multi-sig custody keyed to Nano account activity, with a Lightning Interop Service (LIS) for external Bitcoin Lightning interoperability.**

The core NBLCS backend replaces LN's channel-based state with a simple custodial model: customer balances are Bitcoin-denominated claims tracked by Nano account activity; Bitcoin sits in a federation-controlled multi-sig custody pool; internal payments are final Nano transfers; on-chain Bitcoin deposits and withdrawals are plain Bitcoin transactions.

External Lightning interoperability is provided by a separate **Lightning Interop Service**, which uses independent Lightning Liquidity Providers (LLPs) as transit. The gateway itself never opens a Lightning channel, never holds an HTLC, and never has a force-close path. The LLPs are external service providers that already operate Lightning nodes; they are used as a transport utility, not as the customer backend.

This is the only architecture that satisfies the hard constraint "customer-facing Bitcoin interface stays exactly the same" for real-world Lightning users while keeping pinning, jamming, and force-close races out of the customer backend.

### 6.2 High-Level Shape

- **Customer Bitcoin Gateway**: Presents a byte-identical Bitcoin interface to customers. Generates BOLT11 invoices, accepts `payinvoice` calls, generates unique Bitcoin deposit addresses, monitors incoming Bitcoin transactions, and broadcasts withdrawal transactions to customer-provided Bitcoin addresses.
- **Nano Settlement Rail**: Handles all internal value movement between user balances. Monitors the live Nano network for transfers to federation-controlled Nano accounts and sends Nano transfers when users make internal payments or request withdrawals.
- **Bitcoin Custody Pool**: Holds all Bitcoin reserves in a multi-sig address (e.g., 3-of-5) controlled by the federation. Funds move for deposits, withdrawals, and net settlement with LLPs.
- **Reconciliation Ledger**: Maintains the authoritative mapping between Nano account balances and Bitcoin claims. Records invoices, payments, deposits, withdrawals, LLP receivables/payables, exchange rates, and audit events.
- **Lightning Interop Service (LIS)**: Connects the gateway to multiple independent Lightning Liquidity Providers. Handles external BOLT11 invoice creation, external invoice payment, LLP callbacks, LLP selection, idempotency, and LLP collateral management.
- **Reserve Manager**: Maintains the BTC/XNO exchange rate and reserve health.
- **Monitoring & Alerting**: Continuously verifies all components, including LLP health and collateral.

---

## 7. Bitcoin Interface Compatibility Design

### 7.1 Compat Mechanisms

#### BOLT11 Invoice

**Customer behavior (unchanged):** A customer calls `createinvoice` and receives a normal `lnbc...` invoice. The invoice can be shown to an external payer, and an external Lightning wallet can pay it. Alternatively, another gateway customer can pay it with the gateway's `payinvoice` call.

**Backend mechanism:**

1. The gateway generates a random 32-byte preimage and computes `payment_hash = SHA256(preimage)`.
2. The ledger stores the invoice record: payment_hash, preimage, amount, description, expiry, payee customer, status=`PENDING_SINGLE_USE`.
3. The LIS selects an active, collateralized LLP.
4. The gateway calls the LLP API with the payment_hash, preimage, amount, description, and expiry. The LLP creates and signs a real BOLT11 invoice using its own Lightning node pubkey and returns the BOLT11 string.
5. The gateway returns that BOLT11 string to the customer. To the customer, it is indistinguishable from a normal LN invoice.

**Internal payment:** If another gateway customer calls `payinvoice` with that invoice, the gateway settles it internally via a Nano transfer from payer to payee, reveals the stored preimage, and marks the invoice `PAID`. The gateway also calls `cancel_invoice` on the LLP so an external duplicate payment is rejected.

**External inbound:** If an external Lightning wallet pays the invoice, the LLP's Lightning node receives the payment, settles with the preimage, and sends a signed callback to the gateway with the payment_hash, preimage, and amount. The gateway verifies `SHA256(preimage) == payment_hash`, verifies the amount and single-use status, credits the customer's Nano balance, and marks the invoice `PAID`.

**External outbound:** If a gateway customer calls `payinvoice` with an arbitrary external BOLT11 invoice, the LIS selects an LLP and calls `pay_invoice`. The LLP routes the payment over the public Lightning Network to the external recipient. Real BTC reaches the external recipient from the LLP's channel liquidity. The LLP reports success with the preimage and fee, and the gateway debits the customer's Nano balance.

**Shim implementation:**

- `createinvoice` returns a BOLT11 string from an LLP.
- `payinvoice` accepts both gateway-generated invoices and external invoices.
- The response shape is `{preimage, payment_hash, status}` for both cases.

#### On-Chain Deposit

**Customer behavior (unchanged):** Generate a native SegWit (bc1) Bitcoin address and receive BTC.

**Backend mechanism:** The gateway derives a unique Bitcoin address per customer using BIP32 from the federation master public key. It monitors the Bitcoin blockchain for transactions to that address. After 6 confirmations, it credits the customer's Nano balance with an equivalent amount of Nano, pegged to BTC at the current federation-set rate.

**Shim implementation:** `getnewaddress` returns a valid `bc1...` address. The gateway runs a Bitcoin full node (or uses an external API as fallback) to watch for UTXOs.

#### Confirmation Feedback

**Customer behavior (unchanged):** Instant balance credit on payment receipt, with history entry treated as final.

**Backend mechanism:** Internal payments settle via Nano transfers, which are near-instant and final. External inbound payments credit the customer immediately after the LLP's signed callback is verified. On-chain deposits show a pending state until 6 Bitcoin confirmations, then credit.

**Shim implementation:** The gateway updates the customer's balance in real time as ledger events occur. The history API returns the same structure as Bitcoin's `listtransactions`.

#### Outbound Send

**Customer behavior (unchanged):** Scan/paste a BOLT11 invoice or a Bitcoin address, display amount/description, confirm, and send BTC atomically.

**Backend mechanism:**

- If the invoice is a gateway-generated invoice, the gateway settles internally via Nano.
- If the invoice is an external BOLT11 invoice, the LIS pays it through an LLP over the public Lightning Network. Real BTC reaches the external recipient.
- If the destination is a raw Bitcoin address, the gateway debits the customer's Nano balance and broadcasts a Bitcoin withdrawal from the custody pool.

**Shim implementation:** `payinvoice` and `sendtoaddress` RPCs. The ledger enforces a hold on funds before the external attempt; if the attempt definitively fails, the hold is released and the customer sees the same failure semantics as before.

#### Balance Display

**Customer behavior (unchanged):** Real-time total in BTC/satoshis reflecting actual spendable amount.

**Backend mechanism:** The ledger stores the customer's BTC claim directly. Nano balances are updated in parallel as the settlement rail. The displayed balance is the BTC claim.

**Shim implementation:** `getbalance` returns `{total_btc, confirmed_btc, unconfirmed_btc}` with 8-decimal BTC formatting.

#### Payment History

**Customer behavior (unchanged):** Chronological list of all payments with invoice details, amounts, timestamps, and status.

**Backend mechanism:** The ledger records every internal Nano transfer, external inbound settlement, external outbound payment, Bitcoin deposit, and Bitcoin withdrawal as an immutable event.

**Shim implementation:** `listtransactions` formats events into Bitcoin Core-compatible output. For internal payments, pseudo-txid = Nano block hash. For external payments, the LLP's payment hash is included.

#### Invoice Expiry

**Customer behavior (unchanged):** Configurable expiry (default 1 hour) with clear indication when expired and unpayable.

**Backend mechanism:** The gateway stores the invoice creation timestamp and expiry. The LLP also enforces expiry and rejects payments after expiry. Internal `payinvoice` checks expiry before settling.

**Shim implementation:** `createinvoice` accepts `expiry_seconds`; `payinvoice` returns the standard expired-invoice error.

#### Amountless Invoice

**Customer behavior (unchanged):** Customer can enter the satoshi amount at payment time; payment succeeds only if the entered amount matches the expected value.

**Backend mechanism:** For amountless invoices, the invoice record stores no fixed amount. When an external payer pays, the LLP reports the actual amount and the gateway credits that amount. When a gateway customer pays an external amountless invoice, the customer enters the amount and the LLP routes exactly that amount. For fixed-amount invoices, the LLP rejects any mismatch.

**Shim implementation:** `createinvoice` may omit the amount. `payinvoice` accepts an amount parameter.

#### Error Handling

**Customer behavior (unchanged):** Identical error messages and retry options for expired invoice, insufficient balance, route failure, etc.

**Backend mechanism:** The gateway maps all failures to Bitcoin/LND-style JSON-RPC error codes.

- Insufficient balance → `insufficient funds`
- Expired invoice → `invoice expired`
- Invalid invoice → `invalid invoice`
- LLPs exhausted → `route not found`
- LLP timeout with unknown result → `payment timeout`, with a hold left in place until reconciliation

**Shim implementation:** The RPC layer catches exceptions and returns Bitcoin-compatible JSON-RPC error objects.

---

## 8. Lightning Interop Service (LIS) Specification

The LIS is the external Bitcoin Lightning interoperability path. It is not a Lightning backend; it is a broker layer over independent Lightning Liquidity Providers.

### 8.1 Lightning Liquidity Providers

An LLP is an external, independently operated Lightning service provider with:

- Public Lightning node(s),
- Public API for `create_invoice`, `pay_invoice`, `cancel_invoice`, and callbacks,
- Posted collateral or escrow,
- Public attestation of solvency and operational status.

NBLCS does not run the LLP nodes and does not hold customer balances at LLPs.

### 8.2 LLP Selection Policy

- At least 3 LLPs must be active at all times.
- No single LLP handles more than 33% of external volume.
- Each LLP must post collateral at least equal to its maximum outstanding receivable from inbound settlements.
- Each LLP must have a verifiable, signed API key and TLS endpoint.
- LLPs are selected round-robin for invoice creation, but only among healthy LLPs with available capacity.

### 8.3 LLP Data Model

```
llps (
  id,
  name,
  api_url,
  node_pubkey,
  api_public_key,
  collateral_sat,
  max_inbound_sat,
  max_outbound_sat,
  prepaid_balance_sat,
  receivable_balance_sat,
  health_score,
  enabled
)

outbound_payments (
  id,
  customer_id,
  bolt11,
  payment_hash,
  amount_msat,
  status,             -- PENDING, SUCCESS, FAILED, UNRESOLVED
  llp_id,
  llp_request_id,
  llp_fee_msat,
  created_at,
  settled_at
)

inbound_settlements (
  payment_hash,
  llp_id,
  amount_msat,
  preimage,
  callback_signature,
  status,
  created_at
)
```

### 8.4 External Inbound Flow

1. Customer calls `createinvoice`.
2. Ledger stores the invoice record with preimage and payment_hash.
3. LIS selects an LLP.
4. Gateway sends `create_invoice` to the LLP API with:
   - payment_hash,
   - preimage,
   - amount_msat,
   - description,
   - expiry,
   - signed gateway authorization.
5. LLP returns a signed BOLT11 invoice with its own node pubkey and route hint.
6. Gateway returns the BOLT11 string to the customer.
7. External payer pays the invoice over the public Lightning Network.
8. LLP settles and sends a signed callback to the gateway:
   - payment_hash,
   - preimage,
   - amount_msat,
   - timestamp.
9. Gateway verifies:
   - `SHA256(preimage) == payment_hash`,
   - invoice status is `PENDING_SINGLE_USE`,
   - amount is acceptable,
   - callback signature is valid.
10. Ledger credits the customer's BTC claim and creates an issuance Nano send from a reserve account to the customer's Nano account.
11. The LLP receivable is recorded as a backing asset.
12. Net settlement later moves BTC from the LLP to the custody pool or reduces a prepaid balance.

### 8.5 External Outbound Flow

1. Customer calls `payinvoice` with an external BOLT11 invoice.
2. Gateway validates the invoice and creates an `OUTBOUND_PENDING` hold on the customer's balance.
3. LIS selects a single LLP with outbound capacity and prepaid balance.
4. Gateway calls `pay_invoice` with:
   - a unique request id,
   - the BOLT11 invoice,
   - amount_msat,
   - max_fee_msat,
   - signed authorization.
5. The LLP routes the payment over its Lightning channels. The external recipient receives real BTC and releases the preimage.
6. LLP sends a signed success callback with:
   - payment_hash,
   - preimage,
   - amount_msat,
   - fee_msat.
7. Gateway marks the payment `SUCCESS`, debits the customer's balance, and executes a Nano send from the customer's Nano account to a reserve account.
8. Customer sees the same success response as before.

**Idempotency rule:** Only one LLP may ever be asked to pay a given external invoice. If an LLP returns a definitive failure, the payment is marked `FAILED` and the hold is released. If the LLP times out without a definitive answer, the payment is marked `UNRESOLVED` and the hold remains until reconciliation confirms whether the payment succeeded. This prevents accidental double payment of an external invoice.

### 8.6 LLP Settlement and Collateral

- The gateway maintains a small prepaid transit balance at each LLP.
- Inbound settlements increase the LLP receivable.
- Outbound settlements decrease the prepaid balance.
- When prepaid balance drops below 0.5 BTC, the gateway sends Bitcoin from the custody pool to the LLP.
- When prepaid balance exceeds 5 BTC, the LLP returns the excess to the custody pool.
- When receivable balance exceeds the LLP collateral threshold, inbound settlement is paused until the LLP settles.
- Each LLP posts collateral of at least 2× its maximum receivable cap, so default loss is bounded.

### 8.7 External LN Failure Semantics

If the public Lightning Network is degraded by jamming or routing attacks, external inbound/outbound payments may become slow or unavailable. This is an availability event for external transit, not a safety event for customer balances.

- Internal payments continue over Nano.
- On-chain Bitcoin deposits and withdrawals continue.
- No customer funds are ever locked in an LLP channel.
- If all external LN transit is down, BOLT11 invoices that include a Bitcoin fallback address (`f` field) can optionally be paid on-chain from the custody pool, if the customer approves the higher confirmation delay.

---

## 9. Nano Integration Design

### 9.1 Account Topology

- **User accounts**: each customer gets a unique Nano account, derived from a federation master seed using BIP32-Ed25519, path `m/44'/195'/0'/0/index`. Private keys are held by the system.
- **Reserve accounts**: a set of 3 federation-controlled hot reserve accounts plus cold reserve accounts hold the Nano used for issuance and redemption.
- **Derivation scheme**: BIP32-Ed25519 with master seed; index maps to customer ID.

### 9.2 Block Flow

- **Deposit**: Bitcoin deposit detected and confirmed → system creates a send block from a reserve account to the customer's Nano account with amount = BTC_deposit × exchange_rate → customer's Nano account receives → customer's Nano balance increases.
- **Internal payment**: Customer A pays Customer B → system creates a send block from A's Nano account to B's Nano account → B receives → both balances updated.
- **External inbound**: LLP confirms inbound Lightning payment → reserve account sends Nano to customer's Nano account, increasing the customer's claim.
- **External outbound**: LLP confirms outbound Lightning payment → customer's Nano account sends Nano to a reserve account, reducing the customer's claim.
- **Withdrawal**: User requests withdrawal → user's Nano account sends Nano to reserve → after Nano Instant confirmation, custody pool broadcasts Bitcoin withdrawal.

### 9.3 PoW Policy

- All PoW for send/receive blocks is computed by a dedicated GPU PoW service.
- Target: <5 seconds per block.
- Difficulty: network minimum (currently 8 leading zero bits), adjusted dynamically.
- If PoW exceeds 5 seconds, the block is queued with higher priority.

### 9.4 Representative Policy

- Every account is assigned a representative from a curated list of 10+ trusted, independently operated representatives, each with <10% voting weight.
- The list is reviewed monthly.
- The federation does not run its own Nano node and does not vote.

### 9.5 Confirmation Threshold

- A Nano block is final when cemented and at least 2 seconds have passed since cementing was observed from a majority of public RPCs.
- Ledger-affecting operations require 10 cemented Nano Instant confirmations and cross-checking with at least 3 public nodes.

### 9.6 Public Infrastructure Usage

- At least 3 public Nano RPC nodes: rpc.nano.org, nanocrawler.cc, mynano.ninja.
- Majority agreement required.
- WebSocket subscriptions for live confirmations.
- No custom Nano node is run.

### 9.7 Exchange Rate Mechanism

- The BTC/XNO rate is updated every 10 minutes from an average of major exchange prices (e.g., CoinGecko, Binance).
- Nano amounts are derived from BTC amounts using the current rate.
- Customer-facing balances remain BTC-denominated; Nano is the internal settlement rail.

---

## 10. Security Model

### 10.1 Threat Model

#### BTC-side Attacker

**Capabilities:** Can broadcast Bitcoin transactions, attempt RBF/pinning, flood the mempool, or delay confirmations. Can attempt to double-spend Bitcoin deposits or withdrawals.

**Limitations:** Cannot affect Nano consensus. There are no HTLCs, no channel states, and no timelocks in the NBLCS backend. Bitcoin transactions are only:
- deposits to the custody pool,
- withdrawals from the custody pool,
- net settlement with LLPs.

#### Nano Consensus Attacker

**Capabilities:** Can attempt to double-spend Nano, control representatives, or perform a 51% attack.

**Limitations:** Must overcome Nano's ORV quorum and the confirmation depth enforced by the system. Cannot forge Nano signatures.

#### Custody Pool Attacker

**Capabilities:** Can attempt to steal Bitcoin from the multi-sig pool by compromising federation signers or manipulating the reconciliation ledger.

**Limitations:** Cold funds require 3-of-5 signers; hot wallet is 2-of-3 with transaction limits. Ledger invariants force detection of imbalance.

#### LLP Attacker / Compromised LLP

**Capabilities:** Can fail to deliver an outbound Lightning payment, steal an inbound Lightning payment, or report false callbacks.

**Limitations:** LLPs do not control customer Nano accounts or Bitcoin custody keys. Each LLP is collateralized, capped, and independently audited. A compromised LLP cannot steal more than its cap; losses are covered by collateral and insurance.

#### Infrastructure Spoofing Attacker

**Capabilities:** Can impersonate public Nano RPC nodes, LLP endpoints, or intercept communication.

**Limitations:** All endpoints use TLS and signed API requests. Nano state is cross-checked across multiple nodes. LLP callbacks are signed and verified.

### 10.2 Defeat Controls

1. No HTLCs in the customer backend: Bitcoin transactions are only deposits and withdrawals. Nothing is pin-capable.
2. Independent per-payment Nano sends: each internal payment is a single Nano transaction directly from payer to payee. No shared liquidity pool, no HTLC slots.
3. No force-close path: there is no channel state to close.
4. Nano representative diversification: 10+ representatives, each <10% vote weight.
5. Confirmation depth: 10 Nano Instant confirmations + cross-check with 3 public nodes.
6. Cold multi-sig custody with spend limits.
7. Proof-of-reserve: weekly Merkle tree of Bitcoin pool UTXOs and LLP receivables published and verified against total claims.
8. Authenticated RPC: TLS, API keys, signed LLP callbacks, multi-node Nano queries, blacklisting of inconsistent nodes.

### 10.3 Structural Attack Analysis for External LN

- **Pinning:** Customer settlement never depends on a Bitcoin transaction being mined. Internal customer payments are Nano blocks. External inbound credits occur on LLP callback, not on Bitcoin mempool inclusion.
- **Jamming:** There is no shared customer liquidity pool to jam. Each customer's claim is directly spendable by that customer. LLP channel jamming may delay an external invoice, but cannot freeze customer balances.
- **Force-close races:** Customers have no Lightning channels. There is no unilateral exit, no commitment transaction, no penalty path, and no challenge window.

---

## 11. Full Stack Specification

```
NANO-BASED BITCOIN LIGHTNING COMPATIBILITY STACK (NBLCS)
========================================================
```

### 11.1 Component Specifications

#### 11.1.1 Bitcoin Gateway API

**Role:** Customer-facing Bitcoin-compatible interface.

**Interfaces:**

- `createinvoice(payment_hash, amount_msat, description, expiry_seconds=3600) -> BOLT11 string`
- `payinvoice(bolt11_invoice, amount_msat_optional) -> {preimage, payment_hash, status}`
- `getnewaddress() -> bc1... address`
- `sendtoaddress(address, amount_btc) -> {txid, status}`
- `getbalance() -> {total_btc, confirmed_btc, unconfirmed_btc}`
- `listtransactions(limit=100, offset=0) -> [{txid, category, amount_btc, confirmations, timestamp, description}]`

**Implementation:**

- BOLT11 generation delegated to LLPs for external inbound; internal invoices use the same payment_hash/preimage record.
- Invoice storage in PostgreSQL.
- Address generation via BIP32 `m/44'/1'/0'/0/index`.
- Balance display from the ledger.

#### 11.1.2 Nano Settlement Service (NSS)

**Role:** All Nano network interaction.

**Interfaces:**

- Public Nano RPC nodes (minimum 3, majority agreement): rpc.nano.org, nanocrawler.cc, mynano.ninja.
- WebSocket confirmations.
- Local GPU PoW service.

**Operations:**

- `create_account(index)`
- `send(from_account, to_account, amount_raw, work)`
- `receive(account, block_hash)`
- `get_confirmation(block_hash)`
- `get_pending(account)`

**Confirmation policy:**

- 10 cemented Nano Instant confirmations + 2-second buffer + majority agreement across 3 public nodes.

#### 11.1.3 Bitcoin Custody Pool

**Role:** Holds all Bitcoin reserves.

**Structure:**

- Cold storage: 3-of-5 multi-sig, keys offline.
- Hot wallet: 2-of-3 multi-sig, keys online but restricted.
- LLP settlement hot wallet: 2-of-3 multi-sig, capped at 5 BTC per LLP.

**Operations:**

- `deposit_monitor()`
- `broadcast_withdrawal(address, amount_sat)`
- `rebalance()`
- `settle_llp(llp_id, amount_sat)`

#### 11.1.4 Reconciliation Ledger

**Role:** Authoritative mapping between Nano balances and BTC claims.

**Schema:**

- `users`
- `invoices`
- `transactions`
- `deposits`
- `withdrawals`
- `exchange_rates`
- `outbound_payments`
- `inbound_settlements`
- `llps`
- `audit_log`

**Invariants:**

1. Sum of customer BTC claims equals total user Nano balances converted at the current exchange rate.
2. Sum of customer BTC claims ≤ (Bitcoin custody pool + LLP collateral + LLP net receivables) × 1.1.
3. Nano is never created or destroyed except through reserve-account issuance/redemption events matched to BTC claim changes.
4. Every invoice is single-use.
5. Every external outbound invoice is attempted by at most one LLP.

#### 11.1.5 Lightning Interop Service (LIS)

**Role:** External Bitcoin Lightning interoperability through LLPs.

**Interfaces:**

- LLP API client (REST/HTTPS, signed).
- Callback receiver.
- LLP registry and health monitor.
- Idempotency enforcement.

#### 11.1.6 Reserve Manager

**Role:** Maintains exchange rate and reserve health.

**Interfaces:**

- Rate source: CoinGecko + Binance average every 10 minutes.
- Rebalance scheduler: daily.
- LLP settlement scheduler: every 6 hours or on threshold.

#### 11.1.7 Monitoring & Alerting

**Metrics:**

- Nano Instant confirmation rate
- Bitcoin confirmation rate
- Hot wallet balance
- Reserve ratio
- Ledger invariant violations
- PoW queue depth
- RPC node availability
- LLP health
- LLP collateral ratio
- External LN success rate

**Alerts:**

- Reserve ratio < 110% → pause withdrawals, alert ops.
- Hot wallet < 2% of reserve → trigger rebalance.
- LLP collateral below threshold → pause that LLP.
- LLP callback missing → alert and reconcile.
- Nano Instant confirmation > 5s → alert PoW service.
- RPC node inconsistency → blacklist node, alert.
- Ledger invariant violation → freeze operations, alert.

### 11.2 End-to-End Flows

#### 11.2.1 Deposit

1. Customer calls `getnewaddress`.
2. Customer sends Bitcoin.
3. Bitcoin node detects UTXO.
4. After 6 confirmations, gateway marks deposit confirmed.
5. Ledger computes `nano_amount = btc_amount × rate`.
6. NSS sends Nano from reserve to customer's Nano account.
7. NSS creates receive block on customer's account.
8. After Nano Instant confirmations, ledger credits customer's BTC balance.
9. Customer sees updated `getbalance`.

#### 11.2.2 Internal Payment

1. Customer B calls `createinvoice` → gateway stores invoice and, if externally payable, asks an LLP to create a BOLT11 invoice.
2. Customer A calls `payinvoice`.
3. Gateway validates invoice.
4. Ledger debits A and credits B.
5. NSS sends Nano from A to B.
6. NSS creates receive block on B.
7. Gateway reveals preimage to A.
8. Both customers see updated balances.

#### 11.2.3 External Inbound Payment

1. Customer B calls `createinvoice`.
2. LIS selects an LLP and creates a real BOLT11 invoice with the LLP's node.
3. External payer pays the invoice from any Lightning wallet.
4. LLP settles and sends a signed callback with preimage.
5. Gateway verifies the preimage and amount.
6. Ledger credits Customer B's BTC claim.
7. NSS sends Nano from reserve to Customer B's Nano account.
8. Customer B sees the payment as final.

#### 11.2.4 External Outbound Payment

1. Customer A calls `payinvoice` with an external BOLT11 invoice.
2. Gateway validates the invoice and holds funds.
3. LIS selects one LLP.
4. LLP pays the external invoice over public Lightning.
5. External recipient receives BTC.
6. LLP sends success callback with preimage and fee.
7. Ledger debits Customer A's balance.
8. NSS sends Nano from Customer A to reserve.
9. Customer A sees the payment as successful.

#### 11.2.5 Withdrawal

1. Customer calls `sendtoaddress` with a Bitcoin address and amount.
2. Gateway checks balance and debits the customer's Nano balance.
3. NSS sends Nano from customer to reserve.
4. After 10 Nano Instant confirmations, ledger marks withdrawal pending.
5. If amount ≤ 5 BTC, hot wallet signs and broadcasts.
6. If amount > 5 BTC, cold wallet signs with manual approval.
7. Ledger marks withdrawal complete.

#### 11.2.6 LLP Net Settlement

1. Every 6 hours, LIS computes each LLP's net position:
   - `net_position = prepaid_balance - receivable_balance`
2. If net position is negative and above collateral threshold, pause inbound volume.
3. If net position is positive above 5 BTC, request LLP return excess to custody pool.
4. If net position is below 0.5 BTC, send Bitcoin from custody pool to LLP.

### 11.3 Configuration Values

| Parameter | Value |
|---|---|
| Nano Instant confirmation threshold | 10 confirmations + 2s buffer |
| Bitcoin deposit confirmations | 6 |
| Bitcoin withdrawal confirmations | 1 (broadcast) |
| Exchange rate update | every 10 minutes |
| Reserve ratio | 110% |
| Hot wallet max | 10% of reserve |
| Hot wallet min | 2% of reserve |
| Hot wallet per-tx limit | 5 BTC |
| Cold multi-sig | 3-of-5 |
| Hot multi-sig | 2-of-3 |
| PoW max time | 5 seconds |
| PoW difficulty | network minimum (8 bits) |
| Representative count | 10+ |
| Nano RPC nodes | 3 minimum |
| Proof-of-reserve | weekly |
| Rate source | CoinGecko + Binance average |
| LLP minimum count | 3 |
| LLP max volume share | 33% per LLP |
| LLP prepaid min | 0.5 BTC |
| LLP prepaid max | 5 BTC |
| LLP collateral | ≥ 2× receivable cap |
| External outbound timeout | 60 seconds |
| External outbound hold | until SUCCESS, FAILED, or reconciled UNRESOLVED |
| LLP settlement interval | every 6 hours |

### 11.4 Operational Runbook

#### Daily

- Check reserve ratio ≥ 110%.
- Verify hot wallet balance within 2–10% of reserve.
- Check Nano Instant confirmation rates.
- Check LLP health and collateral ratios.
- Review PoW queue depth.

#### Weekly

- Publish proof-of-reserve Merkle tree, including custody pool UTXOs and LLP collateral/receivables.
- Audit ledger invariants.
- Review representative list.
- Rotate API keys.

#### Monthly

- Full audit of cold storage keys.
- Verify rate peg accuracy.
- Update LLP registry.
- Review security controls.
- Third-party LLP audit.

#### Incident Response

- RPC node failure: switch to backup node, blacklist failed node.
- Nano network issue: wait for confirmations, do not credit.
- Reserve ratio < 110%: pause withdrawals, alert, trigger rebalance.
- Ledger invariant violation: freeze operations, audit log, manual review.
- Bitcoin node down: use external API for monitoring.
- LLP failure: remove LLP from selection, use remaining LLPs.
- LLP timeout with unknown outbound result: hold funds, contact LLP, reconcile ledger before release.
- LLP collateral breach: pause that LLP, require settlement or fresh collateral.
- Hot wallet compromised: move funds to cold, rotate hot keys.

### 11.5 Economics

- Nano PoW: $0.0001 per transaction.
- Bitcoin reserve opportunity cost: $0.0137 per transaction at 5% on $10M reserve.
- Infrastructure: $120,000/year.
- Staff: $400,000/year.
- Custody: $100,000/year.
- Total: $1,120,000/year.
- Internal transactions: 36.5M/year.
- Cost per internal transaction: $0.0308.
- External LN routing fees are passed through to the customer as part of the external payment; they are not applied to internal Nano payments.

---

## 12. Implementation Roadmap

1. **Milestone 1 — Custody pool and multi-sig.** Bitcoin Core synced; 3-of-5 cold and 2-of-3 hot multi-sig created on testnet; PSBT signing flow tested; deposit monitor works.
2. **Milestone 2 — Nano rail with confirmation policy.** NSS derives accounts, sends/receives on live Nano network, PoW service works, 3 public RPC nodes agree on cementing, 10-confirmation policy enforced.
3. **Milestone 3 — Interface shim.** BOLT11 invoices, `getnewaddress`, `payinvoice`, `sendtoaddress`, `getbalance`, and `listtransactions` return Bitcoin-compatible JSON-RPC responses.
4. **Milestone 4 — Reconciliation ledger.** PostgreSQL schema, atomic debit/credit, invariants, exchange rate updates.
5. **Milestone 5 — Lightning Interop Service.** Integrate at least 3 testnet LLPs; external inbound and outbound BOLT11 invoices work; idempotency enforced; LLP callbacks verified; no gateway LN channels.
6. **Milestone 6 — Monitoring and alerting.** Dashboards and alerts for all components, including LLP health.
7. **Milestone 7 — Security hardening.** Cold keys in hardware; LLP collateral verified; proof-of-reserve published weekly; incident drills pass.
8. **Milestone 8 — Pilot with staged traffic.** Internal payments; on-chain deposits/withdrawals; external inbound payments from real Lightning wallets; external outbound payments to real Lightning invoices; reserve ratio stays ≥ 110%; invariants hold.

---

## 13. Verification Plan

1. **Interface Conformance Suite**: 100 randomly generated BOLT11 invoices parse correctly; payment_hash matches `SHA256(preimage)`; addresses are valid bc1; `payinvoice` error codes match LND; `sendtoaddress` txids are 64-hex; `listtransactions` structure matches Bitcoin Core.
2. **External Inbound Test**: Generate a gateway invoice from each LLP; pay it from an external Lightning wallet; assert customer is credited exactly once, with correct preimage verification, and no double credit on duplicate attempt.
3. **External Outbound Test**: Create invoices on an external LND node; pay them through the gateway; assert external recipient receives BTC, preimage is returned, customer balance is debited once, and no second LLP is ever asked to pay the same invoice.
4. **Synthetic Attack Drills**:
   - Pinning drill: Bitcoin deposit pinned; assert no credit before 6 confirmations of canonical chain.
   - Jamming drill: 10,000 fake payment requests; assert valid payments settle within PoW + confirmation targets.
   - Force-close race drill: simultaneous withdrawal and internal payment; assert no double-spend and clean rollback.
   - LLP failure drill: take one LLP offline; assert invoice creation fails over to another LLP; outbound failures roll back cleanly.
5. **Nano Settlement Latency Measurement**: Median total latency < 10 seconds; 95th percentile < 30 seconds; 100% of blocks achieve 10 confirmations within 120 seconds; zero reversals after 10 confirmations.
6. **Proof-of-Reserve Audit**: Weekly Merkle tree of custody pool and LLP collateral/receivables; reserve ratio ≥ 110%; Nano supply invariant holds.
7. **Failover Drills**: Nano RPC failure, Bitcoin node failure, PoW service failure, LLP API failure. All recover within RTO; zero lost payments; zero incorrect ledger credits.

---

## 14. Scope Statement

NBLCS is a hybrid wallet/payment-processor architecture. It makes the customer-facing Lightning backend immune to pinning, jamming, and force-close races by replacing that backend with Nano settlement and federation custody. It does not claim to make the public Lightning Network itself trustless.

External Lightning interoperability is achieved through independent LLPs as transit providers. Customer funds never reside in LLP channels. If the public Lightning Network is globally unavailable, external BOLT11 payments cannot be delivered over Lightning by any system, but NBLCS customers can still:

- send and receive internal payments instantly,
- deposit and withdraw Bitcoin on-chain,
- pay BOLT11 invoices that include a Bitcoin fallback address via on-chain settlement,
- retain full control of their balances with no exposure to channel attacks.

This satisfies the unchanged-Bitcoin-interface constraint for real-world Lightning users while keeping the Nano backend, not a Lightning channel backend, as the system of record.

---

**End of NBLCS Specification.**

---
