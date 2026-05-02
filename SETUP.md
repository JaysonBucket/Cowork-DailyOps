# Daily-Ops Ext — Setup Guide

> **Generic variant** without company-specific terminology — portable across industries and companies. Functionally equivalent to the internal `daily-ops-ms` variant.

**Version 1.6.0** — see [CHANGELOG.md](CHANGELOG.md) for changes.

A personal daily-operations system for customer-facing roles (account managers, consultants, solution architects, project leads, customer success). Initial setup takes about 30–60 minutes after import; from there it largely maintains itself.

> **Language note:** The skill produces briefings, mail copy, and Excel headers in German by default. Repository documentation is English. To localize the skill output, edit the `Sprache:` line at the bottom of each prompt in `scheduled-prompts/` and translate the Excel sheet headers.

## What's new in 1.6.0

**Work/personal separation** — the skill now distinguishes work and personal items:

- `Tasks-Inbox.xlsx` has a new column **Bereich** ("Area" — Inbox column 17 / Done column 10) with dropdown `Beruf | Privat` (work | personal). Old rows without a value are treated as work — existing tasks remain functional.
- Personal tasks get their own capture-code namespace: **P-MMDD-NN** (parallel to T-MMDD-NN for work).
- In the Mon–Fri morning briefing, personal sections are compact at the bottom; the top-3 stays strictly work-only.
- **Weekend briefing** as a third scheduled prompt (`scheduled-prompts/weekend-briefing.md`, Sat/Sun 07:00) — strictly personal-only, with a sanity check on work trigger words (workshop, sync, review, customer, customer domains, ...) and the ground rule "when in doubt, EXCLUDE".
- Six hard personal-detection criteria in the briefing: personal mail addresses, family names, mail category "Privat"/"Familie", entry in `wichtige-daten.xlsx`, area="Privat", or personal indicators in the meeting title.

**Security filter in the afternoon sync (STEP 0)** — protection against DOPS mail injection:

- The afternoon sync only processes DOPS captures whose From address is on the allowlist: `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`, optionally `<SAFE_SENDERS>`.
- Captures from other senders are NOT processed but moved to `DOPS-Capture/_rejected/YYYY-MM-DD/` and flagged in the sync report under "Klärungsbedarf" ("Open questions").
- Conservative default: when the header is unclear → reject, do not guess. Prevents the risk of someone sending DOPS-formatted mails to forge tasks/stakeholders/meeting notes.

**Ad-hoc task schema extended** with a `Bereich: [Beruf | Privat]` field in the body. Default is work if empty or invalid.

**Migration from 1.5.x:**
- `Tasks-Inbox.xlsx`: with existing data, manually add the Area column (column 17 Inbox / column 10 Done) or re-import the template. Empty Area cells are interpreted as work.
- Set up **three** scheduled prompts instead of two: reload morning briefing + afternoon sync (with the 1.6.0 texts) and add the weekend briefing for Sat/Sun 07:00.
- **Required** in all three prompts: fill in `<PERSONAL_MAIL_1..3>` and `<FAMILY_NAMES>`, otherwise the personal filter does not engage.
- **Required** in the afternoon sync: review `<SAFE_SENDERS>` once — leave empty if your own + personal addresses are sufficient.

## What's new in 1.5.0

**Trigger schema redesigned — `DOPS` prefix as namespace:**

The built-in `daily-briefing` skill and Daily-Ops Ext had colliding triggers ("morning briefing", "daily overview"). Result: often the wrong skill fired. From 1.5.0 onward, Daily-Ops Ext uses a clear `DOPS` prefix namespace — analogous to the already-established `DOPS-` in the capture subject.

**Command cheatsheet:**

| Command | What it does |
|---------|--------------|
| `DOPS start` / `DOPS morning` / `DOPS briefing` | Trigger the morning briefing manually |
| `DOPS sync` / `DOPS close` | Trigger the afternoon sync manually |
| `DOPS check` | Status check without sending mail |
| `DOPS review` | Review preparation (highlights + numbers since the last cycle) |
| `DOPS care` | Ad-hoc customer stakeholder maintenance |

Distinctive natural phrases continue to work: *"Daily-Ops briefing"*, *"pull forward before meeting"*, *"review preparation"*, *"stakeholders at [customer]"*.

**Intentionally removed** from the trigger list: "morning briefing" (alone), "daily overview", "new tasks for me", "daily ops". These now intentionally fire the built-in `daily-briefing` — when you want full Daily-Ops Ext, say `DOPS start` or "Daily-Ops briefing".

**Migration from 1.4.x:** no Excel changes, no scheduled-prompt changes. Just reload `SKILL.md` so the new description is picked up. The two scheduled prompts (07:00 / 15:30) keep running unchanged — the trigger redesign only affects manual requests.

## What's new in 1.4.1

- **Identifier change**: `DOPS-` instead of `DOPS#` as the subject prefix. The hyphen tokenizes reliably in every mail client (Outlook Classic Desktop, new Outlook, Web, Mac, Gmail) — the hash was a risk in some clients.
- **Mailto length logic**: when the capture link would get too long (>1800 chars), the skill automatically trims the context block and inserts a hint that the full context is in the daily briefing file. This way tasks with many people or meetings with large attendee lists go through cleanly.
- **Version sync with the ms variant**: the ext variant jumps directly from 1.0.0 to 1.4.1 so both variants align. See `CHANGELOG.md` for the in-between steps (1.1, 1.2, 1.3, 1.4).

## What's new in 1.4

- **Mail capture loop**: the 07:00 briefing now arrives as a mail with clickable capture links per task and per meeting. Click → mail opens pre-filled → type status or note → send. At 15:30 the skill folds in every capture.
- **Capture code schema**: `T-MMDD-NN` for tasks, `M-MMDD-XX` for meetings, `NEW` for ad-hoc tasks. In the subject as `DOPS-<CODE>` — unique, no collision with regular mail.
- **New scheduled prompt** `afternoon-sync.md` (15:30) — reads the DOPS-Capture folder, updates the tasks inbox, drops meeting notes, sends the sync report mail.
- **Schema extension**: `Tasks-Inbox.xlsx` now has a "Capture-Code" column (Inbox + Done). On update from 1.0.x: the column is populated automatically on the first 07:00 run.

## 1. Import the skill

Create the skill directory according to your environment. Common paths:

```
~/.claude/skills/daily-ops-ext/                   # Claude Code CLI, local
<OneDrive>/Apps/Claude/skills/daily-ops-ext/      # CLI with cloud sync
<Cowork-Cloud-Mount>/skills/daily-ops-ext/        # Cowork cloud container
```

Note the actual path — you need it during scheduled-prompt setup as the replacement for the `<SKILL_PATH>` placeholder.

Important: the Excel files must stay in **xlsx format (PK-zip)**. If you open them in Excel, explicitly choose "Excel Workbook (.xlsx)" when saving — not the legacy `.xls` format.

## 2. Initial population (one-off, ca. 30–60 min)

### a) Cowork-Hub.xlsx

| Sheet | What to fill in |
|-------|-----------------|
| **Kunden** (Customers) | One row per customer. Just name + industry + status — keep the rest minimal |
| **Interne Stakeholder** (Internal Stakeholders) | One row per customer + role. Roles are free-form (e.g. account manager, solution architect, project lead, customer success manager) |
| **Watchlist** | Manager name + relevant buzzwords (e.g. "renewal", "escalation", "complaint"). The skill scans mails/Teams for them |
| **Kunden-Stakeholder** (Customer Stakeholders) | Grows automatically — but you can add important people up front |

Example rows are shaded gray — overwrite or delete them.

### b) Tasks-Inbox.xlsx

Leave empty or pre-fill with your current open tasks. Important:
- **Quelle-Typ** (Source type): mail / Teams / meeting / self / contract
- **Personen (Mail)**: comma-separated, so people-meeting matching works
- **Personen (Namen)**: human-readable form
- **Capture-Code**: leave empty — assigned automatically on the first morning briefing run

### c) wichtige-daten.xlsx (important dates)

| Sheet | What to fill in |
|-------|-----------------|
| **Personen & Anlässe** (People & Occasions) | Birthdays, anniversaries. Lead-time days e.g. `14,7,2` for 14/7/2 days ahead |
| **Ferien & Frei** (Holidays & Time Off) | Public holidays and school holidays for your region(s) |

### d) highlights.xlsx

| Sheet | What to fill in |
|-------|-----------------|
| **Review-Cycles** | Date of your last performance review as the anchor (the skill computes the next cycle from there) |
| **Numbers-Tracker** | Adjust the quarter labels, then let the skill fill the rest |
| **Highlights-Log** | The skill maintains it itself — but you can backfill the past |

### e) Bulk-Sources.md

Leave empty — the skill fills it itself over time. You can pre-enter known mass sources:
- Distribution lists in your company (`*@lists.company.com`)
- Known newsletter senders
- Large Teams group chats (channel names)

Treatment options per entry:
- **Ignore** — not in the briefing at all, archived directly
- **KeyPoints** — 1–2 sentences in the briefing, then archived
- **FullText** — exception, treat like a normal mail (e.g. manager even on DLs)

Do not forget the allowlist: add manager mail and customer domains so they are never classified as bulk — even if they arrive via DLs.

### f) Customer markdowns

For every active customer, create a file in `data/Kunden/`. The template is `data/Kunden/_TEMPLATE.md` — copy it and rename with the customer name (e.g. `data/Kunden/ACME GmbH.md`).

## 3. Set up the mail rule for DOPS-Capture (NEW in 1.4 — required for mail capture)

To filter capture mails cleanly, create a mail rule (Outlook, Gmail, Apple Mail — all support this):

1. Open mail client → Rules/Filters → New rule
2. Condition: **Subject contains specific words** → enter: `DOPS-`
3. Action: **Move to folder** → create new folder `DOPS-Capture` (at inbox level)
4. Additional action: **Mark as read** (otherwise every capture pops as unread)
5. Optional, if your client supports categories/labels: **Assign category/label** → "DOPS-Capture" (yellow)
6. Apply rule to existing mails — in case you already ran tests
7. **Important**: rule name e.g. "DOPS-Capture-Filter" — easier to find if you need to adjust it later

Verify: send yourself a test mail with subject `DOPS-TEST-SETUP` — should land in `DOPS-Capture` immediately.

During the afternoon sync, processed mails are moved to `DOPS-Capture/_processed/YYYY-MM-DD/`. The subfolder is created automatically on the first run.

## 4. Set up scheduled prompts (three prompts in 1.6)

**Morning briefing (Mon–Fri 07:00):**
1. Open your AI assistant
2. Say: "Set up the morning briefing from the Daily-Ops-Ext skill as a scheduled task at 07:00 Mon–Fri"
3. Paste the prompt text from `scheduled-prompts/morning-briefing.md`
4. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<INTERNAL_DOMAIN>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`

**Afternoon sync (Mon–Fri 15:30):**
1. Say: "Set up the afternoon sync from the Daily-Ops-Ext skill as a scheduled task at 15:30 Mon–Fri"
2. Paste the prompt text from `scheduled-prompts/afternoon-sync.md`
3. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`; `<SAFE_SENDERS>` is optional (leave empty if not applicable)
4. Adjust the sync time if your day ends differently — should be BEFORE your last meeting so updates from the last meeting can still flow into the next morning briefing

**Weekend briefing (Sat/Sun 07:00) — NEW in 1.6:**
1. Say: "Set up the weekend briefing from the Daily-Ops-Ext skill as a scheduled task at 07:00 Sat/Sun"
2. Paste the prompt text from `scheduled-prompts/weekend-briefing.md`
3. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`
4. Strictly personal-only — no work topics, no customer mails, no review/numbers sections

**Smoke test of the capture pipeline:**
- After the first 07:00 run, send yourself a test mail: subject `DOPS-T-XXXX-01 done`, body `Status: done\nNotiz: Setup test`
- On the 15:30 run the test task should move to the Done sheet and show up in the sync report

What the routine does daily:
- reads your calendar (today + 14 days)
- reads new mails and Teams messages since yesterday
- classifies mass mails / newsletters / large group chats → key points in the briefing, full text archived
- matches tasks against people calendars → recommends pulling forward before meetings
- maintains customer stakeholders from mails/Teams (customer mapping via mail domain)
- detects watchlist hits and lead-time reminders
- writes the briefing as markdown to `data/Briefings/YYYY-MM-DD-morning.md`
- sends the briefing as an HTML mail with capture links

Optional: on Mondays an additional backup step runs to cloud storage `daily-ops-ext/_backup/YYYY-MM-DD/` (8 weeks retention).

## 5. Data flow & responsibilities

### The skill maintains automatically:
- Tasks-Inbox: new tasks from mails/Teams (with people columns filled)
- Customer-Stakeholders sheet: new contacts from mail/Teams/meetings
- Customers / Last contact
- Highlights-Log (as "proposed")
- Bulk-Sources.md (new bulk sources — after your confirmation in the briefing)
- Briefing file
- Meeting notes (from the afternoon sync)

### You maintain manually:
- Customers, initial population
- Internal stakeholders on team changes
- For new customer contacts: add role and decision-maker level (the skill marks the gaps)
- Hours log: the skill suggests, you verify and book externally
- Review cycles when your manager changes

## 6. Backup recommendation

- Weekly automatic via the backup step in the scheduled prompt
- Manual: just copy the `data/` folder

## 7. Common customizations

- **Different region**: fill `wichtige-daten.xlsx → Ferien & Frei` with your local holidays
- **Different roles**: enter any roles in the "Interne Stakeholder" sheet — no fixed list
- **Different language**: sheet headers and section titles are all freely editable
- **Internal domain**: in the scheduled-prompt text, replace `<INTERNAL_DOMAIN>` with your real domain

## Feedback

Adapt this template freely for your own use. If you have improvements, feel free to share.
