---
name: new-project
description: >
  Use this skill when the user says "/new-project", "add a project", "track a new project",
  "I want Claude to work on X", "give Claude a project", "start a project", or describes
  something they want Claude to autonomously execute over multiple days with a goal.
  This skill is the entry point for Princeps — it captures what the user wants done,
  drafts a day-by-day execution plan, classifies tasks as autonomous or manual,
  creates schedules for autonomous tasks, and commits Claude to reaching the goal.
metadata:
  version: "0.4.0"
  author: "Rajveer Kapoor"
---

# /new-project

Register a new autonomous project in Princeps. Claude doesn't just track this —
Claude will actually *execute* it, day by day, until the goal is reached.

Read `references/project-schema.md` for the full data schema and file path logic.

---

## Step 1 — Capture the goal and timeline

Ask the user for the essentials using AskUserQuestion (one round only):

| Field | What to ask | Notes |
|-------|-------------|-------|
| `goal` | "What's the outcome that MUST be achieved?" | Concrete and verifiable. |
| `duration` | "How many days should Claude work on this?" | 1–30 days. |
| `domain` | Infer if obvious, ask if not | writing, research, code, content, analysis, outreach, other |
| `priority` | High / Medium / Low | Ask only if not obvious from context |
| `name` | Short display name | Infer from goal if not given |

Don't ask for tasks — you'll generate those yourself in Step 2.

---

## Step 2 — Draft the execution plan

Generate a day-by-day plan. Each day's task must be something Claude will
actually do. For each task, also classify it as **autonomous** or **manual**.

**Autonomous** (`autonomous: true`) — Claude can execute this fully on its own:
- Research and writing to files
- Drafting content (articles, threads, emails, queues)
- Generating strategy documents, calendars, analyses
- Building code, running scripts, creating deliverables
- Anything Claude can do without user approval or external action

**Manual** (`autonomous: false`) — Requires user involvement before or during:
- Tasks that require approval before going live (e.g., posting to social media)
- Tasks that need the user to click, send, or post something
- Tasks using Chrome MCP to publish/send (not just draft)
- Tasks requiring human judgement or real-world confirmation

Good task examples:
- ✅ "Research top 5 competitors and write a 500-word summary to `research.md`" → autonomous: true
- ✅ "Draft Substack article #1 and save to `post-1-draft.md`" → autonomous: true
- ✅ "Write 7 queued TWIE posts and save to `twie-queue-week1.md`" → autonomous: true
- ⚠️ "Post the drafted article to Substack via Chrome" → autonomous: false
- ⚠️ "Send outreach emails to newsletter partners" → autonomous: false
- ❌ "Work on the essay" (too vague)
- ❌ "User should review" (can't hand off mid-plan)

Rules for a good plan:
- Each day builds on the previous one. Reference prior deliverables explicitly.
- The last day must be a **verification day** — Claude checks whether the goal
  has been met, fills gaps, and declares done or flags blockers.
- Spread the work realistically. Don't front-load everything into day 1.
- Each deliverable must be a file Claude can create (`.md`, `.txt`, `.py`, etc.)
  saved in the project's output folder.

Set `date` for each day starting from today:
```bash
date -d "+N days" +%Y-%m-%d
```

---

## Step 3 — Write to data file

Locate or create the data file (see `references/project-schema.md` for path logic).

Build the project entry:
- `id`: kebab-case of name, e.g. "my-essay"
- `color`: first unused color from the palette
- `pct`: 0
- `startDate`: today
- `deadline`: startDate + duration days
- `addedOn` and `lastUpdated`: today
- `executions`: [] (empty — filled by /execute)
- `outputFiles`: [] (empty — filled by /execute)
- Each `dayPlan` entry MUST include the `autonomous` field

Also create the output folder:
```bash
mkdir -p "$DATA_DIR/<project-id>/"
```

Write the full updated JSON back to the data file.

---

## Step 4 — Create scheduled tasks for autonomous days

After saving the project, create a **daily schedule** that automatically
executes autonomous tasks.

Use the `create_scheduled_task` MCP tool to create ONE recurring schedule:

```
taskName: "princeps-<project-id>-daily"

description: |
  You are the Princeps autonomous execution engine for the project: "<project name>".

  Goal: <full goal text>

  Data file: /sessions/{SESSION}/mnt/claude-cowork/Scheduled/princeps-projects.json
  Project ID: <project-id>

  Every time this task runs:
  1. Read the data file and find the project with id "<project-id>"
  2. Find today's date (bash: date +%Y-%m-%d)
  3. Find the dayPlan entry where date == today AND done == false AND autonomous == true
  4. If found: execute that task fully, save the deliverable to the project's output
     folder, update the dayPlan entry to done: true, add an execution log entry,
     update lastUpdated, and write the data file back.
  5. If today's task is autonomous: false — do NOT execute it. Instead, add a note
     to the execution log: "Day X task is manual — the user must run /execute."
  6. If no task is due today (already done, or it's a non-task day): log "No task
     due today." and exit cleanly.
  7. If all dayPlan entries are done: set project status to "Done" and log
     "Goal verification complete — all tasks finished."

  Output folder: /sessions/{SESSION}/mnt/claude-cowork/Scheduled/<project-id>/

cron: "0 9 * * *"   ← 9am local time, every day
```

This schedule runs at 9am daily and handles all autonomous tasks automatically.
Manual tasks are skipped and flagged — the user runs `/execute` for those.

If the project has NO autonomous tasks at all, skip schedule creation and note
"All tasks are manual — use /execute each day."

---

## Step 5 — Regenerate the kanban dashboard HTML

**This step is mandatory every time a new project is saved.**

After writing the JSON data file, regenerate `princeps-status.html` so the new
project column appears immediately. Follow the exact same process as the
`/status` skill — read ALL projects from the data file and rebuild the full HTML.

The HTML file lives at:
```
/sessions/$SESSION/mnt/claude-cowork/Scheduled/princeps-status.html
```

Follow the full design spec in `../status/SKILL.md`:
- Nova colour palette + DM Sans / Geist Mono fonts
- One vertical kanban column per project (sorted Active → Done)
- Pixel character chosen by `domain` peeking from each column's bottom
- Ghost "add project" column always last
- Page locked to 100vh, tasks scroll inside each column

After writing the file, surface it:
```
[View your Princeps dashboard](computer:///sessions/{SESSION}/mnt/claude-cowork/Scheduled/princeps-status.html)
```

---

## Notes

- If a project with the same name already exists, ask if they want to update it
  and route to /update.
- The goal is a hard commitment. Every day's task should visibly move toward it.
- The autonomous/manual split is the most important classification in this skill.
  When in doubt: if it requires clicking, posting, sending, or human approval
  → manual. If Claude can do it fully in the background → autonomous.
