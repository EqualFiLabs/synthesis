# Agentic Mailbox SDK & Cryptography Spec

**Status:** Draft v0.2  
**Date:** 2026-03-10  
**Applies to:**
- `compute-orchestration-spec.md` (credential handoff)
- `venice-adapter-spec.md` (provider payload encryption)

---

## 1) Purpose

Enable secure on-chain credential handoff between borrowing agents and the Equalis Orchestration Node without requiring custom cryptography in app code.

This spec is aligned to the currently implemented SDK:
- package: `@equalfi/mailbox-sdk`
- implementation: TypeScript
- repository: `EqualFiLabs/mailbox-sdk`

---

## 2) Cryptographic Profile (SDK-Aligned)

Mailbox encryption uses ECIES-style primitives from `eth-crypto` on `secp256k1`.

### 2.1 Key formats

- **Curve:** `secp256k1`
- **Private key:** `0x` + 64 hex chars
- **Public key (uncompressed, SDK return):** `0x` + 128 hex chars (no `04` prefix)
- **Public key (compressed, registry storage):** `0x02/0x03` + 64 hex chars

### 2.2 Supported receiver public key inputs (SDK)

`encryptPayload(receiverPubKeyHex, payload)` accepts:

1. compressed key (`0x02..` / `0x03..`, 66 hex chars after `0x`)
2. uncompressed no-prefix key (`0x` + 128 hex chars)
3. uncompressed with `04` prefix (`0x04...`, 130 hex chars after `0x`)

Any other format MUST revert with invalid public key error.

---

## 3) Envelope Format (Actual SDK Output)

`encryptPayload(...)` returns the output of `EthCrypto.cipher.stringify(...)`.

Canonical envelope object shape:

```json
{
  "iv": "<hex>",
  "ephemPublicKey": "<hex>",
  "ciphertext": "<hex>",
  "mac": "<hex>"
}
```

### 3.1 On-chain transport

The mailbox facet stores `bytes envelope`.

Transport convention:
- producer encrypts to **stringified envelope JSON**
- producer ABI-encodes that UTF-8 string as `bytes` for `publishBorrowerPayload` / `publishProviderPayload`
- consumer decodes `bytes -> string` then calls `decryptPayload(privateKey, envelopeString)`

This keeps storage generic while preserving SDK compatibility.

---

## 4) `@equalfi/mailbox-sdk` Interface (Current)

### 4.1 `generateKeys()`

```typescript
function generateKeys(): {
  privateKey: string;
  publicKey: string;
  compressedPublicKey: string;
}
```

### 4.2 `encryptPayload(receiverPubKeyHex, payload)`

```typescript
async function encryptPayload(
  receiverPubKeyHex: string,
  payload: string | object
): Promise<string>
```

Returns: **stringified envelope** (JSON string), not raw bytes.

### 4.3 `decryptPayload(privateKeyHex, encryptedPayloadString)`

```typescript
async function decryptPayload(
  privateKeyHex: string,
  encryptedPayloadString: string
): Promise<string>
```

Returns plaintext string. If original payload was JSON object, caller parses JSON after decryption.

---

## 5) Integration with Orchestration Node

1. Node keeps a mailbox private key and registers compressed pubkey in `AgentEncPubRegistryFacet`.
2. On `BorrowerPayloadPublished(agreementId, envelopeBytes)`:
   - decode bytes to string envelope
   - call `decryptPayload(nodePrivateKey, envelopeString)`
3. After provider provisioning:
   - call `encryptPayload(borrowerRegisteredPubKey, providerConnectionDetails)`
   - encode returned string as bytes
   - submit via `publishProviderPayload(agreementId, envelopeBytes)`

---

## 6) Security Requirements

- **Key separation:** mailbox keys should be independent from treasury signing keys.
- **No plaintext secret logs:** decrypted payloads and provider credentials must never be logged in plaintext.
- **Ephemeral encryption context:** each encryption operation produces a fresh ephemeral pubkey path via underlying library behavior.
- **Tamper detection:** envelope MAC must be validated by successful decrypt; malformed/tampered payloads must fail closed.

---

## 7) Compatibility Notes

- This spec intentionally matches the current TS SDK semantics and test suite.
- A future Python SDK MUST preserve envelope compatibility (`iv/ephemPublicKey/ciphertext/mac`) to ensure cross-language interoperability.

---

**End of Draft v0.2**
