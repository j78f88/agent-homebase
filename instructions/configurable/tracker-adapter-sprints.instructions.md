---
description: "Sprint and cycle operations in tracker mode — Linear cycles replace sprints/sprint-N/ folders. Retros become Linear Documents. Read alongside retro-report.instructions.md, sprint-docs-format.instructions.md, and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Sprints and Cycles

Active when `{{tracker.type}}` is not `"filesystem"`. Sprint-level operations (composition, kickoff, retro) map to Linear's native cycles or milestones, not to per-sprint folders.

## Sprint composition (replaces sprint-N/PLAN.md)

Filesystem mode writes `sprints/sprint-N/PLAN.md` listing tasks. Tracker mode uses Linear cycle membership.

### Compose a sprint

1. Identify the issues for the sprint by reading their `## Draft Plan` comments (already promoted per `tracker-adapter-drafts.instructions.md`).
2. For each, assign to the cycle:
   ```
   save_issue({ id: "<identifier>", cycle: "<cycle name or number>" })
   ```
3. Optionally update issue priority to control ordering within the cycle (`save_issue({ id, priority: N })`).

The cycle membership **IS** the sprint plan. No separate `PLAN.md` exists. Each issue's `## Draft Plan` comment is its piece of the cycle plan.

### Read the current cycle's plan

```
list_issues({
  project: "{{tracker.project_slug}}",
  cycle: "<current cycle>",
  state: { type: { in: ["unstarted", "started"] } }
})
```

Each returned issue has its draft plan as a comment. Iterate and read.

## Sprint kickoff (replaces SPRINTS.md status update)

No equivalent action needed in Linear. Cycles auto-track. When an issue in the cycle transitions to `In Progress`, the cycle is implicitly active.

If you want an explicit "sprint kicked off" record, post a single comment on a designated "Cycle <N> Tracker" issue (or on the first cycle issue) with header `## Cycle <N> — Kickoff`. This is optional; most teams skip it.

## Sprint retrospective (replaces sprint-N/RETRO.md)

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Retrospective — Cycle <N>",
  content: <retro body following retro-report.instructions.md sections>
})
```

The retro is a Document because retros are reference material — cited in future planning, surfaced in onboarding, etc. Not workflow.

If the project doesn't use cycles, post the retro as a comment on a dedicated "Sprint N" tracker issue with header `## Retrospective` instead. Fallback only.

## Reading prior retros

```
list_documents({ project: "{{tracker.project_slug}}" })
```

Filter titles starting with "Retrospective — ". `@sprint-lead`'s session start reads the prior cycle's retro (forecast) to drive Phase 6 comparison.

## Engagement-level work

If the project uses engagement tracking (ENG-NNN), see `tracker-adapter-reference-docs.instructions.md` for engagement document conventions. Engagements map to either a separate Linear project or a labelled issue group, depending on size.

## Branch naming convention

Where filesystem mode uses `sprint-N` branch names, tracker mode uses the Linear identifier:

- `<issue.identifier-lowercased>-<slug>` (e.g. `ver-11-add-write-permit-token`, `ver-42-fix-login-redirect`)

Each issue produces one branch. No per-sprint branch.

## Anti-patterns specific to sprints

- Writing `## Plan` or `## Sprint Plan` as a comment on a "cycle tracker" issue. Sprint plans live on per-issue draft comments; the cycle is just membership.
- Using `sprint-N` branch naming in tracker mode. Tracker mode uses issue identifier in branch names.
- Skipping the retro at end of cycle because "there's no RETRO.md anymore". Retros are Documents in tracker mode; still required if your project uses retros at all.
- Treating cycle assignment as state. Cycle membership is orthogonal to issue state. An issue can be in cycle 5 AND in Backlog state (planned for that cycle but not started).
