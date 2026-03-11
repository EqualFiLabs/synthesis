# Equalis Agentic Financing — Canonical Specification

**Status:** Canonical Draft v1.10  
**Date:** 2026-03-10  
**Protocol:** Equalis (EqualFi)  
**Replaces:**
- `specs/agentic-financing-current.md`
- `specs/compute-credit-spec.md`

---

## 1) Purpose

Define one unified financing framework for autonomous agents with four product surfaces:

1. **Solo Agentic Financing** (single lender ↔ single agent)
2. **Pooled Agentic Financing** (many lenders ↔ single agent)
3. **Solo Compute/Inference Lending** (single compute lender ↔ single agent)
4. **Pooled Compute/Inference Lending** (many lenders ↔ compute demand)

All four products must share:
- One proposal lifecycle
- One accounting model
- One risk state machine
- One fee routing model
- One identity model (ERC-8004 via agent-wallet-core integration)

---

## 2) Design Goals

1. **One canonical framework** across money-credit and compute-credit.
2. **Deterministic state transitions** for request, approval, draw, repay, delinquency, default, resolution.
3. **Composable with Position NFTs** and native encumbrance rails.
4. **Capital-efficient for lenders** while preserving solvency boundaries.
5. **Simple to implement in current Diamond architecture.**
6. **No module dependency** in canonical financing invariants (module bridging is optional).
7. **Optional collateralization toggle** for lender protection without forcing capital lock on all agreements.

---

## 3) Product Matrix

| Product | Capital Source | Underwriting | Release Style | Repayment Source |
|---|---|---|---|---|
| Solo Agentic Financing | Single lender position | Lender policy | Revolving or milestone | Agent revenue + manual repay |
| Pooled Agentic Financing | Pooled lender capital | Governance + policy | Revolving or milestone | Agent revenue + manual repay |
| Solo Compute/Inference Lending | Single lender prepaid compute budget | Lender policy | On-demand compute draw | Agent revenue + manual repay |
| Pooled Compute/Inference Lending | Pooled prepaid compute budget | Governance + policy | On-demand compute draw | Agent revenue + manual repay |

---

## 4) Shared Primitives

### 4.1 Identity Model

- Borrower identity is represented by ERC-8004 coordinates:
  - `agentRegistry` (`{namespace}:{chainId}:{identityRegistry}`)
  - `agentId` (ERC-721 token id within that identity registry)
- Borrower execution address is resolved from ERC-8004 wallet semantics:
  1. `getAgentWallet(agentId)` when set
  2. fallback to identity registry `ownerOf(agentId)` when wallet is unset
- Authoritative borrower address is the resolved wallet at execution time (not cached indefinitely).
- Deployment profile note: v1 assumes one canonical identity registry per deployment environment; multi-registry indexing MAY be added later via hashed agent keys.

### 4.2 Proposal Primitive

All products begin with a proposal.

```solidity
enum ProposalType {
    SoloAgentic,
    PooledAgentic,
    SoloCompute,
    PooledCompute
}

enum ProposalStatus {
    Pending,
    Approved,
    Rejected,
    Expired,
    Cancelled,
    Activated
}

struct FinancingProposal {
    uint256 id;
    ProposalType proposalType;
    string agentRegistry;         // ERC-8004 coordinate: {namespace}:{chainId}:{identityRegistry}
    uint256 agentId;
    uint256 borrowerPositionId;   // Position NFT anchor for borrower obligations
    uint256 lenderPositionId;     // solo only (0 for pooled)
    uint256 fundingPoolId;        // pool supplying financing liquidity

    // Optional collateral profile
    bool collateralEnabled;
    uint16 minCollateralRatioBps; // required at activation when collateralEnabled=true
    address collateralAsset;      // 0x0 means position-native collateral only

    address settlementAsset;      // e.g. USDC for accounting/repayment
    uint256 requestedAmount;      // money-denominated cap for Agentic products
    uint256 requestedUnits;       // compute units cap for Compute products
    uint40 createdAt;
    uint40 expiresAt;
    bytes32 termsHash;            // hash-commit to canonical terms payload
    string uri;                   // human/machine readable term details
    bytes32 uriHash;              // canonical payload hash
    uint16 uriSchemaVersion;
    address counterparty;         // lender for solo, zero for pooled
    ProposalStatus status;
}
```

### 4.3 Agreement Primitive

When approved, proposal becomes an active agreement.

```solidity
enum AgreementMode {
    Revolving,
    Milestone,
    MeteredUsage
}

enum AgreementStatus {
    Active,
    Delinquent,
    Defaulted,
    Closed,
    WrittenOff
}

enum TrustMode {
    DiscoveryOnly,
    ReputationOnly,
    ValidationRequired,
    Hybrid
}

struct FinancingAgreement {
    uint256 id;
    uint256 proposalId;
    string agentRegistry;         // ERC-8004 coordinate: {namespace}:{chainId}:{identityRegistry}
    uint256 agentId;
    uint256 borrowerPositionId;
    uint256 lenderPositionId;     // solo only (0 for pooled)
    bytes32 borrowerPositionKey;
    bytes32 lenderPositionKey;    // solo only (0x0 for pooled)
    uint256 fundingPoolId;
    AgreementMode mode;
    AgreementStatus status;

    // Funding limits
    uint256 creditLimit;          // money-denominated limit
    uint256 unitLimit;            // compute/inference unit limit

    // Current balances
    uint256 principalDrawn;
    uint256 principalRepaid;
    uint256 interestAccrued;
    uint256 feesAccrued;

    // Native encumbrance tracking
    uint256 principalEncumbered;
    uint256 unitsEncumbered;

    // Payment schedule
    uint256 minPaymentPerPeriod;
    uint32 paymentInterval;
    uint40 firstDueAt;
    uint32 gracePeriod;

    // Risk
    uint16 reserveBps;
    uint16 liquidationPenaltyBps;
    uint16 writeOffThresholdBps;

    // Optional collateral profile (toggle)
    bool collateralEnabled;
    address collateralAsset;        // 0x0 means position-native collateral only
    uint16 minCollateralRatioBps;   // activation requirement when collateralEnabled=true
    uint16 maintenanceCollateralRatioBps;
    uint256 collateralPosted;
    uint256 collateralSeized;

    // Lender-protection covenant (net draw coverage)
    uint16 minNetDrawCoverageBps;  // e.g. 10000 = payment must at least match net draw component
    uint256 principalFloorPerPeriod;
    uint32 covenantCurePeriod;
    bool drawTerminated;           // once set, new draws are permanently disabled for this agreement

    // Metadata
    bytes32 termsHash;
    bool pooled;

    // ERC-8004 trust profile
    TrustMode trustMode;
    int128 minReputationValue;
    uint8 minReputationValueDecimals;
    address requiredValidator;
    uint8 minValidationResponse;   // 0-100 from ERC-8004 validation registry

    // ERC-8183 interoperability
    bool acpEnabled;
    address acpAdapter;           // integration adapter contract
    address evaluator;            // evaluator used for linked ACP jobs
}
```

### 4.4 Capital Backing + Native Encumbrance

- Backing capital remains represented inside Equalis accounting.
- Drawn capital is tracked as explicit utilization against agreement limits.
- Encumbered backing is tracked through **native encumbrance rails** (not module-required paths).
- Canonical mechanism for financing encumbrance is:
  - `LibEncumbrance`
  - `LibActiveCreditIndex`
  - `LibSolvencyChecks`
- Module accounting MAY mirror financing state for optional integrations, but canonical financing correctness MUST NOT depend on module registry state.

### 4.5 Repayment Routing

For each repayment amount `R`:
- `R_lenders = R * 7000 / 10000`
- `R_protocolRail = R - R_lenders`

`R_protocolRail` is routed via `LibFeeRouter` using existing split parameters.

This keeps a single global value rail:
- Treasury share
- ACI share
- FI share

### 4.6 Interest + Fee Accrual

Per agreement:
- Linear interest accrual between checkpoints
- Fee accrual based on configured fee schedule
- Repayment waterfall:
  1. Fees
  2. Interest
  3. Principal

### 4.7 ERC-8183 Interoperability Profile

When financing execution is routed through ERC-8183 jobs, this spec uses the following profile:

- Job standard: ERC-8183 Agentic Commerce (`createJob`, `setBudget`, `fund`, `submit`, `complete`, `reject`, `claimRefund`)
- Role mapping:
  - ACP `client` = agent wallet (borrower)
  - ACP `provider` = external service provider / execution counterparty
  - ACP `evaluator` = agreement evaluator (EOA or policy contract)
- `claimRefund` remains an always-available liveness path for expired jobs.

ACP linkage for each agreement is maintained via `agreementId <-> acpJobId` mappings in `AgenticStorage`.

### 4.8 ERC-8004 Interoperability Profile

This spec integrates ERC-8004 across identity, reputation, and validation rails:

- Identity rail:
  - Canonical borrower key is (`agentRegistry`, `agentId`)
  - Borrower address resolution follows ERC-8004 wallet semantics (`getAgentWallet` then `ownerOf` fallback)
- Reputation rail:
  - Agreement outcomes can emit standardized feedback via ERC-8004 reputation registry
  - Feedback payloads SHOULD use `tag1`/`tag2` taxonomy for repayment quality, delinquency, default, and delivery quality
- Validation rail:
  - High-risk agreements MAY require ERC-8004 validation requests/responses before activation or before key transitions
  - Validator responses use the ERC-8004 `0-100` response scale

Trust modes are explicit per agreement via `TrustMode`:
- `DiscoveryOnly`
- `ReputationOnly`
- `ValidationRequired`
- `Hybrid`

### 4.9 Venue Portability + No-Lock-In Rule

ACP execution venues are adapter-selected and MUST remain portable.

- Core financing state, accounting, risk, and trust logic MUST remain venue-agnostic.
- Venue-specific logic MUST be isolated to adapter contracts.
- No single venue (including Virtuals) may become a required dependency for protocol correctness.
- Canonical no-lock-in adapter profile is defined in:
  - `specs/agentic-financing/virtuals-adapter-spec.md`

### 4.10 Native Encumbrance Namespace

Agentic financing reserves a dedicated native encumbrance namespace.

```solidity
bytes32 constant AGENTIC_ENCUMBRANCE_NAMESPACE = keccak256("equalis.agentic.encumbrance.v1");
```

Rules:
- Financing encumbrance is tracked per agreement and position key in native storage.
- Core transitions (`draw`, `repay`, `default`, `write-off`, `refund`) MUST mutate native encumbrance deterministically.
- Module pause/inactive states MUST NOT block canonical financing accounting transitions.
- Optional module bridges MAY read native encumbrance and mirror it, but never become source of truth.

### 4.11 Net Draw Coverage Covenant (Lender Protection)

Each agreement MAY enforce a periodic payment-vs-draw covenant.

For period `p`:

- `grossDraw_p` = total draws in period
- `refunds_p` = total refunds/reversals in period (including ACP reject/expired paths)
- `netDraw_p = max(0, grossDraw_p - refunds_p)`

Required minimum payment for the period:

`requiredPayment_p = feesDue_p + interestDue_p + (netDraw_p * minNetDrawCoverageBps / 10000) + principalFloorPerPeriod`

Rules:
- `minNetDrawCoverageBps` MUST be `>= 10000` for protected agreements.
- If `actualPayment_p < requiredPayment_p`, covenant is breached.
- On first breach: mark delinquent and freeze new draws immediately.
- If breach remains unresolved past `covenantCurePeriod`: terminate draw rights (`drawTerminated = true`) and transition to default workflow.
- Repayments and refunds remain callable after draw termination.

### 4.12 Optional Collateralization Toggle

Collateralization is optional and policy-driven.

- If `collateralEnabled=false`, agreement runs covenant-only risk controls.
- If `collateralEnabled=true`, agreement must satisfy `minCollateralRatioBps` at activation.
- Collateral is a first-loss buffer and does **not** disable covenant checks.
- Collateral MAY be position-native or asset-denominated depending on policy.
- Collateral shortfall freezes new draws; unresolved shortfall transitions through delinquency/default workflow.

Design intent:
- preserve borrower utility in default path (no mandatory collateral across all agreements)
- allow lenders/pools to demand stronger protection on selected agreements

---

## 5) State Machines

### 5.1 Proposal State Machine

```
Pending -> Approved -> Activated
Pending -> Rejected
Pending -> Expired
Pending -> Cancelled
```

Rules:
- `expiresAt > createdAt`
- Approval requires valid `termsHash` binding.
- Trust-gated flows MUST satisfy configured `TrustMode` constraints (reputation and/or validation) before activation.
- Activation creates `FinancingAgreement`.

### 5.2 Agreement State Machine

```
Active -> Delinquent -> Defaulted -> WrittenOff
Active -> Closed
Delinquent -> Active (if cured)
Defaulted -> Closed (if fully recovered)
```

### 5.3 Delinquency Logic

Base payment delinquency:
- `periodsElapsed`
- `requiredCumulative = periodsElapsed * minPaymentPerPeriod`
- `actualCumulative` applied via waterfall
- `pastDue = max(0, requiredCumulative - actualCumulative)`

Coverage covenant delinquency (when enabled):
- compute `requiredPayment_p` using Section 4.11
- `coverageShortfall_p = max(0, requiredPayment_p - actualPayment_p)`

Collateral delinquency (when collateralEnabled=true):
- compute live collateral ratio from policy-defined valuation source
- `collateralShortfall = (collateralRatio < maintenanceCollateralRatioBps)`

Then:
- If `pastDue > 0` or `coverageShortfall_p > 0` or `collateralShortfall` after due boundary: `Delinquent`
- On covenant breach or collateral shortfall, new draws are frozen immediately.
- If breach remains unresolved past `covenantCurePeriod` (or `gracePeriod` when larger):
  - set `drawTerminated = true`
  - transition to `Defaulted`
- `Delinquent -> Active` only if all shortfalls are cured.

---

## 6) Product A — Solo Agentic Financing

### 6.1 Definition

Single lender provides financing line to one agent identity.

### 6.2 Supported Modes

- Revolving line
- Milestone tranches

### 6.3 Core Flow

1. Agent submits `SoloAgentic` proposal.
2. Target lender approves/rejects before expiry.
3. On approval, agreement activates.
4. Agent draws funds up to `creditLimit`.
5. Agent repays from revenue/manual payments.
6. Agreement closes when debt is fully repaid and no obligations remain.

### 6.4 Solo Controls

- Per-tx draw cap
- Daily draw cap
- Allowed recipient policy (optional)
- Milestone release gates (for milestone mode)

---

## 7) Product B — Pooled Agentic Financing

### 7.1 Definition

Many lenders back one agreement framework through pooled capital.

### 7.2 Governance

- Proposal approved by weighted pool voting (snapshot at vote start)
- Quorum and threshold per pool policy
- Optional guardian veto during emergency pause windows

### 7.3 Core Flow

1. Agent submits `PooledAgentic` proposal.
2. Pool opens vote.
3. If approved, agreement activates with pooled backing.
4. Agent draws under pooled risk limits.
5. Repayments distributed:
   - direct lender-share accounting
   - protocol rail via `LibFeeRouter`
6. On failure, pooled default handling executes recovery and write-off policy.

### 7.4 Pooled Risk Controls

- Max exposure per agent
- Max exposure per sector/category
- Pool utilization ceiling
- Reserve floor

---

## 8) Product C — Solo Compute/Inference Lending

### 8.1 Definition

Single lender allocates prepaid compute/inference budget to one agent.

### 8.2 Unit Model

Compute credit is tracked in abstract units with provider mapping:

```solidity
struct ComputeUnitConfig {
    bytes32 unitType;            // e.g. GPU_HOUR_A100, INFERENCE_TOKEN
    uint256 unitPrice;           // settlement asset per unit
    address providerAdapter;     // provider bridge adapter
    bool active;
}
```

### 8.3 Core Flow

1. Agent submits `SoloCompute` proposal with requested units + budget cap.
2. Lender approves/rejects.
3. Agreement activates in `MeteredUsage` mode.
4. Agent draws compute units; usage converts to monetary liability.
5. Agent repays in settlement asset.

### 8.4 Settlement Rule

Each usage event:
- `debtDelta = usedUnits * unitPrice`
- add to principal/usage debt ledger
- optional provider surcharge as fee component

For compute-heavy agreements, covenant periods SHOULD be short (e.g. 7 days) rather than monthly to prevent fast negative carry accumulation.

### 8.5 Provider Strategy (No-Lock-In)

Compute providers are routed through adapter abstraction and are never canonical sources of truth.

- Dedicated capacity financing path: Lambda-first
- Burst/serverless financing path: RunPod-first
- Managed API inference financing path: Venice-first
- Routing and fallback are policy-configurable and reversible via adapter allowlists

Detailed scoring and integration sequencing are defined in:
- `specs/agentic-financing/compute-provider-decision-spec.md`
- `specs/agentic-financing/venice-adapter-spec.md`

---

## 9) Product D — Pooled Compute/Inference Lending

### 9.1 Definition

Many lenders fund a pooled compute budget that agents draw from under policy.

### 9.2 Core Flow

1. Agent submits `PooledCompute` proposal.
2. Pool governance approves/rejects.
3. Agreement activates with pooled unit and monetary caps.
4. Agent draws compute/inference units.
5. Debt accrues from metered usage.
6. Repayments route via shared repayment rails.

### 9.3 Pool Policies

- Unit-type allowlist
- Provider allowlist
- Max unit burn per day per agent
- Max concurrent active compute agreements

### 9.4 ACP Lifecycle Synchronization (ERC-8183)

For ACP-linked agreements, each linked job terminal state MUST synchronize to agreement accounting:

| ACP terminal state | Agreement accounting action |
|---|---|
| `Completed` | No refund; financed amount remains utilized debt and follows standard repayment schedule. |
| `Rejected` | Escrow refund applied back to agreement utilization; reduce outstanding principal by refunded amount. |
| `Expired` | `claimRefund` proceeds; refunded amount reduces utilization/principal identically to `Rejected`. |

Canonical linkage events MUST be emitted for observability and indexers.

### 9.5 ERC-8004 Trust Signal Synchronization

Agreement lifecycle SHOULD synchronize trust rails:

- On healthy completion / repayment milestones:
  - emit positive reputation feedback (`giveFeedback`) with agreement-derived tags
- On delinquency/default/write-off:
  - emit negative or neutral feedback according to policy
- On trust-gated agreements requiring validation:
  - emit/record validation requests (`validationRequest`) and consume validator responses (`validationResponse`) before gated transitions

Trust synchronization logic must be deterministic and auditable from on-chain events.

---

## 10) Risk and Recovery

### 10.1 Default Recovery

On `Defaulted`:
1. Freeze new draws
2. If covenant breach caused default, permanently terminate draw rights (`drawTerminated = true`)
3. Keep repayments and refunds open
4. Apply penalty schedule
5. If `collateralEnabled=true`, apply collateral seizure path up to policy limits
6. Attempt recovery from remaining configured sources
7. If unresolved past threshold, move to `WrittenOff`

### 10.2 Write-Off Accounting

- Write-offs are explicit, evented, and bounded.
- Loss is socialized according to product type:
  - Solo: lender-borne
  - Pooled: pool-borne pro-rata by share logic

### 10.3 Circuit Breakers

Independent pause switches:
- New proposals
- New approvals
- New draws
- Governance finalization

Repayment and settlement remain enabled in pause mode.
For ACP-linked flows, expired-job refunds (`claimRefund`) MUST remain unblocked and cannot be paused.

---

## 11) Canonical Events

```solidity
event ProposalCreated(uint256 indexed proposalId, ProposalType proposalType, uint256 indexed agentId);
event ProposalApproved(uint256 indexed proposalId, address indexed approver);
event ProposalRejected(uint256 indexed proposalId, address indexed rejector);
event AgreementActivated(uint256 indexed agreementId, uint256 indexed proposalId, AgreementMode mode);
event DrawExecuted(uint256 indexed agreementId, uint256 amount, uint256 units, address recipient);
event NativeEncumbranceUpdated(uint256 indexed agreementId, bytes32 indexed positionKey, uint256 principalEncumbered, uint256 unitsEncumbered, bytes32 reason);
event RepaymentApplied(uint256 indexed agreementId, uint256 amount, uint256 toFees, uint256 toInterest, uint256 toPrincipal);
event AgreementDelinquent(uint256 indexed agreementId, uint256 pastDue);
event CollateralProfileSet(uint256 indexed agreementId, bool collateralEnabled, address collateralAsset, uint16 minCollateralRatioBps, uint16 maintenanceCollateralRatioBps);
event CollateralPosted(uint256 indexed agreementId, address indexed asset, uint256 amount, bytes32 sourcePositionKey);
event CollateralReleased(uint256 indexed agreementId, address indexed asset, uint256 amount, bytes32 targetPositionKey);
event CollateralSeized(uint256 indexed agreementId, address indexed asset, uint256 amount, bytes32 reason);
event CoverageCovenantBreached(uint256 indexed agreementId, uint256 indexed periodId, uint256 requiredPayment, uint256 actualPayment, uint256 netDraw);
event CoverageCovenantCured(uint256 indexed agreementId, uint256 indexed periodId, uint256 curePayment);
event DrawRightsTerminated(uint256 indexed agreementId, bytes32 reason);
event AgreementDefaulted(uint256 indexed agreementId, uint256 pastDue);
event AgreementWrittenOff(uint256 indexed agreementId, uint256 writeOffAmount);

// Credential Handoff (Agentic Mailbox)
event AgentEncPubRegistered(address indexed agentWallet, bytes pubkey);
event BorrowerPayloadPublished(uint256 indexed agreementId, address indexed borrower, bytes envelope);
event ProviderPayloadPublished(uint256 indexed agreementId, address indexed provider, bytes envelope);

// ERC-8183 linkage + lifecycle sync
// terminalState: 1=Completed, 2=Rejected, 3=Expired
event ACPJobLinked(uint256 indexed agreementId, uint256 indexed acpJobId, address indexed adapter);
event ACPJobFunded(uint256 indexed agreementId, uint256 indexed acpJobId, uint256 budget);
event ACPJobSubmitted(uint256 indexed agreementId, uint256 indexed acpJobId, bytes32 deliverable);
event ACPJobResolved(uint256 indexed agreementId, uint256 indexed acpJobId, uint8 terminalState, bytes32 reason);

// ACP venue registry / routing
event ACPVenueAdapterSet(bytes32 indexed venueKey, address indexed adapter, bool enabled);
event ACPAgreementVenueSet(uint256 indexed agreementId, bytes32 indexed venueKey, address indexed adapter);

// ERC-8004 trust profile + signal synchronization
event TrustProfileSet(uint256 indexed agreementId, uint8 trustMode, int128 minReputationValue, uint8 minReputationValueDecimals, address requiredValidator, uint8 minValidationResponse);
event ReputationFeedbackPosted(uint256 indexed agreementId, string agentRegistry, uint256 indexed agentId, int128 value, uint8 valueDecimals, string tag1, string tag2);
event ValidationRequested(uint256 indexed agreementId, bytes32 indexed requestHash, address indexed validator);
event ValidationRecorded(uint256 indexed agreementId, bytes32 indexed requestHash, uint8 response, string tag);
```

---

## 12) Contract Architecture (Diamond Mapping)

### 12.1 New Facets

- `AgenticProposalFacet`
  - create/cancel/query proposals
- `AgenticApprovalFacet`
  - solo approvals + pooled vote finalization
- `AgenticAgreementFacet`
  - activate agreements, draw, repay, close
- `ComputeUsageFacet`
  - register metered usage, provider adapter hooks
- `AgentEncPubRegistryFacet`
  - secp256k1 public key registry for agent secure messaging
- `AgenticMailboxFacet`
  - per-agreement encrypted payload slots for credential handoff
- `ACPAdapterFacet`
  - ERC-8183 job creation/linking/funding/submission/resolution sync
- `AdapterRegistryFacet`
  - venue key -> adapter routing, allowlist governance, agreement venue selection
- `AgentTrustFacet`
  - ERC-8004 identity resolution, reputation sync, validation gating
- `AgenticRiskFacet`
  - delinquency/default transitions, write-offs
- `AgenticGovernanceFacet`
  - pooled vote config + quorum/threshold policies

### 12.2 New Libraries

- `LibAgenticStorage`
- `LibAgenticProposal`
- `LibAgenticAgreement`
- `LibAgenticInterest`
- `LibAgenticRepayment`
- `LibComputeMetering`
- `LibACPLinkage`
- `LibAdapterRegistry`
- `LibAgentTrust`
- `LibAgenticRisk`

### 12.3 Existing Integrations

- `LibEncumbrance` (canonical financing encumbrance)
- `LibActiveCreditIndex`
- `LibFeeRouter`
- `LibPositionNFT`
- `LibSolvencyChecks`
- ERC-8183 adapter interface (`IACP8183Adapter`)
- ERC-8004 identity/reputation/validation adapters (`IERC8004IdentityAdapter`, `IERC8004ReputationAdapter`, `IERC8004ValidationAdapter`)
- Companion no-lock-in venue profile for Virtuals route: `specs/agentic-financing/virtuals-adapter-spec.md`
- Optional bridge only: `LibModuleEncumbrance` mirror path (non-canonical)

### 12.4 ERC-8183 Adapter Interface (Canonical)

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

### 12.5 ERC-8004 Adapter Interfaces (Canonical)

```solidity
interface IERC8004IdentityAdapter {
    function resolveAgentWallet(string calldata agentRegistry, uint256 agentId) external view returns (address wallet);
    function resolveOwner(string calldata agentRegistry, uint256 agentId) external view returns (address owner);
}

interface IERC8004ReputationAdapter {
    function giveFeedback(
        string calldata agentRegistry,
        uint256 agentId,
        int128 value,
        uint8 valueDecimals,
        string calldata tag1,
        string calldata tag2,
        string calldata endpoint,
        string calldata feedbackURI,
        bytes32 feedbackHash
    ) external;

    function getSummary(
        string calldata agentRegistry,
        uint256 agentId,
        address[] calldata clientAddresses,
        string calldata tag1,
        string calldata tag2
    ) external view returns (uint64 count, int128 summaryValue, uint8 summaryValueDecimals);
}

interface IERC8004ValidationAdapter {
    function validationRequest(
        string calldata agentRegistry,
        address validatorAddress,
        uint256 agentId,
        string calldata requestURI,
        bytes32 requestHash
    ) external;

    function getValidationStatus(
        string calldata agentRegistry,
        bytes32 requestHash
    ) external view returns (address validatorAddress, uint256 agentId, uint8 response, bytes32 responseHash, string memory tag, uint256 lastUpdate);
}
```

### 12.6 Venue Adapter Registry (Canonical)

ACP adapters are selected by venue key, not hardcoded venue dependency.

```solidity
interface IACPAdapterRegistry {
    function setVenueAdapter(bytes32 venueKey, address adapter, bool enabled) external;
    function getVenueAdapter(bytes32 venueKey) external view returns (address adapter, bool enabled);
    function setAgreementVenue(uint256 agreementId, bytes32 venueKey) external;
    function getAgreementVenue(uint256 agreementId) external view returns (bytes32 venueKey, address adapter);
}
```

Notes:
- `venueKey` SHOULD be deterministic (`keccak256("VIRTUALS_ACP_BASE")`, etc.)
- Registry + agreement venue mapping MUST allow future non-Virtuals ERC-8183 adapters without storage migration.
- Virtuals adapter MUST comply with no-lock-in profile in `specs/agentic-financing/virtuals-adapter-spec.md`.

---

## 13) Storage Layout (Canonical)

```solidity
bytes32 constant AGENTIC_STORAGE_POSITION = keccak256("equalis.agentic.financing.storage.v1");

struct AgenticStorage {
    uint256 nextProposalId;
    uint256 nextAgreementId;

    mapping(uint256 => FinancingProposal) proposals;
    mapping(uint256 => FinancingAgreement) agreements;

    mapping(uint256 => uint256[]) agentToProposals;     // agentId (canonical registry profile) => proposalIds
    mapping(uint256 => uint256[]) agentToAgreements;    // agentId (canonical registry profile) => agreementIds

    // Solo lender linkage
    mapping(address => uint256[]) lenderToProposals;
    mapping(address => uint256[]) lenderToAgreements;

    // Pooled governance/voting state
    mapping(uint256 => bytes32) proposalVoteConfigHash;
    mapping(uint256 => bool) proposalVoteFinalized;

    // ERC-8183 linkage
    mapping(uint256 => uint256[]) agreementToAcpJobs;              // agreementId => acpJobIds
    mapping(uint256 => uint256) acpJobToAgreement;                 // acpJobId => agreementId
    mapping(uint256 => bytes32) acpJobDeliverable;                 // acpJobId => deliverable hash
    mapping(uint256 => bytes32) acpJobResolutionReason;            // acpJobId => resolution reason hash
    mapping(uint256 => uint8) acpJobTerminalState;                 // 1=Completed,2=Rejected,3=Expired

    // ACP venue adapter registry + per-agreement routing
    mapping(bytes32 => address) acpVenueAdapter;                   // venueKey => adapter
    mapping(bytes32 => bool) acpVenueEnabled;                      // venueKey => enabled
    mapping(uint256 => bytes32) agreementAcpVenueKey;              // agreementId => venueKey

    // ERC-8004 trust synchronization state
    mapping(uint256 => bytes32) agreementValidationRequestHash;    // agreementId => requestHash
    mapping(uint256 => uint8) agreementLastValidationResponse;     // agreementId => 0-100
    mapping(uint256 => uint40) agreementLastValidationAt;          // agreementId => timestamp
    mapping(uint256 => int128) agreementLastReputationValue;       // agreementId => latest posted value
    mapping(uint256 => uint8) agreementLastReputationDecimals;     // agreementId => decimals

    // Native encumbrance state (canonical)
    mapping(uint256 => uint256) agreementPrincipalEncumbered;      // agreementId => settlement principal encumbered
    mapping(uint256 => uint256) agreementUnitsEncumbered;          // agreementId => compute units encumbered
    mapping(uint256 => bytes32) agreementBorrowerPositionKey;      // agreementId => borrower position key
    mapping(uint256 => bytes32) agreementLenderPositionKey;        // agreementId => lender position key (0x0 for pooled)

    // Optional collateral tracking
    mapping(uint256 => uint256) agreementCollateralPosted;          // agreementId => total posted collateral amount
    mapping(uint256 => uint256) agreementCollateralSeized;          // agreementId => cumulative seized collateral
    mapping(uint256 => uint16) agreementLastCollateralRatioBps;     // agreementId => latest evaluated collateral ratio
    mapping(uint256 => uint40) agreementLastCollateralCheckAt;      // agreementId => latest collateral ratio checkpoint

    // Compute metering
    mapping(bytes32 => ComputeUnitConfig) computeUnits;
    mapping(uint256 => mapping(bytes32 => uint256)) agreementUnitUsage;

    // Payment tracking
    mapping(uint256 => uint256) cumulativePayments;
    mapping(uint256 => uint40) lastAccrualAt;

    // Net draw coverage covenant tracking
    mapping(uint256 => uint256) agreementCurrentPeriodId;          // agreementId => rolling period id
    mapping(uint256 => mapping(uint256 => uint256)) periodGrossDraw;    // agreementId => periodId => gross draw
    mapping(uint256 => mapping(uint256 => uint256)) periodRefunds;      // agreementId => periodId => refunds/reversals
    mapping(uint256 => mapping(uint256 => uint256)) periodPayments;     // agreementId => periodId => applied payments
    mapping(uint256 => uint8) agreementCovenantStrikes;            // agreementId => unresolved breach count
}
```

---

## 14) Canonical Policy Defaults (v1)

- Repayment split: **70/30** (lender/protocol rail)
- Minimum grace period: **3 days**
- Maximum grace period: **30 days**
- Default `minNetDrawCoverageBps`: **10500** (pay 105% of period net draw component)
- Default `principalFloorPerPeriod`: **non-zero governance value** (asset-specific)
- Default `covenantCurePeriod`: **7 days**
- Default `collateralEnabled`: **false**
- When collateral is enabled:
  - default `minCollateralRatioBps`: **11000**
  - default `maintenanceCollateralRatioBps`: **10500**
- Compute agreements SHOULD use shorter `paymentInterval` defaults (e.g. **7 days**)
- Pooled quorum default: **20%**
- Pooled pass threshold default: **55%**
- Default `TrustMode`: **DiscoveryOnly**
- Validation-required agreements use ERC-8004 `response >= 80` by default (unless overridden)
- ACP venue routing is registry-based (`venueKey -> adapter`), defaulting to governance-selected portable adapter key
- Native encumbrance rails are canonical for financing accounting; module mirrors are optional
- ACP-linked expired refunds (`claimRefund`) are always available (non-pausable)
- No-lock-in rule: no hardcoded venue/token dependency in canonical financing logic
- Global circuit-breaker authority: governance/timelock

---

## 15) Migration and File Policy

This file is the single source of truth.

Superseded documents are archived and removed from active spec surface:
- `specs/agentic-financing-current.md`
- `specs/compute-credit-spec.md`

Any future updates to shared financing/risk/accounting invariants must modify this canonical file directly.

Companion integration documents are allowed when they do not override canonical invariants (e.g. `specs/agentic-financing/virtuals-adapter-spec.md`, `specs/agentic-financing/compute-provider-decision-spec.md`, `specs/agentic-financing/venice-adapter-spec.md`, `specs/agentic-financing/compute-orchestration-spec.md`).

---

## 16) Implementation Sequence

1. Implement storage + proposal lifecycle facets.
2. Implement native encumbrance ledger + position-key anchoring for agreements.
3. Implement solo approval + activation + revolving repayment.
4. Implement pooled voting + pooled activation.
5. Add compute metering layer (solo then pooled).
6. Implement compute provider adapter abstraction + routing policy.
7. Implement `LambdaComputeAdapter` for dedicated-capacity financing.
8. Implement `RunPodComputeAdapter` for burst/serverless financing.
9. Implement `VeniceComputeAdapter` for managed API inference financing (`specs/agentic-financing/venice-adapter-spec.md`).
10. Add differential compute adapter tests for accounting parity.
11. Implement ACP adapter registry + per-agreement venue routing (`venueKey`).
12. Implement ERC-8183 adapter linkage + ACP lifecycle sync accounting.
13. Implement `VirtualsACPAdapter` under no-lock-in profile (`specs/agentic-financing/virtuals-adapter-spec.md`).
14. Implement at least one alternate/mock generic ERC-8183 adapter for portability verification.
15. Implement ERC-8004 adapters (identity/reputation/validation) + trust-mode gates.
16. Implement net draw coverage covenant evaluation + draw freeze/termination actions.
17. Implement optional collateral toggle flows (post/release/seize + collateral-ratio checks).
18. Add delinquency/default/write-off logic.
19. Optional: implement module mirror bridge from native encumbrance state (must be non-canonical).
20. Add full test matrix:
   - unit
   - fuzz
   - invariant
   - cross-product accounting invariants
   - native encumbrance conservation + position transfer continuity
   - net draw coverage covenant enforcement (breach, cure, draw termination)
   - optional collateral toggle invariants (post/release/seize and no-forced-collateral paths)
   - compute adapter accounting parity across >=2 providers (including >=1 API-based inference adapter)
   - ACP terminal-state accounting synchronization
   - trust-mode gating + validation threshold enforcement
   - differential portability tests across >=2 ERC-8183 adapters

---

## 17) Success Criteria

- All four products run on one proposal + agreement framework.
- No duplicate financing spec branches remain active.
- Repayment and fee routing remain consistent across products.
- ACP-linked job terminal states synchronize correctly to agreement accounting.
- ERC-8004 identity resolution is deterministic (`agentWallet` then `ownerOf` fallback).
- Trust modes (`DiscoveryOnly/ReputationOnly/ValidationRequired/Hybrid`) gate transitions correctly.
- Native encumbrance is source of truth for financing balances and remains consistent across state transitions.
- Net draw coverage covenant is enforced deterministically (breach -> draw freeze -> cure or terminated draws/default).
- Optional collateral toggle works deterministically and does not become a mandatory dependency for agreement activation.
- Financing correctness does not depend on module registry pause/inactive states.
- ACP execution venue is hot-swappable via registry (`venueKey`) without core storage migration.
- Compute provider routing is hot-swappable via adapter policy without core storage migration.
- Differential tests show identical core accounting outcomes across at least two compute provider adapters (including an API-inference adapter such as Venice) and at least two ERC-8183 adapters.
- Accounting invariants hold under stress and default cases.

---

**End of Canonical Spec v1.10**
