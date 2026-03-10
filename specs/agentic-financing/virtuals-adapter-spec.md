# Equalis Virtuals ACP Adapter Specification (No-Lock-In)

**Status:** Draft v0.2  
**Date:** 2026-03-10  
**Applies to:** `specs/agentic-financing/agentic-financing-spec.md` (Canonical v1.5)

---

## 1) Purpose

Define a Virtuals integration path that enables Equalis Agentic Financing to route execution through Virtuals ACP / ERC-8183-compatible jobs **without introducing protocol lock-in**.

This adapter is a venue connector, not a business-logic owner.

Core financing logic (proposals, agreements, accounting, risk, trust policy) remains in Equalis.
Native encumbrance remains source of truth for financing balances; adapters only synchronize execution lifecycle.

---

## 2) Hard Constraint (No Lock-In)

Integration MUST satisfy all of the following:

1. **Adapter-only coupling**
   - Virtuals-specific logic is isolated to `VirtualsACPAdapter`.
   - Core facets/libraries must not depend on Virtuals-specific schemas.

2. **Standards-first interfaces**
   - Core talks to canonical interfaces (`IACP8183Adapter`, ERC-8004 adapters), not Virtuals proprietary APIs.

3. **Portable accounting**
   - Agreement/accounting state machine is venue-agnostic.
   - ACP terminal state mapping is identical regardless of venue.

4. **Hot-swappable venue path**
   - An agreement’s ACP route can be configured via adapter registry.
   - Future adapters (non-Virtuals ERC-8183 venues) can be added without core storage migrations.

5. **No forced monetary dependency**
   - Equalis settlement and lender accounting are not hard-wired to `$VIRTUAL`.
   - Supported assets remain policy-configurable.

---

## 3) Scope

### In scope
- Adapter contract for ERC-8183 job lifecycle on Virtuals-compatible venue(s).
- Canonical linkage between `agreementId` and external `jobId`.
- Deterministic state synchronization into Equalis accounting.
- Liveness-safe refund path (`claimRefund`) and replay-safe sync.

### Out of scope
- Virtuals tokenization platform logic (Pegasus/Unicorn/Titan).
- Butler UX internals.
- Any Virtuals-exclusive trust/reputation model replacing ERC-8004 rails.

---

## 4) System Architecture

```text
Equalis Core (canonical state machine + accounting + risk)
  ├─ AgenticAgreementFacet / ACPAdapterFacet
  ├─ LibACPLinkage (venue-agnostic)
  └─ AdapterRegistry
       ├─ VirtualsACPAdapter (this spec)
       ├─ <Future ACP Adapter A>
       └─ <Future ACP Adapter B>
```

### 4.1 Adapter registry model

`AdapterRegistry` maps:
- `venueKey => adapterAddress`
- `agreementId => selectedVenueKey`

`venueKey` is bytes32 (e.g. `keccak256("VIRTUALS_ACP_BASE")`).

Core never branches on vendor names; it branches on generic adapter capability.

---

## 5) Canonical Interfaces

Virtuals adapter MUST implement existing canonical interface:

```solidity
interface IACP8183Adapter {
    function createJob(
        uint256 agreementId,
        address provider,
        address evaluator,
        uint256 expiredAt,
        string calldata description,
        bytes calldata hookConfig
    ) external returns (uint256 acpJobId);

    function fundJob(uint256 agreementId, uint256 acpJobId, uint256 expectedBudget) external;
    function submitJob(uint256 agreementId, uint256 acpJobId, bytes32 deliverable) external;
    function completeJob(uint256 agreementId, uint256 acpJobId, bytes32 reason) external;
    function rejectJob(uint256 agreementId, uint256 acpJobId, bytes32 reason) external;
    function claimRefund(uint256 agreementId, uint256 acpJobId) external;
}
```

### 5.1 Optional read interface (recommended)

```solidity
interface IACP8183ReadAdapter {
    function getJobState(uint256 acpJobId) external view returns (uint8 state);
    function getEscrowedBudget(uint256 acpJobId) external view returns (uint256 amount);
    function getLastReason(uint256 acpJobId) external view returns (bytes32 reason);
}
```

Use this to make sync deterministic and pull-based.

---

## 6) Data Model (Adapter-Local)

```solidity
struct VenueLink {
    uint256 agreementId;
    uint256 acpJobId;
    bytes32 venueKey;
    uint64 createdAt;
    bool exists;
}

struct SyncStamp {
    uint8 lastSeenState;      // Open/Funded/Submitted/Completed/Rejected/Expired
    uint40 lastSyncedAt;
    bytes32 lastReason;
}
```

Adapter-local storage may include venue-specific references, but core storage must only hold canonical linkage fields already defined in canonical spec.

---

## 7) Lifecycle Mapping (Canonical)

Terminal state mapping remains fixed:

| ACP state | Equalis accounting action |
|---|---|
| `Completed` | Financed amount remains utilized debt under agreement repayment schedule |
| `Rejected` | Refund reduces agreement utilization / outstanding principal |
| `Expired` | `claimRefund` reduces utilization / principal (same as Rejected) |

No venue-specific deviation is allowed.

---

## 8) Execution Flows

### 8.1 Create + Link
1. Core selects adapter from registry using `venueKey` policy.
2. Core calls `createJob(...)` on adapter.
3. Adapter creates external ACP job.
4. Adapter returns `acpJobId` and emits linkage event.

### 8.2 Fund
1. Core validates expected budget from agreement draw policy.
2. Core calls `fundJob(agreementId, acpJobId, expectedBudget)`.
3. Adapter forwards call to venue ACP.
4. Core records ACP funded event and updates utilization.

### 8.3 Submit / Resolve
- Provider path: `submitJob`
- Evaluator path: `completeJob` or `rejectJob`
- Core syncs terminal state via adapter-read or callback-safe path.

### 8.4 Refund liveness
- `claimRefund` must remain callable when expired.
- Refund path must not be pausable by adapter-specific hook logic.

---

## 9) Security Requirements

1. **CEI + Reentrancy guards** on all state-mutating adapter entry points.
2. **Strict authorization**:
   - only approved core facet(s) can call adapter mutating functions.
3. **Idempotent sync**:
   - repeated state-sync calls must not double-apply accounting.
4. **Budget integrity**:
   - preserve `expectedBudget` anti-front-run check.
5. **Terminal finality guard**:
   - once canonical terminal state is recorded, no backward transitions.
6. **Reason hashing**:
   - persist canonical `reason` hash from terminal transitions for auditability.

---

## 10) Lock-In Prevention Checklist (MUST PASS)

- [ ] No Virtuals-specific enums/structs in core financing storage.
- [ ] No `$VIRTUAL` hardcoded in core or adapter (policy/config only).
- [ ] Adapter is selected by generic `venueKey`, not hardcoded address in core.
- [ ] At least one mock/alt ERC-8183 adapter passes same integration tests.
- [ ] Agreement accounting tests pass identically across adapter implementations.
- [ ] Core financing accounting remains valid when module subsystem is paused or absent.

---

## 11) Test Plan

### 11.1 Unit
- Create/link/fund/submit/complete/reject/claimRefund happy paths.
- Authorization failures.
- Budget mismatch reverts.
- Expiry before/after boundary.

### 11.2 Invariants
- One ACP terminal sync applies accounting once.
- `Rejected` and `Expired` refund actions are equivalent.
- No terminal-to-nonterminal transitions.

### 11.3 Differential portability test
Run same scenario matrix against:
1. `VirtualsACPAdapter`
2. `MockGeneric8183Adapter`

Expected: identical core accounting and risk outcomes.

---

## 12) MVP Delivery Sequence

1. Build `AdapterRegistry` and generic adapter wiring.
2. Implement `VirtualsACPAdapter` using canonical interface.
3. Implement `MockGeneric8183Adapter` for portability tests.
4. Add differential tests and no-lock-in checklist gate in CI.
5. Enable `venueKey` policy config per agreement/product type.

---

## 13) Success Criteria

- Virtuals route works for ACP execution.
- Removing/replacing Virtuals adapter requires **no core storage migration**.
- Core accounting and risk logic remain unchanged across venues.
- Integration remains ERC-8183/8004 portable by design.

---

**End of Draft v0.2**
