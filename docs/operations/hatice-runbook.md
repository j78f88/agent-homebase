# Hatice Operations Runbook

**Audience:** Operator / CTO (Joshua / Hatice)
**Scope:** Day-to-day operation of the Hatice orchestration daemon on the Windows workstation at `D:\Hatice`.
**Last updated:** 2026-05-14

---

## Table of Contents

1. [Start](#start)
2. [Stop](#stop)
3. [Watch](#watch)
4. [Intervene](#intervene)
5. [Recover](#recover)
6. [When to Call for Help](#when-to-call-for-help)

---

## Start

### Prerequisites

Before launching Hatice, confirm the following are in place:

| Item | Expected location / value |
|---|---|
| Environment file | `D:\Hatice\.env` — see [VER-34](https://linear.app/verkv2/issue/VER-34) for required keys (`LINEAR_API_KEY`, `ANTHROPIC_API_KEY` / Azure equivalents, `GITHUB_TOKEN`, `HATICE_COST_CAP_USD`, etc.) |
| Workspace root | `D:\Hatice` exists and is writable |
| Hatice binary | `D:\Hatice\node_modules` populated (run `npm install` once if missing) |
| `WORKFLOW.md` | `D:\Hatice\WORKFLOW.md` present — this is the agent instruction file |
| Node / npx | `node --version` ≥ 18 in the same shell |

Quick pre-flight:

```powershell
Test-Path D:\Hatice\.env              # must be True
Test-Path D:\Hatice\WORKFLOW.md       # must be True
node --version                        # must be >= v18
npx tsx --version                     # must succeed
```

### Command to Launch

Open a PowerShell window and run:

```powershell
cd D:\Hatice
.\start-hatice.ps1
```

`start-hatice.ps1` sources `.env` into the current process environment and then executes:

```powershell
npx tsx bin/hatice.ts start -w ./WORKFLOW.md
```

Do **not** close this window while Hatice is running (unless you set up a persistent daemon — see below).

### Confirming It Is Running

1. **Dashboard** — open <http://127.0.0.1:4000> in a browser. You should see the Hatice SSE dashboard with an "active sessions" pane and a cost counter.

2. **Log lines to expect** on startup:

   ```
   [hatice] Config loaded from D:\Hatice\.env
   [hatice] Workflow loaded from D:\Hatice\WORKFLOW.md (hash: <sha>)
   [hatice] SSE dashboard listening on http://127.0.0.1:4000
   [hatice] Supervisor ready — starting poll loop
   [hatice] Poll cycle 1 — querying Linear for Todo issues…
   ```

3. **First poll cycle** — within a few seconds you should see a line like:

   ```
   [hatice] Poll cycle 1 complete — N issues eligible, M dispatched
   ```

   If `N=0` that is normal when there are no `Todo` issues in the configured Linear project.

### Keeping Hatice Running Across Terminal Restarts (Daemon Mode)

Choose one method:

#### Option A — Task Scheduler (recommended for Windows)

```powershell
# Run once as Administrator
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
             -Argument "-NonInteractive -File D:\Hatice\start-hatice.ps1" `
             -WorkingDirectory "D:\Hatice"
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -RestartCount 3 -RestartInterval (New-TimeSpan -Minutes 1)
Register-ScheduledTask -TaskName "Hatice" -Action $action -Trigger $trigger `
  -Settings $settings -RunLevel Highest -Force
```

Start it immediately without logging out:

```powershell
Start-ScheduledTask -TaskName "Hatice"
```

View its status:

```powershell
Get-ScheduledTask -TaskName "Hatice" | Select-Object State, LastRunTime, LastTaskResult
```

#### Option B — NSSM (Non-Sucking Service Manager)

```powershell
nssm install Hatice powershell.exe
nssm set Hatice AppParameters "-NonInteractive -File D:\Hatice\start-hatice.ps1"
nssm set Hatice AppDirectory D:\Hatice
nssm set Hatice AppStdout D:\Hatice\logs\nssm-stdout.log
nssm set Hatice AppStderr D:\Hatice\logs\nssm-stderr.log
nssm set Hatice Start SERVICE_AUTO_START
nssm start Hatice
```

#### Option C — Foreground with `screen` equivalent (WSL)

If you prefer to stay in WSL:

```bash
# Inside WSL
screen -S hatice
cd /mnt/d/Hatice
pwsh ./start-hatice.ps1
# Detach: Ctrl+A then D
# Reattach: screen -r hatice
```

---

## Stop

### Graceful Shutdown

In the PowerShell window where Hatice is running, press **Ctrl+C**.

Hatice intercepts `SIGINT` / `SIGTERM` and:
1. Stops accepting new dispatches from the poll loop.
2. Waits for all in-flight agent sessions to complete their current tool call and reach a safe checkpoint.
3. Flushes pending log buffers.
4. Exits with code `0`.

Expected log lines during graceful shutdown:

```
[hatice] Shutdown signal received — draining N in-flight session(s)…
[hatice] Session <id> completed cleanly.
[hatice] All sessions drained. Goodbye.
```

This can take seconds to minutes depending on what the agents are doing. **Do not force-kill unless necessary.**

If running as a Task Scheduler service:

```powershell
Stop-ScheduledTask -TaskName "Hatice"
```

### Hard Stop (Kill PID)

Find the PID:

```powershell
# Find the node process listening on port 4000
netstat -ano | findstr ":4000"
# Note the PID from the last column, then:
Get-Process -Id <PID>
```

Kill it:

```powershell
Stop-Process -Id <PID> -Force
```

**What cleanup is skipped:**
- In-flight agent sessions are abandoned mid-run. Their Linear issues remain `In Progress` and must be manually reset (see [Intervene → Move an issue back to Backlog](#move-an-issue-back-to-backlog-to-remove-it-from-the-queue)).
- Pino log buffers may not be fully flushed; the last few lines of each session log might be missing.
- Any workspace clones being written at the moment of kill may be left in a dirty state (see [Recover → Workspace corruption](#workspace-corruption-delete-and-re-clone)).

### Pause Dispatch Without Stopping Running Sessions

Hatice supports a soft-pause that stops new dispatches while letting current sessions finish:

```powershell
# Stop new dispatches
Invoke-RestMethod -Uri http://127.0.0.1:4000/control/pause -Method POST
```

To resume:

```powershell
Invoke-RestMethod -Uri http://127.0.0.1:4000/control/resume -Method POST
```

The dashboard badge changes from **RUNNING** to **PAUSED** when paused. The poll loop continues but no new sessions are started.

---

## Watch

### SSE Dashboard

Open <http://127.0.0.1:4000> in any browser.

| Panel | What it shows |
|---|---|
| **Active sessions** | One row per running agent — issue ID, current tool call, elapsed time, cost so far |
| **Cost summary** | Running total in USD, cap remaining |
| **Recent events** | Last 50 SSE events in real-time |
| **Poll log** | Last 10 poll-cycle summaries |

The dashboard requires no authentication (local-only by default). Do not expose port 4000 externally.

### Reading Per-Session Pino NDJSON Logs

Each session writes a structured NDJSON log to:

```
D:\Hatice\logs\sessions\<issue-id>-<timestamp>.ndjson
```

Stream it live while a session runs:

```powershell
Get-Content "D:\Hatice\logs\sessions\VER-35-*.ndjson" -Wait |
  ForEach-Object { $_ | ConvertFrom-Json | Format-List }
```

Or pipe through `pino-pretty` if installed:

```powershell
Get-Content "D:\Hatice\logs\sessions\VER-35-*.ndjson" -Wait | npx pino-pretty
```

Key log fields:

| Field | Meaning |
|---|---|
| `level` | 30=info, 40=warn, 50=error |
| `msg` | Human-readable message |
| `sessionId` | UUID for this session |
| `issueId` | Linear issue identifier |
| `toolName` | Currently executing tool |
| `cost` | Cumulative USD cost at this log line |
| `commitSha` | Most recent git commit made by this session |

### Cost Monitoring

**Live:** The dashboard shows a running total updated every SSE tick.

**Rolled-up daily totals:**

```powershell
# Summarise maximum cost reached across all session logs today
Get-ChildItem D:\Hatice\logs\sessions\*.ndjson |
  Where-Object { $_.LastWriteTime.Date -eq (Get-Date).Date } |
  ForEach-Object { Get-Content $_ } |
  ConvertFrom-Json |
  Where-Object { $_.cost } |
  Measure-Object -Property cost -Maximum |
  Select-Object Maximum
```

The global cost cap is set in `.env` as `HATICE_COST_CAP_USD`. When the cap is reached, Hatice stops dispatching new sessions and posts a warning on the dashboard. Existing in-flight sessions are not killed.

### Linear-Side View of Running / Completed / Retrying Issues

In the Linear board for the Verk v2 project:

- **In Progress** column — all issues currently being worked by Hatice.
- **Done** column — completed issues; each should have a PR link in the issue comments.
- Issues with a `retry` label — sessions that failed and are queued for automatic retry.

Filter by assignee **Hatice** to see only automation-driven work.

---

## Intervene

### Stop a Single In-Flight Session

```powershell
# Find the session ID from the dashboard or logs, then:
Invoke-RestMethod -Uri "http://127.0.0.1:4000/sessions/<session-id>/stop" -Method POST
```

This sends a graceful stop to that session only. The session drains its current tool call and exits. The Linear issue is left `In Progress` — move it back to Backlog if you do not want it retried (see below).

### Mark an Issue Done Manually (Bypass Hatice)

In Linear:

1. Open the issue.
2. Change its state to **Done**.
3. Optionally add a comment explaining the manual closure.

Hatice will not re-dispatch an issue already in a terminal state (`Done`, `Canceled`, `Duplicate`).

### Move an Issue Back to Backlog to Remove It from the Queue

In Linear:

1. Open the issue.
2. Change its state to **Backlog** to park it without immediate re-dispatch, or **Todo** to allow re-dispatch in the next poll cycle.

> Hatice's poll query targets issues in the **Todo** state only. Issues in **Backlog**, **In Progress** (already claimed), or terminal states are skipped.

### Update `WORKFLOW.md` on the Fly

Hatice watches `D:\Hatice\WORKFLOW.md` for changes using mtime + SHA-256 hash.

1. Edit the file:

   ```powershell
   code D:\Hatice\WORKFLOW.md   # VS Code
   # or
   notepad D:\Hatice\WORKFLOW.md
   ```

2. Save the file.

3. Hatice detects the change at the next poll tick and logs:

   ```
   [hatice] WORKFLOW.md changed (hash: <new-sha>) — hot-reloading
   ```

4. New sessions started after the reload use the updated workflow. Already-running sessions are unaffected.

No restart required.

---

## Recover

### Crash Recovery (Supervisor Wrapper)

Hatice ships with a built-in Supervisor that automatically restarts the daemon after an unexpected crash (exit code ≠ 0).

On crash you will see:

```
[supervisor] Hatice exited with code 1 — restarting in 5 s (attempt 1/5)
```

After 5 consecutive failures the Supervisor backs off to a 60-second retry interval and posts a Linear comment on the most recently active issue.

If you are running Hatice via Task Scheduler or NSSM, those provide an independent restart layer on top.

To force a clean restart manually:

```powershell
Stop-Process -Name "node" -Force       # hard kill if needed
Start-ScheduledTask -TaskName "Hatice" # or just re-run .\start-hatice.ps1
```

### Failed Session Retry from a Known-Good Commit

1. Identify the failed session's last committed SHA from the session log:

   ```powershell
   Get-Content "D:\Hatice\logs\sessions\<issue-id>-*.ndjson" |
     ConvertFrom-Json | Where-Object { $_.commitSha } | Select-Object -Last 1
   ```

2. In the session's workspace directory (path logged as `workspaceDir`), inspect history and reset if needed:

   ```powershell
   cd <workspaceDir>
   git log --oneline -10
   git checkout <known-good-sha>   # review before re-dispatch
   ```

3. Move the Linear issue back to **Todo** to allow Hatice to re-dispatch it with a fresh session:
   - In Linear: change state to **Todo**.
   - Or via management API: `Invoke-RestMethod -Uri "http://127.0.0.1:4000/issues/<issue-id>/retry" -Method POST`

### Workspace Corruption (Delete and Re-Clone)

Each session uses an isolated workspace clone. If it is corrupted:

1. Note the `workspaceDir` from the session log.
2. Stop the session (see [Intervene](#intervene)).
3. Delete the corrupt workspace:

   ```powershell
   Remove-Item -Recurse -Force <workspaceDir>
   ```

4. Move the issue back to **Todo**. Hatice creates a fresh clone on next dispatch.

If the **main** Hatice install directory (`D:\Hatice`) itself is corrupt:

```powershell
# Back up secrets first!
Copy-Item D:\Hatice\.env C:\Backup\hatice.env.bak

# Re-clone and reinstall
Remove-Item -Recurse -Force D:\Hatice
git clone https://github.com/j78f88/agent-homebase D:\Hatice
cd D:\Hatice
npm install

# Restore secrets
Copy-Item C:\Backup\hatice.env.bak D:\Hatice\.env
```

### Rate Limit Recovery

**Linear rate limits** — Hatice backs off automatically with exponential retry (logged as `[hatice] Linear rate limit — retrying in Xs`). No operator action required unless the limit persists for more than 10 minutes (indicates misconfiguration or a runaway poll loop).

**Anthropic / Azure AI Foundry rate limits** — If you see `429 Too Many Requests` in session logs:

1. Check how many concurrent sessions are running on the dashboard.
2. Pause dispatch (`POST /control/pause`) until current sessions drain.
3. If the limit is a daily token cap, wait until the reset time (typically midnight UTC for Azure).

Relevant log pattern to watch for:

```
[session] LLM request failed: 429 — will retry in 30 s (attempt 1/3)
```

After 3 retries the session fails and Hatice marks the issue for retry.

### Credential Rotation Procedure

#### Linear API Key

1. In Linear: **Settings → API → Personal API keys** — revoke the old key and generate a new one.
2. Update `.env`:

   ```powershell
   (Get-Content D:\Hatice\.env) -replace 'LINEAR_API_KEY=.*', "LINEAR_API_KEY=<new-key>" |
     Set-Content D:\Hatice\.env
   ```

3. Restart Hatice (graceful shutdown, then re-launch). Hatice reads `.env` only at startup.

#### Anthropic API Key via Azure AI Foundry

> **Note:** Joshua plans to rotate the Azure key after the first smoke test (per [VER-30](https://linear.app/verkv2/issue/VER-30) ADR exit criteria). Follow this procedure at that time.

1. In the Azure AI Foundry portal: navigate to your Hatice deployment → **Keys and Endpoint** → regenerate **Key 1** (keep Key 2 active during transition to avoid a gap).
2. Update `.env`:

   ```powershell
   (Get-Content D:\Hatice\.env) -replace 'AZURE_OPENAI_API_KEY=.*', "AZURE_OPENAI_API_KEY=<new-key>" |
     Set-Content D:\Hatice\.env
   ```

3. Quick smoke test with a low-cost issue before fully committing.
4. Once confirmed working, revoke the old Azure key.
5. Restart Hatice.

#### GitHub Token

1. In GitHub: **Settings → Developer settings → Personal access tokens** — generate a new token with `repo` scope (add `workflow` if Hatice triggers GitHub Actions CI).
2. Update `.env`:

   ```powershell
   (Get-Content D:\Hatice\.env) -replace 'GITHUB_TOKEN=.*', "GITHUB_TOKEN=<new-token>" |
     Set-Content D:\Hatice\.env
   ```

3. Restart Hatice.

**After any credential rotation:** verify on the dashboard that the next poll cycle completes without auth errors before considering the rotation complete.

---

## When to Call for Help

The following situations exceed normal operator recovery and require deeper investigation or escalation.

### Cost Cap Exceeded with No Visible Progress

**Symptoms:** Dashboard shows cost at or near `HATICE_COST_CAP_USD`; no issues are moving to Done; agents appear to be looping on the same tools.

**Steps:**
1. Pause dispatch: `POST http://127.0.0.1:4000/control/pause`
2. Open the active session log for the looping agent; look for repeated identical tool calls.
3. Stop the looping session manually.
4. Post a comment on the affected Linear issue describing the loop behaviour so the next attempt can avoid it.
5. Only raise `HATICE_COST_CAP_USD` after understanding why the cap was hit.

### Hatice Itself Crashes Repeatedly

**Symptoms:** Supervisor log shows multiple restart attempts; Hatice does not stabilise; dashboard is unreachable at `http://127.0.0.1:4000`.

**Steps:**
1. Stop the Task Scheduler job: `Stop-ScheduledTask -TaskName "Hatice"`
2. Examine `D:\Hatice\logs\hatice-error.log` for the root-cause stack trace.
3. Common causes: malformed `.env`, incompatible Node version, disk full, port 4000 already in use.
4. If the cause is not obvious, collect logs and open a GitHub issue on the Hatice repository.

### An Agent Commits Something Destructive That Escapes the Workspace

**Symptoms:** Unexpected commits or force-pushes appear on protected branches in `j78f88/agent-homebase`; files deleted outside the expected issue scope.

**Steps:**
1. Hard-stop Hatice immediately: `Stop-Process -Name "node" -Force`
2. In GitHub, identify the bad commits and revert:

   ```bash
   git revert <bad-sha>
   git push
   ```

3. If a force-push destroyed history, use `git reflog` on a local clone and contact GitHub Support if the remote refs need restoring.
4. Review `D:\Hatice\WORKFLOW.md` to add guardrails that prevent the destructive action.
5. Do not restart Hatice until the root cause is understood and the workflow is updated.

### Azure AI Foundry Endpoint Returns Auth Errors After `.env` Values Look Correct

**Symptoms:** Session logs show `401 Unauthorized` or `403 Forbidden` from the Azure endpoint even after a recent credential rotation and the values in `.env` appear correct.

This is a known risk documented in the [VER-30 ADR exit criteria](https://linear.app/verkv2/issue/VER-30): the Azure AI Foundry proxy may not be fully API-compatible with the Anthropic SDK.

**Steps:**
1. Pause Hatice dispatch.
2. Test the endpoint directly without Hatice:

   ```powershell
   $key = (Get-Content D:\Hatice\.env | Select-String "AZURE_OPENAI_API_KEY").ToString().Split("=")[1]
   $endpoint = (Get-Content D:\Hatice\.env | Select-String "AZURE_OPENAI_ENDPOINT").ToString().Split("=")[1]
   $headers = @{ "api-key" = $key }
   Invoke-RestMethod -Uri "$endpoint/openai/deployments?api-version=2024-02-01" -Headers $headers
   ```

3. If that call **also** fails, the issue is with the Azure deployment itself (wrong endpoint URL, key not propagated, subscription quota exceeded). Check the Azure portal status.
4. If the direct call **succeeds** but Hatice still fails, the Anthropic SDK may be sending headers or a payload shape that Azure rejects. Check the Hatice GitHub issues for known Azure compatibility bugs and apply any available workaround patch.
5. **Fallback:** switch `.env` to use the direct Anthropic API (`ANTHROPIC_API_KEY`) if Azure remains incompatible. This satisfies the VER-30 ADR exit criterion and unblocks the team while the Azure proxy issue is investigated separately.
