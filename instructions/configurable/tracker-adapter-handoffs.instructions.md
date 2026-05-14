---
description: "Handoff manifests and rejection (REJ) entries in tracker mode — both live as Linear comments on the source issue, with REJs paired to a state transition. Replaces filesystem writes to {{paths.handoffs}}/ and {{paths.rejections}}. Read alongside handoff-rejection-format.instructions.md and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Handoffs and Rejections

Active when `{{tracker.type}}` is not `"filesystem"`. Handoffs between agents are recorded as comments on the source issue. Handoff rejections (REJ-NNN) are comments PLUS a state transition that signals the upstream agent.

## Handoff manifests

### Write a handoff manifest

Where filesystem mode creates `{{paths.handoffs}}<date>-<from>-to-<to>-<slug>.md`:

```
save_comment({
  issueId: "<identifier>",
  body: `## Handoff from @<from> to @<to>

| Field | Value |
|---|---|
| from | @<from> |
| to | @<to> |
| date | <iso date> |
| feature | <slug> |
| artifact | <reference to prior comment header, e.g. "## Validation Record — <slug>"> |
| verdict | <if applicable — VALIDATED / REFRAMED / NEW / DEFERRED / etc.> |
| notes | <one-line context summary> |

> Context for @<to>: <paste-friendly summary the receiving agent can use to orient>
`
})
```

The handoff comment lives on the issue being handed off. The receiving agent picks it up at session start.

### Find incoming handoffs at session start

Where filesystem mode scans `{{paths.handoffs}}` for manifests addressed to it:

```
list_comments({ issueId: "<identifier>" })
```

Filter for headers matching `## Handoff from @<X> to @<self>` where `## [ARCHIVED]` prefix is NOT present. The most recent matching unarchived comment is the active handoff.

If multiple unarchived handoffs to the same agent exist (anti-pattern), use the most recent and flag the duplicate to the user.

### Archive a handoff (after acceptance)

Where filesystem mode moves a manifest to `{{paths.handoffs}}archive/`:

Edit the comment in place to prepend `[ARCHIVED] ` to the header:

```
save_comment({
  id: "<comment id>",
  body: <body with `## Handoff` replaced by `## [ARCHIVED] Handoff`>
})
```

This makes the comment invisible to future session-start scans while preserving the audit trail.

## Handoff Rejections (REJ entries)

A REJ is a comment plus a state transition. Both are required.

### Write a REJ entry

```
save_comment({
  issueId: "<identifier>",
  body: `## REJ — <one-line reason>

**Severity:** <CRITICAL | WARNING | SUGGESTION>
**From:** @<rejecting agent>
**To:** @<upstream agent>
**Raised:** <iso date>
**Status:** OPEN

### Context
<what was handed over to this agent>

### Reason
<specific conflict — cite ADR-NNN, constraint, or prior decision; be concrete>

### Proposed resolution
<split / defer / re-approach / etc.>
`
})

save_issue({
  id: "<identifier>",
  state: "Backlog"  // or "Todo" if re-triage is the next expected step
})
```

The state transition signals to the upstream agent (via next poll) that work is needed. Without the state transition, the REJ is dormant — no one acts on it. **This pairing is non-negotiable.**

### REJ lifecycle

REJ status flows in this order:

- `OPEN` — newly raised, awaiting upstream response
- `RESOLVED-ACCEPTED` — upstream agent revised the handoff, REJ resolved
- `RESOLVED-OVERRIDDEN` — user explicitly overrode the REJ, reason in commit body
- `CANCELLED` — REJ was raised in error

When transitioning the status, update the comment in place:

```
save_comment({
  id: "<REJ comment id>",
  body: <body with `Status: OPEN` replaced by `Status: <new>` plus a Resolution section>
})
```

Add a Resolution section to the bottom:

```
### Resolution
**Resolved:** <iso date>
**By:** @<agent or user>
**Outcome:** <what changed>
```

### `@reviewer` enforcement

`@reviewer` scans for REJs that are OPEN >14 days (SUGGESTION) or REJ entries missing a `Proposed resolution` section (WARNING). For handoffs scaled down or scope dropped without a paired REJ, `@reviewer` flags CRITICAL.

## Anti-patterns specific to handoffs and rejections

- **REJ without state transition.** A REJ comment alone is dormant. Always pair with `save_issue({ state: ... })`.
- **Multiple unarchived handoffs to the same agent on the same issue.** Archive the older one when you create a new one.
- **Editing or deleting prior handoff comments.** Archive in place; don't delete or rewrite.
- **Writing a handoff manifest with no artefact reference.** The `artifact` field must point at a specific prior comment header (e.g. "## Validation Record — <slug>") or a Linear Document URL. Receiving agents rely on this pointer.
- **Posting a handoff in autopilot mode without a paste-friendly Context block.** The context block is the receiving agent's orientation when it has no prior session history.
