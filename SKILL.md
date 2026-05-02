---
name: daily-ops-ext
version: 1.6.0
description: |
  Personal daily-operations system for customer-facing roles (account managers, consultants,
  solution architects, project leads, customer success). Captures tasks, customers, stakeholders,
  projects, hours, and highlights in one structured hub. Connects tasks to people → calendar
  and recommends pulling work forward before shared meetings in the morning briefing.
  Maintains customer stakeholders automatically from mail/Teams/meetings. Filters out mass
  mails and large group chats.

  Day cycle: 07:00 morning briefing as a mail with capture codes and mailto links per task
  and per meeting. During the day the user sends status updates and meeting notes by mail
  to themself (a mail rule matches "DOPS-" in the subject). 15:30 afternoon sync reads all
  captures, updates Tasks-Inbox / customer markdowns, sends the sync report.

  Primary triggers (DOPS-prefix short commands — do NOT collide with built-in daily-briefing):
  "DOPS start", "DOPS morning", "DOPS briefing" (morning briefing); "DOPS sync", "DOPS close"
  (afternoon sync); "DOPS check" (status without mail); "DOPS review" (review preparation);
  "DOPS care" (ad-hoc customer stakeholder maintenance).

  Secondary triggers (distinctive natural phrases): "Daily-Ops briefing", "Daily-Ops sync",
  "pull forward before meeting", "stakeholders at customer X", "review preparation",
  "highlights log", "who is at which customer next week".

  Do NOT use for: generic calendar overview without customer context (use built-in
  daily-briefing/calendar-management); single-meeting recaps (use meeting-intel); ad-hoc
  mail triage without customer context. Generic phrases like "morning briefing",
  "daily overview" are intentionally NOT triggered any more (collision risk with built-in
  daily-briefing) — use the explicit "DOPS" prefix.
---

# Daily-Ops Ext Skill v1.6.0

> **Generic variant** without company-specific terminology. Functionally equivalent to the internal `daily-ops-ms` variant (same version state v1.6.0) — but portable across industries and companies.

> Current version: **1.6.0** — see [CHANGELOG.md](CHANGELOG.md) for changes.

A personal operations system for roles that manage many customers, projects, and stakeholders in parallel. A template for import — all data generic, you populate it with your own world.

## What it solves

- **Tasks without context loss** — every task captures source (mail/Teams/meeting) AND the people involved
- **Pull forward before meeting** — if an open task touches people you have an upcoming meeting with, Cowork suggests handling it before that meeting
- **Customer stakeholders grow automatically** — new contacts from mails/Teams land in the hub + per-customer markdown, with gap markers for role and decision-maker level
- **Review preparation** — highlights log, numbers tracker, review cycles in one central collection
- **Important personal dates** — birthdays, anniversaries, school holidays for your region
- **Bulk filter** — mass mails, newsletters, and large group chats are recognized, summarized as key points in the briefing, and auto-archived
- **Day cycle with mail capture (NEW in 1.4)** — 07:00 briefing mail with clickable capture links, you send updates by mail throughout the day, 15:30 Cowork folds everything in
- **Work/personal separation (NEW in 1.6)** — dedicated area filter, P-MMDD-NN codes for personal tasks, separate weekend briefing for personal-only items (Sat/Sun 07:00). In the Mon–Fri briefing, personal meetings appear compactly at the bottom; the top-3 stays work-only.
- **Security filter for captures (NEW in 1.6)** — the afternoon sync only processes DOPS mails from trusted senders (your own address, personal addresses, optional safe-senders list). Foreign captures are moved to `_rejected/` and flagged in the sync report — no risk of externally injected tasks/stakeholders.

## File structure

```
data/
├── Cowork-Hub.xlsx          # 7 sheets: Customers, Internal Stakeholders, Customer
│                             #          Stakeholders, Partners, Projects & Milestones,
│                             #          Hours Log, Watchlist
├── Tasks-Inbox.xlsx          # 3 sheets: Inbox (people-meeting matching + capture code +
│                             #          area), Done (capture code + area), Backlog
├── wichtige-daten.xlsx       # 2 sheets: People & Occasions, Holidays & Time Off
├── highlights.xlsx           # 3 sheets: Highlights Log, Review Cycles, Numbers Tracker
├── Bulk-Sources.md           # Mass-mail classification (grows automatically)
├── Kunden/                   # one MD per customer — stakeholders, history, notes
├── Meeting-Notes/            # YYYY-MM-DD-<CODE>.md per meeting (NEW in 1.4 — afternoon sync)
└── Briefings/                # morning/afternoon files daily
```

## Day cycle (NEW in 1.4)

```
  07:00 ───────────► During the day ───────────► 15:30
  Briefing mail     Mail capture                Afternoon sync
  + capture codes   DOPS- subject               + sync report mail
  + mailto links    → mail rule                 + tasks/customers update
                    → DOPS-Capture folder
```

**Capture-code schema:**
- `T-MMDD-NN` — work task code (sequential per day), stable across multiple days
- `P-MMDD-NN` — personal task code (NEW in 1.6 — separate sequence per day)
- `M-MMDD-XX` — meeting code (XX = short tag from the meeting title)
- `NEW` — ad-hoc task

**Mail subject format** for captures:
```
DOPS-T-0504-01 Contract renewal Customer X     → task update
DOPS-M-0504-PR Project Review Sync             → meeting note
DOPS-NEW Follow-up order ACME GmbH             → ad-hoc task
```

A mail rule (Outlook/Gmail/etc.) matches `Subject contains DOPS-` → moves to `DOPS-Capture`.

**Mail body format** (pre-filled by the mailto link):

For tasks:
```
Status: [done | progress | paused | reschedule]

Note:

---
Context: <generated from the briefing — do not delete>
```

For meetings:
```
Notes:

New stakeholders: (Name <Mail>, Role)
Follow-up tasks:

---
Context: <generated from the briefing>
```

## Setup

1. **Import the skill** — create the skill directory according to your Cowork environment's conventions (see SETUP.md)
2. **Initial population** — customers, internal stakeholders, watchlist, important dates, review cycles
3. **One MD file per customer** in `data/Kunden/`
4. **Set up the mail rule** for `DOPS-` capture (see SETUP.md, new step for 1.4)
5. **Three scheduled prompts**:
   - Mon–Fri 07:00 — morning briefing (`scheduled-prompts/morning-briefing.md`)
   - Mon–Fri 15:30 — afternoon sync (`scheduled-prompts/afternoon-sync.md`)
   - Sat/Sun 07:00 — weekend briefing (`scheduled-prompts/weekend-briefing.md` — NEW in 1.6)

In all prompts, replace `<SKILL_PATH>`, `<YOUR_MAIL>` and `<INTERNAL_DOMAIN>` once. For the personal filter (NEW in 1.6) additionally fill in `<PERSONAL_MAIL_1..3>` and `<FAMILY_NAMES>`. For the security filter in the afternoon sync (NEW in 1.6) optionally fill in `<SAFE_SENDERS>` with additional trusted senders.

## When to use this skill

**Command cheatsheet (DOPS prefix — recommended, does not collide with built-in daily-briefing):**

| Command | What it does |
|---------|--------------|
| `DOPS start` / `DOPS morning` / `DOPS briefing` | Trigger the morning briefing manually |
| `DOPS sync` / `DOPS close` | Trigger the afternoon sync manually |
| `DOPS check` | Status check without mail (what is open? what is coming today?) |
| `DOPS review` | Review preparation — highlights + numbers since the last cycle |
| `DOPS care` | Ad-hoc customer stakeholder maintenance (gap list) |

**Distinctive natural phrases (also work):**

- "Daily-Ops briefing" / "Daily-Ops sync"
- "Which tasks should I pull forward before my meeting on DD.MM. with X?"
- "Who are the stakeholders at [customer]?"
- "Review preparation" / "Highlights log since the last review"

## When NOT to use this skill

- **Generic daily overview without customer context** → the built-in `daily-briefing` skill is lighter. If you just say "morning briefing" / "daily overview", the built-in skill is intentionally triggered — Daily-Ops Ext only fires on the `DOPS` prefix or "Daily-Ops".
- **Single-meeting recap or prep** → the `meeting-intel` skill is more targeted
- **Pure calendar maintenance** (move, decline) → `calendar-management` / `schedule-meeting`
- **Ad-hoc mail triage without customer context** → use mail tools directly

## How Cowork uses the skill

| Request | Loaded |
|---------|--------|
| Morning briefing (full routine) | all 4 xlsx + all customer MDs + Bulk-Sources |
| Afternoon sync | Tasks-Inbox + codes map from briefing + capture mails + customer MDs |
| "Pull forward before meeting?" | Tasks-Inbox + calendar |
| "Stakeholders at customer X?" | Cowork-Hub + `Kunden/<Customer>.md` |
| "Review preparation" | highlights.xlsx + last 4 briefings |

## Customizations

- **Region**: enter your public holidays and school holidays in `wichtige-daten.xlsx`
- **Roles**: in the "Internal Stakeholders" sheet, roles are free-form — match your setup (e.g., account manager, solution architect, project lead, customer success manager)
- **Language**: templates are in German by default — section headings can be freely renamed
- **Internal domain**: in the scheduled-prompt text, replace `<INTERNAL_DOMAIN>` with your real domain so "external vs. internal people" is detected correctly
- **Sync times**: 07:00 / 15:30 are defaults. If your day is structured differently, adjust in the scheduled-prompt setup
