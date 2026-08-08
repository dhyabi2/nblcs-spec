# NBLCS — Nano-Based Bitcoin Lightning Compatibility Stack

A complete architecture specification for replacing the Bitcoin Lightning Network backend with a **Nano (XNO) settlement rail**, while keeping the customer-facing Bitcoin interface — BOLT11 invoices, `payinvoice`, `sendtoaddress`, `getbalance`, on-chain deposits — byte-for-byte unchanged.

## What's inside

The full spec lives in **[NBLCS-SPEC.md](NBLCS-SPEC.md)** and covers:

- **LN backend inventory** — every Lightning component replaced (channels, commitments, HTLCs, watchtowers, force-close arbitration) and every customer-facing surface preserved
- **Architecture** — Bitcoin Gateway API, Nano Settlement Service, federated multi-sig Bitcoin Custody Pool, Reconciliation Ledger, Lightning Interop Service (LLP broker), Reserve Manager, Monitoring
- **External Lightning interoperability** — paying and receiving arbitrary BOLT11 invoices through collateralized Lightning Liquidity Providers, without the gateway ever opening a channel
- **Security model** — threat model, defeat controls, and structural analysis showing why pinning, jamming, and force-close races have no attack surface in the customer backend
- **Full specification** — component interfaces, data schemas, configuration values, end-to-end flows, operational runbook, economics
- **Implementation roadmap and verification plan** — 8 milestones plus interface-conformance suites and synthetic attack drills

## Key design constraints

1. Customers modify **nothing** — accepting and sending Bitcoin work exactly as before
2. No custom Nano node — only the live public Nano network (multi-RPC majority agreement)
3. Structural immunity, not mitigation: no mempool dependence, no timelocks, no shared liquidity pools, no force-close path

## Status

Specification / design document. Not an implementation, not an operating service. Anyone operating a system like this custodies customer funds and must hold the applicable money-transmission / VASP licenses in their jurisdiction.
