# Hatice Operations Config

Operational parameters for the live hatice deployment against the `j78f88/agent-homebase` repository.
Last updated: 2026-05-14 (VER-34).

---

## Secrets

All secrets are stored as **system environment variables** on the host machine — not in `.env` files committed to the repo.

| Variable | Purpose | Location |
|---|---|---|
| `ANTHROPIC_API_KEY` | Claude API access for dispatched agents | System env var (set 2026-05-13) |
| `LINEAR_API_KEY` | Read issues, post comments, transition states | System env var |
| `GH_TOKEN` | Push branches, open PRs from dispatched workspaces | System env var |

> **Policy:** Actual secret values must never appear in the repository. See `env.example` in the repo root for the required variable names.

---

## Caps and Limits

### Per-session cost cap
- **Hard cap:** $3.00 USD per session
- **Model:** `claude-sonnet-4-6` (default); `claude-opus-4-6` for issues labelled `complex` or `orchestration`
- Agents must monitor cumulative cost and post a `### Cost Cap Approaching` workpad note when approaching $10 (daily envelope; see below); they must stop work if the per-session hard cap is reached.

### Daily cost envelope
- **Total daily cap:** $30.00 USD across all parallel sessions
- This covers up to 3 concurrent agents running full sessions.

### Concurrency
- **Max concurrent agents:** 3
- Tune upward after VER-31 cost data is available.

### Turns per session
- **Max turns:** 30 per dispatched session

### Model selection
| Condition | Model |
|---|---|
| Default | `claude-sonnet-4-6` |
| Issue labelled `complex` or `orchestration` | `claude-opus-4-6` |

---

## Workspace Policy

| Parameter | Value |
|---|---|
| Workspace root | `~/workspaces/verk-v2/` |
| Per-issue workspace | `~/workspaces/verk-v2/<ISSUE-ID>/` (e.g. `VER-34/`) |
| Cleanup policy | Age-based; stale workspaces older than **7 days** are removed |
| Branch naming | `<ISSUE-ID>` — e.g. `VER-34` (lowercase if required by host git) |
| Push target | Feature branch; never push directly to `main` |
| PR policy | Open PR against `main`; apply `hatice` label; require Human Review state transition before merge |

> PR flow confirmed in ADR (VER-30): agent opens PR → Human Review → merge.

---

## Observability

### SSE Dashboard
- **Port:** 4000 (localhost only; do not expose externally)
- URL: `http://localhost:4000`

### Per-session logs
- **Format:** NDJSON (newline-delimited JSON)
- **Location:** `~/workspaces/verk-v2/logs/<ISSUE-ID>-<timestamp>.ndjson`
- Each log entry includes: timestamp, session ID, issue ID, model, tokens used, cumulative cost USD, turn count.

### Alerting
| Event | Action |
|---|---|
| Session completed successfully | Post workpad comment with summary + cost |
| Session failed / errored | Post workpad comment with error details; leave issue in current state |
| Cost cap breach ($3/session) | Post `### Cost Cap Approaching` workpad note; halt session |
| Daily envelope exceeded ($30) | Block new session spawning until next UTC day |

---

## Rate Limits

### Anthropic API
- **Budget allocation:** hatice sessions share the same API key as normal Claude Code use. Treat rate limit headroom as a shared resource.
- **Tier assumption:** Tier 2 or higher (adjust if 429s become frequent).

### Backoff policy on 429s
hatice has built-in retry tracking. Default behaviour:
1. On first 429: wait 60 seconds, retry.
2. On second 429 in same session: wait 120 seconds, retry.
3. On third 429: post a workpad note `### Rate Limit — paused`; halt session and retry on next scheduled run.

---

## Cost Discipline Rules (CLAUDE.md supplement)

The following rules apply to every agent session:

1. Log token counts and estimated cost after each tool-use batch.
2. If cumulative session cost reaches **$2.50**, append a `### Cost Cap Approaching` section to the workpad comment and complete only the current atomic step before stopping.
3. If cumulative session cost reaches **$3.00**, stop immediately; do not start new steps.
4. Report final cost in the workpad closing summary.

---

## Acceptance Checklist (VER-34)

- [x] Secrets confirmed as system env vars (`ANTHROPIC_API_KEY`, `LINEAR_API_KEY`, `GH_TOKEN`)
- [x] Per-session cost cap set ($3.00)
- [x] Daily cost envelope set ($30.00)
- [x] Max concurrent agents: 3
- [x] Max turns per session: 30
- [x] Model selection rules documented
- [x] Workspace root confirmed (`~/workspaces/verk-v2/`)
- [x] Cleanup policy: 7-day age threshold
- [x] Branch naming: `<ISSUE-ID>`
- [x] PR policy: feature branch → PR with `hatice` label → Human Review
- [x] SSE dashboard port: 4000 (localhost)
- [x] NDJSON log location defined
- [x] Alerting events documented
- [x] Backoff policy on 429s documented
- [x] `env.example` created in repo root (note: `.env.example` is gitignored by the repo's `.gitignore`; file is named `env.example`)
