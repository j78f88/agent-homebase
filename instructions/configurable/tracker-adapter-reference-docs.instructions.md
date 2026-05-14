---
description: "Project-level reference documents (NON_GOALS, ROADMAP, FEATURE_MATRIX, engagement briefs) in tracker mode — each lives as a Linear Document under the project. Replaces filesystem files like docs/NON_GOALS.md, docs/planning/ROADMAP.md. Read alongside non-goals-governance.instructions.md and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Reference Documents

Active when `{{tracker.type}}` is not `"filesystem"`. Project-level reference documents — content that's cited across many issues and changes slowly — live as Linear Documents. Each is a single document per project, edited in place over time.

## NON_GOALS (replaces `{{paths.non_goals}}`)

A single project-scoped Linear Document.

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Non-Goals",
  content: `# Non-Goals — <project name>

This list is the explicit set of things this project is NOT doing. Each entry has a rationale; future requests that propose these should reference this document.

| Item | Rationale | Date added |
|---|---|---|
| <item 1> | <why we're not doing it> | <iso> |
| <item 2> | ... | ... |
`
})
```

Updates use `save_document({ id, content })` — replace the whole body with the updated table.

Owned by `@planner`. `@pm` may also add entries after a REJECTED validation (per `non-goals-governance.instructions.md`).

## ROADMAP (replaces `{{paths.roadmap}}`)

A single project-scoped Linear Document.

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Roadmap",
  content: <roadmap body — phases, rationale, parked items, dependencies>
})
```

Owned by `@pm`. Updated when feature validation produces a NEW or DEFERRED verdict, or when a phase completes and the next becomes active.

## FEATURE_MATRIX (replaces `{{paths.feature_matrix}}`)

A single project-scoped Linear Document.

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Feature Matrix",
  content: <matrix body — typically a markdown table of features × platforms × validation status>
})
```

Owned by `@docs`. Updated at sprint completion as features transition status.

## Engagement briefs (replaces `{{paths.engagements}}<id>/brief.md`)

Engagement-level work has two paths depending on size:

### Large engagement (multi-issue initiative)

Use a **separate Linear project** for the engagement. The brief, gate-log, and deployment-log become Documents under that project. Each piece of work is an issue.

```
save_document({
  project: "<engagement-project-slug>",
  title: "Engagement Brief",
  content: <brief body following engagement-format.instructions.md>
})
```

### Small engagement (single-issue or few-issue)

Use a label (`engagement-<NNN>` or similar) on each related issue, plus a "lead" issue that carries the brief as a comment:

```
save_comment({
  issueId: "<lead issue identifier>",
  body: `## Engagement Brief — <slug>

<brief body following engagement-format.instructions.md>
`
})
```

Gate-log entries become comments with headers `## Gate <N> — <decision>` on the lead issue.

## Reading reference documents

```
list_documents({ project: "{{tracker.project_slug}}" })
```

Filter by title prefix:
- "Non-Goals" → the non-goals doc
- "Roadmap" → the roadmap
- "Feature Matrix" → the feature matrix
- "Research:" → research docs (see `tracker-adapter-research.instructions.md`)
- "ADR-" → architecture decision records (see `tracker-adapter-architecture.instructions.md`)
- "Retrospective —" → retrospective docs (see `tracker-adapter-sprints.instructions.md`)
- "Engagement" → engagement-level docs

Then `get_document({ id })` for the full content of any matching doc.

## Update frequency

Reference documents change slowly. Batch updates rather than churning small edits:

- NON_GOALS: typically updated when @pm REJECTS a feature with a "this is a standing no"
- ROADMAP: updated at phase transitions or when @pm reframes a phase
- FEATURE_MATRIX: updated at sprint completion (typically by @docs as part of post-sprint sync)
- Engagement briefs: updated at gate transitions

Each update edits the existing document in place. Never create a "v2" of an existing reference document.

## Anti-patterns specific to reference docs

- Posting NON_GOALS or ROADMAP as an issue comment. These are project-level reference; they live in Documents.
- Creating multiple Non-Goals documents per project. There is exactly one. Edit it.
- Updating reference docs without a corresponding source artefact (e.g. adding to NON_GOALS without a matching REJECTED validation record). Reference docs reflect decisions made elsewhere; they're not the place to declare new decisions.
- Letting reference docs go stale. `@docs` agent's session start should include a quick scan for "documents not updated in >90 days" and flag if any are reachable from current work.
