# Legacy Kotlin App Redesign Guide

> AI-driven UC spec extraction → incremental redesign pipeline
> Constraints: business logic is also up for revision (to-be, not just as-is) · the service is live in production and must be replaced incrementally · the only source of truth is the code itself (no spec docs, no PO to ask) · a solo effort (with AI assistance)

---

## 1. Why This Guide Exists

- **Ad-hoc implementation**: responsibilities are tangled together, and handling is inconsistent from case to case
- **Tests exist, but…**: coverage is low, and some tests simply pin existing bugs as if they were correct behavior
- **No source of truth**: there's no spec document and no PO to ask — intent has to be inferred from the code alone

## 2. The Core Risk: There Is No External Oracle

```
Code (the only evidence that exists) → a single AI interpretation (plausible, but unverified) → ⚠ confirmation bias (a bug hardens into "the spec")
```

With no spec document and no one to ask, the moment you accept an AI's interpretation at face value, you're exposed to confirmation bias.
That's why this guide puts **cross-checking any single AI judgment from multiple angles, and logging the reasoning behind every call (a decision log), at the center of the pipeline** — rather than trusting one pass.

## 3. Three Core Principles

1. **Cross-verification** — never settle a judgment on a single AI pass; re-examine it from different angles
2. **Decision log** — record the reasoning behind each call as you go, so it retroactively becomes the requirements document that never existed
3. **Incremental rollout** — replace module by module, behind feature flags, so anything can be rolled back instantly

## 4. The Pipeline at a Glance

| Phase | Name | One-line summary |
|---|---|---|
| 0 | Inventory & Prioritization | Map the modules, pick a pilot using git churn |
| 1 | Behavior Extraction (as-is) | AI reads code + tests + git history, produces a behavior catalog with confidence tags |
| 2 | Cross-Verification | Re-review flagged items from multiple angles, record the decision log |
| 3 | UC Spec Finalization | Only confirmed items get promoted to formal use-case docs |
| 4 | Architecture Redesign | Design the target structure against the finalized spec |
| 5 | Safety-Net Hardening | Add characterization tests, retire tests that were pinning bugs |
| 6 | Cutover & Monitoring | Roll out gradually via feature flag + shadowing, with instant rollback |
| 7 | Repeat | Carry the lessons from this pilot into the next module |

> Validate the whole process on one pilot module first, then repeat it across the rest.

---

## 5. Worked Example: Order Discount Calculator

This is the example used throughout the guide to make the pipeline concrete. (Not real code — a reconstruction of a typical legacy situation.)

**`order/OrderDiscountCalculator.kt`**
Coupon discounts, tier discounts, and promotional discounts are all tangled together in a single class, inside a chain of if-else statements.
As ownership passed between different developers over time, "just make it work" patches accumulated throughout — a textbook legacy module.

| Metric | Value |
|---|---|
| Change frequency within the package (git churn) | #1 |
| Existing test cases | 12 |
| Test coverage | 60% |

---

### Phase 0 · Inventory & Prioritization

**Method**
- Map the codebase by domain/module
- Surface hot spots and pain points using git log change frequency (churn)
- Get a rough read on test coverage per module
- Pick one pilot module — not too large, not too central

**Applied to the example**
Ranking the `order` package by churn showed `OrderDiscountCalculator.kt` as the most frequently modified file over the last 6 months. It's also a reasonable size (~480 lines) and has fewer dependencies than other modules, making it a good pilot candidate.

### Phase 1 · Behavior Extraction (as-is catalog)

AI reads the code, existing tests, and git history together, and produces a list of "what this actually does right now" — each item tagged with a confidence level.

| ID | Observed behavior | Confidence |
|---|---|---|
| A | VIP-tier customers get an additional 5% discount | Clear |
| B | When a coupon and a promotion apply simultaneously, only the promotion is applied and the coupon is ignored | Suspicious |
| C | If the discounted amount would go negative, it's clamped to 0 | Clear |
| D | Discount rate calculation truncates at the third decimal place instead of rounding | Suspicious |

### Phase 2 · Cross-Verification — Reviewing Behavior B

| Angle | Finding |
|---|---|
| Business consistency | Blocking coupon + promotion from stacking is inconsistent with the general policy |
| Code history | Commit message: "temporarily prioritize promotion (workaround for coupon bug)" |
| Similar-case comparison | The cart preview screen does allow coupon + promotion to stack |

**Verdict: confirmed bug → scheduled for a fix**
Decision log entry: *"All three angles agree — the coupon-exclusion behavior is a bug; stacking is the correct policy."*

### Phase 3 · UC Spec Finalization

**UC-014 — Simultaneous application of coupon and promotion discounts**

- **Given** the customer holds a valid coupon and has a promotion-eligible item in the order
- **When** the order is checked out
- **Then** both the coupon discount and the promotion discount are applied and combined (the combined discount rate is capped at 30%)

### Phase 4 · Architecture Redesign

**Before** — a single `DiscountCalculator` class
```kotlin
if (vip) { ... }
if (coupon) { ... }
if (promotion) { ... }
// conditions tangled together, order-dependent
```

**After** — split via the Strategy pattern
```
DiscountPolicy (composes the strategies)
├── CouponDiscountStrategy
├── GradeDiscountStrategy
└── PromotionDiscountStrategy
```
The existing entry point (`calculateDiscount`) is preserved — callers don't change, only the internals are swapped.

### Phase 5 · Safety-Net Hardening

| Category | Count | Detail |
|---|---|---|
| Existing tests kept as-is | 8 | Characterization tests for behavior that isn't changing |
| Expected values updated | 2 | Replacing tests that had been pinning the bug as "correct" |
| New tests added | 3 | Edge cases like coupon + promotion + VIP applied together |

### Phase 6 · Cutover & Monitoring

```
Feature flag ON (10% shadow traffic) → log diffs between old and new results → monitor for 1 week → cut over to 100% → remove old code
```

If anything goes wrong, flipping the flag off rolls back to the previous behavior instantly.

### Phase 7 · Repeat — On to the Next Module

**What this pilot produced**
- Three review angles that proved useful for cross-verification
- A decision log template
- A draft AI prompt for behavior extraction
- A feature-flag / shadow-deployment checklist

**Next pilot: shipping fee calculator**
Ranked #2 by churn in the Phase 0 priority list. With the refined process, this round should move faster.

---

## 6. All-at-Once vs. This Pipeline

| Aspect | All-at-once approach | This pipeline |
|---|---|---|
| Risk of misjudgment | Trusts a single AI pass outright | Multi-angle cross-verification + decision log |
| Operational risk | One big deployment, hard to roll back | Module-by-module rollout, instant rollback |
| Regression safety net | Trusts existing tests as-is | Coverage hardening + retiring bug-pinning tests |
| Missing documentation | Demands a perfect spec up front | The decision log generates the requirements retroactively |
| Burden of working solo | Have to handle everything at once | Validate on a pilot, then repeat and refine |

## 7. Adoption Checklist

- [ ] Pick the pilot from the most frequently changed modules (by git churn)
- [ ] Always tag each item in the behavior catalog with a confidence level (clear / suspicious)
- [ ] Cross-check suspicious items from 2–3 different angles
- [ ] For hard-to-reverse decisions (money, auth, data deletion), write out the reasoning yourself in full sentences
- [ ] Lock down behavior you're not changing with characterization tests first
- [ ] Deploy behind a feature flag + shadowing, so you can always roll back

---

## Next Step

Pick one pilot module and start at Phase 0.
Validate small, keep a decision log, and deploy so you can always roll back — those three things are enough.
