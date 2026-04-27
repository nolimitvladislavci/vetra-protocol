# VETRA Protocol — Open Document Integrity Standard

> An open, offline-verifiable, Bitcoin-anchored document integrity protocol.

**License:** CC0 1.0 Universal (Public Domain)
**Author:** Ivan Brtan, No Limit Vladislavci udruga
**Status:** Production-ready (vetra.live)

## What is VETRA?

VETRA is a tamper-evident proof layer for digital documents. It does NOT replace
qualified electronic signatures — it complements them with an additional,
mathematically verifiable integrity layer.

## Core Protocol

- **Triple hash:** SHA-256 + SHA3-256 + BLAKE3
- **Dual signature:** Ed25519 (classical) + ML-DSA-65 (NIST FIPS 204, post-quantum)
- **Time anchor:** Bitcoin OpenTimestamps (3 public calendars)
- **Bundle format:** `.dok` — self-contained ZIP with offline viewer.html

## Documents

| Document | Description |
|---|---|
| [VETRA-SEAL-SPEC-1.1.md](VETRA-SEAL-SPEC-1.1.md) | Full protocol specification |
| [VETRA_KRUNICA_WHITEPAPER.md](VETRA_KRUNICA_WHITEPAPER.md) | Philosophical foundation — 4 elements, Krunica chain |
| [RUNBOOK.md](RUNBOOK.md) | Operational runbook, SLA, incident procedures |

## eIDAS Positioning

VETRA is positioned as a **complementary integrity layer** to eIDAS 2.0 qualified
signatures. It provides tamper-evident proof of existence without claiming QTSP status.

## Defensive Publication

This specification is published as prior art under CC0. KONJ-DP-2026-001.
Any patent claim on concepts described herein is blocked by this publication date.

## Live Implementation

- **Platform:** vetra.live
- **Verification:** Any .dok file is self-verifiable offline (open viewer.html)
- **Chain:** KrunicaChain — 59-position document chain with Genesis + Terminal Seal

---
*"Ne treba ti server. Ne treba ti dozvola. Treba ti samo .dok."*
