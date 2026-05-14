---
description: "Core rules for non-filesystem tracker mode (Linear, GitHub Issues, GitLab). Defines the two-tier model, comment header convention, MCP requirements, verification grounding, and per-concern adapter file references. Every agent operating in tracker mode reads this file. Concern-specific operations (backlog, drafts, validations, etc.) live in tracker-adapter-<concern>.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Core

The core ruleset that every agent reads when `{{tracker.type}}` is not `"filesystem"`. Concern-specific operations live in sibling files (`tracker-adapter-backlog`, `tracker-adapter-drafts`, etc.). Each agent reads only the concerns it touches.

## When this applies

Active when `{{tracker.type}}` in the project profile is one of: `linear`, `github`, `gitlab`.

When `{{tracker.type}}` is `filesystem` (the default), this file does not apply. The existing filesystem-coupled instructions (`backlog-ledger`, `sprint-docs-format`, `handoff-rejection-format`, etc.) are authoritative for both format and location.

In tracker mode the existing instructions still govern the **format** of artefacts. The adapter files govern the **location** (Linear comment or document instead of filesystem path). Agents read both.

## Tracker MCP requirements

The dispatched agent must have access to the tracker's MCP server with at minimum:

- `list_issues(filter)` — fetch issues by state, label, team, project
- `get_issue(id)` — fetch one issue with its comments
- `save_issue(id?, ...)` — create or update an issue, including state transitions
- `list_comments(issueId)` — fetch comments on an issue
- `save_comment(issueId, body, id?)` — create or update a comment
- `list_documents(project)`, `get_document(id)`, `save_document(...)` — project-level reference docs (Linear only; GitHub uses `docs/*.md` or wiki; GitLab uses wiki pages)

If MCP access is unavailable: STOP and post a blocker note. Do not fall back to filesystem operations — that would corrupt the agent farm's mental model of where state lives.

## The two-tier model

Every artefact agent-homebase agents produce belongs to one of two tiers:

- **Per-issue artefacts** (workpad, drafts, brainstorms, validations, REJs, handoffs, scoped research, bug intake) live as **Linear comments** on the relevant issue
- **Project-level reference** (ADRs, NON_GOALS, ROADMAP, FEATURE_MATRIX, cross-cutting research) lives as **Linear Documents** under the project

The default is "comment on the issue." Promote to a Document only when an artefact is referenced across multiple issues or has a lifespan longer than any single workstream.

## Comment header convention (required)

All workflow-significant comments use a level-2 markdown header (`##`) with a recognised type prefix. Agents scan comments by these headers to find prior artefacts. **Never use a level-2 header that doesn't fit one of these types for workflow-significant comments.** Casual progress paragraphs without headers are fine.

| Header | Purpose | Producer |
|---|---|---|
| `## Agent Workpad` | The single workpad per issue — edited not duplicated | All agents |
| `## Draft Plan` | @planner draft plan output | `@planner` |
| `## Brainstorm` | Pre-draft ideation | `@planner` |
| `## Validation Record — <slug>` | @pm 5-test validation | `@pm` |
| `## Commercial Validation — <slug>` | @pm commercial framework (post-VER-12) | `@pm` |
| `## Research — <topic>` | Issue-scoped research | `@researcher` |
| `## ADR Reference — ADR-<NNN>` | Pointer to a Linear Document | `@architect` |
| `## Handoff from @<X> to @<Y>` | Handoff manifest | The sending agent |
| `## REJ — <reason>` | Rejection record | The rejecting agent |
| `## Bug Intake` | @bug structured intake | `@bug` |
| `## Deferred — <reason>` | Records a deferral; count = Def | `@planner` |
| `## [ARCHIVED] <original header>` | Prefix on edit when archiving | Any agent |

Use the `id` parameter on `save_comment` to **edit** an existing comment rather than posting a new one for the same artefact type. The workpad in particular is a single comment for the lifetime of the issue.

## Verification grounding (cross-cutting)

In tracker mode, every task in a `## Draft Plan` MUST include an explicit `Verification` field. The verification is a mechanical check:

- A shell command with expected exit code or output (e.g. `grep -c "X" file returns 1`, `npm run typecheck returns 0`)
- A file-state assertion (e.g. `test -f path/to/file.md` returns 0)
- A named test invocation (e.g. `vitest run test/foo.test.ts -t "specific test" passes`)
- A git state check (e.g. `git diff --stat origin/main..HEAD shows N files changed`)
- A specific behavioural observation tied to a UI path (e.g. "navigate to /projects, set filter to Active, only active rows render")

This is non-negotiable. Without per-task verifications, downstream agents (`@in-progress`, `@sprint-lead`, `@qa`, `@reviewer`) cannot ground their work in source-of-truth checks and the workflow degrades to "agent says done, QA accepts the claim." This was a learned failure mode from prior projects.

`@qa` and `@reviewer` in tracker mode re-run each task's verification themselves before signing off. They do NOT accept a subagent's claim of "done" without re-running the verification. The verification output goes into the workpad as evidence.

## Per-concern adapter files

Each concern has its own focused file. Read only what you need.

| File | When to read |
|---|---|
| `tracker-adapter-backlog.instructions.md` | Reading or writing backlog state (status, priority, deferrals). `@planner`, `@bug` |
| `tracker-adapter-drafts.instructions.md` | Writing or reading draft plans and brainstorms. `@planner` |
| `tracker-adapter-validations.instructions.md` | Writing or reading validation records (5-test, commercial). `@pm`, `@reviewer` |
| `tracker-adapter-research.instructions.md` | Writing or reading research outputs (issue-scoped or project-scoped). `@researcher` |
| `tracker-adapter-handoffs.instructions.md` | Writing handoff manifests, finding incoming handoffs, REJ entries. Every agent that hands off |
| `tracker-adapter-sprints.instructions.md` | Composing or running sprints (cycles). `@sprint-lead` |
| `tracker-adapter-architecture.instructions.md` | Writing or referencing ADRs. `@architect` |
| `tracker-adapter-reference-docs.instructions.md` | NON_GOALS, ROADMAP, FEATURE_MATRIX, engagement docs. `@planner`, `@pm` |

## Anti-Patterns (cross-cutting)

- **Mixing modes within a single dispatch.** Filesystem OR tracker, decided by `{{tracker.type}}`. No hybrid. If you call `list_comments`, use comments for every artefact in this session. Don't half-commit.
- **Creating filesystem artefacts when tracker is non-filesystem.** No `docs/planning/drafts/*.md`, no `BACKLOG_LEDGER.md` row, no `HANDOFF_REJECTIONS.md` entry. Comments and documents only.
- **Posting unstructured comments for workflow data.** Workflow-significant comments use the recognised level-2 headers from the table above. Casual notes without headers are fine; structured artefacts always carry a header.
- **Creating duplicate workpads or duplicate-typed comments.** One `## Agent Workpad` per issue, edited via `save_comment(id, ...)`. Same applies to `## Draft Plan` (one per issue), `## Brainstorm` (one per issue). Multiple of the same type break the source-of-truth assumption.
- **Treating Linear Documents as comments or vice versa.** Documents are project-scoped reference. Comments are issue-scoped workflow. Don't put workflow data in documents or reference data in comments.
- **Editing system or orchestrator comments.** Orchestrator-generated comments (e.g. hatice's "**hatice** completed this issue") are read-only. Agents do not modify them.
- **Skipping verification grounding.** Every task in a draft plan has a Verification field. Every QA/Reviewer pass re-runs each task's verification. Without this the workflow degrades.

## How agents reference these files

Each agent that operates in tracker mode adds the following to its Shared Rules section (per `<agent>.skill.md`):

```markdown
- `{{paths.instructions_dir}}/tracker-adapter-core.instructions.md` — required in tracker mode; defines the two-tier model and conventions
- `{{paths.instructions_dir}}/tracker-adapter-<concern>.instructions.md` — one or more per the agent's responsibilities
```

The existing filesystem-coupled rules (e.g. `backlog-ledger.instructions.md`) stay in the Shared Rules list. They define the artefact format. The adapter files define the location. Both are read in tracker mode.

## Migration note

Transitioning a project from filesystem to tracker mode is one-way per project. Past filesystem artefacts (BACKLOG_LEDGER.md, drafts/, validation/, etc.) remain in the repo as historical reference. New artefacts produced after the cutover go through the adapter. No retroactive migration is required. Switching back to filesystem mode after operating in tracker mode would lose artefacts that only exist as Linear comments and documents.
