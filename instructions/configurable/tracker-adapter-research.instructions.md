---
description: "Research outputs (@researcher) in tracker mode — issue-scoped research goes as a Linear comment, project-scoped research goes as a Linear Document. Replaces filesystem writes to {{paths.research}}/. Read alongside researcher.skill.md template and tracker-adapter-core.instructions.md."
applyTo: "**"
---

# Tracker Adapter — Research Outputs

Active when `{{tracker.type}}` is not `"filesystem"`. Research from `@researcher` lives in one of two places depending on scope: issue-scoped research as a Linear comment on the relevant issue; project-scoped research (cross-cutting, multi-issue) as a Linear Document under the project.

## Decide scope first

- **Issue-scoped** — the research informs one specific Linear issue's decision. Post as a comment on that issue.
- **Project-scoped** — the research informs multiple issues, or is reference material with a lifespan beyond any single workstream (e.g. "competitive landscape for makers" surveyed once, referenced repeatedly). Use a Linear Document.

When in doubt, start with issue-scoped (a comment). Promote to a Document later if you find yourself citing the same research from multiple issues.

## Write issue-scoped research

Where filesystem mode creates `{{paths.research}}/<slug>-research.md`:

```
save_comment({
  issueId: "<identifier>",
  body: `## Research — <topic>

**Date:** <iso>
**Scope:** <specific question or segment>

### Apps / sources surveyed
- <app name> — <URL>, <category>, <scale: users/years>

### Patterns found

For each pattern:
- **Name:** <short title>
- **What it is:** <concrete UX or data model — not abstract>
- **Source apps:** <which apps implement it>
- **Adoption scale:** <users, years, success indicators>
- **User complaints:** <reddit/forums/store reviews>
- **Failure mode:** <apps that shipped this and abandoned it>

### Unmet needs observed
<patterns users ask for but no app delivers well>

### Sources
<full citation list with URLs>
`
})
```

Do NOT include a "Recommendations" section. Recommendations are `@pm`'s output, not `@researcher`'s.

## Write project-scoped research

Where filesystem mode might create `{{paths.research}}/<segment>-survey.md` or similar long-lived doc:

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Research: <topic>",
  content: <full research body following the same shape as issue-scoped, just longer>
})
```

Project documents have stable URLs. Cite them from issue comments where relevant.

## Reference a project-scoped doc from an issue

```
save_comment({
  issueId: "<identifier>",
  body: `## Research Reference — <topic>

This issue is informed by [Research: <topic>](<linear-doc-url>) under project verk v2.

### Relevant findings for this issue
- <pattern 1 cited from the document>
- <pattern 2 cited from the document>
`
})
```

Keeps the workflow surface (issue comments) light while pointing at the heavyweight reference.

## Research Index (replaces `{{paths.research}}/INDEX.md`)

Maintain a single Linear Document titled "Research Index" under the project. Lists all project-scoped research with date and topic. Updated whenever a new project-scoped research doc lands. Used by `@researcher` for staleness/overlap checks at session start.

```
save_document({
  project: "{{tracker.project_slug}}",
  title: "Research Index",
  content: `# Research Index

| Date | Topic | Document |
|---|---|---|
| <iso> | <topic> | [<doc title>](<url>) |
...
`
})
```

## Read existing research before starting new work

For issue-scoped: `list_comments({ issueId })` and filter for `## Research —` headers.

For project-scoped: `list_documents({ project: "{{tracker.project_slug}}" })` and look at the Research Index document.

Staleness check: if a topic was researched <30 days ago, do NOT re-run by default. Offer the user a refresh or abandon option per `researcher.skill.md`.

## Anti-patterns specific to research

- Writing a "Recommendations" section in a research record. That's `@pm`'s job. `@researcher` surfaces evidence; it does not recommend.
- Citing popularity without numbers ("very popular", "widely used"). Give figures: users, years, revenue where knowable.
- Skipping the failure mode section because the positive patterns are more interesting. Research without failure data is marketing material.
- Project-scoped research without an update to the Research Index document. Future `@researcher` sessions won't find it.
- Promoting issue-scoped research to project-scoped by copy-paste rather than save_document. Keep one canonical copy.
