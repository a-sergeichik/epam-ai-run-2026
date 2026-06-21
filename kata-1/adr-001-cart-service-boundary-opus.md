# 001: Carve Cart onto commercetools-Native Cart Behind a Façade, Leave Order Creation in the Monolith (For Now)

**Date:** 2026-06-21

**Status:** Draft

---

## Context

**Problem.** The regional .NET monolith owns four entangled capabilities: cart, pricing, promotion application, and order creation. Phase 1 requires unified cart + checkout across 22 regions. We must extract cart without a rip-and-replace, with zero downtime, while SAP ECC remains read-only inventory ground truth and 6 regional CRMs stay where they are. The question is **where to draw the service boundary** around "cart" given how tightly it couples to pricing, promotion, and order creation.

**Decision drivers.**
- Strangler-fig is mandated; we cannot create a second monolith (Tomás Reyes).
- Stores must keep selling throughout — no cutover that risks the buy flow (David Park).
- Target stack includes commercetools, whose native model *already* implements Cart, Price/PriceTier, CartDiscount/DiscountCode, and Order as first-class primitives.
- Internal product team is junior; less bespoke logic = better knowledge transfer and lower run risk (Lena Park).
- Regulatory surface (PCI-DSS L1, PSD2 SCA, local payment methods) lives mostly at checkout/payment, downstream of cart — but promotion and price correctness are revenue-critical and region-specific.

**Options considered.**

### Option A — Thin Cart service (cart state only); pricing, promotion, and order stay in the monolith
A new microservice owns only the cart aggregate (line items, quantities, cart-level metadata). It calls back into the monolith synchronously for every price and promotion calculation.

- **Pros:** Smallest possible first cut; lowest migration risk; pricing/promotion logic untouched so no behavioral regressions; fast to ship.
- **Cons:** The new service is anaemic — the hard logic stays in the monolith, so we haven't actually strangled anything meaningful. Creates a chatty synchronous dependency from new→old (latency, coupling, availability tied to the legacy). Effectively a distributed monolith; Tomás's "no new monolith" concern applies in reverse. Doesn't exploit commercetools at all.

### Option B — Cart + Pricing + Promotion as one bounded context (custom services); order creation stays in monolith
Carve cart together with pricing and promotion into a cohesive new context built as custom microservices, leaving only order creation behind in the monolith.

- **Pros:** Clean domain boundary — the things that change together move together; eliminates the chatty callback of Option A; a genuinely meaningful strangler slice.
- **Cons:** Re-implements pricing and promotion logic that commercetools largely provides out of the box — large custom build, exactly the bespoke complexity a junior team should avoid (Lena). High regression risk on revenue-critical, region-specific promo rules (Marco, Junichi). Slower to Phase-1 value; biggest blast radius for "keep selling" (David).

### Option C (recommended) — Adopt commercetools-native Cart/Price/Promotion behind an Apollo façade; order creation stays in the monolith via anti-corruption layer
Cart, price calculation, and promotion application move onto commercetools' native primitives. A thin Apollo GraphQL façade fronts it for clients. SAP inventory is synced read-only into commercetools (platform never writes to SAP). At checkout, an anti-corruption layer (ACL) hands the finalized cart to the **existing** monolith order-creation path, which stays authoritative until a later, separate ADR addresses Order.

- **Pros:** Uses platform capabilities instead of rebuilding them → least bespoke code, best fit for a junior team (Lena) and Tomás's "no new monolith." Cart/price/promo correctness leans on a supported product. Real strangler progress (three of four capabilities move) while the riskiest, most entangled piece — order creation, where PCI/PSD2/local-payment complexity concentrates — is deferred behind a clean seam. Order flow unchanged means lowest risk to "keep selling" (David). Inventory stays SAP-truth via one-way sync.
- **Cons:** Requires a price/promotion **parity-mapping** exercise from legacy rules to commercetools' model — some regional promo edge cases may not map 1:1 and need redesign or accepted divergence (Marco, Junichi). Introduces an ACL/seam between commercetools and the monolith order path that must be built and owned. Two pricing engines run in parallel during migration; we need shadow-comparison to prove parity before switching traffic.

---

## Decision

**Adopt Option C.** Move Cart, Pricing, and Promotion application onto commercetools' native primitives, exposed through the Apollo gateway, and keep order creation in the .NET monolith behind an anti-corruption layer until a dedicated Order ADR supersedes it.

**Rationale against the constraints:**
- **Strangler-fig / no new monolith (Tomás):** We replace legacy capabilities with platform-native ones rather than rebuilding a large custom service. The monolith is incrementally hollowed, not forked into a second one.
- **Keep selling / no downtime (David):** Order creation — the part wired to live payment and fulfillment — is untouched. We route via shadow-mode and per-region canary, so the legacy buy flow remains the fallback until parity is proven.
- **SAP as inventory truth:** Honored — commercetools receives a read-only inventory projection; it never writes back.
- **Junior team / knowledge transfer (Lena):** Configuration-and-mapping over bespoke pricing/promo engines minimizes the custom surface area the team must own.
- **Regulatory:** PCI-DSS L1 / PSD2 SCA / local methods concentrate at checkout-payment-order; deferring Order keeps that risk in the known, certified legacy path while we stabilize cart.

Options A and B are rejected: A produces a distributed monolith with a chatty new→old dependency and no real strangling; B incurs a large, high-regression custom rebuild of logic commercetools already provides.

---

## Consequences

**Expected outcomes.**
- Unified cart + pricing + promotion across regions on a single supported platform; Phase-1 checkout can compose cart (commercetools) + order (monolith) through the gateway.
- A clean, documented ACL seam at the cart→order handoff that becomes the extraction point for the future Order ADR.
- Kafka events emitted on cart state changes for downstream consumers; SAP inventory consumed as read-only sync.
- Reduced bespoke code and clearer ownership for the internal team.

**Drawbacks / things we accept.**
- A non-trivial promotion/price parity-mapping effort with possible regional edge-case divergence that must be explicitly signed off per region.
- Temporary dual pricing engines requiring shadow-comparison tooling and reconciliation before each region cuts over.
- The ACL is net-new code to build, test, and own; the cart↔order boundary is a known coupling we are deliberately deferring, not eliminating.
- Order creation remaining in the monolith means the strangler is incomplete after Phase 1 — by design, but it must not become permanent (track as explicit follow-up).

---

## Stakeholders

| Stakeholder | Role | Consulted on |
|---|---|---|
| Tomás Reyes | Lead Architect | Boundary shape; confirmed Option C avoids a second monolith and keeps a clean strangler seam |
| Eva Müller | VP Digital (Sponsor) | Phase-1 scope and sequencing; deferral of Order |
| David Park | Head of Retail Ops | Zero-downtime constraint; canary/shadow rollout keeping legacy buy flow as fallback |
| Sarah Chen | Head of CX | Cart/checkout experience continuity across regions |
| Lena Park | Internal Product Lead | Build complexity vs. junior-team capacity; preference for platform-native over bespoke |
| Marco Rossi | Regional GM, IT | Italian promo/local-payment edge cases (Postepay/Satispay); parity-mapping risk |
| Junichi Tanaka | Regional GM, JP | JP promo edge cases and local payment (PayPay); parity sign-off |

*Open items for review:* per-region promotion parity sign-off; agreement on shadow-comparison exit criteria before cutover.

---

## Links

- [Placeholder] Program strangler-fig migration strategy (CTO mandate)
- [Placeholder] SAP ECC → commercetools inventory read-only sync design
- [Placeholder] Promotion/price parity-mapping spec (legacy .NET → commercetools)
- [Placeholder] ADR 002: Order creation extraction (future / supersedes the deferral here)
- [Placeholder] Apollo gateway schema & ACL design for cart→order handoff
- [Placeholder] Phase-1 canary/shadow rollout plan (per-region)

---

*Draft for circulation — comments requested from listed stakeholders before moving to Accepted.*
