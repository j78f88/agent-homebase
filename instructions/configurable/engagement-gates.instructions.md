---
description: "Use when running engagement gates, checking scope upgrade thresholds, building subagent prompts for validation or planning, or determining deployment workflow details. Defines Gate 1-5 criteria, scope upgrade rules, and platform configuration."
applyTo: ".github/agents/delivery-lead.agent.md"
---

# Engagement Gates

Gate definitions, subagent prompt templates, scope upgrade thresholds, and platform configuration for the @delivery-lead engagement lifecycle. Five gates total: Requirements Validation, Plan Approval, Commercial Readiness, Test Deployment, Production.

## Gate Definitions

### Gate 1 — Requirements Validation

**Trigger:** After validation subagent completes (M/L size). Skipped for S-size.

**What to check:**
- Validation subagent verdict (PASS / CONDITIONAL / FAIL)
- Conflicts with {{paths.non_goals}}, {{paths.roadmap}}, {{paths.decisions}}
- Risk assessment

**Present to CTO:**
- Verdict with rationale
- Identified risks and conflicts
- Recommendation

**Decision options:** Approve / Revise brief / Reject / Override

### Gate 2 — Plan Approval

**Trigger:** After planning subagent produces draft PLAN.md.

**What to check:**
- Task list completeness and clarity
- Quality gates defined
- Acceptance criteria mapped to brief
- Compliance with planning-preflight checks

**Present to CTO:**
- Full draft plan in conversation
- Planning subagent's risk assessment
- Compliance issues (if any)

**Decision options:** Approve / Revise / Reject / Split into smaller sprints

### Gate 3 — Commercial Readiness

**Trigger:** After @sprint-lead completes sprint execution. Re-runs the commercial 5-test against actual built scope — not the proposed scope from intake.

**What to check:**
- Scope delta: what was actually built vs. what was proposed at intake
- Commercial 5-test verdicts against the as-built feature set:
  1. Market size and serviceable share — still valid for what shipped?
  2. Pricing model fit and willingness to pay — does the implementation match the monetisation assumption?
  3. CAC and channel viability — acquisition cost unchanged by implementation choices?
  4. LTV by tier and unit economics — unit economics hold for what was built?
  5. Defensibility and regulatory cost — new surface area changes the risk profile?
- Any scope additions or removals that occurred during execution

**Present to CTO:**
- Scope delta summary (proposed vs. built)
- Commercial 5-test re-run verdict (VALIDATED / CONDITIONAL / REJECTED) with per-test reasoning
- Recommendation: proceed to test deployment or revise

**Decision options:** Approve for test deployment / Revise scope / Abort

### Gate 4 — Test Deployment

**Trigger:** After Gate 3 approval and test deployment workflow finishes.

**What to check:**
- Test deployment workflow status (via `gh` CLI)
- Automated test results from workflow summary
- Test environment URL accessibility

**Present to CTO:**
- Deployment status (success/failure)
- Automated test pass/fail count
- Test environment URL for manual verification

**Decision options:** Approve for production / Request fixes / Abort

### Gate 5 — Production

**Trigger:** After merge to {{git.main_branch}} and production deployment workflow finishes.

**What to check:**
- Production deployment workflow status
- Automated smoke test results
- Production URL accessibility

**Present to CTO:**
- Deployment status
- Smoke test results
- Production URL

**Decision options:** Confirm / Rollback / Hold

---

## Scope Upgrade Thresholds

These thresholds trigger a scope upgrade prompt when an S-size engagement's planning subagent returns values exceeding them. @delivery-lead reads these values at runtime — tune by editing this section, zero agent changes needed.

```
taskCount > {{scope_upgrade.task_count}}
filesAffected > {{scope_upgrade.files_affected}}
multiStore: true    # touches >1 store factory
multiArea: true     # touches >2 top-level directories (apps/, packages/, docs/)
```

When triggered, @delivery-lead presents: "This looks bigger than S-size. Upgrade to M (adds requirements validation) or continue as S?"

- Upgrade to M: run Phase 2 (validation) before continuing. Log in gate-log.
- Continue as S: CTO override, logged as `SIZE-OVERRIDE` in gate-log.

**Calibration:** Review SIZE-OVERRIDE entries in gate-logs after 5-10 engagements. If overrides are frequent, raise thresholds. If scope creep is common, lower them.

---

## Subagent Prompt Templates

### Validation Subagent (Phase 2)

```
Read `{{paths.engagements}}{engagement}/brief.md` and
`{{paths.instructions_dir}}/validation-framework.instructions.md`.

Apply the 5-test validation framework against the brief.

Also read:
- {{paths.non_goals}} — check for scope conflicts
- {{paths.roadmap}} — check for roadmap alignment
- {{paths.decisions}} — check for ADR conflicts

Write a validation record to `{{paths.engagements}}{engagement}/validation.md`.

Return a JSON summary:
{
  "verdict": "PASS|CONDITIONAL|FAIL",
  "risks": ["..."],
  "conflicts": ["..."],
  "recommendation": "..."
}
```

### Planning Subagent (Phase 3)

```
Read `{{paths.engagements}}{engagement}/brief.md` and the validation
record at `{{paths.engagements}}{engagement}/validation.md` (if it exists).

Read:
- {{paths.instructions_dir}}/planning-compliance.instructions.md
- {{paths.instructions_dir}}/planning-preflight.instructions.md

Explore the codebase for patterns to reuse relevant to the brief's scope.
Run pre-flight checks ({{paths.technical_debt}}, {{paths.decisions}}, {{paths.future_considerations}}, {{paths.non_goals}}).

Write a draft PLAN.md with tasks, files to modify, quality gates, and
acceptance criteria. Follow the format in docs/templates/SPRINT_PLAN_TEMPLATE.md.

Return a JSON summary:
{
  "sprintNumber": N,
  "taskCount": N,
  "filesAffected": N,
  "risks": ["..."],
  "complianceIssues": ["..."]
}
```

### Commercial Readiness Subagent (Gate 3)

```
[SUBAGENT-MODE] [WRITE:ANALYSIS-ONLY]

Read `{{paths.engagements}}{engagement}/brief.md` to retrieve the proposed scope.
Read the sprint RETRO.md at `{{paths.engagements}}{engagement}/` (or ask @delivery-lead
for the sprint reference) to determine what was actually built.

Re-run the commercial 5-test framework against the AS-BUILT scope:

1. Market size and serviceable share — still valid for what shipped?
2. Pricing model fit and willingness to pay — does the implementation match the monetisation assumption?
3. CAC and channel viability — acquisition cost unchanged by implementation choices?
4. LTV by tier and unit economics — unit economics hold for what was built?
5. Defensibility and regulatory cost — new surface area changes the risk profile?

Also identify:
- Scope delta: features proposed but not built, and features built but not in brief
- Any changes that affect commercial assumptions from the original intake

Return a JSON summary:
{
  "verdict": "VALIDATED|CONDITIONAL|REJECTED",
  "scopeDelta": { "added": ["..."], "dropped": ["..."] },
  "testResults": [
    { "test": "Market size", "verdict": "PASS|FAIL|N/A", "reasoning": "..." },
    { "test": "Pricing fit", "verdict": "PASS|FAIL|N/A", "reasoning": "..." },
    { "test": "CAC/channel", "verdict": "PASS|FAIL|N/A", "reasoning": "..." },
    { "test": "LTV/unit economics", "verdict": "PASS|FAIL|N/A", "reasoning": "..." },
    { "test": "Defensibility", "verdict": "PASS|FAIL|N/A", "reasoning": "..." }
  ],
  "recommendation": "..."
}
```

---

## Platform Configuration

Gate definitions above reference "the platform's deployment workflow" and "automated test suite" generically. This section maps to concrete values per platform.

### Web

- **Test deploy workflow:** `{{platform.test_workflow}}`
- **Production deploy workflow:** `deploy-azure-swa.yml` ({{platform.ci_workflow_display_name}})
- **Automated test suite:** Playwright E2E (chromium + iPhone 14 viewport)
- **Test environment URL:** Azure SWA test instance URL from workflow output
- **Production URL:** `https://gray-glacier-029377c00.7.azurestaticapps.net`
- **CI workflow:** `ci.yml`
- **Deploy check commands:**
  ```bash
  # Test deployment
  gh run list --branch {{git.develop_branch}} --workflow {{platform.test_workflow}} --limit 3 --json databaseId,status,conclusion
  # Production deployment
  gh run list --branch {{git.main_branch}} --workflow "{{platform.ci_workflow_display_name}}" --limit 2 --json databaseId,status,conclusion
  # Failure details
  gh run view <run-id> --log-failed | tail -50
  ```

### Mobile (future)

- TBD — add when mobile enters engagement model
- Same gate structure, different workflow names and test runner
