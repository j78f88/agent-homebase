---
description: "Architecture Decision Records (ADRs) in tracker mode — ADRs are project-level reference, so they live as Linear Documents. Issue comments reference them via header `## ADR Reference — ADR-<NNN>`. Read alongside architect.skill.md template and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Architecture Decisions

Active when `{{tracker.type}}` is not `"filesystem"`. ADRs are project-level reference material with long lifespans, so they live as Linear Documents under the project. Issue-level work references them via comment pointers.

## Write an ADR

Where filesystem mode creates `docs/architecture/decisions/ADR-<NNN>-<title>.md`:

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "ADR-<NNN>: <title>",
  content: `# ADR-<NNN>: <title>

**Status:** <Proposed | Accepted | Deprecated | Superseded by ADR-<XXX>>
**Date:** <iso>
**Deciders:** <user name or agent>

## Context
<what problem are we solving? what constraints apply? what are we NOT solving?>

## Decision
<the choice we made — one paragraph>

## Alternatives considered
<at least two alternatives, each with why it was rejected>

## Consequences
- Positive: <what this unlocks>
- Negative: <what this costs or forecloses>
- Risks: <what could go wrong and how we'd know>

## Principles applied
<general engineering concepts behind this decision — e.g. "Dependency Inversion", "YAGNI", "Open/Closed Principle">
`
})
```

The document URL becomes the canonical ADR reference. Cite it from issue comments and other documents.

## ID assignment

Sequential per project. Read the existing ADR Documents to find the highest existing N, assign N+1. If no ADRs exist yet, start at ADR-001.

```
list_documents({ project: "{{tracker.project_slug}}" })
```

Filter titles matching `ADR-`.

## Reference an ADR from an issue

When an issue's work is governed by or depends on an ADR, post a comment:

```
save_comment({
  issueId: "<identifier>",
  body: `## ADR Reference — ADR-<NNN>

This work is governed by [ADR-<NNN>: <title>](<linear-doc-url>).

### How it applies
<one-or-two-sentence summary of why this ADR is relevant to this issue's scope>
`
})
```

This makes the dependency discoverable in both directions: opening the issue shows the ADR pointer; reading the ADR's "Referenced by" section in Linear shows which issues cite it.

## Supersede an ADR

When a new ADR replaces an older one:

1. Edit the superseded ADR's document to update Status:
   ```
   save_document({
     id: "<superseded ADR doc id>",
     content: <body with Status updated to `Superseded by ADR-<XXX>`>
   })
   ```
2. Create the replacement ADR as a new document with its own ID (e.g. ADR-007 superseded by ADR-014).
3. In the new ADR's body, reference the superseded one explicitly in the Context section.

Both documents remain. History is preserved. Future readers can follow the chain.

## Promote a Proposed ADR to Accepted

```
save_document({
  id: "<ADR doc id>",
  content: <body with Status updated from `Proposed` to `Accepted`>
})
```

`@architect` typically transitions Proposed → Accepted after user approval. After acceptance, `@planner` can incorporate the ADR's decisions into draft plans.

## Anti-patterns specific to architecture

- Posting an ADR as a comment instead of a Document. ADRs are reference, not workflow — they belong in Documents.
- Skipping the Alternatives section. An ADR without alternatives is marketing copy, not engineering. `@reviewer` flags this.
- Skipping Consequences. An ADR without costs listed is incomplete.
- Using vendor names as the decision (e.g. "use Supabase"). Decisions should be in terms of capabilities (e.g. "use a hosted Postgres with row-level auth"). Vendor choice belongs in a separate Reference Doc.
- Editing an Accepted ADR in place (other than to mark it Superseded). Once Accepted, the body is frozen. Changes require a new ADR that supersedes.
- Creating multiple Proposed ADRs for the same decision. One per decision. If you're exploring, use a Linear comment with header `## ADR Exploration — <topic>` first, then write one Proposed ADR when the direction is clear.
