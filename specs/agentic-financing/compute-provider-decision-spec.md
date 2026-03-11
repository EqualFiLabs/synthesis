# Compute Provider Decision Spec (No-Lock-In)

**Status:** Draft v0.2  
**Date:** 2026-03-10  
**Applies to:** `specs/agentic-financing/agentic-financing-spec.md`

---

## 1) Purpose

Select the best initial compute provider integration strategy for Agentic Financing while preserving hard constraints:

- **No lock-in**
- **Adapter-only coupling**
- **Native encumbrance remains source of truth**
- **Deterministic accounting across providers**

---

## 2) Product Surfaces Evaluated

1. **Financed dedicated compute capacity** (instance-backed, longer-lived)
2. **Financed burst inference execution** (job/serverless-backed)
3. **Financed managed API inference** (key-scoped credit-backed usage)

A single provider is unlikely to dominate all three surfaces.

---

## 3) Evaluation Criteria (Weighted)

| Criterion | Weight |
|---|---:|
| Lifecycle API control (provision, inspect, terminate/revoke) | 20 |
| Billing determinism (price/usage visibility for debt accounting) | 20 |
| Capacity reliability / availability | 15 |
| Burst/serverless semantics | 10 |
| Managed API inference ergonomics | 10 |
| Security controls (key scope, limits, expiry, revocation) | 10 |
| Operational maturity (error model, docs, rate limits) | 10 |
| Lock-in risk / portability | 5 |
| **Total** | **100** |

Scoring scale: **1 (poor) → 5 (excellent)**.

---

## 4) Provider Assessment Snapshot

## 4.1 Lambda (Lambda Labs)

Strengths:
- Strong instance lifecycle API (`launch`, `list`, `terminate`, etc.)
- Instance type data includes hourly pricing fields suitable for deterministic debt conversion
- Infra controls (SSH keys, filesystems, firewall rulesets)

Weaknesses:
- Managed inference API appears de-emphasized/winding down; strategy is instance-centric

Score profile:
- Best for: **financed dedicated compute capacity**

## 4.2 RunPod

Strengths:
- Strong serverless lifecycle (`/run`, `/runsync`, `/status`, `/cancel`, etc.)
- Built for bursty queue-based inference and autoscaling
- Broad GPU/serverless operational ergonomics

Weaknesses:
- Less ideal as sole long-lived financed instance rail vs dedicated instance-first providers

Score profile:
- Best for: **financed burst inference execution**

## 4.3 Venice

Strengths:
- OpenAI-compatible inference API with broad model catalog
- Key lifecycle endpoints (`create/update/delete`) support per-agreement scoped access
- Billing telemetry endpoints support deterministic metering and reconciliation
- Supports managed API inference without instance orchestration overhead

Weaknesses:
- API-credit inference model is a different surface than dedicated infrastructure ownership
- Requires strict normalization from provider billing rows to canonical unit schema

Score profile:
- Best for: **financed managed API inference**

## 4.4 Modal

Strengths:
- Excellent developer experience and serverless orchestration
- Strong GPU abstraction and operational tooling

Weaknesses:
- Higher platform coupling risk for protocol-level financing rails
- Better treated as orchestration/provider-of-last-mile than canonical financed capacity substrate

Score profile:
- Best for: optional orchestration layer, not first canonical provider rail

## 4.5 Vast

Strengths:
- Large marketplace supply and API-based provisioning

Weaknesses:
- Marketplace variability complicates underwriting predictability and policy controls

Score profile:
- Better as optional secondary/tertiary source after core rails are stable

---

## 5) Decision

### 5.1 Not a single-provider winner

There is no single provider that is best for all compute financing products.

### 5.2 Recommended strategy

- **Primary rail (Phase 1): Lambda adapter** for financed dedicated instances
- **Secondary rail (Phase 2): RunPod adapter** for financed burst/serverless inference
- **Tertiary rail (Phase 3): Venice adapter** for financed API-based inference
- Keep all rails behind one canonical compute adapter boundary

This satisfies no-lock-in while matching each provider to its strongest product surface.

---

## 6) Exact Integration Order

### Phase 0 — Common adapter contract (must come first)
Define canonical compute adapter interface and event schema:
- quote
- allocate/provision
- usage/metering pull
- release/terminate/revoke
- error normalization

### Phase 1 — LambdaComputeAdapter (primary)
Implement dedicated-instance financing path:
- deterministic debt accrual from instance pricing + measured usage windows
- hard policy bounds (max runtime, max spend, region/type allowlists)

### Phase 2 — RunPodComputeAdapter (secondary)
Implement burst/serverless financing path:
- queue-job funding windows
- sync/async settlement mapping
- bounded TTL and retry semantics into debt model

### Phase 3 — VeniceComputeAdapter (managed API inference)
Implement key-scoped inference financing path:
- per-agreement API key create/update/delete
- consumption limits + expiry tied to agreement policy
- billing/usage normalization into canonical unit types
- revoke/limit-clamp kill switch on delinquency/default

### Phase 4 — Router + policy scheduler
Route by agreement policy:
- `DedicatedCapacity` -> Lambda first
- `BurstInference` -> RunPod first
- `ApiCreditInference` -> Venice first
- fallback path only through approved adapter allowlist

---

## 7) No-Lock-In Acceptance Gates

Must pass before mainnet release:

- [ ] Native financing accounting remains correct when any provider adapter is disabled
- [ ] Differential tests show identical core debt/accounting outcomes across >=2 provider adapters for equivalent workloads
- [ ] At least one differential suite includes API-based inference adapter (Venice) vs non-Venice adapter trace replay
- [ ] Provider-specific metadata is never required to reconstruct canonical agreement state
- [ ] Provider swap (Lambda/RunPod/Venice) requires no storage migration in core contracts

---

## 8) Canonical Policy Defaults (v1)

- Default dedicated-capacity provider: **Lambda**
- Default burst-inference provider: **RunPod**
- Default API-credit inference provider: **Venice**
- Provider routing is governance-configurable and reversible
- Any provider-level fail-open behavior is forbidden; all failures must resolve to explicit agreement state transitions

---

## 9) Out of Scope (for this decision doc)

- Tokenomics changes
- New risk primitives beyond existing canonical financing states
- Exclusive commercial agreements with any compute provider

---

**End of Draft v0.2**
