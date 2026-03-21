# Princeps Project Schema — v3.1

## Data file location

```bash
SESSION=$(ls /sessions/ | grep -v lost 2>/dev/null | head -1)
DATA_DIR="/sessions/$SESSION/mnt/claude-cowork/Scheduled"
DATA_FILE="$DATA_DIR/princeps-projects.json"
```

---

## Full JSON structure

```json
{
  "version": "3.1",
  "lastUpdated": "YYYY-MM-DD",
  "projects": [
    {
      "id": "kebab-case-slug",
      "name": "Display Name",
      "goal": "The non-negotiable outcome. Stated as a concrete, verifiable result.",
      "domain": "writing | research | code | content | analysis | outreach | other",
      "status": "Active | Paused | Done | Blocked",
      "priority": "High | Medium | Low",
      "color": "#hexcolor",
      "startDate": "YYYY-MM-DD",
      "duration": 7,
      "deadline": "YYYY-MM-DD",
      "pct": 0,
      "scheduleName": "princeps-<project-id>-daily",
      "dayPlan": [
        {
          "day": 1,
          "date": "YYYY-MM-DD",
          "task": "Specific action Claude will take — concrete verb, not vague",
          "deliverable": "Exact output: a file, a draft, a list, a decision",
          "autonomous": true,
          "done": false
        }
      ],
      "executions": [
        {
          "day": 1,
          "date": "YYYY-MM-DD",
          "summary": "What Claude actually did",
          "outputFiles": ["path/relative/to/Scheduled/folder"],
          "goalProgress": "Honest 1-sentence assessment of proximity to goal",
          "adjustments": "Changes made to remaining dayPlan as a result",
          "triggeredBy": "schedule | manual"
        }
      ],
      "outputFiles": ["all output paths accumulated across executions"],
      "notes": "Blockers, context, user instructions",
      "addedOn": "YYYY-MM-DD",
      "lastUpdated": "YYYY-MM-DD"
    }
  ]
}
```

---

## The `autonomous` field (new in v3.1)

Every `dayPlan` entry MUST have an `autonomous` field:

| Value | Meaning | Who executes |
|-------|---------|--------------|
| `true` | Claude can do this fully on its own | Daily schedule at 9am |
| `false` | Requires user approval, posting, or real-world action | the user runs `/execute` |

**Autonomous examples:**
- Research and writing to files
- Drafting content (articles, threads, email templates, post queues)
- Generating strategies, calendars, competitor analyses
- Writing scripts, processing data, building deliverables
- Weekly review reports and progress audits

**Manual examples:**
- Posting to Substack, X.com, Reddit (even if draft already exists)
- Sending outreach emails or DMs
- Any task using Chrome MCP to publish/send (not just draft)
- Tasks that require the user's real-world confirmation or approval
- Tasks that require clicking something live

---

## `scheduleName` field

After creating a project, store the kebab-case name of the created schedule:
```
"scheduleName": "princeps-social-media-growth-daily"
```
This lets `/update` and `/execute` reference the schedule for pausing/resuming.

## `triggeredBy` in executions

Track how each execution was triggered:
- `"schedule"` — ran automatically via the daily cron schedule
- `"manual"` — the user explicitly ran `/execute`

---

## Status values and badge colors

| Status  | Meaning                                        | Text     | Background |
|---------|------------------------------------------------|----------|------------|
| Active  | Claude is executing this on schedule           | #1251C0  | #EEF4FF    |
| Paused  | Skipped in /execute runs until resumed         | #64748B  | #F1F5F9    |
| Done    | Goal reached — all done                        | #10B981  | #ECFDF5    |
| Blocked | Claude can't proceed — needs user input        | #7C3AED  | #F5F3FF    |

## Priority badge colors

| Priority | Text     | Background |
|----------|----------|------------|
| High     | #D97706  | #FFFBEB    |
| Medium   | #64748B  | #F1F5F9    |
| Low      | #94A3B8  | #F8FAFC    |

## Color palette (assign in order, first unused)

`#1251C0` · `#E8533C` · `#10B981` · `#F5A623` · `#7C3AED` · `#0891B2` · `#D97706` · `#BE185D`

## pct calculation

```python
pct = round(100 * sum(1 for d in dayPlan if d["done"]) / max(len(dayPlan), 1))
```
