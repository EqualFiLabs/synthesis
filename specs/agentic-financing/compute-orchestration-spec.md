# Compute Orchestration Node Specification

**Status:** Draft v0.3  
**Date:** 2026-03-10  
**Applies to:** `specs/agentic-financing/agentic-financing-spec.md` (Canonical v1.10), `compute-provider-decision-spec.md`, and `venice-adapter-spec.md`

---

## 1) Purpose

The **Compute Orchestration Node** is the trusted off-chain relayer that bridges deterministic Equalis contracts to non-deterministic provider APIs (Lambda, RunPod, Venice).

It is responsible for:
1. Translating on-chain agreement activations into live compute access.
2. Securely exchanging credentials/config with the borrowing agent via on-chain encrypted mailbox flows.
3. Metering off-chain usage and posting it on-chain to accrue debt.
4. Enforcing on-chain risk transitions (delinquency/covenant breach/default) by terminating/revoking off-chain access.

This node is the sole trusted party holding provider admin/API credentials and the relayer wallet private keys.

---

## 2) Architecture Overview

The node runs as a persistent service with four core subsystems:

1. **Ingestion Engine:** Listens to RPC endpoints for canonical Equalis events.
2. **Provider Clients:** Provider-specific clients for Lambda, RunPod, Venice.
3. **Relayer/Submitter:** Signs and broadcasts `registerUsage()` and `publishProviderPayload()` transactions.
4. **State Database:** Local mapping of `agreementId <-> providerResourceId <-> billingState` for outage/replay safety.

---

## 3) The Ingestion Engine (Listening)

### Trigger Events
- `AgreementActivated(agreementId, proposalId, mode)`
- `BorrowerPayloadPublished(agreementId, envelope)`
- `CoverageCovenantBreached(agreementId, ...)`
- `DrawRightsTerminated(agreementId, reason)`
- `AgreementDefaulted(agreementId, pastDue)`
- `AgreementClosed(agreementId)`

### Handling Logic
- **Idempotency:** Store `blockNumber` + `logIndex` for every processed event.
- **Filtering:** Only process compute-linked agreements.
- **Ordering:** Enforce deterministic local transition ordering when events arrive from reorg-prone windows.

---

## 4) Credential Handoff (On-Chain Mailbox)

Credential handoff is on-chain using:
- `AgentEncPubRegistryFacet` (agent encryption public key registry)
- `AgenticMailboxFacet` (per-agreement encrypted payload channel)

### 4.1 Required flow
1. Agent registers encryption pubkey.
2. Agent publishes borrower payload after activation.
3. Orchestration node provisions provider resources / scoped credentials.
4. Orchestration node encrypts provider payload to borrower and publishes on-chain.
5. Borrower decrypts and consumes provider access details.

Compatibility requirement (`mailbox-sdk-spec.md` v0.2):
- mailbox events carry `bytes envelope`
- envelope bytes MUST decode to a UTF-8 stringified ECIES envelope (`iv/ephemPublicKey/ciphertext/mac`)
- decrypt path MUST call `decryptPayload(privateKey, envelopeString)`

Security invariant:
- Node never publishes plaintext secrets on-chain.

---

## 5) Provisioning & Provider Clients

Triggered by `BorrowerPayloadPublished`.

### 5.1 Lambda (Dedicated Capacity)
- Create instance (`/v1/instances` path).
- Map canonical `unitType -> instance_type`.
- Inject borrower SSH public key into instance config.
- Persist `instance_id` in local state.

### 5.2 RunPod (Burst Inference)
- Create endpoint/pod through GraphQL API.
- Persist `pod_id` / `endpoint_id` in local state.
- Publish endpoint/token details via mailbox.

### 5.3 Venice (Managed API Inference)
- Create scoped INFERENCE API key (`POST /api_keys`) with policy limits.
- Optional: validate model policy against `/models`.
- Persist `apiKeyId` + non-secret metadata.
- Publish encrypted provider payload containing:
  - `baseUrl` (`https://api.venice.ai/api/v1`)
  - scoped API key
  - expiry/consumption limits
  - model policy metadata

Security rule:
- plaintext API key lifetime in memory should be minimal and never logged.

---

## 6) Metering Relayer (Usage Oracle)

Compute financing requires metered off-chain usage to become on-chain debt.

### Flow
1. Cron worker runs periodically (e.g., 15m–1h depending on agreement policy).
2. Provider query:
   - **Lambda:** instance status + elapsed runtime windows.
   - **RunPod:** endpoint/pod usage / inference telemetry.
   - **Venice:** `GET /billing/usage` with pagination/time windows.
3. Normalize provider rows into canonical `unitType` deltas.
4. Batch `registerUsage(agreementId, unitType, amount)` submissions.
5. Update local reconciliation state after confirmation.

### Replay safety
- Every metering row gets a deterministic digest for dedupe.
- High-watermark checkpoints are persisted per agreement.

---

## 7) Enforcement (Kill Switch)

On risk triggers (`CoverageCovenantBreached`, `DrawRightsTerminated`, `AgreementDefaulted`), the node must stop off-chain spend quickly.

### Flow
1. Detect trigger event.
2. Lookup provider resource linkage from local state.
3. Execute provider shutdown/revocation:
   - **Lambda:** terminate instance.
   - **RunPod:** terminate pod/endpoint.
   - **Venice:** clamp key limit to zero, set immediate expiry, revoke key (`DELETE /api_keys`).
4. Perform final metering sync up to stop/revoke boundary.
5. Mark agreement as terminated in local state.

### Edge handling
- Provider outage -> enqueue retries with exponential backoff + jitter.
- On-chain draw rights are already frozen, but off-chain resource stop/revoke must continue until confirmed.

---

## 8) Local State Model (Minimum)

- Agreement linkage:
  - `agreementId`
  - `providerType` (`lambda|runpod|venice`)
  - `providerResourceId` (`instance_id|pod_id|apiKeyId`)
- Metering checkpoint:
  - `lastSyncedAt`
  - `lastDigest`
  - provider cursor/page marker
- Enforcement status:
  - `active|termination_pending|terminated`

---

## 9) Operational Requirements

- Rate-limit-aware provider clients (especially billing/list endpoints).
- Structured logs with request correlation IDs (`CF-RAY` for Venice when available).
- Dead-letter queues for failed metering and failed shutdown tasks.
- Alerting for prolonged `termination_pending` states.

---

## 10) Success Criteria

- End-to-end activation -> provisioning -> mailbox delivery works across all enabled provider adapters.
- Metering submissions are deterministic and idempotent.
- Kill-switch reliably halts new off-chain spend for delinquent/defaulted agreements.
- Provider outages do not corrupt canonical on-chain accounting.

---

**End of Draft v0.3**
