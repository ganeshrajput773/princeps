---
name: status
description: >
  Use this skill when the user says "/status", "show my projects", "Princeps status",
  "what's Claude working on", "show me the dashboard", "how are my projects doing",
  "give me a visual", "show graphs", "project overview", or "what's been done".
  Renders a Nova-style kanban dashboard — one vertical column per project with a
  pixelated character peeking from the bottom, waving.
metadata:
  version: "0.4.0"
  author: "Rajveer Kapoor"
---

# /status

Render the Princeps kanban dashboard and write it to `princeps-status.html`.
Each project gets a vertical column. A pixel character matching the project's
domain peeks up from the column bottom with a bob + wave animation.

This same HTML generation is also called by `/new-project` after every save,
so the dashboard always stays current.

Read `../new-project/references/project-schema.md` for the data file path.

---

## Step 1 — Load data

Always write the dashboard to this canonical file on the user's Mac:
```
HTML_OUT = /Users/rvk/Desktop/Claude/claude-cowork/Scheduled/princeps-status.html
```
Use a dynamic MNT finder so this works across sessions:
```bash
MNT=$(find /sessions/ -maxdepth 2 -name mnt -type d 2>/dev/null | head -1)
DATA_FILE="$MNT/claude-cowork/Scheduled/princeps-projects.json"
HTML_OUT="$MNT/claude-cowork/Scheduled/princeps-status.html"
TODAY=$(date +%Y-%m-%d)
TODAY_DISPLAY=$(date "+%A, %-d %B %Y")
```

Read the file. If missing or empty, write the empty-state HTML and surface it.

Recalculate `pct` for every project:
```python
pct = round(100 * sum(1 for d in dayPlan if d["done"]) / max(len(dayPlan), 1))
```

Compute days until deadline:
```python
days_left = (date(deadline) - date(today)).days
dl_color = "#B91C1C" if days_left < 0 else "#D97706" if days_left <= 3 else "#10B981"
dl_text  = f"{abs(days_left)}d overdue" if days_left < 0 else "Due today" if days_left == 0 else f"{days_left}d left"
```

---

## Step 2 — Write the HTML dashboard

Write a **fully self-contained** HTML file to:
```
$HTML_OUT
```

---

## Design system

### Palette & fonts (always hardcoded — not CSS variables)

```css
:root {
  --bg:     #faf9f5;
  --card:   #ffffff;
  --accent: #D97757;
  --text:   #141413;
  --muted:  #5c5c5a;
  --faint:  #8c8c8a;
  --border: #e5e4df;
}
```

Always load both fonts:
```html
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:ital,opsz,wght@0,9..40,300;0,9..40,400;0,9..40,500;0,9..40,600;0,9..40,700&display=swap" rel="stylesheet"/>
<style>
@font-face {
  font-family: 'Geist Mono';
  src: url('https://cdn.jsdelivr.net/npm/geist@1.3.0/dist/fonts/geist-mono/GeistMono-Regular.woff2') format('woff2');
}
</style>
```

### Status badge colours

| Status  | color    | background |
|---------|----------|------------|
| Active  | #1251C0  | #EEF4FF    |
| At Risk | #D97706  | #FFFBEB    |
| Behind  | #B91C1C  | #FEF2F2    |
| Blocked | #7C3AED  | #F5F3FF    |
| Paused  | #64748B  | #F1F5F9    |
| Done    | #10B981  | #ECFDF5    |

---

## Page layout

Page must NOT scroll. Everything fits in 100vh. Task lists scroll inside columns.

```css
html, body { height: 100%; overflow: hidden; }
body {
  font-family: 'DM Sans', sans-serif;
  background: var(--bg); color: var(--text);
  height: 100%; display: flex; flex-direction: column;
  padding: 32px 40px 24px;
}
```

### Header

```html
<header> <!-- flex, justify-content: space-between, margin-bottom: 28px -->
  <div class="header-left"> <!-- flex, gap: 8px, align-items: center -->
    <div class="p-dot"></div>
    <!-- 8px circle, border-radius: 50%, background: var(--accent) -->
    <span class="header-title">Princeps · Projects</span>
    <!-- 13px, font-weight 600, color: var(--accent), uppercase, letter-spacing 0.04em -->
  </div>
  <span class="header-date">{TODAY_DISPLAY}</span>
  <!-- 13px, color: var(--muted) -->
</header>
```

### Summary strip

```html
<div class="summary-strip">
  <!-- flex, gap: 24px, border-bottom: 1px solid var(--border),
       padding-bottom: 20px, margin-bottom: 24px -->
  <div class="s-stat">
    <div class="s-num">{N}</div>
    <!-- Geist Mono, 32px, color: var(--accent), line-height: 1 -->
    <div class="s-label">active projects</div>
    <!-- 12px, color: var(--muted) -->
  </div>
  <!-- repeat for: avg progress (N%), days to nearest deadline, overdue count -->
</div>
```

### Board

```css
.board {
  flex: 1; display: flex; gap: 16px;
  align-items: stretch;
  overflow-x: auto; overflow-y: hidden; min-height: 0;
}
.board::-webkit-scrollbar { height: 4px; }
.board::-webkit-scrollbar-thumb { background: var(--border); border-radius: 2px; }
```

---

## Column card structure

Sort order: Active → At Risk → Behind → Blocked → Paused → Done.

```css
.col {
  flex: 0 0 280px;
  background: var(--card);
  border: 1px solid var(--border);
  border-radius: 12px;
  box-shadow: 0 1px 4px rgba(0,0,0,0.04), 0 2px 12px rgba(0,0,0,0.03);
  overflow: hidden;
  display: flex; flex-direction: column;
}
```

Each column has these sections top-to-bottom:

```
┌─────────────────────────────┐
│ 3px accent bar              │  style="background:{project.color}"
├─────────────────────────────┤
│ DOMAIN LABEL  (11px, faint) │  uppercase, letter-spacing 0.07em
│ Project Name  (16px, 600)   │
│ [Active] [Medium] [⚡ auto] │  badges — see badge colours above
│ [progress bar ──────  0%]   │  Geist Mono % label
├─────────────────────────────┤
│ UP NEXT                     │  11px uppercase faint label
│ 01  Task name      [Auto]   │  grid: 24px 1fr auto
│     Day N · description     │  11px faint meta
│ 02  ...            [Manual] │
│ 03  ...                     │  show first 5 tasks
│ +N more · M manual          │  italic faint
│  ↕ this section scrolls     │  flex:1; overflow-y:auto; min-height:0
├─────────────────────────────┤
│ 31d left       Apr 20, 2026 │  border-top, flex space-between
├─────────────────────────────┤
│  [pixel character, peeking] │  .col-char-zone, h:62px, overflow:hidden
└─────────────────────────────┘
```

**Task tag colours:**
- Auto:   `background:#fff8ee; color:#a05c00`
- Manual: `background:#eef4ff; color:#1554c0`
- Done:   `background:#eefff3; color:#1a7a3a`

**Character zone:**
```css
.col-char-zone {
  border-top: 1px solid var(--border);
  height: 62px; overflow: hidden; flex-shrink: 0;
  display: flex; justify-content: center; align-items: flex-start;
  background: linear-gradient(to bottom, var(--card) 60%, #f5f4ef);
  position: relative;
}
.col-char-zone::after {
  content: ''; position: absolute;
  bottom: 0; left: 0; right: 0; height: 22px;
  background: linear-gradient(to top, #f5f4ef, transparent);
  pointer-events: none;
}
```

**Ghost "add project" column** (always the last column):
```css
.col-ghost {
  flex: 0 0 180px; align-self: stretch;
  border: 1.5px dashed var(--border); border-radius: 12px;
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  gap: 8px; padding: 32px 16px; opacity: 0.45;
}
```
Contains a `+` circle SVG and the text "say /new-project to add another".

---

## Pixel character animations

Always include these keyframes and classes:

```css
@keyframes bob {
  0%, 100% { transform: translateY(0px); }
  50%       { transform: translateY(-5px); }
}
@keyframes wave {
  0%   { transform: rotate(0deg); }
  15%  { transform: rotate(-38deg); }
  30%  { transform: rotate(6deg); }
  45%  { transform: rotate(-32deg); }
  60%  { transform: rotate(8deg); }
  75%  { transform: rotate(-18deg); }
  100% { transform: rotate(0deg); }
}
.char-body {
  animation: bob 2.8s ease-in-out infinite;
  image-rendering: pixelated; display: block; margin-top: 4px;
}
.wave-arm {
  transform-origin: 3px 3px;
  animation: wave 2.2s ease-in-out infinite;
  animation-delay: 0.4s;
}
```

Every character SVG uses: `width="78" height="102" viewBox="0 0 26 34"`
The 62px-tall `.col-char-zone` clips everything below row ~20, showing only
the head and waving arm — the "peeking" effect.

---

## Domain → character mapping

```
content / social / marketing / growth    → marketing_girly
business / finance / outreach / strategy → businessman
code / tech / engineering                → dev
writing / analysis / docs / research     → scholar
health / fitness / habit                 → athlete
other / (anything else)                  → friendly_person
```

---

## Pixel character library

All characters share the same SVG container and `.wave-arm` class.
The waving arm group is always: `<g transform="translate(19,12)"><g class="wave-arm">…</g></g>`

### marketing_girly
Pink hair · EC407A crop top · BA68C8 skirt · pink heels · phone in left hand

```svg
<!-- Hair -->
<rect x="8"  y="0" width="10" height="1" fill="#F06292"/>
<rect x="7"  y="1" width="12" height="1" fill="#EC407A"/>
<rect x="6"  y="2" width="14" height="3" fill="#E91E8C"/>
<rect x="6"  y="5" width="2"  height="5" fill="#E91E8C"/>
<rect x="18" y="5" width="2"  height="5" fill="#E91E8C"/>
<rect x="9"  y="2" width="3"  height="1" fill="#F48FB1" opacity="0.6"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#FFCBA4"/>
<rect x="9"  y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="14" y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="11" y="5" width="1"  height="1" fill="#ffffff"/>
<rect x="16" y="5" width="1"  height="1" fill="#ffffff"/>
<rect x="9"  y="7" width="2"  height="1" fill="#FFB3C6" opacity="0.8"/>
<rect x="15" y="7" width="2"  height="1" fill="#FFB3C6" opacity="0.8"/>
<rect x="10" y="8" width="6"  height="1" fill="#c97a5a"/>
<rect x="11" y="9" width="4"  height="1" fill="#c97a5a"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="1" fill="#FFCBA4"/>
<!-- Body -->
<rect x="7"  y="11" width="12" height="6" fill="#EC407A"/>
<rect x="10" y="11" width="6"  height="1" fill="#F8BBD0"/>
<rect x="7"  y="15" width="12" height="1" fill="#C2185B" opacity="0.4"/>
<!-- Left arm (static, holds phone) -->
<rect x="4" y="11" width="3" height="5" fill="#FFCBA4"/>
<rect x="3" y="15" width="2" height="2" fill="#FFCBA4"/>
<!-- Right arm (waving) -->
<g transform="translate(19,12)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#FFCBA4"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#FFCBA4"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#FFCBA4"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#FFCBA4"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#FFCBA4"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#FFCBA4"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#FFCBA4"/>
  </g>
</g>
<!-- Skirt -->
<rect x="6"  y="17" width="14" height="2" fill="#CE93D8"/>
<rect x="5"  y="19" width="16" height="3" fill="#BA68C8"/>
<rect x="4"  y="22" width="18" height="2" fill="#AB47BC"/>
<rect x="5"  y="23" width="16" height="1" fill="#E1BEE7" opacity="0.5"/>
<!-- Legs + heels -->
<rect x="8"  y="24" width="4" height="6" fill="#FFCBA4"/>
<rect x="14" y="24" width="4" height="6" fill="#FFCBA4"/>
<rect x="6"  y="30" width="6" height="2" fill="#E91E8C"/>
<rect x="6"  y="32" width="1" height="1" fill="#C2185B"/>
<rect x="14" y="30" width="6" height="2" fill="#E91E8C"/>
<rect x="19" y="32" width="1" height="1" fill="#C2185B"/>
<!-- Phone -->
<rect x="2" y="14" width="4" height="6" fill="#212121"/>
<rect x="3" y="15" width="2" height="4" fill="#4FC3F7"/>
<rect x="4" y="14" width="1" height="1" fill="#616161"/>
<!-- Sparkles -->
<rect x="1"  y="4" width="1" height="1" fill="#FFD700"/>
<rect x="24" y="3" width="1" height="1" fill="#FFD700"/>
<rect x="0"  y="9" width="1" height="1" fill="#FF69B4" opacity="0.7"/>
```

---

### businessman
Black suit · white shirt · black tie · dark shades · briefcase

```svg
<!-- Hair -->
<rect x="8"  y="0" width="10" height="1" fill="#2c2c2c"/>
<rect x="7"  y="1" width="12" height="2" fill="#1a1a1a"/>
<rect x="6"  y="3" width="2"  height="4" fill="#1a1a1a"/>
<rect x="18" y="3" width="2"  height="4" fill="#1a1a1a"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#FFCBA4"/>
<!-- Sunglasses -->
<rect x="8"  y="5" width="4"  height="2" fill="#1a1a2e"/>
<rect x="14" y="5" width="4"  height="2" fill="#1a1a2e"/>
<rect x="12" y="5" width="2"  height="1" fill="#333"/>
<!-- Mouth -->
<rect x="10" y="8" width="6"  height="1" fill="#b07850"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="2" fill="#FFCBA4"/>
<!-- White shirt + collar -->
<rect x="9"  y="11" width="8" height="1" fill="#f5f5f5"/>
<rect x="7"  y="12" width="12" height="6" fill="#1a1a2e"/>
<rect x="11" y="12" width="4"  height="5" fill="#f5f5f5"/>
<rect x="12" y="12" width="2"  height="5" fill="#111"/>
<rect x="7"  y="12" width="4"  height="4" fill="#212140"/>
<rect x="15" y="12" width="4"  height="4" fill="#212140"/>
<!-- Left arm (static) -->
<rect x="4" y="12" width="3" height="5" fill="#1a1a2e"/>
<rect x="3" y="16" width="2" height="2" fill="#FFCBA4"/>
<!-- Right arm (waving) -->
<g transform="translate(19,13)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#1a1a2e"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#1a1a2e"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#FFCBA4"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#FFCBA4"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#FFCBA4"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#FFCBA4"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#FFCBA4"/>
  </g>
</g>
<!-- Trousers -->
<rect x="7"  y="18" width="12" height="6" fill="#111827"/>
<rect x="7"  y="18" width="12" height="1" fill="#4B1C1C"/>
<rect x="11" y="18" width="4"  height="1" fill="#8B7355"/>
<!-- Legs + shoes -->
<rect x="8"  y="24" width="4" height="6" fill="#111827"/>
<rect x="14" y="24" width="4" height="6" fill="#111827"/>
<rect x="6"  y="30" width="6" height="2" fill="#0a0a0a"/>
<rect x="14" y="30" width="6" height="2" fill="#0a0a0a"/>
<rect x="6"  y="32" width="7" height="1" fill="#1a1a1a"/>
<rect x="14" y="32" width="7" height="1" fill="#1a1a1a"/>
<!-- Briefcase -->
<rect x="1" y="18" width="5" height="4" fill="#8B5E3C"/>
<rect x="2" y="17" width="3" height="2" fill="#6B4226"/>
<rect x="1" y="20" width="5" height="1" fill="#6B4226"/>
<rect x="3" y="18" width="1" height="4" fill="#6B4226"/>
```

---

### dev
Dark hoodie · glasses · laptop in hand

```svg
<!-- Hair -->
<rect x="7"  y="0" width="12" height="1" fill="#5D4037"/>
<rect x="6"  y="1" width="14" height="2" fill="#4E342E"/>
<rect x="6"  y="3" width="2"  height="4" fill="#4E342E"/>
<rect x="18" y="3" width="2"  height="4" fill="#4E342E"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#FFDAB9"/>
<!-- Glasses -->
<rect x="8"  y="5" width="3"  height="2" fill="#87CEEB" opacity="0.35"/>
<rect x="8"  y="5" width="1"  height="2" fill="#333"/>
<rect x="10" y="5" width="1"  height="2" fill="#333"/>
<rect x="8"  y="5" width="3"  height="1" fill="#333"/>
<rect x="8"  y="6" width="3"  height="1" fill="#333"/>
<rect x="14" y="5" width="3"  height="2" fill="#87CEEB" opacity="0.35"/>
<rect x="14" y="5" width="1"  height="2" fill="#333"/>
<rect x="16" y="5" width="1"  height="2" fill="#333"/>
<rect x="14" y="5" width="3"  height="1" fill="#333"/>
<rect x="14" y="6" width="3"  height="1" fill="#333"/>
<rect x="11" y="6" width="3"  height="1" fill="#333"/>
<!-- Mouth -->
<rect x="11" y="8" width="4"  height="1" fill="#b07850"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="2" fill="#FFDAB9"/>
<!-- Hoodie -->
<rect x="6"  y="11" width="14" height="7" fill="#37474F"/>
<rect x="10" y="11" width="6"  height="1" fill="#546E7A"/>
<rect x="9"  y="15" width="8"  height="3" fill="#2E3A40"/>
<!-- Left arm -->
<rect x="3"  y="12" width="3" height="5" fill="#37474F"/>
<rect x="2"  y="16" width="3" height="2" fill="#FFDAB9"/>
<!-- Right arm (waving) -->
<g transform="translate(19,13)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#37474F"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#37474F"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#FFDAB9"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#FFDAB9"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#FFDAB9"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#FFDAB9"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#FFDAB9"/>
  </g>
</g>
<!-- Jeans -->
<rect x="7"  y="18" width="12" height="6" fill="#1565C0"/>
<rect x="13" y="18" width="1"  height="6" fill="#1976D2"/>
<!-- Legs + sneakers -->
<rect x="8"  y="24" width="4" height="6" fill="#1565C0"/>
<rect x="14" y="24" width="4" height="6" fill="#1565C0"/>
<rect x="6"  y="30" width="6" height="2" fill="#ECEFF1"/>
<rect x="14" y="30" width="6" height="2" fill="#ECEFF1"/>
<rect x="6"  y="32" width="7" height="1" fill="#90A4AE"/>
<rect x="14" y="32" width="7" height="1" fill="#90A4AE"/>
<!-- Laptop -->
<rect x="0" y="17" width="6" height="4" fill="#263238"/>
<rect x="1" y="17" width="4" height="2" fill="#4FC3F7" opacity="0.7"/>
<rect x="0" y="20" width="6" height="1" fill="#1a1a2e"/>
```

---

### scholar
Burgundy sweater · round glasses · book in hand

```svg
<!-- Hair -->
<rect x="7"  y="0" width="12" height="1" fill="#8D6E63"/>
<rect x="6"  y="1" width="14" height="3" fill="#795548"/>
<rect x="6"  y="4" width="2"  height="4" fill="#795548"/>
<rect x="18" y="4" width="2"  height="4" fill="#795548"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#FFCBA4"/>
<!-- Round glasses -->
<rect x="9"  y="5" width="3"  height="2" fill="#D4A853" opacity="0.3"/>
<rect x="9"  y="5" width="1"  height="2" fill="#8B6914"/>
<rect x="11" y="5" width="1"  height="2" fill="#8B6914"/>
<rect x="9"  y="5" width="3"  height="1" fill="#8B6914"/>
<rect x="9"  y="6" width="3"  height="1" fill="#8B6914"/>
<rect x="14" y="5" width="3"  height="2" fill="#D4A853" opacity="0.3"/>
<rect x="14" y="5" width="1"  height="2" fill="#8B6914"/>
<rect x="16" y="5" width="1"  height="2" fill="#8B6914"/>
<rect x="14" y="5" width="3"  height="1" fill="#8B6914"/>
<rect x="14" y="6" width="3"  height="1" fill="#8B6914"/>
<rect x="12" y="6" width="2"  height="1" fill="#8B6914"/>
<!-- Mouth -->
<rect x="10" y="8" width="6"  height="1" fill="#b07850"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="2" fill="#FFCBA4"/>
<!-- Burgundy sweater -->
<rect x="6"  y="11" width="14" height="7" fill="#7B1FA2"/>
<rect x="9"  y="11" width="8"  height="2" fill="#9C27B0"/>
<!-- Left arm -->
<rect x="3"  y="12" width="3" height="5" fill="#7B1FA2"/>
<rect x="2"  y="16" width="3" height="2" fill="#FFCBA4"/>
<!-- Right arm (waving) -->
<g transform="translate(19,13)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#7B1FA2"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#7B1FA2"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#FFCBA4"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#FFCBA4"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#FFCBA4"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#FFCBA4"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#FFCBA4"/>
  </g>
</g>
<!-- Trousers -->
<rect x="7"  y="18" width="12" height="6" fill="#A1887F"/>
<rect x="13" y="18" width="1"  height="6" fill="#8D6E63"/>
<!-- Legs + shoes -->
<rect x="8"  y="24" width="4" height="6" fill="#A1887F"/>
<rect x="14" y="24" width="4" height="6" fill="#A1887F"/>
<rect x="6"  y="30" width="6" height="2" fill="#4E342E"/>
<rect x="14" y="30" width="6" height="2" fill="#4E342E"/>
<rect x="6"  y="32" width="7" height="1" fill="#3E2723"/>
<rect x="14" y="32" width="7" height="1" fill="#3E2723"/>
<!-- Book -->
<rect x="0" y="16" width="5" height="6" fill="#D32F2F"/>
<rect x="0" y="16" width="1" height="6" fill="#B71C1C"/>
<rect x="1" y="17" width="3" height="1" fill="#EF9A9A" opacity="0.6"/>
<rect x="1" y="19" width="3" height="1" fill="#EF9A9A" opacity="0.4"/>
<rect x="1" y="21" width="3" height="1" fill="#EF9A9A" opacity="0.3"/>
```

---

### athlete
Orange tracksuit · sweatband · water bottle

```svg
<!-- Hair -->
<rect x="8"  y="0" width="10" height="2" fill="#212121"/>
<rect x="7"  y="2" width="12" height="2" fill="#1a1a1a"/>
<rect x="6"  y="4" width="2"  height="3" fill="#1a1a1a"/>
<rect x="18" y="4" width="2"  height="3" fill="#1a1a1a"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#F5CBA7"/>
<rect x="9"  y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="14" y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="11" y="5" width="1"  height="1" fill="#fff"/>
<rect x="16" y="5" width="1"  height="1" fill="#fff"/>
<rect x="10" y="8" width="6"  height="1" fill="#b07850"/>
<!-- Sweatband -->
<rect x="6"  y="2" width="14" height="1" fill="#FF5722"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="2" fill="#F5CBA7"/>
<!-- Orange tracksuit top -->
<rect x="6"  y="11" width="14" height="7" fill="#FF5722"/>
<rect x="10" y="11" width="6"  height="2" fill="#BF360C"/>
<rect x="6"  y="13" width="2"  height="5" fill="#FF8A65"/>
<rect x="18" y="13" width="2"  height="5" fill="#FF8A65"/>
<!-- Left arm -->
<rect x="3"  y="12" width="3" height="5" fill="#FF5722"/>
<rect x="2"  y="16" width="3" height="2" fill="#F5CBA7"/>
<!-- Right arm (waving) -->
<g transform="translate(19,13)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#FF5722"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#FF5722"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#F5CBA7"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#F5CBA7"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#F5CBA7"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#F5CBA7"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#F5CBA7"/>
  </g>
</g>
<!-- Tracksuit bottoms -->
<rect x="7"  y="18" width="12" height="6" fill="#BF360C"/>
<rect x="6"  y="19" width="2"  height="5" fill="#FF8A65"/>
<rect x="18" y="19" width="2"  height="5" fill="#FF8A65"/>
<!-- Legs + sneakers -->
<rect x="8"  y="24" width="4" height="6" fill="#BF360C"/>
<rect x="14" y="24" width="4" height="6" fill="#BF360C"/>
<rect x="6"  y="30" width="6" height="2" fill="#FFFFFF"/>
<rect x="14" y="30" width="6" height="2" fill="#FFFFFF"/>
<rect x="7"  y="30" width="4" height="1" fill="#FF5722"/>
<rect x="15" y="30" width="4" height="1" fill="#FF5722"/>
<rect x="6"  y="32" width="7" height="1" fill="#E0E0E0"/>
<rect x="14" y="32" width="7" height="1" fill="#E0E0E0"/>
<!-- Water bottle -->
<rect x="1" y="16" width="3" height="6" fill="#29B6F6"/>
<rect x="1" y="15" width="3" height="1" fill="#0288D1"/>
<rect x="2" y="22" width="1" height="1" fill="#29B6F6"/>
<rect x="1" y="18" width="3" height="1" fill="#81D4FA" opacity="0.5"/>
```

---

### friendly_person (default)
Teal t-shirt · jeans · simple friendly face

```svg
<!-- Hair -->
<rect x="7"  y="0" width="12" height="2" fill="#6D4C41"/>
<rect x="6"  y="2" width="14" height="2" fill="#5D4037"/>
<rect x="6"  y="4" width="2"  height="4" fill="#5D4037"/>
<rect x="18" y="4" width="2"  height="4" fill="#5D4037"/>
<!-- Face -->
<rect x="8"  y="3" width="10" height="7" fill="#FFCBA4"/>
<rect x="9"  y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="14" y="5" width="3"  height="2" fill="#1a1a2e"/>
<rect x="11" y="5" width="1"  height="1" fill="#fff"/>
<rect x="16" y="5" width="1"  height="1" fill="#fff"/>
<rect x="9"  y="7" width="2"  height="1" fill="#FFB3C6" opacity="0.6"/>
<rect x="15" y="7" width="2"  height="1" fill="#FFB3C6" opacity="0.6"/>
<rect x="10" y="8" width="6"  height="1" fill="#c97a5a"/>
<rect x="11" y="9" width="4"  height="1" fill="#c97a5a"/>
<!-- Neck -->
<rect x="11" y="10" width="4" height="2" fill="#FFCBA4"/>
<!-- Teal t-shirt -->
<rect x="7"  y="11" width="12" height="7" fill="#00897B"/>
<rect x="10" y="11" width="6"  height="2" fill="#00796B"/>
<!-- Left arm -->
<rect x="4"  y="11" width="3" height="5" fill="#FFCBA4"/>
<rect x="3"  y="15" width="2" height="2" fill="#FFCBA4"/>
<!-- Right arm (waving) -->
<g transform="translate(19,12)">
  <g class="wave-arm">
    <rect x="0"  y="-1" width="3" height="2" fill="#FFCBA4"/>
    <rect x="2"  y="-3" width="2" height="3" fill="#FFCBA4"/>
    <rect x="3"  y="-5" width="2" height="3" fill="#FFCBA4"/>
    <rect x="4"  y="-6" width="3" height="2" fill="#FFCBA4"/>
    <rect x="3"  y="-7" width="1" height="2" fill="#FFCBA4"/>
    <rect x="5"  y="-8" width="1" height="2" fill="#FFCBA4"/>
    <rect x="6"  y="-7" width="1" height="1" fill="#FFCBA4"/>
  </g>
</g>
<!-- Jeans -->
<rect x="7"  y="18" width="12" height="6" fill="#1565C0"/>
<rect x="13" y="18" width="1"  height="6" fill="#1976D2"/>
<!-- Legs + shoes -->
<rect x="8"  y="24" width="4" height="6" fill="#1565C0"/>
<rect x="14" y="24" width="4" height="6" fill="#1565C0"/>
<rect x="6"  y="30" width="6" height="2" fill="#424242"/>
<rect x="14" y="30" width="6" height="2" fill="#424242"/>
<rect x="6"  y="32" width="7" height="1" fill="#212121"/>
<rect x="14" y="32" width="7" height="1" fill="#212121"/>
```

---

## Step 3 — Surface the dashboard

```
[View your Princeps dashboard](computer:///Users/rvk/Desktop/Claude/claude-cowork/Scheduled/princeps-status.html)
```

One-line summary per project:
```
Social Media Growth    Active    Day 1/31    0%    Goal: Hit 1,000 subs...
```

Then:
```
⚡ Auto-scheduled tasks run on their own. Say /execute for manual tasks, or /new-project to add a project.
```
