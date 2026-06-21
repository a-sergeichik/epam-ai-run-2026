# Model Selection Note

**Date:** 2026-06-21
**Author:** Andrei Siarheichyk — Architecture Team
**Project:** Headless commerce consolidation (course case study — client details omitted per project policy)
**Task:** Draft a first-cut service-boundary ADR when carving a capability (here: Cart) out of a regional .NET monolith under a strangler-fig mandate.
**Committed location:** https://github.com/a-sergeichik/epam-ai-run-2026/blob/main/kata-1/model-selection-note.md

---

## Evaluation Criteria

| # | Criterion | Why it matters for this task |
|---|-----------|------------------------------|
| 1 | Business alignment | The boundary *is* the business decision; an ADR that contradicts a hard constraint (e.g. SAP read-only) isn't lower-quality, it's the wrong system — and the error propagates to every downstream ADR and ticket. |
| 2 | Boundary correctness | The first carve sets the precedent five squads copy; single write-ownership, no dual-write, and strangler-reversibility are what keep 1,400 stores selling. |
| 3 | Trade-off completeness | Junior squads and skeptical regional GMs act on this record without re-deriving it, so honest pros/cons — especially the chosen option's downsides — are what make it trustworthy. |
| 4 | Structure adherence + usability | The ADR circulates across three SIs and non-technical stakeholders and is consumed by a pipeline, so consistent sections + a stakeholder→concern mapping make it auditable and machine-parseable. |

---

## Prompt Used

> You are on the architecture team for a multi-category retailer ($8.2B revenue, 1,400 stores, 22 countries) consolidating 22 regional e-commerce stacks onto a single headless commerce platform.
>
> CONSTRAINTS (ground truth — do not contradict or invent beyond these):
> - Strangler-fig pattern is MANDATED by the CTO. No rip-and-replace.
> - No acceptable downtime window — stores must keep selling throughout.
> - Must coexist with legacy SAP ECC = inventory ground truth (read-only sync to the platform; the platform does NOT write to SAP).
> - 6 regional CRMs will NOT be migrated in this program.
> - Target stack: commercetools (headless commerce), Apollo GraphQL gateway, microservices on AWS EKS, Kafka for eventing, Auth0 for identity.
> - Phase 1 (in build now): unified identity + cart + checkout.
> - Regulatory: GDPR, CCPA, PCI-DSS L1, PSD2 SCA (EU), local payment methods (Postepay/Satispay IT, PayPay JP, Klarna Nordics).
>
> TASK:
> Write a first-draft Architecture Decision Record for the SERVICE BOUNDARY decision when carving the CART capability out of the regional .NET monolith (which today owns cart, pricing, promotion application, and order creation). Present at least three boundary options with genuine pros AND cons for each, recommend one, and justify it against the constraints above.
>
> REQUIRED ADR STRUCTURE (use these exact sections):
> - Title (format: "001: <decision summary>")
> - Date
> - Status (Draft/Accepted/Superseded)
> - Context (problem, decision drivers, options considered)
> - Decision (the choice + rationale)
> - Consequences (expected outcomes AND drawbacks)
> - Stakeholders (who was consulted and on what)
> - Links (related/supporting docs — placeholders allowed)
>
> Relevant stakeholders: Lead Architect (wants clean strangler-fig, no new monolith), VP Digital (sponsor), Head of Retail Ops (stores must keep selling), Head of CX, Internal Product Lead (junior team, knowledge transfer), regional GMs for Italy and Japan.
>
> Be concise enough to circulate. Do not invent constraints, stakeholders, or facts not given above.

*Identical input pasted into both models, no edits between runs.*

---

## Output Comparison

### Model A: Haiku 4.5

> ### Option 1: Strangler Façade with Gradual Logic Migration (RECOMMENDED)
> **Boundary:** Cart service owns shopping-cart state and orchestration. Pricing/promotion logic initially stays in legacy; cart service calls legacy as a policy engine via synchronous RPC.
> - Order completion syncs bidirectionally: new service → legacy DB → SAP (for store consistency)
> ...
> ✗ Network latency: every cart operation calls legacy synchronously

### Model B: Opus 4.8

> ### Option C (recommended) — Adopt commercetools-native Cart/Price/Promotion behind an Apollo façade; order creation stays in the monolith via anti-corruption layer
> ... SAP inventory is synced read-only into commercetools (platform never writes to SAP). At checkout, an anti-corruption layer (ACL) hands the finalized cart to the existing monolith order-creation path, which stays authoritative until a later, separate ADR addresses Order.
> - **Cons:** Requires a price/promotion parity-mapping exercise... Introduces an ACL/seam... Two pricing engines run in parallel during migration; we need shadow-comparison to prove parity before switching traffic.

---

## Scorecard

| Criterion | Model A score (1–3) | Model A evidence | Model B score (1–3) | Model B evidence |
|-----------|---------------------|------------------|---------------------|------------------|
| 1. Business alignment | 2 | Recommended option syncs order completion to SAP — violates SAP read-only ground truth. | 3 | States "it never writes back"; localizes PCI/PSD2 to checkout/order and reasons from constraints. |
| 2. Boundary correctness | 2 | Reversible strangler, but its own recommendation does a bidirectional dual-write — the anti-pattern it rejects in Option 2. | 3 | Single write-ownership, one-way SAP sync, no dual-write (shadow-compare before cutover), per-region canary. |
| 3. Trade-off completeness | 3 | Genuine pros/cons per option; recommended option carries 4 honest cons + mitigations table. | 3 | Genuine pros/cons; flags the sharpest risk — "strangler is incomplete… must not become permanent." |
| 4. Structure + usability | 2 | Stakeholders are a flat list, not mapped to "consulted on"; title-format drift; extra non-spec sections. | 3 | Exact title format; Stakeholders→"Consulted on" table; all 8 sections; concise. |
| **Total** | **9** | | **12** | |

---

## Decision

**Selected model:** Opus 4.8

**Rationale:** Opus wins 12–9. Haiku's decisive shortcoming was a **factual-grounding failure inside its recommended option** — it had order completion write to SAP (a bidirectional dual-write), contradicting the SAP read-only constraint while presenting it confidently as strangler-compliant. For a downstream pipeline where the ADR is committed and feeds the next ADR/tickets without human review, that silent error is disqualifying — so Opus is the default for this task.

---

## Active Constraint

**What could change this decision within 30 days:** If a per-ADR token/cost cap is imposed, Haiku becomes the right call for any variant of this task that has a human reviewer in the loop before anything acts on the output.

---

## Revision history

| Version | Date | Change |
|---------|------|--------|
| 1.0 | 2026-06-21 | Initial commit |
