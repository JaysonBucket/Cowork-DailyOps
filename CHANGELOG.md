# Changelog

All relevant changes to the Daily-Ops Ext skill. Format: [Semantic Versioning](https://semver.org/) — MAJOR.MINOR.PATCH.

- **MAJOR** — breaking changes (Excel schema breaks, incompatible briefing logic). Manual migration required.
- **MINOR** — new features, backwards-compatible. Existing data remains usable as-is.
- **PATCH** — bugfixes, documentation, cosmetic fixes.

> **Note on the version jump 1.0.0 → 1.4.1**: the ext variant is pulled up to the same version state as the internal `daily-ops-ms` (1.4.1) so both variants stay consistent. The features described under [1.1.0]–[1.4.1] were already implemented in `daily-ops-ms` and are ported to ext in one step (without the MS specifics).

---

## [1.6.0] — 2026-05-02

### New

- **Work/personal separation** (logic 0b in the morning briefing, dedicated prompt for the weekend):
  - `Tasks-Inbox.xlsx` Inbox sheet gets a 17th column **Bereich** (Area, dropdown: `Beruf | Privat`). Done sheet gets the same as column 10. Old rows without a value are treated as work.
  - **P-MMDD-NN capture codes** for personal tasks — separate per-day numbering, parallel to T-codes.
  - **Six hard personal-detection criteria** in the morning briefing: personal mail addresses, family names, mail category "Privat"/"Familie", entry in `wichtige-daten.xlsx`, area="Privat", or personal indicators in the meeting title.
  - **Mon–Fri layout**: personal sections at the bottom, compact. Top-3 stays work-only.
  - **Weekend briefing** as a third scheduled prompt (`scheduled-prompts/weekend-briefing.md`, Sat/Sun 07:00) — strictly personal-only, with a sanity check on work trigger words (workshop, sync, review, customer, customer domains, ...) and ground rule "when in doubt, EXCLUDE".
- **Security filter in the afternoon sync** (STEP 0):
  - Allowlist matching on the From address: `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`, `<SAFE_SENDERS>` (optional).
  - Captures from senders outside the allowlist are NOT processed but moved to `DOPS-Capture/_rejected/YYYY-MM-DD/` and reported under "Klärungsbedarf" in the sync report.
  - Prevents injection risk if someone sends DOPS-formatted mails to you — tasks/stakeholders/meeting notes are only folded in from your own source.
  - Conservative default: when the header is unclear → reject, do not guess.
- **Ad-hoc task schema extended** with an Area field: `Bereich: [Beruf | Privat]` in the body. Default work if empty or invalid. The afternoon sync carries the field into the new inbox row.

### Changed

- **Morning briefing structure**: two new sections 13 (personal meetings today & next 7 days) and 14 (personal tasks open) at the bottom, compact.
- **Capture code assignment (logic 3)**: P-prefix added for personal, personal meetings explicitly get NO code (see logic 0b).
- **People-meeting matching (logic 1)** is now applied only to work tasks — personal tasks do not match against the work calendar.

### Migration from 1.5.x

- **Tasks-Inbox.xlsx**: with existing data, manually add the Area column (column 17 Inbox / column 10 Done) or re-import the template. Empty Area cells are interpreted as work — existing tasks remain functional.
- **Set up three scheduled prompts instead of two**: reload morning briefing AND afternoon sync (with the 1.6.0 texts), and add the weekend briefing for Sat/Sun 07:00.
- **Required**: in all three prompts, fill in `<PERSONAL_MAIL_1..3>` and `<FAMILY_NAMES>`, otherwise the personal filter does not engage.
- **Required for the security loop**: in the afternoon sync, review `<SAFE_SENDERS>` once — leave empty if your own + personal addresses are sufficient.
- **No Excel schema breaks** in Cowork-Hub, wichtige-daten, highlights — only Tasks-Inbox gets one additional column.

---

## [1.5.0] — 2026-05-02

### Changed

- **Trigger schema redesigned to a `DOPS` prefix namespace**: Daily-Ops Ext and the built-in `daily-briefing` skill had colliding triggers ("morning briefing", "daily overview", "daily ops") — the built-in skill often fired when the user actually wanted Daily-Ops (or vice versa). Solution: `DOPS` prefix as the namespace for all primary commands. Builds on the already-established `DOPS-` from the capture subject — one mental model, one prefix.

### New

- **DOPS prefix command set** for manual triggering:
  - `DOPS start` / `DOPS morning` / `DOPS briefing` → morning briefing
  - `DOPS sync` / `DOPS close` → afternoon sync
  - `DOPS check` → status check without mail (what is open? what is coming today?)
  - `DOPS review` → review preparation (highlights + numbers since the last cycle)
  - `DOPS care` → ad-hoc customer stakeholder maintenance
- **Secondary triggers** (distinctive natural phrases — stay active): "Daily-Ops briefing", "Daily-Ops sync", "pull forward before meeting", "stakeholders at customer X", "review preparation", "highlights log".
- **Command cheatsheet** as a table in `SKILL.md` ("when to use this skill") and in `README.md` ("what's new in 1.5.0") — user sees at a glance what they can say.

### Removed from the trigger list

The following generic phrases intentionally no longer fire Daily-Ops Ext but the built-in `daily-briefing` skill:

- "morning briefing" (alone, without "Daily-Ops" in front)
- "daily overview"
- "new tasks for me"
- "daily ops" (lowercase, without context)

User value: when someone just wants a quick daily overview, they get the lightweight built-in skill. When they want the full Daily-Ops routine, they say `DOPS start` or "Daily-Ops briefing". Clear disambiguation.

### Version sync with the ms variant

- ext and ms move to 1.5.0 in parallel — both variants share the same trigger schema with adjusted command sets (ext: `DOPS review` instead of `DOPS connect`/`DOPS renewal`).

### Migration from 1.4.x

- **No Excel changes**, no data loss.
- **No scheduled-prompt changes**: 07:00 morning briefing and 15:30 afternoon sync continue unchanged. The trigger redesign only affects manual requests to the skill.
- **Required**: reload `SKILL.md` (or re-import the skill) so the skill loader picks up the new `description` with the DOPS prefix trigger set. Otherwise the old 1.4.1 trigger list keeps firing.
- **Recommended**: take a quick look at the cheatsheet in `README.md` — the new commands are memorized in 30 seconds.

---

## [1.4.1] — 2026-05-02

### Changed

- **Subject identifier moved to maximum mail-client compatibility**: `DOPS-` instead of `DOPS#`. The hash was not reliably recognized as part of a word in some mail tokenizers (new Outlook for Web/Mac) — the hyphen is consistently treated as part of the subject word in every client (Outlook Classic Desktop, new Outlook, Web, Mac, Mobile, Gmail, Apple Mail). Mail rule now matches "Subject contains DOPS-".
- **URL encoding for `#` is no longer needed**: the `%23` encoding directive in the mailto link logic is removed together with the hash-specific warning. The hyphen needs no encoding and is more robust.

### New

- **Mailto length limit actively enforced** (3-tier logic):
  - Tier 1 (default): full context block in the body.
  - Tier 2 (URL > 1800 chars): context block trimmed to title + date, hint in body "full context in the daily briefing".
  - Tier 3 (URL > 1900 chars): context block dropped entirely, 📎 marker next to the entry in the briefing HTML.
  - Tasks with long people lists or meetings with large attendee lists go through reliably — mail clients do not accept mailto URLs beyond ~2000 chars.

### Version sync with the ms variant

- The ext variant jumps from 1.0.0 to 1.4.1 in one step. The features described in 1.1, 1.2, 1.3, and 1.4 are now fully present in ext — see the historical entries below for details.

### Migration from 1.0.x

- **Required**: set up the mail rule — "Subject contains" set to `DOPS-`, move to folder `DOPS-Capture` (see SETUP.md step 3).
- **Required**: re-set both scheduled prompts with the 1.4.1 texts from `scheduled-prompts/`. In both prompts, replace `<SKILL_PATH>`, `<YOUR_MAIL>` AND `<INTERNAL_DOMAIN>`.
- **Excel update**: `Tasks-Inbox.xlsx` now has an additional "Capture-Code" column (Inbox + Done). Existing data remains usable as-is — the new column is auto-filled on the first 07:00 run. If you maintain your own copy of the xlsx: either swap in the shipped file, or manually add a "Capture-Code" column as the 16th column in Inbox and 9th column in Done.
- **Cowork-Hub.xlsx, wichtige-daten.xlsx, highlights.xlsx, Bulk-Sources.md**: no changes.

---

## [1.4.0] — adopted from ms 1.4.0

### New

- **Mail capture loop** — operational day cycle in three steps:
  - **07:00 morning briefing** now arrives as an HTML mail with `mailto:` links per task and per meeting. Click → mail opens pre-filled with subject `DOPS-<CODE> <title>` and body template (status / note or notes / stakeholders / follow-up tasks).
  - **During the day**, the user sends status updates and meeting notes by mail to themself. The mail rule matches `DOPS-` in the subject and moves to the `DOPS-Capture` folder.
  - **15:30 afternoon sync** reads `DOPS-Capture`, parses captures, updates Tasks-Inbox + customer markdowns + meeting notes, archives processed mails to `DOPS-Capture/_processed/YYYY-MM-DD/`, sends the sync report mail.

- **Capture code schema** as a robust identifier for the mail rule and mapping:
  - `T-MMDD-NN` for tasks (NN sequential per day, persisted in `Tasks-Inbox.xlsx` → stable across multiple days)
  - `M-MMDD-XX` for meetings (XX = short tag from the meeting title, max 6 chars)
  - `NEW` for ad-hoc tasks
  - Subject format `DOPS-<CODE> <free text>`

- **New scheduled prompt** `scheduled-prompts/afternoon-sync.md` (15:30 Mon–Fri) — complements the morning briefing with the end-of-day loop.

- **Schema extension Tasks-Inbox.xlsx**: "Capture-Code" column in Inbox sheet (now 16 columns) and Done sheet (now 9 columns).

- **New folder `data/Meeting-Notes/`** — the afternoon sync drops a file `YYYY-MM-DD-<CODE>.md` per processed meeting note; the customer markdown gets a link under "## Meeting recaps".

- **Cheatsheet block** at the end of every briefing mail explains the four status words (done / progress / paused / reschedule) and the ad-hoc mail format.

---

## [1.3.0] — adopted from ms 1.3.0

### New

- **Trigger phrases in description**: the description now explicitly names which requests should fire the skill ("morning briefing", "daily overview", "pull forward before meeting", "maintain customer stakeholders", etc.).
- **"When NOT to use this skill" section** in the SKILL.md body: cleanly delineates against built-in `daily-briefing`, `meeting-intel`, `calendar-management`, and `schedule-meeting`.
- **Per-request load map** in SKILL.md: a table documents which files are loaded for which request.
- **Tilde-free paths**: all tilde paths replaced with the `<SKILL_PATH>` placeholder — works in cloud containers without tilde expansion as well.

---

## [1.2.0] — adopted from ms 1.2.0

### New

- **Bulk filter for mass communication** (`data/Bulk-Sources.md`)
  - New markdown maintains a list of known mass sources with three treatment options: **Ignore** / **KeyPoints** / **FullText**
  - Auto-detection heuristics: >20 recipients, DL sender (`noreply@*`, `*@lists.*`), typical newsletter subjects, large group chats without @-mention
  - Allowlist with priority: manager mail and customer domains are never classified as bulk
  - Auto-archive to `Archiv/Bulk` for classified bulk mails

- **Briefing logic 0 — bulk filtering** runs BEFORE all other logics
- **Two new briefing sections**: "Bulk key points" and "New bulk sources detected"

---

## [1.1.0] — adopted from ms 1.1.0

### New

- **People-meeting matching** (logic 1)
  - Tasks-Inbox with columns "People (Mail)", "People (Names)", "Next shared meeting", "Meeting date"
  - The skill automatically finds the next meeting with task people in the briefing and suggests "pull forward before meeting"
  - Source per task is recorded: mail ID, Teams chat link, or event ID

- **Customer stakeholder maintenance** (logic 2)
  - Cowork-Hub.xlsx sheet "Kunden-Stakeholder" extended with columns: decision-maker level, first-contact date, first-contact channel, first-contact occasion
  - External people from mail/Teams/meetings are mapped via mail domain → customer and entered automatically into the hub + customer markdown

- **New briefing sections "Pull forward before meeting"** and "New customer contacts"

---

## [1.0.0] — 2026-05-01

### Initial release

Generic variant of the Daily-Ops system — derived from the internal `daily-ops-ms` (v1.2.0), without company-specific terminology and structures.

**What is different from the internal variant:**
- Lightweight **customer sheet** (name, industry, status, last contact, note) instead of an end-customer sheet with corporate hierarchy and contract details
- **Internal stakeholders** with free-form roles instead of a fixed role list
- **Projects & milestones** instead of a CRM-specific opportunity pipeline
- **Hours log** without external booking-system references
- **Review cycles** instead of company-specific employee cycles
- Generic **watchlist buzzwords** (renewal, escalation, complaint) instead of company-internal terminology
- No corporate-hierarchy sheet
- No licenses sheet (too product-specific)
- Briefing logic without hardcoded internal domain — configurable via the scheduled prompt text

**What is functionally identical:**
- People-meeting matching (logic 1)
- Customer stakeholder maintenance with decision-maker level (logic 2)
- Bulk filter for mass communication (logic 0)
- Tasks-Inbox with source tracking
- Bulk-Sources.md
- wichtige-daten.xlsx (people & occasions + holidays & time off)
- Highlights log
- Customer markdowns as a growing knowledge base
- Scheduled prompt for the daily briefing
