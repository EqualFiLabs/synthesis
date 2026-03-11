# Synthesis — Agentic Financing Specs

This repository contains the canonical specification documents for EqualFi’s **agentic financing** architecture.

The goal is simple: define financing rails for autonomous agents that are deterministic, auditable, and portable across execution venues.

---

## What this repo contains

```text
specs/
  agentic-financing/
    agentic-financing-spec.md
    virtuals-adapter-spec.md
    compute-provider-decision-spec.md
    compute-orchestration-spec.md
    mailbox-sdk-spec.md
    venice-adapter-spec.md
```

### 1) `agentic-financing-spec.md`
Canonical framework for:
- Solo Agentic Financing
- Pooled Agentic Financing
- Solo Compute/Inference Lending
- Pooled Compute/Inference Lending

It defines shared primitives, state machines, accounting, risk transitions, ERC-8183 interoperability, and ERC-8004 trust/identity integration.

### 2) `virtuals-adapter-spec.md`
No-lock-in adapter profile for routing ERC-8183 job execution through Virtuals-compatible ACP venues.

This adapter spec is intentionally written so Virtuals can be integrated **without becoming a required dependency** of core protocol logic.

### 3) `compute-provider-decision-spec.md`
Provider-selection and sequencing spec for compute financing adapters.

It defines weighted evaluation criteria and the current no-lock-in rollout order:
- Lambda-first for dedicated-capacity financing
- RunPod-first for burst/serverless financing
- Venice adapter for managed API-based inference financing

### 4) `venice-adapter-spec.md`
No-lock-in adapter profile for integrating Venice as a managed inference compute rail.

Defines per-agreement API key lifecycle, mailbox credential handoff, billing-to-unit normalization,
and kill-switch revocation behavior under delinquency/default transitions.

---

## Core design principles

- **No lock-in:** venue integrations are adapter-based and hot-swappable.
- **Standards-first:** ERC-8183 (agent commerce) + ERC-8004 (agent identity/trust).
- **Deterministic accounting:** explicit terminal-state synchronization and auditable transitions.
- **Separation of concerns:** financing core remains independent of UX/application-layer systems.

---

## Intended audience

- Protocol engineers
- Smart contract auditors
- Integrators building agent commerce or compute-financing flows
- Partners evaluating adapter-based interoperability

---

## Status

These are active specification drafts and may evolve as implementation and testing progress.

When updating specs, preserve canonical invariants:
- portable accounting,
- no-lock-in adapter boundaries,
- deterministic risk + settlement behavior.

---

## License

TBD

---

**Author:** Eve (`agent-eve`)  
**Maintainer context:** EqualFi Labs
