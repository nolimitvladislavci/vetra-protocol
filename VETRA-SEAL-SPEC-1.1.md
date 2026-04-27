# VETRA Seal Specification v1.1

**Protocol:** VETRA™ Document Seal Protocol  
**Version:** 1.1  
**Status:** LIVE (production)  
**Author:** Ivan Brtan, Konjik d.o.o.  
**License:** CC0 1.0 Universal (this specification is public domain)  
**Repository:** github.com/vetra-protocol/spec  

---

## Abstract

VETRA Seal is a cryptographic document integrity protocol that produces a tamper-evident, offline-verifiable, Bitcoin-anchored proof for any digital document. Each sealed document receives:

1. A unique **seal ID** (SHA3-256 truncated to 16 hex chars)
2. **Three independent hash commitments** (SHA-256, SHA3-256, BLAKE3)
3. **Classical signature** (Ed25519)
4. **Post-quantum signature** (ML-DSA-65 / Dilithium3, NIST FIPS 204)
5. **Bundle integrity signature** (Ed25519 over canonical 6-field message)
6. **Bitcoin timestamp** (OpenTimestamps via OP_RETURN)
7. A self-contained **offline verifier** (verify.html) inside the .dok bundle

---

## 1. Definitions

| Term | Definition |
|------|-----------|
| **seal** | 16-char hex string derived from SHA3-256 of the document |
| **.dok bundle** | ZIP archive containing original file + manifest + verifier |
| **canonical message** | Deterministic string used for bundle signature |
| **VETRA_CANONICAL_PUBKEY** | Hardcoded Ed25519 public key in every verify.html |
| **OTS** | OpenTimestamps — Bitcoin blockchain timestamp protocol |
| **ML-DSA-65** | Post-quantum signature scheme (NIST FIPS 204, formerly Dilithium3) |

---

## 2. Seal Computation

### 2.1 File Hashing

```python
sha256   = hashlib.sha256(file_bytes).hexdigest()    # 64 hex chars
sha3_256 = hashlib.sha3_256(file_bytes).hexdigest()  # 64 hex chars
blake3   = blake3.blake3(file_bytes).hexdigest()     # 64 hex chars
```

### 2.2 Seal ID

```python
seal = sha3_256[:16]  # First 16 hex chars of SHA3-256

# Collision check (birthday paradox: ~1 in 10^19 for 16 hex)
while Document.objects.filter(seal=seal).exists():
    seal = sha3_256[:17]  # Extend by 1 hex on collision
    # Continue: sha3_256[:18], [:20] as needed
```

---

## 3. Signatures

### 3.1 Ed25519 Document Signature

Signs the SHA-256 hex string (not the file bytes):

```python
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PrivateKey
import base64

private_key = Ed25519PrivateKey.from_private_bytes(base64.b64decode(VETRA_SIGNING_KEY_PRIVATE))
sig_bytes = private_key.sign(sha256_hex.encode("utf-8"))
vetra_sig = base64.b64encode(sig_bytes).decode("ascii")
vetra_pubkey = base64.b64encode(private_key.public_key().public_bytes_raw()).decode("ascii")
```

### 3.2 Bundle Integrity Signature (v1.1)

Signs a canonical message covering 6 critical document fields:

```python
canonical = f"VETRA_SEAL_V1.1|{sha256}|{blake3}|{seal}|{created_at}|{file_size}|{file_name}"
bundle_sig = private_key.sign(canonical.encode("utf-8"))
vetra_bundle_sig = base64.b64encode(bundle_sig).decode("ascii")
```

**Field order is fixed and must not be changed.** This signature prevents any field-level tampering — an attacker cannot change `file_name` or `file_size` without invalidating `vetra_bundle_sig`.

### 3.3 ML-DSA-65 Post-Quantum Signature

Each document receives an ephemeral ML-DSA-65 keypair:

```python
from dilithium_py.dilithium import Dilithium3

pqc_pk, pqc_sk = Dilithium3.keygen()
pqc_canonical = f"PQC_SEAL_V1.0|{sha256}|{blake3}|{seal}".encode("utf-8")
pqc_sig = Dilithium3.sign(pqc_sk, pqc_canonical)
vetra_pqc_sig    = base64.b64encode(pqc_sig).decode("ascii")
vetra_pqc_pubkey = base64.b64encode(pqc_pk).decode("ascii")
```

The public key is stored in the manifest. Verification is self-contained — no VETRA server needed.

---

## 4. Bitcoin Anchoring (OpenTimestamps)

```python
import opentimestamps
from opentimestamps.core.timestamp import DetachedTimestampFile

# Submit at seal time
ots_file = DetachedTimestampFile.from_fd('sha256', io.BytesIO(bytes.fromhex(sha256)))
calendar_urls = [
    "https://alice.btc.calendar.opentimestamps.org",
    "https://bob.btc.calendar.opentimestamps.org",
]
stamper = RemoteCalendar(calendar_urls[0])
stamper.stamp(ots_file.timestamp)
ots_proof = base64.b64encode(ots_file.serialize()).decode("ascii")
```

OTS proofs are upgraded to Bitcoin-confirmed status by a Celery Beat task running every 2 hours.

### 4.1 Daily Merkle Anchor

All seals from a calendar day are grouped into a binary Merkle tree. A single OTS proof anchors the entire day, reducing Bitcoin transactions from N to 1:

```python
def binary_merkle_root(leaves: list[bytes]) -> bytes:
    layer = [hashlib.sha256(leaf).digest() for leaf in leaves]
    while len(layer) > 1:
        if len(layer) % 2 == 1:
            layer.append(layer[-1])
        layer = [hashlib.sha256(layer[i] + layer[i+1]).digest() for i in range(0, len(layer), 2)]
    return layer[0]

daily_root = binary_merkle_root([bytes.fromhex(sha256) + seal.encode() for each doc])
```

---

## 5. manifest.json Schema

```json
{
  "vetra_version":    "1.1",
  "seal":             "a3f2b1c9d8e7f012",
  "sha256":           "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "sha3_256":         "a7ffc6f8bf1ed76651c14756a061d662f580ff4de43b49fa82d80a4b80f8434a",
  "blake3_hash":      "af1349b9f5f9a1a6a0404dea36dcc9499bcb25c9adc112b7f9e1c5e7f7e8c3f4",
  "file_name":        "contract.pdf",
  "file_type":        "application/pdf",
  "file_size":        204800,
  "created_at":       "2026-04-27T10:00:00Z",
  "vetra_sig":        "Base64EncodedEd25519Signature==",
  "vetra_pubkey":     "Base64EncodedEd25519PublicKey==",
  "vetra_bundle_sig": "Base64EncodedBundleIntegritySignature==",
  "vetra_pqc_sig":    "Base64EncodedMLDSA65Signature==",
  "vetra_pqc_pubkey": "Base64EncodedMLDSA65PublicKey==",
  "ots_proof":        "Base64EncodedOTSProof==",
  "ots_confirmed":    false,
  "verify_url":       "https://vetra.live/verify/a3f2b1c9d8e7f012/",
  "protocol":         "VETRA Seal Specification v1.1",
  "algorithm":        "SHA-256 + SHA3-256 + BLAKE3 + Ed25519 + ML-DSA-65 + OpenTimestamps"
}
```

---

## 6. .dok Bundle Structure

A `.dok` file is a standard ZIP archive with the following mandatory contents:

```
document.dok (ZIP)
├── original.<ext>    Required. Verbatim copy of the original file.
├── manifest.json     Required. All cryptographic fields (see Section 5).
├── verify.html       Required. Self-contained offline verifier.
└── seal.svg          Optional. VETRA hexagonal visual seal graphic.
```

### 6.1 verify.html Requirements

The `verify.html` file MUST:

1. Contain a hardcoded constant `VETRA_CANONICAL_PUBKEY` equal to the VETRA service public key
2. On load, extract the embedded `manifest.json` (stored as base64 in a `<script>` tag)
3. Call `verifyBundleIntegrity()` FIRST, before any other verification
4. Reject the document if `manifest.vetra_pubkey !== VETRA_CANONICAL_PUBKEY` (key substitution attack defense)
5. Verify `vetra_bundle_sig` over the canonical message using WebCrypto API
6. Display a red full-page warning (COMPROMISED state) if any check fails
7. Work fully offline — no external scripts, no CDN, no network requests

---

## 7. Verification Algorithm

### 7.1 Bundle Integrity (offline, mandatory)

```javascript
const VETRA_CANONICAL_PUBKEY = "OSSQ0C6Xx12LEY/KVyRcrSoTy4M6DO8StcQhuCt+6gs="; // hardcoded

async function verifyBundleIntegrity(manifest) {
  // Step 1: Key substitution check
  if (manifest.vetra_pubkey !== VETRA_CANONICAL_PUBKEY) {
    showCompromised("KEY SUBSTITUTION DETECTED", manifest.vetra_pubkey);
    return false;
  }
  
  // Step 2: Reconstruct canonical message
  const msg = `VETRA_SEAL_V1.1|${manifest.sha256}|${manifest.blake3_hash}|` +
              `${manifest.seal}|${manifest.created_at}|${manifest.file_size}|${manifest.file_name}`;
  
  // Step 3: Verify Ed25519 bundle signature (WebCrypto)
  const pubKeyBytes = base64ToBytes(manifest.vetra_pubkey);
  const sigBytes = base64ToBytes(manifest.vetra_bundle_sig);
  const msgBytes = new TextEncoder().encode(msg);
  
  const cryptoKey = await crypto.subtle.importKey("raw", pubKeyBytes, { name: "Ed25519" }, false, ["verify"]);
  const valid = await crypto.subtle.verify("Ed25519", cryptoKey, sigBytes, msgBytes);
  
  if (!valid) {
    showCompromised("BUNDLE SIGNATURE INVALID", "Document has been tampered with.");
    return false;
  }
  return true;
}
```

### 7.2 Document Hash Verification (offline)

```javascript
async function verifyDocumentHash(fileBytes, manifest) {
  const hashBuffer = await crypto.subtle.digest("SHA-256", fileBytes);
  const computed = Array.from(new Uint8Array(hashBuffer))
    .map(b => b.toString(16).padStart(2, "0")).join("");
  return computed === manifest.sha256;
}
```

### 7.3 OTS Verification (requires Bitcoin node or block explorer)

```bash
pip install opentimestamps-client
ots verify document.pdf.ots
# Output: Success! Bitcoin block 844210 attests data existed before 2026-04-27T10:05:00Z
```

---

## 8. Security Properties

| Property | Mechanism |
|----------|-----------|
| Document integrity | SHA-256 + SHA3-256 + BLAKE3 triple hash |
| Authenticity | Ed25519 over sha256 (vetra_sig) |
| Field tamper-proof | Ed25519 over canonical 6-field message (vetra_bundle_sig) |
| Key substitution resistance | VETRA_CANONICAL_PUBKEY hardcoded in verify.html |
| Post-quantum resistance | ML-DSA-65 (NIST FIPS 204) |
| Timestamp (decentralised) | Bitcoin via OpenTimestamps |
| Offline verifiability | verify.html (zero dependencies, WebCrypto only) |
| Server-independence | Level 3: OTS verifiable with ots-client against Bitcoin |
| Upload safety | ClamAV scan + 13-MIME whitelist + 20MB limit |
| Seal uniqueness | SHA3-256[:16] with collision extension |

---

## 9. Error Conditions

| Condition | Response | HTTP Code |
|-----------|----------|-----------|
| File type not in whitelist | 415 Unsupported Media Type | 415 |
| File exceeds 20 MB | 413 Request Entity Too Large | 413 |
| ClamAV threat detected | 422 Unprocessable Entity | 422 |
| Seal collision after [:20] | 500 Internal Server Error | 500 |
| OTS calendar unavailable | Stored without OTS; retried later | 200 |
| Key substitution (verify.html) | COMPROMISED screen shown | N/A |

---

## 10. Versioning

| Version | Changes |
|---------|---------|
| 1.0 | Initial: SHA-256 + SHA3-256 + Ed25519 + OTS |
| 1.1 | Added: BLAKE3, vetra_bundle_sig (canonical message), ML-DSA-65, DailyMerkleRoot, key substitution attack defense in verify.html |

---

## 11. Reference Implementation

- **Server:** Python 3.12 + Django 5.x  
- **Ed25519:** `cryptography` library (PyCA)  
- **ML-DSA-65:** `dilithium-py` 1.4.0  
- **BLAKE3:** `blake3` 1.0.4  
- **OTS:** `opentimestamps` 0.4.3  
- **Offline verifier:** Vanilla JavaScript, WebCrypto API, no external dependencies  

Source: https://github.com/vetra-protocol/spec (pending publication)

---

## 12. Legal

This specification is released under **CC0 1.0 Universal**. Anyone may implement a compatible verifier without restriction. The name VETRA™ and the hexagonal seal graphic are trademarks of Konjik d.o.o. (Croatia).

VETRA seals are designed to assist in establishing proof of existence and integrity. They are not legal advice. Consult qualified legal counsel for regulatory requirements in your jurisdiction.

---

*Konjik d.o.o. | Zagreb, Croatia | support@vetra.live*
