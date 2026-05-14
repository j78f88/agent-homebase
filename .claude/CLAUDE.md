# agent-homebase — Project Instructions for Dispatched Agents

## Project Identity

This repository is **agent-homebase**, the skills and instructions library for the Verk platform. It contains agent definitions, skill files, instruction sets, and configuration used by all dispatched Claude Code agents.

The repository is currently being upgraded for **Verk v2** per the Linear project `verk-v2`. Work is tracked in Linear and dispatched by **hatice** (the orchestration agent).

---

## Agent Invocation Pattern

Use the correct agent for each class of work. Do not invoke specialist agents directly — route through the appropriate lead.

| Situation | Invoke |
|---|---|
| Engagement-level work (multi-issue features) | `@delivery-lead` |
| Sprint scoping and planning | `@planner` |
| Spec proposals from vision docs | `@spec` *(once VER-16 lands)* |
| Commercial validation | `@pm /validate-commercial` *(once VER-13 lands)* |
| Implementation orchestration | `@sprint-lead` |

**Specialist agents** (`@qa`, `@reviewer`, `@docs`, `@a11y`, `@perf`, `@security`, `@bug`) are called via `@sprint-lead` delegation — not invoked directly.

---

## File Edit Approval Rule

Each Linear issue moved to **Todo** by Joshua is implicit approval for hatice to dispatch a Claude Code agent, and for that agent to edit files **within the scope of the issue**.

The agent must not edit files outside the issue scope without raising a new Linear issue per the WORKFLOW.md out-of-scope protocol.

---

## Coding Conventions

- **Commit messages**: follow `instructions/generic/commit-conventions.instructions.md`
  - Format: `<type>: <description> (ISSUE-ID)`
  - Types: `feat`, `fix`, `test`, `docs`, `chore`, `refactor`
- **Planning files**: always draft before promotion, per `skills/planner/planner.skill.md`
- **Validation records**: required before sprint hand-offs, per `instructions/configurable/validation-framework.instructions.md`

---

## Linear Interaction Protocol

1. Read the issue body; respect its `File:` and `Action:` fields literally.
2. Post and maintain a `## Agent Workpad` comment on the issue (WORKFLOW.md Step 1).
3. Transition issue states per the WORKFLOW.md Status Map.
4. For out-of-scope improvements discovered during work, create a separate Backlog issue — do not fix in the current session.

---

## Cost Discipline

- hatice tracks per-session USD cost against the cap defined in VER-34.
- If a session approaches the cost cap, stop work and post a `### Cost Cap Approaching` note to the issue workpad.
- Prefer focused, scoped edits over speculative refactors.
