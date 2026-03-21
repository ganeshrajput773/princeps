---
name: execute
description: >
  Use this skill when the user says "/execute", "run today's task", "do the work",
  "execute the plan", "what's Claude doing today", "run Princeps", or when a scheduled
  Princeps execution fires automatically. This is the autonomous engine of Princeps —
  Claude reads the day's plan, actually does the work, saves outputs, and updates the log.
  Also trigger when the user says "continue the project" or "run [project name]".
metadata:
  version: "0.3.0"
  author: "Rajveer Kapoor"
---

# /execute

The heart of Princeps. Claude reads the plan, does the work, saves outputs, logs what
happened.

Read `../new-project/references/project-schema.md` for data file path and schema.

---

## Step 1 — Load projects and find today's work

```bash
TODAY=$(date +%Y-%m-%d)
SESSION=$(ls /sessions/ | grep -v lost 2>/dev/null | head -1)
DATA_FILE="/sessions/$SESSION/mnt/claude-cowork/Scheduled/princeps-projects.json"
```

Read the data file. For each project where `status == "Active"`:
1. Find the dayPlan entry where `date == TODAY` and `done == false`.
2. If found → this project has work due today.
3. If no entry for today but future undone entries exist → skip (not due yet).
4. If all entries done → mark `status = "Done"`, `pct = 100`.
5. If today is past the last day and goal not met → trigger a **goal recovery run**.

If user named a specific project ("/execute social-media"), only run that one.
Otherwise run all active projects with work due today.

---

## Step 2 — Autonomous vs manual: who does what

Each dayPlan entry has an `autonomous` field:

| `autonomous` | Executed by | When |
|---|---|---|
| `true` | Daily cron schedule | 9am automatically |
| `false` | the user runs `/execute` | Manual trigger |

**When `/execute` is triggered manually by the user:**
- Look for today's task where `autonomous: false` and `done: false` → **execute it**
- Also execute any `autonomous: true` task that the schedule missed (done: false, date ≤ today)
- Log `"triggeredBy": "manual"` in the execution record

**When the scheduled cron calls `/execute`:**
- Only execute today's task if `autonomous: true`
- If today's task is `autonomous: false`: skip it and log:
  `"Day N task is manual — the user must run /execute to proceed."`
- Log `"triggeredBy": "schedule"` in the execution record

---

## Step 3 — Execute the day's task

**Do the actual work.** This is not a planning step — it's execution:

- Read the `task` and `deliverable` fields for the day.
- Check `$DATA_DIR/<project-id>/` for prior day outputs and build on them.
- Execute fully:
  - **Writing**: produce the actual text, not an outline or placeholder.
  - **Research**: actually research and write findings.
  - **Code**: write working code, not pseudocode.
  - **Drafts**: produce the full draft, ready for review.
  - **Queues**: write all queued items, not summaries.
- Save output to `$DATA_DIR/<project-id>/day-<N>-<slug>.md` (or appropriate extension).

**The goal is the north star.** If today's written task doesn't fully serve the goal,
adapt it. The goal must be reached — the plan is a means, not a constraint.

For **manual tasks** (autonomous: false), Claude still writes and saves the full
deliverable. The distinction is that the *output going live* (posting, sending) is
what requires the user's action — the *creation* of the content is Claude's job.

---

## Step 4 — Goal check and plan adjustment

After executing, ask honestly: **will the project reach its goal if the remaining
plan is followed?**

- If the plan is missing something → update the remaining dayPlan entries now.
- If today's work revealed a blocker → set `status = "Blocked"`, write blocker to `notes`.
- If no adjustments needed → continue.

Never leave a stale plan in place.

---

## Step 5 — Update the data file

Mark today's dayPlan entry `done: true`.

Append to `executions`:
```json
{
  "day": <N>,
  "date": "<TODAY>",
  "summary": "One paragraph — what was done, how, what file was produced",
  "outputFiles": ["<project-id>/day-N-filename.md"],
  "goalProgress": "Honest one-sentence: how close is the project to its goal now?",
  "adjustments": "Any changes to remaining plan, or 'None'",
  "triggeredBy": "manual | schedule"
}
```

Add new output files to the top-level `outputFiles` array.

Recalculate `pct`:
```python
pct = round(100 * sum(1 for d in dayPlan if d["done"]) / max(len(dayPlan), 1))
```

If pct == 100 and goal is met: `status = "Done"`.

Set `lastUpdated = TODAY`. Write the full JSON back.

---

## Step 6 — Confirm with a summary card

```
✅ [Project Name] — Day N of M complete  [⚡ auto | ↗ manual]
Task: [what was done]
Output: [computer:// link to file]
Goal progress: [one sentence]
[If adjustments]: Plan adjusted: [what changed]
[If blocked]: 🚫 Blocked: [what you need from the user]
```

If Done:
```
🏁 [Project Name] — GOAL REACHED
[Goal text]
All outputs: [list with computer:// links]
```

If today's task was manual and needed the user:
```
↗ [Project Name] — Day N manual task complete
[deliverable file linked]
Next: [what you need to do with this draft]
```

---

## Goal Recovery Run (overdue projects)

If today is past the last planned day and goal is not met:
1. Read all prior execution summaries.
2. Assess the gap: what still needs to happen?
3. Append new dayPlan entries for today onward with `autonomous` classified appropriately.
4. Execute today's new task immediately.
5. Log `"adjustments": "Goal recovery run — extended plan by N days"`.

The goal cannot be abandoned. If truly unreachable (needs external data only the user
can provide), set `status = "Blocked"` with a specific explanation.

---

## Notes

- Never skip work just because a task is hard or vague. Make a reasonable interpretation.
- Don't truncate outputs — produce the full deliverable.
- Always check `$DATA_DIR/<project-id>/` for prior work before starting.
- Multiple active projects: execute in priority order (High first).
