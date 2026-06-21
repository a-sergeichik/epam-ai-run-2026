# ADR-001: Cart Service Boundary Strategy During Strangler-Fig Migration

**Date:** 2026-06-21  
**Status:** Draft  
**Stakeholders:** Tomás Reyes (Lead Architect), Eva Müller (VP Digital), David Park (Head of Retail Ops), Sarah Chen (Head of CX), Lena Park (Internal Product Lead), regional GMs (Marco Rossi IT, Junichi Tanaka JP)

---

## Context

The legacy regional .NET monoliths own cart, pricing, promotion application, and order creation as a single cohesive domain. We are extracting cart to become a commercetools-backed microservice as part of Phase 1 (unified identity + cart + checkout).

**Decision drivers:**
- CTO mandate: strangler-fig pattern (no rip-and-replace, coexistence required)
- Zero acceptable downtime: 1,400 stores must keep selling throughout transition
- Pricing/promotion logic is complex; regional teams hold deep knowledge
- SAP ECC remains inventory ground truth (read-only sync only)
- Phase 1 focuses on cart; pricing optimization is later-phase work
- Lena Park's junior team has capacity constraints; knowledge transfer must be staged

**The core question:** How do we carve cart out of the monolith without forcing immediate extraction of pricing/promotion logic, while maintaining strangler-fig integrity?

---

## Options Considered

### Option 1: Strangler Façade with Gradual Logic Migration (RECOMMENDED)

**Boundary:** Cart service owns shopping-cart state and orchestration. Pricing/promotion logic initially stays in legacy; cart service calls legacy as a policy engine via synchronous RPC (REST/gRPC).

**Architecture:**
- New cart microservice (commercetools-backed) deployed in parallel to legacy cart endpoints
- API gateway routes new traffic → new service; legacy POS traffic → legacy (for duration of transition)
- Cart service calls legacy pricing/promotion engine synchronously
- Order completion syncs bidirectionally: new service → legacy DB → SAP (for store consistency)
- Once new pricing/promotion service is proven, migrate that logic and sunset synchronous calls

**Pros:**
- ✓ True strangler fig: legacy continues operating unchanged during transition; new service gradually takes traffic
- ✓ Zero downtime: store POS can remain on legacy path indefinitely until cutover is proven safe
- ✓ Staged knowledge transfer: pricing team can move incrementally; Lena's team focuses on cart first
- ✓ Clear rollback: if new cart fails, stores revert to legacy with no data loss
- ✓ Validates cart domain in isolation before tackling pricing complexity
- ✓ Operational simplicity during transition: no dual-write sync complexity

**Cons:**
- ✗ Extended transition period: pricing will be decoupled later, not now
- ✗ Network latency: every cart operation calls legacy synchronously (mitigation: cache pricing rules, batch calls)
- ✗ Dual maintenance: cart logic exists in two places temporarily
- ✗ Regional teams support two paths during overlap (training/tooling burden)

---

### Option 2: Full Service Extraction with Event-Driven Sync

**Boundary:** Cart service extracts completely (cart + pricing + promotion logic); legacy pricing data syncs via Kafka events. New service writes to both commercetools and legacy DB during transition.

**Architecture:**
- Extract cart, pricing, promotion as single microservice immediately
- Dual writes: new service commits to commercetools AND legacy DB (via event sink)
- Legacy POS continues reading from legacy DB (unaware of new service)
- Events from legacy (inventory updates, rule changes) replicate to new service via Kafka
- At cutover, switch POS to new service and deprecate legacy

**Pros:**
- ✓ Faster path to full platform decoupling
- ✓ Smaller legacy monolith sooner
- ✓ Single source of truth for cart/pricing/promotion earlier
- ✓ No synchronous RPC dependency on legacy during normal operation

**Cons:**
- ✗ **Does not follow CTO's strangler-fig mandate:** full extraction is a rip-and-replace pattern in miniature
- ✗ **High operational risk:** dual writes create data consistency exposure; if sync fails, stores may see stale pricing
- ✗ Requires complex event ingestion and reconciliation logic (Kafka consumer lag, idempotence)
- ✗ Accelerates knowledge transfer demand: pricing team must move at same pace as cart (high cognitive load on Lena's junior team)
- ✗ Rollback is costly: reverting cart + pricing back to legacy requires dual-write reversal and data reconciliation

---

### Option 3: Functional Domain Split (Minimal Boundary)

**Boundary:** Cart service owns only shopping-cart state machine (add/remove items, serialization). Pricing/promotion stay in legacy. Cart does not cache; it calls legacy pricing for every operation.

**Architecture:**
- New cart microservice is minimal: item list, quantity tracking, session state
- Every price calculation, discount application, cart validation calls legacy synchronously
- Legacy remains the authoritative engine for all commerce rules
- Order creation still happens in legacy; cart service passes validated cart to legacy for conversion

**Pros:**
- ✓ Smallest service extraction; lowest risk of new code bugs
- ✓ Legacy pricing logic unchanged; no knowledge transfer for pricing team yet
- ✓ Clear domain boundary (state ≠ rules)
- ✓ Easy to rollback: if new cart service fails, stores use legacy cart entirely

**Cons:**
- ✗ Extreme latency and chattiness: cart is now unusable for stores (every operation requires legacy RPC)
- ✗ Tight coupling remains: cart cannot be deployed without legacy pricing availability
- ✗ Does not advance Phase 1 goal of decoupling cart from legacy pricing
- ✗ Creates false microservice: cart is just a thin state layer; no independent value
- ✗ Harder to optimize later (promotions engine is permanently legacy-dependent)

---

## Decision

**We adopt Option 1: Strangler Façade with Gradual Logic Migration.**

### Rationale

1. **Mandated pattern:** CTO requires strangler fig. Option 2 violates this directly. Option 1 is the only true strangler approach.

2. **Zero-downtime requirement:** Option 1 allows regional teams (esp. David Park's Retail Ops) to keep legacy cart running indefinitely until new cart is proven in production. Options 2 and 3 force faster cutover with higher operational risk.

3. **Staged knowledge transfer:** Lena's junior team extracts cart first (well-scoped, testable independently). Pricing/promotion logic follows once cart team is confident. Option 2 forces both simultaneously.

4. **Pricing complexity:** Pricing/promotion rules are regional, promotion-driven, and fragile (store marketing runs promotions weekly). Keeping this in legacy initially reduces blast radius while new platform stabilizes.

5. **Rollback safety:** If new cart fails at scale, Option 1 lets stores revert to legacy cart with no data loss. Options 2 and 3 require reconciliation.

---

## Implementation Guidelines

- **Phase 1a (Cart Extraction, T+0 to T+3m):**  
  Deploy new cart service (commercetools-backed) in parallel. Route new checkout flows to it; legacy POS remains on old cart. New cart calls legacy pricing/promotion RPC synchronously.

- **Phase 1b (Hardening & Rollback Drills, T+3m to T+6m):**  
  Monitor new cart in production. Run regular failover drills: switch live traffic back to legacy. Build observability around legacy RPC latency (cache rules to <100ms P99).

- **Phase 2 (Pricing Extraction, T+6m+):**  
  Once cart is stable, extract pricing/promotion logic. Migrate regional rules to new service. Sunset legacy RPC dependency.

---

## Consequences

### Expected Outcomes
- Cart is decoupled from monolith faster than pricing/promotion
- Strangler fig remains intact: new service coexists with legacy, no forced cutover
- Phase 1 delivers cart + checkout on commercetools; pricing/promotion follows in Phase 2
- Regional GMs (Marco, Junichi) can keep stores selling throughout transition
- Lena's team gains confidence with cart before tackling pricing domain

### Drawbacks & Mitigations
| Drawback | Mitigation |
|----------|-----------|
| Extended transition (pricing still in legacy) | Pricing extraction is Phase 2; acceptable delay given CTO mandate for strangler fig |
| Synchronous RPC latency (cart → legacy pricing) | Cache pricing rules locally; batch price lookups; monitor legacy response time |
| Dual maintenance during overlap | Document cart logic once in new service; legacy pricing is read-only from cart's perspective |
| Knowledge transfer sequencing | Schedule pricing team training AFTER cart stabilizes (T+6m); reduces parallel load |

---

## Links & References

- **Related:** ADR-002 (Pricing/Promotion Service Boundary — future)
- **Related:** ADR-003 (Order Creation & SAP Sync — future)
- **Architecture Diagram:** [Strangler Fig Phase 1 Topology — Confluence TBD]
- **Commercetools Integration Plan:** [TBD]
- **Legacy Cart Specification:** [Regional Monolith Cart Module — Confluence TBD]

---

## Approval & Sign-Off

| Role | Name | Status |
|------|------|--------|
| Lead Architect | Tomás Reyes | Pending |
| VP Digital (Sponsor) | Eva Müller | Pending |
| Head of Retail Ops | David Park | Pending |
| Internal Product Lead | Lena Park | Pending |
