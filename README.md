# daily-ops-ext

> **A Cowork Custom Skill for customer-facing roles.**
> Tasks, customers, stakeholders, projects, hours, highlights — in one hub, with a daily morning briefing and an end-of-day sync.

A personal daily-operations system for account managers, consultants, solution architects, project leads, and customer success managers — anywhere lots of customers run in parallel and context is easy to lose.

Current version: **1.6.0** — see [CHANGELOG.md](CHANGELOG.md).

---

## What it solves

- **Tasks without context loss** — every task captures source (mail/Teams/meeting) AND the people involved
- **Pull-forward before a meeting** — if an open task touches people you have an upcoming meeting with, the skill suggests handling it before that meeting
- **Customer stakeholders grow automatically** — new contacts from mails/Teams land in the hub + per-customer markdown, with gap markers for role and decision-maker level
- **Review preparation** — highlights log, numbers tracker, review cycles in one place
- **Important personal dates** — birthdays, anniversaries, school holidays for your region
- **Bulk filter** — mass mails, newsletters, and large group chats are recognized, summarized as key points in the briefing, and auto-archived
- **Day cycle with mail capture** — 07:00 briefing mail with clickable capture links, you send updates to yourself by mail throughout the day, 15:30 the skill folds everything in
- **Work/personal separation** — dedicated filter, P-codes for personal tasks, separate weekend briefing for personal-only items (Sat/Sun 07:00)
- **Security filter for captures** — the afternoon sync only processes DOPS mails from trusted senders. Foreign captures are moved to `_rejected/` and flagged

---

## Day cycle

```
  07:00 ───────────► During the day ───────────► 15:30
  Briefing mail      Mail capture                Afternoon sync
  + capture codes    DOPS-<CODE> subject         + sync report mail
  + mailto links     → mail rule                 + tasks/customers update
                     → DOPS-Capture folder
```

**Capture codes:**
- `T-MMDD-NN` — work task (sequential per day, stable across multiple days)
- `P-MMDD-NN` — personal task (separate sequence, NEW in 1.6)
- `M-MMDD-XX` — meeting (XX = short tag derived from the meeting title)
- `NEW` — ad-hoc task captured by mail

**Example subjects:**

```
DOPS-T-0504-01 Contract renewal Customer X     → task update
DOPS-M-0504-PR Project Review Sync             → meeting note
DOPS-NEW Follow-up order ACME GmbH             → ad-hoc task
```

A mail rule (Outlook/Gmail/etc.) matches on `Subject contains DOPS-` and drops everything into a folder called `DOPS-Capture`. The afternoon sync works through it at 15:30 — status word in the first line, done.

---

## Command cheatsheet

DOPS prefix as a namespace — does not collide with the built-in `daily-briefing` skill.

| Command | What it does |
|---------|--------------|
| `DOPS start` / `DOPS morning` / `DOPS briefing` | Trigger the morning briefing manually |
| `DOPS sync` / `DOPS close` | Trigger the afternoon sync manually |
| `DOPS check` | Status check without sending mail — what is open, what is coming today |
| `DOPS review` | Review preparation — highlights + numbers since the last cycle |
| `DOPS care` | Ad-hoc customer stakeholder maintenance (gap list) |

Distinctive natural phrases work too: *"Daily-Ops briefing"*, *"Which tasks should I pull forward before my meeting on DD.MM. with X?"*, *"Stakeholders at customer Y?"*, *"Highlights since the last review"*.

---

## Quick start

1. **Unpack the ZIP** and import the skill folder following your Cowork environment's conventions
2. **Populate `data/Cowork-Hub.xlsx`** initially — customers, internal stakeholders, watchlist
3. **Set up the mail rule**: `Subject contains "DOPS-"` → move to folder `DOPS-Capture`
4. **Create three scheduled prompts** (texts in `scheduled-prompts/`):
   - Mon–Fri 07:00 → morning briefing
   - Mon–Fri 15:30 → afternoon sync
   - Sat/Sun 07:00 → weekend briefing (personal-only)
5. **Replace placeholders** in all three prompts: `<SKILL_PATH>`, `<YOUR_MAIL>`, `<INTERNAL_DOMAIN>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`, optionally `<SAFE_SENDERS>`

Detailed walkthrough with migration notes and troubleshooting → [SETUP.md](SETUP.md).

---

## What's in the ZIP

```
daily-ops-ext/
├── SKILL.md                    # Skill manifest (name, version, triggers, description)
├── SETUP.md                    # Detailed setup guide with migration & troubleshooting
├── CHANGELOG.md                # Version history (Semantic Versioning)
├── README.md                   # This file
├── LICENSE                     # MIT
│
├── scheduled-prompts/
│   ├── morning-briefing.md     # Mon–Fri 07:00 — briefing mail with capture codes & mailto links
│   ├── afternoon-sync.md       # Mon–Fri 15:30 — read capture mails, update tasks/customers
│   └── weekend-briefing.md     # Sat/Sun 07:00 — personal-only with strict sanity check
│
└── data/
    ├── Cowork-Hub.xlsx         # 7 sheets: Customers, Internal Stakeholders, Customer
    │                           #          Stakeholders, Partners, Projects & Milestones,
    │                           #          Hours Log, Watchlist
    ├── Tasks-Inbox.xlsx        # 3 sheets: Inbox (with people-meeting matching, capture
    │                           #          code, area work/personal), Done, Backlog
    ├── wichtige-daten.xlsx     # 2 sheets: People & Occasions, Holidays & Time Off
    ├── highlights.xlsx         # 3 sheets: Highlights Log, Review Cycles, Numbers Tracker
    ├── Bulk-Sources.md         # Mass-mail classification — grows over time
    ├── Kunden/
    │   └── _TEMPLATE.md        # Per-customer template — stakeholders, current status,
    │                           # history, meeting recaps, doc index, open items
    ├── Meeting-Notes/          # YYYY-MM-DD-<CODE>.md per meeting (created by the sync)
    └── Briefings/              # YYYY-MM-DD-morning.md & -afternoon.md, daily
```

### What each file does

**`SKILL.md`** — Cowork reads name, version, description, and trigger phrases from here. Only edit it if you want to change triggers or scope.

**`SETUP.md`** — step-by-step guide for first install and updates between versions. Contains the mail-rule configuration, all placeholder replacements, and a troubleshooting section.

**`scheduled-prompts/*.md`** — the three daily routines as prompt texts to paste into your Cowork schedule configuration. Each file lists the recommended frequency/time at the top and the actual prompt in a code block at the bottom.

**`data/Cowork-Hub.xlsx`** — the heart: all customers, internal and external stakeholders, partners, projects and milestones, an hours log, and a watchlist (people/buzzwords the skill watches for in mails/Teams).

**`data/Tasks-Inbox.xlsx`** — inbox sheet with open tasks. Columns hold source, people, next shared meeting, capture code, and area (work/personal). Done sheet as archive, backlog sheet for later.

**`data/wichtige-daten.xlsx`** — personal dates outside the calendar: birthdays, anniversaries, holidays, school holidays. Powers the lead-time reminders in the briefing. *(Filename kept in German — sheet headers are German too; rename freely if you localize.)*

**`data/highlights.xlsx`** — highlights log (customer wins, completed milestones), review cycles (when is your next review due?), numbers tracker (KPIs you show in reviews).

**`data/Bulk-Sources.md`** — list of mass-mail sources with treatment (`Ignore` / `KeyPoints` / `FullText`). The skill maintains it itself — new sources are proposed in the briefing for confirmation.

**`data/Kunden/_TEMPLATE.md`** — template for a customer markdown. Create one file per active customer at `Kunden/<CustomerName>.md` — stakeholders, current status, history, meeting recaps, doc index, open qualitative items. The skill appends recaps after every meeting.

**`data/Meeting-Notes/`** and **`data/Briefings/`** — populated automatically by the afternoon sync and the morning briefing. The folders stay in the repo via `.gitkeep`; your real content is excluded by `.gitignore`.

> **Note on language:** the skill itself runs in German by default (briefings, mail copy, and Excel headers). The repository documentation (README, SETUP, CHANGELOG) is English. To localize the skill output, edit the `Sprache:` line at the bottom of each prompt in `scheduled-prompts/` and translate the Excel sheet headers.

---

## Architecture — the six logics

The morning briefing runs six logics in this order. Order matters: bulk filter first, otherwise newsletters end up in stakeholder maintenance.

| # | Logic | Function |
|---|-------|----------|
| 0 | **Bulk filter** | Detect mass mails, newsletters, and large group chats. Hits from `Bulk-Sources.md` — otherwise heuristics (>20 recipients, DL sender, buzzword subjects, unsubscribe link, missing personal greeting). Treatment: Ignore / KeyPoints / FullText. Bulk must NOT feed into the other logics. |
| 0b | **Work/personal separation** *(NEW in 1.6)* | Classify meetings and tasks as work vs. personal. Six personal-detection criteria (personal mail addresses, family names, category/label, entry in `wichtige-daten.xlsx`, area column, personal indicator in title). Personal meetings get no code, personal tasks get P-codes. |
| 1 | **People-meeting matching** | For every open work task: scan the next 14 days of calendar for meetings with the task people. If the shared meeting is BEFORE the deadline → recommendation "pull forward before meeting". |
| 2 | **Customer stakeholder maintenance** | Match external people from mails/Teams/meetings against `Cowork-Hub.xlsx`. Match → "Last contact" set to today, counter +1. No match → new row with gap markers for role and decision-maker level, plus an entry in the customer markdown. |
| 3 | **Capture code assignment** | Every entry gets a unique code: `T-MMDD-NN` (work task), `P-MMDD-NN` (personal task), `M-MMDD-XX` (meeting), `NEW` (ad-hoc). Task codes stay stable across multiple days; meeting codes are regenerated per briefing. |
| 4 | **Mailto link generation** | One pre-filled mailto link per code. Three-tier length limit (~2000-char browser limit): tier 1 full context, tier 2 trimmed context + pointer to briefing file, tier 3 stub only with a "📎 context trimmed" marker in the briefing. |

The afternoon sync runs complementary in six steps: **step 0** security filter (allowlist on the From address) → **1** read capture mails → **2** load codes map → **3** classify and fold in (task update / meeting note / ad-hoc) → **4** archive → **5** highlight detection → **6** sync report mail.

The weekend briefing (Sat/Sun) is a stand-alone personal-only routine with a hard sanity check: it scans the finished mail before send for work trigger words ("workshop", "sync", "customer", internal domain) — any hit ⇒ drop the entry, recompose. Ground rule: *when in doubt, exclude*.

---

## When NOT to use this skill

- **Generic daily overview without customer context** → the built-in `daily-briefing` skill is lighter. If you just say "morning briefing" / "daily overview", the built-in skill is intentionally triggered. daily-ops-ext only fires on the `DOPS` prefix or "Daily-Ops".
- **Single-meeting recap or prep** → `meeting-intel` is more targeted
- **Pure calendar maintenance** (move, decline) → `calendar-management` / `schedule-meeting`
- **Ad-hoc mail triage without customer context** → just use mail tools directly

---

## Two variants

There are two versions with the same functional state:

- **`daily-ops-ext`** *(this one)* — generic, portable across industries and companies, MIT-licensed, public.
- **`daily-ops-ms`** *(not public)* — internal variant with company-specific terminology, customer templates, and stakeholder schemas. Stays internal.

Both are developed in parallel — version 1.6.0 here matches 1.6.0 in the internal variant.

---

## Frontier note

Cowork Custom Skills currently run as a **Frontier feature** in Cowork environments where custom-skill import is enabled. If you can't import the skill on your end, check with your admin whether the feature is turned on.

---

## License

MIT — see [LICENSE](LICENSE). Use it, fork it, carve out pieces, build your own variant.

---

## Feedback

I have been running this skill in production for a few weeks and keep evolving it whenever something shows up in the workflow. If you try it and notice anything — a logic missing, a category that does not catch, a briefing section that is awkward — feel free to open an [issue](../../issues) or get in touch via [jaysons.dev](https://jaysons.dev).

Community work, no SLA, plenty of love. ✌️

---

**Source:** [github.com/JaysonBucket/Cowork-DailyOps](../../) · **License:** MIT · **Web:** [jaysons.dev](https://jaysons.dev)

⭐ If the skill helps you, give the repo a star — it helps others find it.
