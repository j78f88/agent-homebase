---
applyTo: "{{paths.validation}}**"
---

# Commercial Validation Framework

Single source of truth for the 5-test commercial validation applied to **Feature-class** recommendations **before** they reach `@planner`. Owned by `@pm`. Runs in parallel with the product-fit validation framework — both must be on record before a feature handoff is accepted.

> **Why this exists:** product-fit tests guard against echo-chamber feature copying, but they do not verify whether a feature is commercially viable. This framework forces explicit checks on market size, pricing dynamics, acquisition economics, unit economics, and defensibility so that a technically sound feature is not shipped into a commercially incoherent position.

## When to apply

- Any Feature-class issue being handed off from `@pm` to `@planner`.
- Any roadmap item that affects monetisation, pricing tiers, or go-to-market channel mix.
- Any feature touching NDIS workflows, AT registration, or accessibility compliance obligations.

Routine bug fixes, UX polish, and internal-tooling changes **do not** need this framework.

## The 5 Tests

For each, record a **Verdict** (PASS / FAIL / N/A) and a one-or-two-sentence **Reasoning**. Do not auto-pass without reasoning.

1. **Market size and serviceable share.** Is the addressable market large enough to justify the investment? Estimate TAM, SAM, and what is realistically serviceable today given distribution constraints, geography, and NDIS participant eligibility.

2. **Pricing model fit and willingness to pay.** Does the proposed pricing model (subscription vs one-time) match how this segment buys? Check NDIS rate caps, the Maker tier one-time conversion event, and any evidence of willingness to pay at the intended price point.

3. **CAC and channel viability.** Can customers be acquired at a cost that produces a viable payback period? Evaluate organic SEO, paid acquisition, and NDIS coordinator referrals as distinct channels; call out which channel assumption the payback calculation relies on.

4. **LTV by tier and unit economics.** Does lifetime value exceed CAC with a healthy margin across all tiers (free, Maker one-time, Pro, NDIS Pro)? Account for voice-token economics against ElevenLabs cost and any per-seat or per-use variable costs.

5. **Defensibility and regulatory cost.** What prevents a well-funded competitor from replicating this feature quickly? Assess NDIS AT provider registration requirements, accessibility audit obligations, and payment compliance costs as barriers; note whether they are durable or merely time-delayed.

## Labels

Exactly one label per commercial validation record:

- **VALIDATED** — all five tests pass. Commercial record is cleared; feature may proceed to `@planner`.
- **REFRAMED** — one test fails but the failure is rescuable by restating the feature scope or pricing approach. Record both the original and the reframed version with the reason the reframe rescues it.
- **REJECTED** — one or more tests fail and the failure is not rescuable at this time. If the failure is structural, update `{{paths.non_goals}}`.
- **DEFERRED** — commercially viable in principle but blocked by missing data (e.g. NDIS pricing review outcome, ElevenLabs contract renegotiation). Add to `{{paths.roadmap}}` under "Parked" with the unblock condition.

## Output location

All commercial validation records go to `{{paths.validation}}<feature-slug>-commercial.md` using the template in the `/validate-commercial` prompt.

## Enforcement (for @reviewer)

- **CRITICAL:** Hand off to `@planner` from `@pm` on a Feature-class issue without a corresponding commercial validation record in `{{paths.validation}}`.
- **WARNING:** Commercial validation record where any test has a verdict but no reasoning sentence.
- **SUGGESTION:** Commercial validation record labelled REFRAMED without both the original and the reframed version present.
