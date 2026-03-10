# Agentic Mailbox SDK & Cryptography Spec

**Status:** Draft v0.1  
**Date:** 2026-03-10  
**Applies to:** `compute-orchestration-spec.md` (Credential Handoff Phase)

---

## 1) Purpose

To enable secure, on-chain credential handoff between borrowing agents and the Equalis Orchestration Node without requiring developers to write custom cryptographic routines. 

This spec defines the exact encryption standard (ECIES) and the minimal SDK (`@equalfi/mailbox-sdk`) required to generate keys, format payloads, and handle on-chain decryption.

---

## 2) Cryptographic Standard (ECIES)

To maintain maximum compatibility with existing Ethereum tooling, the mailbox will use the **Elliptic Curve Integrated Encryption Scheme (ECIES)** as implemented by `eth-crypto` and `@metamask/eth-sig-util`.

### 2.1 Keys
- **Curve:** `secp256k1`
- **Private Key:** 32-byte hex string.
- **Public Key (Registered):** 33-byte compressed hex string (stored in `AgentEncPubRegistryFacet`).

### 2.2 Encryption Flow (ECDH + AES)
When Sender (A) wants to send a message to Receiver (B):
1. **A** generates an ephemeral `secp256k1` keypair.
2. **A** performs ECDH (Elliptic Curve Diffie-Hellman) using their ephemeral private key and **B**'s registered public key to derive a shared secret.
3. The shared secret is hashed to generate an AES encryption key and a MAC key.
4. The plaintext payload is encrypted using AES-256-CBC (or AES-128-CTR depending on the strict MetaMask standard profile).
5. A MAC (Message Authentication Code) is computed over the ciphertext and IV.

### 2.3 The Envelope Structure
The resulting encrypted payload must be formatted as a structured JSON string, which is then ABI-encoded as `bytes` before being pushed to the smart contract.

```json
{
  "version": "x25519-xsalsa20-poly1305", 
  "nonce": "<base64_IV>",
  "ephemPublicKey": "<base64_ephemeral_pubkey>",
  "ciphertext": "<base64_ciphertext>"
}
```
*Note: The version string historically maps to standard MetaMask `eth_decrypt` payloads, ensuring cross-compatibility if a human wallet needs to read the mailbox.*

---

## 3) The `@equalfi/mailbox-sdk` Interface

The SDK will be published in TypeScript and Python to serve both the Orchestration Node and the borrowing agents. It wraps the raw ECIES logic into three core operations.

### 3.1 `generateKeys()`
Generates a new `secp256k1` keypair specifically for mailbox communications. 

```typescript
function generateKeys(): {
    privateKey: string;
    publicKey: string;           // Uncompressed (for local ECDH)
    compressedPublicKey: string; // 33-bytes (for registry)
}
```
*Best Practice: Agents should generate ephemeral keypairs per-agreement rather than reusing their main EVM transaction signing key for encryption.*

### 3.2 `encryptPayload(receiverPubKey, payload)`
Takes the receiver's compressed public key (fetched from the registry) and a JSON/String payload, returning the ABI-encodable `bytes` hex string.

```typescript
async function encryptPayload(
    receiverPubKeyHex: string, 
    payload: string | object
): Promise<string> // Returns hex string starting with 0x
```
**Example Payload (Borrower -> Node):**
```json
{
  "sshPublicKey": "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC...",
  "regionPreference": "us-east-1"
}
```

### 3.3 `decryptPayload(privateKey, encryptedHex)`
Takes the agent's private key and the `bytes` hex string pulled from the `BorrowerPayloadPublished` or `ProviderPayloadPublished` events.

```typescript
async function decryptPayload(
    privateKeyHex: string, 
    encryptedHex: string
): Promise<string> // Returns the decoded plaintext
```
**Example Payload (Node -> Borrower):**
```json
{
  "instanceIp": "192.168.1.100",
  "sshUser": "ubuntu",
  "providerId": "i-0abcd1234efgh5678"
}
```

---

## 4) Integration with the Orchestration Node

The Orchestration Node will utilize this SDK heavily:

1. **Bootstrapping:** The Node possesses its own master Mailbox Private Key. Its corresponding compressed public key is registered in `AgentEncPubRegistryFacet` under the Node's wallet address.
2. **Ingestion:** When `BorrowerPayloadPublished` is detected, the Node calls `decryptPayload(nodePrivateKey, event.envelope)`.
3. **Delivery:** After provisioning Lambda, the Node calls `encryptPayload(borrowerRegisteredPubKey, connectionDetails)` and pushes it via `publishProviderPayload`.

---

## 5) Security Considerations

- **Forward Secrecy:** Because the sender uses a new ephemeral key for every encryption operation, the compromise of a single message does not compromise past messages.
- **Key Separation:** By maintaining a separate `AgentEncPubRegistryFacet`, we ensure that agents are not forced to export the private key that controls their actual treasury funds just to read a message.
- **Malleability:** The MAC included in the ECIES envelope ensures that the ciphertext cannot be tampered with while at rest on the blockchain.

---

**End of Draft v0.1**
