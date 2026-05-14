---
description: "Drafts and brainstorms in tracker mode — write as Linear comments on the corresponding issue. Replaces filesystem writes to {{paths.drafts}}/*.md. Read alongside sprint-docs-format.instructions.md (defines section structure) and tracker-adapter-core.instructions.md (defines conventions)."
applyTo: "**"
---

# Tracker Adapter — Drafts and Brainstorms

Active when `{{tracker.type}}` is not `"filesystem"`. Drafts and brainstorms live as Linear comments on the corresponding issue. The format of each section follows `sprint-docs-format.instructions.md`; only the location changes.

## Write a draft plan

Where filesystem mode creates `{{paths.drafts}}/<slug>-draft-plan.md`:

```
save_comment({
  issueId: "<identifier>",
  body: `## Draft Plan

**Status:** DRAFT
**Sources:** <upstream artefacts — see sprint-docs-format.instructions.md for source types>

### Goals
...

### Why This Sprint
...

### Technical Tasks
- 1. <task title>
  - File: <path>
  - Action: <create | edit | delete>
  - Verification: \`<command or behavioural check>\`

### Files to Create/Modify
...

### Success Criteria
- [ ] <criterion 1>
- [ ] <criterion 2>

### Pre-flight Findings
...

### Compliance Notes
...
`
})
```

The `Verification` field per task is **non-negotiable in tracker mode** — see `tracker-adapter-core.instructions.md` § Verification grounding.

## Write a brainstorm

Where filesystem mode creates `{{paths.drafts}}/<slug>-brainstorm.md`:

```
save_comment({
  issueId: "<identifier>",
  body: `## Brainstorm

### Candidates
- [SELECTED] <idea 1>
- <idea 2>
- <idea 3>
- [SELECTED] <idea 4>

### Notes
...
`
})
```

`[SELECTED]` markers signal which candidates `@planner` should draft next. Same convention as filesystem brainstorms.

## Read existing drafts and brainstorms for an issue

Where filesystem mode scans `{{paths.drafts}}` for `<slug>*`:

```
list_comments({ issueId: "<identifier>" })
```

Filter by header:
- `## Draft Plan` → existing draft
- `## Brainstorm` → existing brainstorm
- `## [ARCHIVED] Draft Plan` or `## [ARCHIVED] Brainstorm` → ignore (already promoted or discarded)

If multiple `## Draft Plan` comments exist on the same issue (anti-pattern), use the most recent. Flag the duplicate to the user via the workpad — only one active draft per issue.

## Update an existing draft (revision)

Where filesystem mode would overwrite the draft file:

```
save_comment({
  id: "<existing draft comment id>",
  body: <updated draft body>
})
```

Edit in place. Never post a second `## Draft Plan` comment as a "v2".

## Promote a draft to a sprint plan

In Linear there is no separate `PLAN.md` file. Promotion means:

1. Edit the draft comment to update its header line: `## Draft Plan (Promoted YYYY-MM-DD)`
2. Update issue state to `Todo` via `save_issue({ id, state: "Todo" })`
3. If using Linear cycles, assign the issue to the current cycle: `save_issue({ id, cycle: "<cycle>" })` per `tracker-adapter-sprints.instructions.md`

The state transition and (optionally) cycle membership are the promotion signal. The draft comment remains as the plan; no separate `PLAN.md` is created.

## Anti-patterns specific to drafts and brainstorms

- Posting a second `## Draft Plan` comment when one already exists. Edit the existing one via `save_comment(id, ...)`.
- Writing a draft without per-task `Verification` fields. Required in tracker mode.
- Using `## Plan` or `## Plan Draft` or `## V2 Plan` instead of the canonical `## Draft Plan`. Agents scan by exact header.
- Treating brainstorm `[SELECTED]` markers as resolved when the corresponding draft hasn't been written. `@planner`'s session start checks for `[SELECTED]` candidates with no corresponding draft and prompts to harvest.
