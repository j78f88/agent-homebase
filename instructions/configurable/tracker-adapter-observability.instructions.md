---
description: "Observability layer for tracker mode — defines three artefacts that surface strategic signals from the agent farm's high-volume output: Strategic Brief (curated weekly summary), Decision Log (append-only chronological audit), and Attention Required (a Linear view + label scheme). Maintained by @docs in its post-sprint cadence. Read alongside tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Observability

Active when `{{tracker.type}}` is not `"filesystem"`. The agent farm produces high-volume output (validations, REJs, deferrals, ADRs, NON_GOALS additions). Without curation, the operator (typically the CTO) cannot keep up. This file defines three observability artefacts that turn the firehose into something readable.

Owned by `@docs` in tracker mode, updated on its existing post-sprint cadence.

## The three artefacts

### 1. Strategic Brief — what the operator reads

A single Linear Document per project, titled `"Strategic Brief"`, edited in place. Joshua opens this when he wants to know "what's happening strategically and what needs me."

**Update cadence:** weekly OR on sprint completion (whichever comes first). `@docs` overwrites the document body each update; Linear's edit history preserves prior versions.

**Required structure:**

```markdown
# Strategic Brief — <project name>

**Last updated:** <iso>
**Period covered:** <start iso> to <end iso>

## Needs your decision

Items requiring the operator's strategic call. List concrete items with links. If none, write "No decisions pending."

- <REJ-NNN: <one-line description>. Open <N> days. [link to issue]>
- <Validation REJECTED for <feature>: needs roadmap update. [link]>
- <ITEM-NNN deferred <N>+ times: must resolve or kill. [link]>

## Current roadmap phase

Snapshot of the ROADMAP Linear Document's current phase. One paragraph. Link to the full Roadmap document.

## Decisions made this period

### Validations
- <feature-slug>: VALIDATED → in current cycle. [link]
- <feature-slug>: REJECTED → added to Non-Goals. [link]
- <feature-slug>: REFRAMED → see new framing. [link]

### Architecture
- ADR-NNN: <title> — Accepted on <date>. [link]
- ADR-NNN: <title> — Proposed (awaiting operator review). [link]

### Deferrals and escalations
- ITEM-NNN: deferred <N> times, re-evaluation date <iso>. [link]

## Active work

- Issues in `In Progress`: <count>
- Issues in `In Review`: <count>
- Issues in `Todo` awaiting dispatch: <count>
- Current cycle: <name> — <N> issues, <M> complete

## Stale or stuck

- <Issue>: in `Todo` >14 days without activity. [link]
- <REJ>: OPEN >14 days. [link]
- <Validation>: pending @pm response >7 days. [link]
```

When a section has no entries, write "(none this period)" rather than removing the heading. The shape stays stable across updates so readers can scan the same place.

### 2. Decision Log — append-only audit trail

A single Linear Document per project, titled `"Decision Log"`, appended to (never rewritten) by `@docs`.

**Update cadence:** appended on every Strategic Brief update. Also: triggered immediately when a significant decision lands (Validation REJECTED, ADR Accepted, NON_GOALS addition, ROADMAP phase change). Operator does not read regularly; this is the audit trail.

**Required structure:**

```markdown
# Decision Log — <project name>

Append-only chronological log of every decision made by the agent farm. Most recent at the top.

## <YYYY-MM-DD>

- <HH:MM> — <type>: <description>. <link to source artefact>
- <HH:MM> — Validation REJECTED for <slug> (5-test framework). Reason: failed <test name>. Added to Non-Goals. [link to Validation Record comment]
- <HH:MM> — ADR-007 Accepted: "<title>". [link to ADR Document]
- <HH:MM> — ITEM-042 deferred (Def=3). Reason: blocked by ADR pending. Re-evaluate after <iso>. [link]
- <HH:MM> — ROADMAP phase 2 entered. Phase 1 completed with <N>/<M> features shipped. [link]

## <YYYY-MM-DD>
...
```

Each entry is one line: timestamp, decision type, one-sentence description, link to source. The Decision Log is the comprehensive log; the Strategic Brief is the curated summary. Both link back to source artefacts (Linear comments or Documents).

`@docs` MUST NOT delete or rewrite past entries. Append-only. The audit trail is the value.

### 3. Attention Required — a Linear view + label scheme

Not a document. A saved Linear view that the operator bookmarks. Powered by labels that `@docs` applies based on scan rules.

**Label scheme** (`@docs` applies and removes during digest updates):

- `attention:cto-review` — REJ open >14 days OR Validation REJECTED with no follow-up roadmap update OR ADR-Proposed pending operator approval
- `attention:stale-rej` — REJ comment marked OPEN that has been on the issue >14 days
- `attention:def-escalation` — issue with deferral count >= `{{escalation.def_p0_threshold}}` (typically 3) requiring P0 inclusion or kill
- `attention:def-kill` — issue with deferral count >= `{{escalation.def_kill_threshold}}` (typically 5) requiring resolution or kill, no more deferral permitted

**View spec** (Joshua creates this once in Linear UI; `@docs` doesn't manipulate views):

```
Filter:
  Project: <project slug>
  Label: any of (attention:cto-review, attention:stale-rej, attention:def-escalation, attention:def-kill)
Group by: Label
Sort: Priority descending, Created ascending
```

The view shows only issues `@docs` has tagged for operator attention. Bookmarked once, opens to a focused worklist on demand.

`@docs` MUST also REMOVE attention labels when the underlying condition resolves (e.g. REJ status moves OPEN → RESOLVED, deferral count drops below threshold, validation gets a follow-up). Stale labels degrade the view.

## When and how `@docs` updates these artefacts

### Trigger 1: Sprint completion

After processing a sprint completion in tracker mode:

1. Append new entries to the Decision Log Document for every decision that landed in the sprint (validations from `@pm`, ADRs from `@architect`, deferrals from `@planner`, NON_GOALS additions).
2. Edit the Strategic Brief Document with the new period's snapshot:
   - Recompute the `## Needs your decision` section by scanning open REJs, REJECTED validations without follow-ups, items above deferral thresholds
   - Update `## Current roadmap phase` from the ROADMAP Document
   - Populate `## Decisions made this period` from the new Decision Log entries
   - Recount `## Active work` from `list_issues`
   - Scan for `## Stale or stuck` items per the thresholds in the brief structure above
3. Update issue labels per the Attention Required scheme: apply labels where conditions newly fire, remove labels where conditions resolved.

### Trigger 2: Significant decision lands (between sprints)

When a particularly weighty decision lands mid-sprint:
- ADR Accepted
- Validation REJECTED on a Feature-class issue
- NON_GOALS addition

`@docs` appends one entry to the Decision Log immediately. Does NOT update the Strategic Brief until the next sprint trigger (Brief is weekly cadence; Log is realtime).

### Trigger 3: Operator on-demand

Operator can invoke `@docs` with `/observability-refresh` to force a Strategic Brief update outside the sprint cadence. Useful before a strategic review meeting or after a long absence.

## `@docs`'s extended responsibilities in tracker mode

The existing `@docs` flow (SPRINTS, FEATURE_MATRIX, changelog, user guide updates) continues in tracker mode adapted via `tracker-adapter-reference-docs.instructions.md` and `tracker-adapter-sprints.instructions.md`.

In addition, in tracker mode `@docs` is responsible for:

1. Maintaining the Strategic Brief Document (overwrite on each cadence trigger)
2. Appending to the Decision Log Document (append-only)
3. Applying and removing Attention Required labels on issues

These are new responsibilities specific to tracker mode. Filesystem mode does not have an equivalent (in filesystem mode the human operator typically reads commits and BACKLOG_LEDGER directly).

## Anti-patterns specific to observability

- **Rewriting the Decision Log.** Append-only. Past entries are immutable. If a decision was wrong, the correction is a NEW entry referencing the old one, not an edit.
- **Letting attention labels go stale.** When the underlying condition resolves, remove the label. Otherwise the Attention Required view fills with noise and the operator stops trusting it.
- **Putting workflow data in the Strategic Brief.** The brief is curated summary. Detailed workflow lives in issue comments and the per-issue workpad. The brief is for "what you need to know" not "every step that happened."
- **Skipping the `## Needs your decision` section.** Even if there are no decisions pending, the section header stays with the text "No decisions pending." Operators scan for this section first; if it's missing they assume the brief didn't update.
- **Updating the Strategic Brief more often than sprint cadence.** Brief is weekly/per-sprint. Realtime updates go to the Decision Log. The brief's value is its bounded, predictable surface.
- **Treating Strategic Brief as historical.** It's a snapshot. Each update replaces the prior body. Linear's edit history preserves the trail if needed, but no one reads back through versions. Decision Log is the historical record.

## Operational notes

- Strategic Brief and Decision Log are both Linear Documents. They're located by title under the project (`list_documents({ project })` filtered by title).
- The Decision Log grows unbounded over time. Once it exceeds a manageable size (say 200 entries or 50,000 chars), `@docs` can split: archive the older entries into a second document titled `"Decision Log — Archive YYYY-Q[1-4]"`, keep the current quarter in the main document.
- The Attention Required view is operator-created in Linear UI once. `@docs` does not create or modify views.
- For GitHub Issues / GitLab mode: Strategic Brief and Decision Log can live as files in the repo's `docs/observability/` folder; Attention Required maps to GitHub issue labels + a saved search URL.
