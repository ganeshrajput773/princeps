# Princeps

**Autonomous project execution engine for Claude Cowork.**

Princeps lets you hand off long-running goals to Claude and have it work on them day by day — automatically. You define the goal, Claude drafts the plan, and then it executes one task per day, saves deliverables to your folder, and keeps a live kanban dashboard up to date.

## What it does

- `/new-project` — Tell Claude what you want to achieve and how many days to work on it. Claude drafts a day-by-day execution plan, classifies each task as autonomous (runs automatically) or manual (requires you), and sets up a daily schedule.
- `/execute` — Run today's task manually, or catch up on missed days. Claude does the actual work — writing, researching, coding, drafting — and saves the output to your folder.
- `/status` — Renders a live kanban dashboard with one column per project, pixel characters, progress bars, and deadline countdowns.
- `/update` — Pause, resume, adjust goals, add days, or mark projects done. Also auto-syncs scheduled task statuses and detects completed deliverables from disk.

## How it works

Each project gets a day-by-day plan stored in `princeps-projects.json`. Autonomous tasks run on a 9am daily cron schedule. Manual tasks wait for you to run `/execute`. Every execution saves output files and logs progress. The kanban dashboard is regenerated automatically after every change.

## Requirements

- Claude Cowork (desktop app)
- A workspace folder selected in Cowork

## Installation

Install via the Claude plugin directory or download the `.plugin` file and upload it manually in Cowork settings.

## Skills

| Skill | Trigger |
|-------|---------|
| `/princeps:new-project` | Add a new autonomous project |
| `/princeps:execute` | Run today's tasks |
| `/princeps:status` | View the kanban dashboard |
| `/princeps:update` | Adjust or override any project |

## Author

[Rajveer Kapoor](https://github.com/RajveerKapoor)

## License

MIT
