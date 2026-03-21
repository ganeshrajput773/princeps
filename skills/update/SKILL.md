---
name: update
description: >
  Use this skill when the user says "/update", "update project X", "pause X",
  "resume X", "X is blocked", "change the goal of X", "add a day to X",
  "adjust the plan for X", "mark X done", or wants to manually override
  anything in a Princeps project. This is the manual override hatch — for when
  the user wants to intervene in what Claude is doing autonomously.
metadata:
  version: "0.4.0"
  author: "Rajveer Kapoor"
---

# /update

Manual override for any Princeps project. Also runs two **automatic syncs on
every call** before applying any manual change: automation status sync and
file-based progress detection.

Read `../new-project/references/project-schema.md` for data file path and schema.

---

## Step 0 — Automatic syncs (always run first, on every /update call)

Run both syncs silently before touching anything the user asked to change.

### 0a — Automation status sync

Call `list_scheduled_tasks` to get live state for all scheduled tasks.

For each task, determine its today-status:

| Condition | Status |
|-----------|--------|
| `lastRunAt` is today, session transcript shows clean completion | `ran_clean` |
| `lastRunAt` is today, transcript shows partial/mid-step cutoff | `partial` |
| `lastRunAt` is today, transcript shows rate limit hit | `rate_limited` |
| No `cronExpression` (manual-only) | `manual` |
| `nextRunAt` is today but hasn't run yet | `upcoming` |
| `lastRunAt` not today, `nextRunAt` is a future date | `upcoming` |


Then update `princeps-status.html` — Automations tab only:
- Replace each `.auto-row` status dot and label with the live data.
- Update timeline SVG dot colours: green=ran_clean, amber=partial, red=rate_limited, grey=upcoming/manual.
- Rewrite the four summary strip numbers (total, ran clean, partial/rate-limited, upcoming).
Write the updated HTML back.

### 0b — File-based progress sync

For each active project in `princeps-projects.json`:

1. Determine the project output dir: `$DATA_DIR/<project-id>/`
2. List all files present in that directory.
3. For each `dayPlan` entry where `done == false`:
   - Parse the filename(s) from the `deliverable` field.
     Examples: `"aigm/aigm-architecture.md"` → look for `aigm-architecture.md`
               `"aigm/main.py + aigm/agent_base.py"` → look for both files
   - If **all primary deliverable files for that day exist on disk** → set `done: true`.
4. Recalculate `pct`:
   ```
   pct = round(100 * done_count / max(total_days, 1))
   ```
5. If `pct == 100` and project was `Active` → set `status = "Done"`.

This catches days completed by Claude Code, external agents, or the user themselves
that wrote files without updating the JSON.

Write the synced JSON back after both 0a and 0b complete.

---

## Step 1 — Identify the project (for manual changes)

If the user named a project or requested a change, fuzzy-match to the project.
"essay" matches "My Scholarship Essay". If ambiguous, show a short list.

If the user said "all" or only asked for a status update with no specific
project change, skip to Step 4.

---

## Step 2 — Determine what to update

| What they say | What to change |
|---------------|----------------|
| "pause X" | `status = "Paused"` |
| "resume X" | `status = "Active"` |
| "X is blocked" / "stuck on X" | `status = "Blocked"`, write blocker to `notes` |
| "X is done" / "goal reached" | `status = "Done"`, `pct = 100`, mark all dayPlan done |
| "change the goal to..." | Update `goal` field, ask if plan needs adjusting |
| "add N more days" | Append N new dayPlan entries. Classify each autonomous/manual. |
| "change today's task to..." | Update the matching dayPlan entry `task` field |
| "skip day N" | Mark that dayPlan entry `done: true`, log "Skipped by user" in executions |
| "add a note: ..." | Append to `notes` |
| "make day N autonomous" | Set `dayPlan[N-1].autonomous = true` |
| "make day N manual" | Set `dayPlan[N-1].autonomous = false` |
| Priority / deadline changes | Update the relevant field directly |

Multiple updates in one message: apply all in a single write.


---

## Step 3 — Write back

Apply manual changes on top of the auto-sync results. Set `lastUpdated` to today.
Write the full JSON back in a single write.

---

## Step 4 — Confirm

```
✅ Auto-synced
  • Automations: [N] statuses refreshed in dashboard
  • Progress:    [list any days newly marked done via file scan]
                 e.g. "AIGM Day 1 → done (aigm-architecture.md found)"
                 or   "No new completions detected"

✅ [Project Name] updated        ← only shown if a manual change was applied
  [one line per change]
```

Then: `Run /status to see your full dashboard.`

---

## Notes

- Never delete projects. To remove: set `status = "Archived"`.
- Never delete dayPlan entries or executions — they're the project's history.
- If the goal changes significantly, offer to regenerate remaining dayPlan entries.
- When adding new days, always classify each as `autonomous: true/false`.
- Step 0 syncs are silent and automatic — never ask for confirmation before running them.
- If `list_scheduled_tasks` is unavailable, skip Step 0a and note it in the confirm block.
- File scan (Step 0b) is non-destructive: only flips `done` from false → true, never the reverse.
- For deliverables listed as "X + Y" (multiple files), all must exist to mark done.
