# /validate-commercial \<feature\>

Run the commercial 5-test framework against a proposed or VALIDATED feature. Produces a `<slug>-commercial.md` artifact in `{{paths.validation}}`. Required before any `@pm → @planner` handoff for VALIDATED features.

---

## When to invoke

- After `/validate-feature` returns **VALIDATED** or **REFRAMED** and a `@planner` handoff is approaching
- When evaluating whether a feature is commercially viable regardless of product-fit status
- When `@reviewer` flags a missing commercial validation as a CRITICAL finding

Do **not** invoke on REJECTED features — if the product-fit validation already rejects the feature, commercial validation is unnecessary.

---

## Step-by-step workflow

### 1. Identify the feature slug

Derive `<slug>` from the feature name (lowercase, hyphenated). This slug must match the product-fit validation file (`<slug>-validation.md`) if one exists.

### 2. Confirm scope with the user

Before running, confirm:
- The exact feature name and slug
- Whether a product-fit validation already exists (check `{{paths.validation}}<slug>-validation.md`)
- Any known pricing tiers, target markets, or revenue assumptions to factor in

Use `#tool:askQuestions` if any of the above is unclear.

### 3. Run the 5 commercial tests

For each test, record a **Verdict** (PASS / FAIL / N/A) and one-or-two-sentence **Reasoning**. Do not auto-pass without reasoning.

#### Test 1 — Market size and serviceable share

Is the addressable market large enough to justify building and maintaining this feature? Estimate the serviceable addressable market (SAM) for the specific user segment this feature targets. A feature that appeals to 2 % of a niche market may be valid; one that targets a broad market but has no realistic acquisition path is not.

Fail if: the target segment is too small to recoup development cost at realistic conversion rates, or if there is no credible path to reaching that segment.

#### Test 2 — Pricing model fit and willingness to pay

Does this feature belong in the current pricing tier(s), or does it require a new tier? Is there evidence that the target user segment is willing to pay for this specific value? Consider whether the feature is a differentiator that increases conversion/upgrade, or table-stakes that must be free to stay competitive.

Fail if: the feature would need a new pricing tier with no evidence of willingness to pay, or if it belongs in a free tier where it creates value for users but no revenue path for the product.

#### Test 3 — CAC and channel viability

Can this feature be used as an acquisition driver, or does it serve only existing users? If it can drive acquisition, is there a realistic channel (SEO, word-of-mouth, B2B sales, app store optimisation) where it would be discoverable? If it serves only existing users, does it reduce churn meaningfully enough to justify the CAC indirectly?

Fail if: the feature is invisible to prospective users, does not reduce churn, and offers no acquisition leverage.

#### Test 4 — LTV by tier and unit economics

Does building this feature improve lifetime value (LTV) for the tiers it targets? Consider whether it increases upgrade rates, reduces cancellation, or enables upsell. Weigh estimated implementation cost against realistic LTV improvement across the likely user cohort.

Fail if: the projected LTV lift across the affected cohort does not cover implementation cost within a reasonable payback period (typically 12–18 months for a feature of this scope).

#### Test 5 — Defensibility and regulatory cost

Does this feature create a durable competitive advantage, or is it easily replicated by competitors? Are there compliance, legal, or data-handling costs (GDPR, accessibility, platform policy) that significantly raise the true cost of ownership?

Fail if: competitors can ship an equivalent in one sprint, or if regulatory/compliance overhead materially erodes the unit economics calculated in Test 4.

---

### 4. Determine the verdict

Apply exactly one verdict:

| Verdict | Criteria |
|---------|----------|
| **VALIDATED** | All five tests pass. Feature is commercially viable. Cleared to proceed to `@planner` handoff (assuming product-fit validation also exists). |
| **DEFERRED** | Commercially viable in principle but blocked by a specific, nameable condition (e.g., "needs pricing tier restructure", "requires market to mature"). Add to `{{paths.roadmap}}` under "Parked" with the unblock condition. |
| **REJECTED** | One or more tests fail and the failure is not rescuable. If it is a standing no for commercial reasons, note it but do not add to `{{paths.non_goals}}` (product non-goals belong to product-fit validation). |

> Note: there is no REFRAMED verdict in the commercial framework. If a reframe would rescue the feature commercially, record it as VALIDATED with a note explaining the reframe required — and ensure the product-fit validation record is updated to match.

---

### 5. Write the artifact

**Filesystem mode** — save to `{{paths.validation}}<slug>-commercial.md`:

```markdown
---
feature: <slug>
verdict: <VALIDATED | DEFERRED | REJECTED>
date: YYYY-MM-DD
framework: commercial (5-test)
---

# Commercial Validation — <slug>

**Verdict:** <VALIDATED | DEFERRED | REJECTED>
**Date:** YYYY-MM-DD
**Framework:** commercial (5-test)

## Test results

1. **Market size and serviceable share** — Verdict: <PASS|FAIL|N/A>. <reasoning>
2. **Pricing model fit and willingness to pay** — Verdict: <PASS|FAIL|N/A>. <reasoning>
3. **CAC and channel viability** — Verdict: <PASS|FAIL|N/A>. <reasoning>
4. **LTV by tier and unit economics** — Verdict: <PASS|FAIL|N/A>. <reasoning>
5. **Defensibility and regulatory cost** — Verdict: <PASS|FAIL|N/A>. <reasoning>

## If DEFERRED
**Unblock condition:** <what must be true before revisiting>

## Sources / inputs
- <research links, competitor pricing data, market reports>
```

**Tracker mode** (when `{{tracker.type}}` != `"filesystem"`) — post as a Linear comment per `tracker-adapter-validations.instructions.md` § "Write a commercial validation record".

---

### 6. Post-validation actions

| Verdict | Action |
|---------|--------|
| VALIDATED | Confirm product-fit validation also exists. If both exist, show the `@planner` handoff button. |
| DEFERRED | Note the unblock condition on `{{paths.roadmap}}` under "Parked". Do not show handoff button. |
| REJECTED | Record rejection. Inform user; no handoff. |

Always end with "what do you think?" or a concrete choice for the user to make. Never make the handoff decision for the user.

---

## Subagent mode

When invoked with `[SUBAGENT-MODE] [WRITE:COMMERCIAL-VALIDATION]`:

1. Skip session lifecycle (no scope gate, no roadmap read, no askQuestions)
2. Execute tests 1–5 using any context provided in the prompt
3. Write `{{paths.validation}}<slug>-commercial.md` using the template above
4. Return Tier 2 JSON:

```json
{
  "tier": 2,
  "agent": "pm",
  "status": "complete",
  "summary": "<verdict and one-sentence rationale>",
  "artifactPath": "docs/planning/validation/<slug>-commercial.md",
  "artifactType": "validation",
  "findings": [],
  "flaggedDecisions": []
}
```
