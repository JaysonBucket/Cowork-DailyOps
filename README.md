[back home](https://github.com/JaysonBucket)  
Content freshness: 04.05.2026 (v2.1 release)

# daily-ops-ext v2.1 (beta)

> **A Cowork Custom Skill for customer-facing roles.**
> Tasks, customers, stakeholders, projects, hours, highlights — in one hub, with a daily morning briefing and an end-of-day sync.

A personal daily-operations system for account managers, consultants, solution architects, project leads, and customer success managers — anywhere lots of customers run in parallel and context is easy to lose.

Current version: **2.1.0** — see [CHANGELOG.md](CHANGELOG.md). **2.1.0 is additive on top of 2.0.0** — adds personal goals tracking and Outlook-category intake. No data migration needed; just replace the three weekday scheduled prompts and create a few new Outlook categories. See [SETUP.md](SETUP.md) for details. (If upgrading from 1.7.x or earlier, read the 2.0.0 migration section first — that one was a breaking xlsx → JSON migration.)

</br>

## What you get: A Cowork skill that runs your day on rails

- 07:00 morning briefing — Top 3 for today, today's meetings with capture codes, watchlist hits, lead-time reminders, vacation alerts, goal adherence (Soll/Ist).
- 10:00 / 12:00 / 14:00 capture drains — fold quick updates from the day into your task list; silent unless something needs review.
- 15:30 afternoon sync — closes the loop: tasks updated, meeting notes archived, new stakeholders folded in, OneDrive mirror, daily snapshot, end-of-day proposal for tomorrow.
- Sat/Sun weekend briefing — personal-only, with strict filtering against work content.

How you feed it: 
- any Outlook mail with DOPS- in the subject. 
- Codes (T-MMDD-NN for tasks, M-MMDD-XX for meetings, P- for personal, NEW for ad-hoc) are pre-baked into mailto links in the briefing — one click, type a status word, send. 
- Or just tag a mail/event with the DOPS-Intake category and it gets picked up automatically.

What it tracks: customers, stakeholders (with first-contact + chronicle), projects, partners, watchlist, vacation balance with carryover-deadline alerts, and active goals (cadence, time-budget, streak, or anti-goals like a meeting limit).

Storage: plain JSON under the skill folder, mirrored daily to OneDrive with versioned snapshots.
Net effect: capture-by-mail in seconds during the day, structured state at the end of the day, no double-entry.

</br>

## What it solves

- **Tasks without context loss** — every task captures source (mail/Teams/meeting) AND the people involved
- **Pull-forward before a meeting** — if an open task touches people you have an upcoming meeting with, the skill suggests handling it before that meeting
- **Customer stakeholders grow automatically** — new contacts from mails/Teams land in the hub + per-customer JSON, with gap markers for role and decision-maker level
- **Notes + Chronicle on every record** *(NEW in 2.0)* — every record carries free-text `notes` plus a timestamped `chronicle` array with optional `event_id` links to calendar events. Stakeholder contact events log automatically.
- **Review preparation** — highlights log, numbers tracker, review cycles in one place
- **Important personal dates** — birthdays, anniversaries, school holidays for your region
- **Bulk filter** — mass mails, newsletters, and large group chats are recognized, summarized as key points in the briefing, and auto-archived
- **Day cycle with mail capture** — 07:00 briefing mail with clickable capture links, you send updates to yourself by mail throughout the day, 15:30 the skill folds everything in
- **Work/personal separation** — dedicated filter, P-codes for personal tasks, separate weekend briefing for personal-only items (Sat/Sun 07:00)
- **Security filter for captures** — afternoon sync and capture drain only process DOPS mails from trusted senders. Foreign captures are moved to `_rejected/` and flagged
- **Capture drain** — three lightweight mid-day runs (10:00/12:00/14:00) fold new captures into `tasks.json` between morning briefing and afternoon sync. Silent unless something needs your attention.
- **Vacation tracking** — `urlaub` block with year-by-year balance and entry list. Status `geplant` (booked externally but not yet officially submitted) triggers an escalating alarm cadence (60d → 30d → 14d → P1). Carryover deadline alerts before the carryover deadline. Year rollover routine on the first weekday of each new year.
- **OneDrive mirror** *(NEW in 2.0)* — every JSON is mirrored to a personal OneDrive folder `daily-ops-readonly/` after each scheduled run (morning, weekend, afternoon). Hash-based diff so only changed files are pushed. Read-access from mobile/web/manager — write stays in the skill.
- **Daily snapshots** *(NEW in 2.0)* — at 15:30 the afternoon sync copies the 10 living JSONs to `data/_snapshots/YYYY-MM-DD/`. Time-travel recovery without ZIP overhead. ~60 MB after a year.
- **Personal goals tracking** *(NEW in 2.1)* — `goals.json` with four goal types: `kadenz` (e.g. "3× sport per week"), `zeitbudget` (e.g. "4h focus per week"), `streak` (e.g. "read every day"), and `anti_goal` (e.g. "no more than 25 meetings per week"). Matches calendar events by Outlook category or title keyword over a configurable fenster. Morning briefing computes Soll/Ist, surfaces unmet goals with a slot suggestion (no auto-create — you book it manually), and escalates anti-goal warnings/breaches. Manual hits via `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>`.
- **Outlook-category intake** *(NEW in 2.1)* — additive intake channel alongside the DOPS-mailto pipeline. Tag any Outlook mail or event with `DOPS-Intake` (auto-classify) or a typed variant (`DOPS-Intake-Task` / `-Customer` / `-Highlight` / `-Goal`) — the next capture-drain or afternoon sync folds it in and swaps the category to `DOPS-Intake-Done` so nothing gets re-processed. Same trusted-sender security filter as the mailto pipeline.

---

## Day cycle

```
  07:00 ────► 10/12/14 ────► During the day ────► 15:30
  Briefing    Capture drain  Mail capture         Afternoon sync
  mail        (silent unless DOPS-<CODE> subj.    + sync report mail
  + codes     attention)     → mail rule          + tasks/customers update
  + mailto    → fold into    → DOPS-Capture       + meeting-notes archival
  + mirror →  tasks.json                          + mirror → OneDrive
  OneDrive                                        + daily snapshot
```

**Capture codes:**
- `T-MMDD-NN` — work task (sequential per day, stable across multiple days)
- `P-MMDD-NN` — personal task (separate sequence)
- `M-MMDD-XX` — meeting (XX = short tag derived from the meeting title)
- `NEW` — ad-hoc task captured by mail

**Example subjects:**

```
DOPS-T-0504-01 Contract renewal Customer X     → task update
DOPS-M-0504-PR Project Review Sync             → meeting note
DOPS-NEW Follow-up order ACME GmbH             → ad-hoc task
DOPS-NEW Urlaub-Recalc                         → flag vacation balance for recalculation
DOPS-NEW Mirror-Now                            → trigger interim OneDrive mirror
DOPS-NEW Goal-Done goal_sport 2026-05-04       → log a manual goal hit
```

A mail rule (Outlook/Gmail/etc.) matches on `Subject contains DOPS-` and drops everything into a folder called `DOPS-Capture`. The afternoon sync works through it at 15:30 — status word in the first line, done.

---

## Command cheatsheet

DOPS prefix as a namespace — does not collide with the built-in `daily-briefing` skill.

| Command | What it does |
|---------|--------------|
| `DOPS start` / `DOPS morning` / `DOPS briefing` | Trigger the morning briefing manually |
| `DOPS drain` | Trigger the capture drain manually |
| `DOPS sync` / `DOPS close` | Trigger the afternoon sync manually |
| `DOPS check` | Status check without sending mail — what is open, what is coming today |
| `DOPS review` | Review preparation — highlights + numbers since the last cycle |
| `DOPS care` | Ad-hoc customer stakeholder maintenance (gap list) |
| `DOPS urlaub` | Vacation balance + open submissions |

Distinctive natural phrases work too: *"Daily-Ops briefing"*, *"Which tasks should I pull forward before my meeting on DD.MM. with X?"*, *"Stakeholders at customer Y?"*, *"Highlights since the last review"*.

---

## Quick start

1. **Unpack the ZIP** and import the skill folder following your Cowork environment's conventions
2. **Populate the JSONs** initially — replace example rows in `customers.json`, `stakeholders-internal.json`, `watchlist.json`, `wichtige-daten.json` with your own world
3. **Set up the mail rule**: `Subject contains "DOPS-"` → move to folder `DOPS-Capture`
4. **Create the OneDrive mirror folder** `daily-ops-readonly/` in your personal OneDrive (or let the first run auto-create it)
5. **Create four scheduled prompts** (texts in `scheduled-prompts/`):
   - Mon–Fri 07:00 → morning briefing
   - Mon–Fri 10:00 / 12:00 / 14:00 → capture drain
   - Mon–Fri 15:30 → afternoon sync
   - Sat/Sun 07:00 → weekend briefing (personal-only)
6. **Replace placeholders** in all prompts: `<SKILL_PATH>`, `<YOUR_MAIL>`, `<INTERNAL_DOMAIN>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`, optionally `<SAFE_SENDERS>` (capture drain shares the afternoon sync's allowlist)

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
│   ├── capture-drain.md        # Mon–Fri 10/12/14 — silent inbox-fold between morning & afternoon
│   ├── afternoon-sync.md       # Mon–Fri 15:30 — read capture mails, update tasks/customers, mirror, snapshot
│   └── weekend-briefing.md     # Sat/Sun 07:00 — personal-only with strict sanity check
│
└── data/
    ├── _meta.json              # Schema/skill version, hashes per file, mirror/snapshot timestamps
    ├── customers.json
    ├── stakeholders-internal.json
    ├── stakeholders-customer.json
    ├── partners.json
    ├── projects.json
    ├── hours-log.json
    ├── watchlist.json
    ├── tasks.json              # Inbox/active/blocked/done in one file (status field)
    ├── wichtige-daten.json     # personen_anlaesse + ferien_frei + urlaub block
    ├── highlights.json         # highlights + review_cycles + numbers_tracker
    ├── goals.json              # personal goals (kadenz / zeitbudget / streak / anti_goal)
    ├── Bulk-Sources.json       # sources + heuristics + allowlist
    ├── Kunden/
    │   └── _TEMPLATE.json      # Per-customer template — content_md + structured fields
    ├── Meeting-Notes/          # YYYY-MM-DD-<CODE>.json per meeting (created by the sync)
    ├── Briefings/              # YYYY-MM-DD-morning.json | -afternoon.json | -weekend.json
    └── _snapshots/             # YYYY-MM-DD/ — daily folder copy of the 10 living JSONs
```

### What each file does

**`SKILL.md`** — Cowork reads name, version, description, and trigger phrases from here. Only edit it if you want to change triggers or scope.

**`SETUP.md`** — step-by-step guide for first install and updates between versions. Contains the mail-rule configuration, all placeholder replacements, the OneDrive mirror setup, and a troubleshooting section.

**`scheduled-prompts/*.md`** — the four daily routines as prompt texts to paste into your Cowork schedule configuration. Each file lists the recommended frequency/time at the top and the actual prompt in a code block at the bottom.

**`data/_meta.json`** — schema/skill version, per-file MD5 hashes (for the mirror diff), `last_mirror` / `last_snapshot` timestamps, snapshot target list. Updated by every scheduled run that mirrors.

**`data/customers.json`** — all customers with status (`active` / `paused` / `ended`), industry, contact dates, and chronicle.

**`data/stakeholders-internal.json` / `stakeholders-customer.json`** — internal team contacts and external customer contacts respectively. Customer stakeholders auto-grow from mail/Teams/meetings; the skill creates entries with gap markers when role/decision-maker level is unknown.

**`data/tasks.json`** — single file with `status` field (`inbox` / `active` / `blocked` / `done` / `backlog` / `cancelled`). Each task carries source (mail/Teams/meeting), people, capture code, area (work/personal), notes and chronicle.

**`data/wichtige-daten.json`** — personal dates outside the calendar (birthdays, anniversaries), holidays/school holidays, plus a vacation `urlaub` block with year-by-year balance, config (default annual entitlement, max-rollover days), and entry list with status enum (`geplant` / `eingereicht` / `genehmigt` / `storniert`). Powers the lead-time reminders and vacation alerts in the briefing.

**`data/highlights.json`** — highlights log (customer wins, completed milestones), review cycles (when is your next review due?), numbers tracker (KPIs you show in reviews).

**`data/goals.json`** — personal goals with four types: `kadenz` (count hits per fenster, e.g. 3× sport per week), `zeitbudget` (sum durations, e.g. 240 min focus per week), `streak` (consecutive daily hits, e.g. read every day), `anti_goal` (max-cap per fenster, e.g. ≤25 meetings per week). Fenster types: `rolling-7d` / `calendar-week` / `rolling-30d` / `calendar-month` / `daily`. Matching by Outlook category (`kalender_kategorien`) or title keyword (`title_keywords`, case-insensitive); optional `min_duration_min`, `show_as_filter`, `exclude_categories`. Morning briefing computes Soll/Ist per active goal, suggests free slots for unmet goals with a `preferred_slot` (no auto-create — user books manually), escalates anti-goal `warnung` (last 20% of cap) and `überschritten`. Manual hits via `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` capture.

**`data/Bulk-Sources.json`** — list of mass-mail sources with treatment (`Ignore` / `KeyPoints` / `FullText`) plus heuristics and allowlist. The skill maintains it itself — new sources are proposed in the briefing for confirmation.

**`data/Kunden/_TEMPLATE.json`** — template for a customer JSON. Create one file per active customer at `Kunden/<slug>.json` — `content_md` for the human-readable narrative, structured fields for stakeholder links and chronicle.

**`data/Meeting-Notes/`** and **`data/Briefings/`** — populated automatically by the afternoon sync and the morning briefing. Date-keyed JSONs with a `content_md` wrapper.

**`data/_snapshots/`** — daily folder copy of the 10 living JSONs, written by the afternoon sync. Time-travel recovery without ZIP overhead.

> **Note on language:** the skill itself runs in German by default (briefings, mail copy, JSON values where noted). The repository documentation (README, SETUP, CHANGELOG) is English. To localize the skill output, edit the `Sprache:` line at the bottom of each prompt in `scheduled-prompts/`.

---

## Architecture — the eight logics

The morning briefing runs eight logics in this order. Order matters: bulk filter first, otherwise newsletters end up in stakeholder maintenance.

| # | Logic | Function |
|---|-------|----------|
| 0 | **Bulk filter** | Detect mass mails, newsletters, and large group chats. Hits from `Bulk-Sources.json` — otherwise heuristics (>20 recipients, DL sender, buzzword subjects, unsubscribe link, missing personal greeting). Treatment: Ignore / KeyPoints / FullText. Bulk must NOT feed into the other logics. |
| 0b | **Work/personal separation** | Classify meetings and tasks as work vs. personal. Six personal-detection criteria (personal mail addresses, family names, category/label, entry in `wichtige-daten.json`, `bereich="Privat"`, personal indicator in title). Personal meetings get no code, personal tasks get P-codes. |
| 1 | **People-meeting matching** | For every open work task: scan the next 14 days of calendar for meetings with the task people. If the shared meeting is BEFORE the deadline → recommendation "pull forward before meeting". |
| 2 | **Customer stakeholder maintenance** | Match external people from mails/Teams/meetings against `stakeholders-customer.json`. Match → `last_contact_event_id` set, `chronicle_log` appended. No match → new entry with gap markers for role and decision-maker level, plus an entry in the customer JSON. |
| 3 | **Capture code assignment** | Every entry gets a unique code: `T-MMDD-NN` (work task), `P-MMDD-NN` (personal task), `M-MMDD-XX` (meeting), `NEW` (ad-hoc). Task codes stay stable across multiple days; meeting codes are regenerated per briefing. |
| 4 | **Mailto link generation** | One pre-filled mailto link per code. Three-tier length limit (~2000-char browser limit): tier 1 full context, tier 2 trimmed context + pointer to briefing file, tier 3 stub only with a "📎 context trimmed" marker in the briefing. |
| 5 | **Vacation tracking** | Year rollover (5b) on the first weekday of a new calendar year — adds a bilanz row with carryover from the previous year clamped at `max_rollover_tage`. Stale-flag recalculation (5a) — recompute when `stale=true` (set by capture-drain on `Urlaub-Recalc` capture or by afternoon-sync on calendar change). Daily alerts (5c): `geplant` escalates 60d → 30d → 14d (P1) → daily; `eingereicht` only at ≤14d. Carryover deadline (5d): 60d → 30d → 14d (P1) before carryover deadline. |
| 6 | **Goal Adherence Check** *(NEW in 2.1)* | For every active goal in `goals.json`: resolve fenster, count/sum hits from calendar events matching `kalender_kategorien` / `title_keywords` (case-insensitive). Compute Soll/Ist gap. Surfaces in section "Goals this week". Unmet goals with `preferred_slot`: suggest the next free 60-min window matching weekdays + earliest time + duration (no auto-create — user books manually). Anti-goals: `warnung` (last 20% of cap) shown inline; `überschritten` escalates to Top-3 area as P2 advisory. Chronicle hooks for target-reached / streak-broken / limit-exceeded. |

The capture drain (10/12/14 Mon–Fri) runs a restricted subset of the afternoon sync — security filter → find new captures since last run → fold task updates and ad-hoc tasks into `tasks.json` → scan Outlook for `DOPS-Intake*` tagged mails and events, fold them in, swap category to `DOPS-Intake-Done` → leave M-code mails and full archival to the afternoon sync. Silent unless something needs attention. **No mirror, no snapshot** — those run only in the afternoon sync.

The afternoon sync runs in ten steps: **0** security filter → **1** read captures → **2** load codes map → **3** classify and fold in → **4** archive → **5** highlight detection → **6** vacation stale-flag handling → **6.5** Category-Intake re-scan (full archival on AUTO past calendar events, full Kunden/<slug>.json updates) → **7** mirror to OneDrive → **8** daily snapshot → **9** sync report mail.

The weekend briefing (Sat/Sun) is a stand-alone personal-only routine with a hard sanity check: it scans the finished mail before send for work trigger words ("workshop", "sync", "customer", internal domain) — any hit ⇒ drop the entry, recompose. Ground rule: *when in doubt, exclude*.

---

## When NOT to use this skill

- **Generic daily overview without customer context** → the built-in `daily-briefing` skill is lighter. If you just say "morning briefing" / "daily overview", the built-in skill is intentionally triggered. daily-ops-ext only fires on the `DOPS` prefix or "Daily-Ops".
- **Single-meeting recap or prep** → `meeting-intel` is more targeted
- **Pure calendar maintenance** (move, decline) → `calendar-management` / `schedule-meeting`
- **Ad-hoc mail triage without customer context** → just use mail tools directly

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

**Source:** [github.com/JaysonBucket/Cowork-DailyOps](../../) · **License:** MIT · **Web:** [jaysons.dev](https://jaysons.dev)

⭐ If the skill helps you, give the repo a star — it helps others find it.
