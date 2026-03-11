# Equalis Venice Compute Adapter Specification (No-Lock-In)

**Status:** Draft v0.1  
**Date:** 2026-03-10  
**Applies to:**
- `specs/agentic-financing/agentic-financing-spec.md` (Canonical v1.10)
- `specs/agentic-financing/compute-provider-decision-spec.md`
- `specs/agentic-financing/compute-orchestration-spec.md`

---

## 1) Purpose

Define a Venice integration path for financed inference workloads using Venice API keys and usage telemetry, while preserving canonical Agentic Financing invariants:

- no provider lock-in,
- adapter-only coupling,
- deterministic accounting,
- native encumbrance as source of truth.

This adapter treats Venice as a **managed inference credit rail**, not as protocol accounting authority.

---

## 2) Hard Constraints (MUST HOLD)

1. **Adapter-only coupling**
   - Venice-specific API logic is isolated to `VeniceComputeAdapter` + orchestration provider client.
   - Core facets/libraries do not branch on provider-specific payloads.

2. **Portable accounting**
   - Core debt/usage accounting is provider-agnostic.
   - Venice response formats are normalized into canonical unit records before on-chain submission.

3. **No permanent external key dependency**
   - Core agreement state is reconstructible without retaining plaintext API keys.
   - Adapter stores Venice `apiKeyId` and normalized metering checkpoints; not raw key material.

4. **Fail-closed enforcement**
   - On covenant/default triggers, draw rights freeze on-chain first.
   - Off-chain key revocation/limit clamp is retried until confirmed.

5. **No model/vendor hardcoding in core**
   - Venice model IDs and feature flags are agreement policy/config inputs.

---

## 3) Scope

### In scope
- Per-agreement Venice inference key lifecycle (create/update/revoke).
- Secure key delivery to borrower via Agentic Mailbox.
- Usage metering from Venice billing endpoints into `registerUsage()`.
- Kill-switch behavior for delinquency/default.
- Differential accounting parity tests vs other compute adapters.

### Out of scope
- Venice staking/token workflows as mandatory path (`generate_web3_key` remains optional for future autonomous key issuance).
- Direct changes to canonical agreement/risk semantics.
- Any replacement of ERC-8004/8183 trust and venue abstractions.

---

## 4) Adapter Architecture

```text
Equalis Core (canonical agreement + accounting + risk)
  └─ ComputeAdapterRegistry
      ├─ LambdaComputeAdapter
      ├─ RunPodComputeAdapter
      └─ VeniceComputeAdapter  <-- this spec

Compute Orchestration Node
  ├─ Event Ingestion
  ├─ Venice Provider Client (api_keys + billing + models)
  ├─ Mailbox Relayer
  └─ Metering Submitter (registerUsage)
```

`VeniceComputeAdapter` MUST remain swappable with no core storage migration.

---

## 5) Provider API Mapping (Venice)

Base URL:
- `https://api.venice.ai/api/v1`

Primary endpoints used:
- `POST /api_keys` (create per-agreement key)
- `PATCH /api_keys` (clamp/update limits, expiry)
- `DELETE /api_keys` (revoke key)
- `GET /billing/usage` (usage telemetry)
- `GET /billing/balance` (health/guardrail checks)
- `GET /models` (optional model allowlist sanity checks)

Operational metadata to capture:
- `CF-RAY`
- `x-ratelimit-*`
- `x-venice-balance-*`

---

## 6) Key Lifecycle + Credential Handoff

### 6.1 Activation flow

1. Agreement activates with Venice-routed compute policy.
2. Orchestration Node creates a scoped Venice INFERENCE key:
   - `apiKeyType = INFERENCE`
   - `consumptionLimit` set from agreement policy (USD/DIEM guardrail)
   - `expiresAt` aligned to agreement TTL or shorter rolling window
3. Node stores `apiKeyId` in adapter-local state.
4. Node publishes encrypted provider payload via `publishProviderPayload(...)` containing:
   - provider name (`venice`)
   - base URL
   - API key (secret)
   - optional model allowlist/policy metadata
   - key expiry/limit metadata

Transport compatibility rule:
- encrypt with `@equalfi/mailbox-sdk` (`encryptPayload`)
- resulting stringified envelope is UTF-8 encoded to `bytes` for on-chain mailbox events
- receiver decodes `bytes -> string` before `decryptPayload`

### 6.2 Security requirements

- API key plaintext MUST never be logged.
- API key plaintext MUST only exist in process memory long enough to envelope-encrypt and deliver.
- Payload encryption MUST use mailbox ECIES standard from `mailbox-sdk-spec.md`.

---

## 7) Usage Metering and Canonical Unit Conversion

### 7.1 Metering source

Use `GET /billing/usage` with pagination and time windows.

### 7.2 Deduplication/checkpointing

Each ingested usage row MUST be deduplicated using a deterministic composite key, e.g.:

`hash(timestamp | sku | inferenceDetails.requestId | amount | units)`

Persist high-watermark checkpoints per agreement to survive restarts.

### 7.3 Canonical conversion

Adapter normalizes Venice usage rows into canonical `unitType` buckets, e.g.:

- `VENICE_TEXT_TOKEN_IN`
- `VENICE_TEXT_TOKEN_OUT`
- `VENICE_IMAGE_GEN`
- `VENICE_AUDIO_TTS_CHAR`
- `VENICE_AUDIO_STT_SEC`

Then submits aggregated deltas through `registerUsage(agreementId, unitType, amount)`.

Rules:
- Conversion policy must be deterministic and versioned.
- Any billing row not mappable to an allowed `unitType` must be quarantined and flagged; never silently dropped.

---

## 8) Enforcement (Kill Switch)

Triggered by canonical risk events:
- `CoverageCovenantBreached`
- `DrawRightsTerminated`
- `AgreementDefaulted`

Execution sequence:
1. Freeze new draws on-chain (canonical state transition).
2. Clamp Venice key limits to zero and set immediate expiry.
3. Revoke key (`DELETE /api_keys`) when safe.
4. Perform final metering pass and submit final usage delta.
5. Mark adapter-local agreement state as terminated.

If Venice API is unavailable:
- enqueue retry with exponential backoff and jitter,
- alert ops channel,
- keep agreement in terminated-enforcement-pending local state until confirmed.

---

## 9) Adapter-Local State (Minimum)

```solidity
struct VeniceLink {
    uint256 agreementId;
    string apiKeyId;
    bytes32 keyFingerprint;      // never store plaintext key
    uint40 createdAt;
    uint40 expiresAt;
    bool active;
}

struct VeniceMeterCheckpoint {
    uint40 lastUsageTimestamp;
    bytes32 lastUsageDigest;
    uint40 lastSyncedAt;
}
```

---

## 10) No-Lock-In Acceptance Checklist (MUST PASS)

- [ ] Core contracts contain no Venice-specific business logic branches.
- [ ] Disabling Venice adapter does not break canonical agreement accounting.
- [ ] Equivalent workload traces produce matching accounting outcomes across Venice and at least one non-Venice adapter.
- [ ] Provider key material is not required to reconstruct canonical agreement state.
- [ ] Swapping away from Venice requires no core storage migration.

---

## 11) Test Plan

### Unit tests
- key create/update/delete success and error normalization
- mailbox payload encryption/decryption roundtrip
- usage pagination + dedupe + checkpoint replay
- kill-switch behavior under partial provider outage

### Invariant tests
- no double-application of usage deltas
- final usage sync on termination is idempotent
- on-chain draw freeze always precedes off-chain revoke attempts

### Differential tests
- replay identical synthetic workload traces through:
  1. `RunPodComputeAdapter` (or Lambda path)
  2. `VeniceComputeAdapter`

Expected: identical canonical debt/accounting outcomes after normalization.

---

## 12) Implementation Sequence

1. Add `VeniceComputeAdapter` to compute adapter registry.
2. Implement orchestration provider client for Venice key + billing APIs.
3. Add policy schema for Venice model allowlist and consumption limits.
4. Implement metering normalization table (`sku/model -> unitType`).
5. Implement kill-switch revoke path.
6. Add differential accounting tests and no-lock-in gate in CI.

---

## 13) Success Criteria

- Venice-backed financed inference runs end-to-end via canonical compute agreement flow.
- Metered usage is posted on-chain deterministically with replay safety.
- Default/delinquency stops new inference spend via key clamp/revoke.
- Canonical financing state remains venue-agnostic and migration-free.

---

**End of Draft v0.1**
