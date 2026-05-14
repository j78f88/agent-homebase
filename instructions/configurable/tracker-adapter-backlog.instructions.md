---
description: "Backlog operations in tracker mode — read, create, update status, record deferrals. Replaces filesystem reads/writes against BACKLOG_LEDGER.md. Read alongside backlog-ledger.instructions.md (which defines the row format) and tracker-adapter-core.instructions.md (which defines conventions)."
applyTo: "**"
---

# Tracker Adapter — Backlog Operations

Active when `{{tracker.type}}` is not `"filesystem"`. Replaces `{{paths.backlog_ledger}}` reads and writes with tracker-native operations. Format of each entry is still defined by `backlog-ledger.instructions.md`.

## Read backlog

Where filesystem mode reads `BACKLOG_LEDGER.md` to enumerate open items:

```
list_issues({
  project: "{{tracker.project_slug}}",
  team: "{{tracker.team_key}}",
  state: { type: { in: ["backlog", "unstarted", "started"] } },
  includeArchived: false
})
```

Each issue maps to a ledger row:

| BACKLOG_LEDGER column | Linear field |
|---|---|
| ID | `identifier` (e.g. `VER-11`) |
| Type | First label among `feature` / `bug` / `improvement` / `debt` / `audit` |
| Source | Parent issue identifier or origin label |
| Draft | Comment with header `## Draft Plan` if present |
| Status | `state.type` |
| Priority | `priority` (1=Urgent → 4=Low) |
| Created | `createdAt` |
| Age | `currentDate - createdAt` |
| Def (deferral count) | Count of `## Deferred — <reason>` comments on the issue |

## Create a backlog entry

Where filesystem mode appends a row to the ledger:

```
save_issue({
  team: "{{tracker.team_key}}",
  project: "{{tracker.project_slug}}",
  title: "<intent title>",
  description: "<user story / intent body>",
  labels: ["<type>"],
  priority: <1-4>,
  assignee: "{{team.cto_name}}"
})
```

The returned `identifier` (e.g. `VER-42`) is the new ID. Reference it in commits and other artefacts.

## Update status

```
save_issue({ id: "<identifier>", state: "<target state name>" })
```

State mapping (Linear default workflow):

| filesystem status | Linear state |
|---|---|
| `open` / `unscheduled` | `Backlog` |
| `ready` | `Todo` |
| `active` | `In Progress` |
| `awaiting-review` | `In Review` |
| `complete` | `Done` |
| `closed` (not-done) | `Cancelled` or `Duplicate` |

For non-Linear trackers, see platform-specific state lists in your tracker's MCP docs.

## Record a deferral

Where filesystem mode increments the `Def` column on a ledger row:

```
save_comment({
  issueId: "<identifier>",
  body: `## Deferred — <reason>

**Date:** <iso>
**Deferred by:** @<agent>
**Reason:** <one-line explanation>
**Re-evaluate at:** <iso date or condition>
`
})
```

Deferral count is the number of `## Deferred —` comments on the issue. When this count reaches `{{escalation.def_p0_threshold}}` (typically 3), the issue becomes P0 mandatory per `composition-rules.instructions.md`. When it reaches `{{escalation.def_kill_threshold}}` (typically 5), the issue must be resolved or killed — no further deferral.

## Composition note

Sprint composition (deciding which issues go into the next cycle) uses Linear's native `cycle` field, not a separate composition document. See `tracker-adapter-sprints.instructions.md` for cycle assignment.

## Anti-patterns specific to backlog

- Posting "I deferred this" as a casual comment without the `## Deferred —` header. Agents scanning for deferral count rely on the header.
- Updating priority via comment instead of `save_issue(priority: ...)`. Priority lives on the issue, not in a comment.
- Reading state from comment content instead of `state.type`. State is structured Linear data; don't shadow it in prose.
