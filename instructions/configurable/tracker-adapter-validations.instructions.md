---
description: "Validation records (@pm 5-test, commercial framework) in tracker mode — write as Linear comments on the corresponding issue. Replaces filesystem writes to {{paths.validation}}/*-validation.md. Read alongside validation-framework.instructions.md and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Validation Records

Active when `{{tracker.type}}` is not `"filesystem"`. Validation records from `@pm` (and `@reviewer` enforcement) live as Linear comments on the validated issue. The 5-test framework (product-fit) and commercial framework (post-VER-12) each get their own comment.

## Write a product-fit validation record (5-test framework)

Where filesystem mode creates `{{paths.validation}}/<slug>-validation.md`:

```
save_comment({
  issueId: "<identifier>",
  body: `## Validation Record — <slug>

**Verdict:** <VALIDATED | REFRAMED | NEW | REJECTED | DEFERRED>
**Date:** <iso>
**Framework:** product-fit (5-test)

### Test results

1. **Causation vs correlation** — Verdict: <PASS|FAIL|N/A>. <one-or-two-sentence reasoning>
2. **Frequency match** — Verdict: <PASS|FAIL|N/A>. <reasoning>
3. **Survivorship bias** — Verdict: <PASS|FAIL|N/A>. <reasoning>
4. **Anti-pattern / engagement-at-cost** — Verdict: <PASS|FAIL|N/A>. <reasoning>
5. **Complexity cost** — Verdict: <PASS|FAIL|N/A>. <reasoning>

### If REFRAMED
**Original:** <original feature framing>
**Reframed:** <new framing and the reason the reframe rescues it>

### Sources / inputs
- <comment references or research links>
`
})
```

## Write a commercial validation record (post-VER-12)

For Feature-class issues, post a separate comment with the commercial framework:

```
save_comment({
  issueId: "<identifier>",
  body: `## Commercial Validation — <slug>

**Verdict:** <VALIDATED | REJECTED | DEFERRED>
**Date:** <iso>
**Framework:** commercial (5-test)

### Test results

1. **Market size and serviceable share** — Verdict: <PASS|FAIL|N/A>. <reasoning>
2. **Pricing model fit and willingness to pay** — Verdict: <PASS|FAIL|N/A>. <reasoning>
3. **CAC and channel viability** — Verdict: <PASS|FAIL|N/A>. <reasoning>
4. **LTV by tier and unit economics** — Verdict: <PASS|FAIL|N/A>. <reasoning>
5. **Defensibility and regulatory cost** — Verdict: <PASS|FAIL|N/A>. <reasoning>
`
})
```

## Read existing validation records on an issue

```
list_comments({ issueId: "<identifier>" })
```

Filter by header:
- `## Validation Record — <slug>` → product-fit validation
- `## Commercial Validation — <slug>` → commercial validation

For `@reviewer` enforcement: when a handoff from `@pm` to `@planner` happens (transition or comment), check that both a product-fit Validation Record AND (if Feature-class) a Commercial Validation comment exist on the issue. Missing = CRITICAL finding.

## Update an existing validation record

Where filesystem mode would re-write the validation file:

```
save_comment({
  id: "<existing validation comment id>",
  body: <updated body>
})
```

Edit in place. Never post a second `## Validation Record — <same slug>` as a "v2".

## Anti-patterns specific to validations

- Posting a validation record without naming the verdict at the top. Verdict line is the first thing `@reviewer` reads.
- Verdict line present but no reasoning per test. Each test must have a one-or-two-sentence reasoning, not just PASS/FAIL.
- Marking REFRAMED without both the original and the reframed version. `@reviewer` flags SUGGESTION on REFRAMED records missing original/reframed pair.
- Skipping the commercial validation for Feature-class issues (post-VER-12). The commercial framework is mandatory for Feature-class.
