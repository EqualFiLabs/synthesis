# Compute Orchestration Node Specification

**Status:** Draft v0.2  
**Date:** 2026-03-10  
**Applies to:** `specs/agentic-financing/agentic-financing-spec.md` (Canonical v1.8) and `compute-provider-decision-spec.md`

---

## 1) Purpose

The **Compute Orchestration Node** is the trusted off-chain relayer that bridges the deterministic Equalis smart contracts to the non-deterministic Web2 APIs of compute providers (Lambda, RunPod).

It is responsible for:
1. Translating on-chain agreement activations into live hardware/endpoints.
2. Securely exchanging access credentials with the borrowing agent via on-chain encrypted mailboxes.
3. Metering off-chain usage and posting it on-chain to accrue debt.
4. Enforcing on-chain risk transitions (delinquency/covenant breach) by tearing down off-chain resources.

This node is the sole trusted party holding the provider API keys and the relayer wallet private keys.

---

## 2) Architecture Overview

The node runs as a persistent service with four core subsystems:

1. **Ingestion Engine:** Listens to RPC endpoints for Equalis canonical events.
2. **Provider Client:** Interacts with Lambda Labs and RunPod APIs.
3. **Relayer/Submitter:** Signs and broadcasts `registerUsage()` and `publishProviderPayload()` transactions to the blockchain.
4. **State Database:** A local mapping of `agreementId <-> providerInstanceId <-> billingState` to handle RPC drops and provider API outages cleanly.

---

## 3) The Ingestion Engine (Listening)

The node must subscribe to the Equalis smart contract events to drive the state machine.

### Trigger Events:
- `AgreementActivated(agreementId, proposalId, mode)`
- `BorrowerPayloadPublished(agreementId, envelope)`
- `CoverageCovenantBreached(agreementId, ...)`
- `DrawRightsTerminated(agreementId, reason)`
- `AgreementDefaulted(agreementId, pastDue)`
- `AgreementClosed(agreementId)`

### Handling Logic:
- **Idempotency:** The ingestion engine must store the `blockNumber` and `logIndex` of processed events to survive restarts without double-provisioning or double-terminating.
- **Filtering:** Only process `AgreementActivated` events where the agreement involves the `ComputeUsageFacet`.

---

## 4) Credential Handoff (The On-Chain Mailbox)

Credential handoff is entirely on-chain, utilizing Diamond facets adapted from the EqualX Atomic Mailbox architecture. This removes the need for IPFS or centralized secret passing.

### 4.1 Required Facets
- **`AgentEncPubRegistryFacet`**: Allows agents (via ERC-8004 wallets) to register their compressed secp256k1 encryption public keys.
- **`AgenticMailboxFacet`**: An encrypted, per-agreement message board.

### 4.2 The Handoff Flow
1. **Agent Registration:** Before proposing compute, the borrowing agent registers their encryption public key in the `AgentEncPubRegistryFacet`.
2. **Borrower Payload (The Request):** Upon `AgreementActivated`, the borrowing agent posts their own SSH public key (for Lambda) or environment config to the `AgenticMailboxFacet` via `publishBorrowerPayload(agreementId, envelope)`.
3. **Provisioning (The Node):** 
   - The Orchestration Node detects `BorrowerPayloadPublished`.
   - The node provisions the hardware on Lambda/RunPod, injecting the borrower's provided SSH key/config directly.
4. **Provider Payload (The Delivery):**
   - The node retrieves the instance IP address or generated RunPod API token.
   - The node fetches the borrower's public encryption key from the registry.
   - The node encrypts the connection details and posts them back via `publishProviderPayload(agreementId, envelope)`.
5. **Consumption:** The borrowing agent decrypts the payload and connects to their compute.

*Security Note: The Orchestration Node never generates or holds the private SSH key used to access the Lambda instance. The agent generates it and passes the public key in.*

---

## 5) Provisioning & The Provider Client

Triggered by the `BorrowerPayloadPublished` event.

### 5.1 Lambda (Dedicated Capacity)
- **Action:** POST to `/v1/instances`
- **Mapping:** Map the on-chain `unitType` (e.g., `GPU_A100_1X`) to the Lambda `instance_type` (e.g., `gpu_1x_a100_sxm4`).
- **Injection:** Pass the decoded SSH public key from the borrower's payload into the `ssh_key_names` or startup script payload.
- **State Update:** Save the returned `instance_id` to the local database, linked to the `agreementId`.

### 5.2 RunPod (Burst Inference)
- **Action:** POST to `/v2/graphql` to create a Serverless endpoint or Pod.
- **State Update:** Save the `pod_id` or `endpoint_id`. Connect the endpoint details into the `providerPayload`.

---

## 6) Metering Relayer (The Oracle)

Compute financing relies on metered usage converting into on-chain debt. The node acts as the oracle for this conversion.

### Flow:
1. **Polling:** A cron worker runs periodically (e.g., every 1 hour).
2. **Provider Query:** 
   - **Lambda:** Query `/v1/instances/{id}` to verify the instance is `active`. Calculate the hours elapsed since the last poll.
   - **RunPod:** Query the billing/usage API for inference tokens consumed by the specific endpoint/pod.
3. **Transaction Batching:** The node packages the usage deltas into a `registerUsage(agreementId, unitType, amount)` payload.
4. **On-Chain Submission:** The node signs and broadcasts the transaction to the Equalis `ComputeUsageFacet`.
5. **Reconciliation:** Upon successful transaction confirmation, the node updates its local database to mark those units as "billed on-chain."

---

## 7) Enforcement (The Kill Switch)

If a borrower fails to meet their payment covenants, the on-chain risk transitions must immediately halt the off-chain compute.

### Triggers:
- `CoverageCovenantBreached`
- `DrawRightsTerminated`
- `AgreementDefaulted`

### Flow:
1. **Ingestion:** The node detects one of the trigger events for a tracked `agreementId`.
2. **Lookup:** The node retrieves the associated `providerInstanceId` from the local database.
3. **Execution:**
   - **Lambda:** POST to `/v1/instances/{id}/terminate`.
   - **RunPod:** Execute the GraphQL mutation to terminate the pod/endpoint.
4. **Final Metering:** The node performs one final usage calculation up to the exact moment of termination and submits a final `registerUsage()` transaction to capture all outstanding debt.
5. **State Update:** The `agreementId` is marked as `terminated` in the local database.

### Edge Case Handling:
- **Provider API Outage:** If the terminate call fails (e.g., 500 error from Lambda), the node must queue the request and retry with exponential backoff. The on-chain draw rights are already frozen, so no *new* debt is authorized, but the hardware must be stopped to prevent unrecoverable leakage.

---

**End of Draft v0.2**
