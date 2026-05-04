# Daily-Ops Ext — Setup Guide

> **Generic variant** without company-specific terminology — portable across industries and companies. Functionally equivalent to the internal `daily-ops-ms` variant.

**Version 2.1.0** — see [CHANGELOG.md](CHANGELOG.md) for changes. **2.1.0 is additive on top of 2.0.0** — no data migration, just new files and prompt updates. If you are upgrading from 1.7.x or earlier, read the 2.0.0 migration section below first; **2.0.0 was a breaking migration from xlsx to JSON**.

A personal daily-operations system for customer-facing roles (account managers, consultants, solution architects, project leads, customer success). Initial setup takes about 30–60 minutes after import; from there it largely maintains itself.

> **Language note:** The skill produces briefings, mail copy, and JSON values where noted in German by default. Repository documentation is English. To localize the skill output, edit the `Sprache:` line at the bottom of each prompt in `scheduled-prompts/` and translate the section headings inside the JSON templates as you populate them.

## What's new in 2.1.0

**Additive on top of 2.0.0** — no breaking changes, no data migration. Two new feature blocks:

**Personal goals tracking** — `data/goals.json` with four goal types:

- `kadenz` — count hits per fenster (e.g. "3× sport per week"). Soll = `soll_pro_fenster`.
- `zeitbudget` — sum durations per fenster (e.g. "240 min focus per week"). Soll = `soll_minuten_pro_fenster`.
- `streak` — consecutive daily hits (e.g. "read every day"). Tracks `current_streak` + `longest_streak`.
- `anti_goal` — max-cap per fenster (e.g. "≤25 meetings per week"). Status `ok` / `warnung` (last 20% of cap) / `überschritten`.

Fenster types: `rolling-7d`, `calendar-week`, `rolling-30d`, `calendar-month`, `daily`.

Matching: by Outlook category (`goal.match.kalender_kategorien`) or case-insensitive title-keyword match (`goal.match.title_keywords`); plus optional `min_duration_min` and `show_as_filter` (default excludes `free`). Anti-goals support `exclude_categories` to carve out exceptions.

The morning briefing runs a Goal Adherence Check (logic 6) — counts/sums hits per active goal in the active fenster, computes Soll/Ist gap, surfaces in new section "**Goals this week**". For unmet goals with a configured `preferred_slot`: suggests the next free 60-min window matching the slot's weekdays + earliest time + duration. **No auto-create — you manually book the suggested slot in Outlook.** Anti-goals: `warnung` shows inline; `überschritten` escalates to the Top-3 area as a P2 advisory line. Chronicle hooks: `Target reached` flip, streak-break, anti-goal `Limit exceeded`.

Manual hits can be logged via `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` capture (with optional `Duration:` and `Note:` body fields) — for activities done outside the calendar (spontaneous run, reading offline). The capture is fully actioned (history append + chronicle_log) without leaving an inbox task.

**Outlook-category intake** — additive intake channel alongside the DOPS-mailto pipeline:

- **AUTO category**: `DOPS-Intake` — Cowork classifies based on context. Mail from own/personal address with verb-like first line → Task; customer-domain sender with contact change → Customer-Update; customer-domain generic → Task; past calendar event with external attendees → Meeting Note + Stakeholder fold; future event with deliverable-implying title → Task with due_date=event.start.
- **TYPED override categories**: `DOPS-Intake-Task` / `DOPS-Intake-Customer` / `DOPS-Intake-Highlight` / `DOPS-Intake-Goal`. TYPED overrides AUTO if both are present on the same item.
- **Lifecycle**: after successful processing, the original `DOPS-Intake*` category is removed and replaced with `DOPS-Intake-Done` (visual confirmation in Outlook). Mails additionally archive to `DOPS-Capture/_processed/YYYY-MM-DD/`. Failed processing leaves the category untouched — the next drain retries.
- **Same trusted-sender security filter** as the DOPS-mailto pipeline. External-sender items surface under "Needs review — not processed for security".

Capture-drain (10/12/14) and afternoon-sync (15:30) both scan for tagged items. The drain folds quickly (deferring full meeting-note archival and customer-md updates); the afternoon sync repeats the scan more thoroughly (full archival on AUTO past calendar events, full Kunden/<slug>.json updates) — catches anything tagged after the last drain.

**Two new ad-hoc captures**:
- `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` — log a manual goal hit (incl. optional `Duration:` and `Note:` body fields)
- (existing) `DOPS-NEW Mirror-Now`, `DOPS-NEW Urlaub-Recalc` unchanged

**New triggers** in the SKILL.md frontmatter description: "personal goals", "goal adherence", "Soll/Ist", "tag in Outlook" — natural-phrase entries; the explicit `DOPS` commands stay unchanged.

## Migration from 2.0.x to 2.1.0

**Fully additive — no data migration needed.** Existing JSONs remain usable as-is. Required steps:

1. **Replace the three weekday scheduled prompts** (morning-briefing, capture-drain, afternoon-sync) with the v2.1.0 prompt texts from `scheduled-prompts/`. Placeholder substitutions (`<SKILL_PATH>`, `<YOUR_MAIL>`, `<INTERNAL_DOMAIN>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`, `<SAFE_SENDERS>`) are unchanged. **Weekend briefing is unchanged** — no v2.1.0 prompt update needed there.
2. **Populate `data/goals.json`** with your real goals. The shipped file ships with four example goals (sport-kadenz, focus-zeitbudget, read-streak, meeting-anti-goal) — replace freely or set `active=false` to disable. To start cautiously: leave `active=false` on all goals, populate one at a time, and verify the morning briefing surfaces the line as expected.
3. **Create the Outlook categories** (see new section 4 below): `DOPS-Intake`, `DOPS-Intake-Task`, `DOPS-Intake-Customer`, `DOPS-Intake-Highlight`, `DOPS-Intake-Goal`, `DOPS-Intake-Done`.

**No mail-rule changes** — the DOPS-mailto pipeline keeps working exactly as before. **No `_meta.json` edits needed** — the v2.1.0 release ships an updated `_meta.json` that already lists `goals.json` in `files` and in `snapshot_targets`. If you carried over a custom `_meta.json` from v2.0.x: add `"goals.json": {"hash": null, "last_mirror": null}` under `files`, and append `"goals.json"` to `snapshot_targets`.

## What's new in 2.0.0

**Breaking migration from xlsx to JSON.** Two pain points triggered the rewrite:

1. **No mobile/manager access.** Cowork has no mobile app, and the xlsx files only lived inside the skill sandbox. There was no way for the user (or their manager) to get a read-only look at the data on the go.
2. **Excel-lock fragility.** xlsx-in-mantle was brittle: openpyxl could not read every workbook reliably from the sandbox; capture-drain runs frequently skipped slots because the workbook was open on the desktop.

The fix: full migration to JSON, plus a daily push to a personal OneDrive folder so the data is reachable from any device.

**JSON data model** — all 4 xlsx workbooks (Cowork-Hub, Tasks-Inbox, wichtige-daten, highlights) decompose into 11 small JSON files in `data/`:

- `_meta.json` — schema/skill version, MD5 hashes per file, `last_mirror` / `last_snapshot` timestamps, snapshot target list
- `customers.json`, `stakeholders-internal.json`, `stakeholders-customer.json`, `partners.json`, `projects.json`, `hours-log.json`, `watchlist.json` — Cowork-Hub split into per-purpose files
- `tasks.json` — single file with `status` field (`inbox` / `active` / `blocked` / `done` / `backlog` / `cancelled`); replaces the three Tasks-Inbox sheets
- `wichtige-daten.json` — `personen_anlaesse` / `ferien_frei` / `urlaub` (bilanz + config + eintraege) in one file
- `highlights.json` — `highlights` / `review_cycles` / `numbers_tracker` in one file
- `Bulk-Sources.json` — sources + heuristics + allowlist (was: free-form Bulk-Sources.md)

Customer files (`Kunden/<slug>.json`), meeting notes (`Meeting-Notes/YYYY-MM-DD-<CODE>.json`), and briefings (`Briefings/YYYY-MM-DD-{morning,afternoon,weekend}.json`) all use a `content_md` wrapper holding the human-readable narrative plus structured fields.

**Universal Notes + Chronicle pattern** — every record (customer, stakeholder, project, partner, task, watchlist item, holiday, vacation entry, highlight, customer file) carries `notes` (free text, single field) and `chronicle` (timestamped array). Operations vocabulary across the briefing/sync logic: `note_replace`, `note_append`, `note_clear`, `chronicle_log`, `chronicle_edit`, `chronicle_link`. Stakeholder contact events log automatically with the linked calendar event.

**OneDrive mirror** — `data/` is mirrored to a personal OneDrive folder `daily-ops-readonly/` after each scheduled run that does heavy synthesis (morning briefing, weekend briefing, afternoon sync). Hash-based diff: only files with a changed MD5 are pushed via Graph API. `_meta.json` is updated last after all hashes settle.

**Daily snapshots** — at 15:30 the afternoon sync copies the 10 living JSONs to `data/_snapshots/YYYY-MM-DD/`. No ZIP, no pruning. Briefings, customer JSONs, and meeting-notes are date-keyed and self-historical, so they need no extra snapshot. ~250 KB per day, ~60 MB after a year.

**Event-driven vacation recalculation** — `urlaub.bilanz[year]` carries a `stale` flag and a `trigger_recalc_on` list. The afternoon sync sets `stale=true` when calendar events with category "Urlaub" change; the user can also flag it via `DOPS-NEW Urlaub-Recalc` capture. The morning briefing recomputes when `stale=true` or `last_calculated` is older than 7 days.

**Two new ad-hoc captures**:
- `DOPS-NEW Mirror-Now` — force an interim OneDrive mirror in the next capture-drain run (for travel-day exceptions)
- `DOPS-NEW Urlaub-Recalc` — flag the vacation balance for recalculation in the next morning briefing

**Removed:**
- Excel-lock-aware retry logic (no longer applicable)
- Monday-only Excel backup (replaced by daily snapshots)
- All `.xls` / `.xlsx` files from the data folder

## Migration from 1.7.x to 2.0.0

The data format is incompatible. There is no auto-upgrade path. Manual migration:

1. **Back up the old `data/` folder** (xlsx + Bulk-Sources.md + Kunden/*.md) outside the skill so you can reference your real content during the migration.
2. **Re-import the v2.0.0 skill** — overwrites `data/` with the new JSON template and the empty `Briefings/`, `Meeting-Notes/`, `_snapshots/` folders.
3. **Move your real data into the JSONs**:
   - Cowork-Hub sheet 1 → `customers.json` `customers` array
   - Sheets 2/3 → `stakeholders-internal.json` / `stakeholders-customer.json`
   - Sheets 4–7 → `partners.json` / `projects.json` / `hours-log.json` / `watchlist.json`
   - Tasks-Inbox sheets → `tasks.json` (preserve `status` field per row)
   - `wichtige-daten.xlsx` → `wichtige-daten.json` (Personen & Anlässe → `personen_anlaesse`; Ferien & Frei → `ferien_frei`; Urlaub → `urlaub.bilanz` + `urlaub.eintraege`; copy `Default-Anspruch` / `Max-Rollover-Tage` into `urlaub.config`)
   - `highlights.xlsx` → `highlights.json` (3 arrays)
   - `Bulk-Sources.md` → `Bulk-Sources.json` (`sources` array; `heuristics` and `allowlist` blocks)
   - One `Kunden/<slug>.json` per customer, copying the old `.md` content into `content_md`
4. **Update all four scheduled prompts** with the v2.0.0 prompt texts from `scheduled-prompts/`. Replace placeholders as before.
5. **Create the OneDrive mirror folder** `daily-ops-readonly/` in your personal OneDrive (or let the first run auto-create it).
6. **First run** — manually trigger a morning briefing. Verify the mirror runs and `_meta.json` `files.<filename>.hash` fills in for every JSON. Check the OneDrive folder shows all files.
7. **Optional**: trigger a `DOPS-NEW Urlaub-Recalc` capture once to validate the stale-flag recalculation path.

## 1. Import the skill

Create the skill directory according to your environment. Common paths:

```
~/.claude/skills/daily-ops-ext/                   # Claude Code CLI, local
<OneDrive>/Apps/Claude/skills/daily-ops-ext/      # CLI with cloud sync
<Cowork-Cloud-Mount>/skills/daily-ops-ext/        # Cowork cloud container
```

Note the actual path — you need it during scheduled-prompt setup as the replacement for the `<SKILL_PATH>` placeholder.

The data folder contains JSON files only — no xlsx, no .doc/.docx, no Excel formulas. You can hand-edit any JSON file in any text editor; the skill reads/writes them directly.

## 2. Initial population (one-off, ca. 30–60 min)

### a) customers.json + stakeholders + partners + projects + hours-log + watchlist

| File | What to fill in |
|------|-----------------|
| **`customers.json`** | One entry per customer in the `customers` array. Just name + industry + status (`active` / `paused` / `ended`) — keep the rest minimal. Each entry has `notes` (string) and `chronicle` (array) — leave both empty initially |
| **`stakeholders-internal.json`** | One entry per teammate + role. Roles are free-form (e.g. account manager, solution architect, project lead, customer success manager) |
| **`stakeholders-customer.json`** | Grows automatically — but you can add important people up front. Each entry needs `customer_id`, `name`, `email`, `role` (or `?`), `decision_level` (or `?`) |
| **`partners.json`** / **`projects.json`** / **`hours-log.json`** | Optional — fill in only if you actively use them |
| **`watchlist.json`** | Manager name + relevant buzzwords (e.g. "renewal", "escalation", "complaint"). The skill scans mails/Teams for them |

Example entries are present — overwrite or remove them.

### b) tasks.json

Leave the `tasks` array empty or pre-fill with your current open work. Important per task:
- **`source_type`**: `"mail"` / `"teams"` / `"meeting"` / `"self"` / `"contract"`
- **`people_emails`**: array of mail addresses, so people-meeting matching works
- **`people_names`**: array of human-readable names
- **`bereich`**: `"Beruf"` (work) or `"Privat"` (personal)
- **`status`**: `"inbox"` / `"active"` / `"blocked"` / `"done"` / `"backlog"` / `"cancelled"`
- **`capture_code`**: leave empty — assigned automatically on the first morning briefing run

### c) wichtige-daten.json (important dates + vacation)

| Block | What to fill in |
|-------|-----------------|
| **`personen_anlaesse`** | Birthdays, anniversaries. Lead-time days e.g. `[14, 7, 2]` for 14/7/2 days ahead |
| **`ferien_frei`** | Public holidays and school holidays for your region(s) |
| **`urlaub.config`** | Adjust `default_anspruch_tage` (default 30) and `max_rollover_tage` (default 5) if your contract differs |
| **`urlaub.bilanz`** | One entry per year — fill `anspruch` (annual entitlement) and `uebertrag_aus_vorjahr` for the current year. `verbraucht` and `verbleibend` are recomputed on the fly by the morning briefing logic 5 |
| **`urlaub.eintraege`** | One entry per booking — `von`, `bis` as `YYYY-MM-DD`; `tage` as decimal (half-days allowed); `status` ∈ {`geplant`, `eingereicht`, `genehmigt`, `storniert`} |

### d) highlights.json

| Block | What to fill in |
|-------|-----------------|
| **`review_cycles`** | Date of your last performance review as the anchor (the skill computes the next cycle from there) |
| **`numbers_tracker`** | Adjust the quarter labels, then let the skill fill the rest |
| **`highlights`** | The skill maintains it itself — but you can backfill the past |

### e) Bulk-Sources.json

Leave `sources` empty — the skill fills it itself over time. You can pre-enter known mass sources:
- Distribution lists in your company (`*@lists.company.com`)
- Known newsletter senders
- Large Teams group chats (channel names)

Treatment options per entry:
- **`Ignore`** — not in the briefing at all, archived directly
- **`KeyPoints`** — 1–2 sentences in the briefing, then archived
- **`FullText`** — exception, treat like a normal mail (e.g. manager even on DLs)

Do not forget the `allowlist` block: add manager mail and customer domains so they are never classified as bulk — even if they arrive via DLs.

### f) Customer JSONs

For every active customer, create a file in `data/Kunden/`. The template is `data/Kunden/_TEMPLATE.json` — copy it and rename to `<slug>.json` (e.g. `data/Kunden/acme-gmbh.json`). The `content_md` field holds the free-form narrative; `chronicle` and stakeholder cross-links are structured fields.

### g) goals.json (NEW in 2.1)

The shipped `data/goals.json` contains four example goals — one per type — so you can see the schema in action. Replace freely or set `active=false` to disable individual goals.

| Field | Purpose |
|-------|---------|
| **`id`** | Stable identifier — used in `DOPS-NEW Goal-Done <goal_id> ...` captures and chronicle entries |
| **`type`** | `kadenz` / `zeitbudget` / `streak` / `anti_goal` |
| **`active`** | `true` to evaluate in the morning briefing; `false` to disable temporarily |
| **`bereich`** | `Beruf` or `Privat` — controls which briefing surfaces the goal (work goals: morning briefing only; private goals: morning briefing on weekdays) |
| **`fenster`** | `rolling-7d` / `calendar-week` / `rolling-30d` / `calendar-month` / `daily` |
| **`soll_pro_fenster`** (kadenz) / **`soll_minuten_pro_fenster`** (zeitbudget) / **`max_pro_fenster`** (anti_goal) | Target / cap |
| **`match`** | `kalender_kategorien` (array of Outlook category names) and/or `title_keywords` (array of substrings, case-insensitive). Optional `min_duration_min`, `show_as_filter`, `exclude_categories` |
| **`preferred_slot`** (optional) | Days-of-week + earliest time + duration. The morning briefing suggests the next free slot matching these for unmet goals — **no auto-create**, you book it manually in Outlook |

**Recommended onboarding**:
1. Pick one goal type to start with (e.g. `kadenz` for "3× sport per week")
2. Set `active=true` on that goal, leave others `false`
3. Tag a couple of past calendar events with the matching category or rename them so a `title_keyword` matches
4. Trigger a manual `DOPS start` and verify the "Goals this week" section surfaces the goal correctly
5. Then add more goals one at a time

Manual hits (activities done outside the calendar — spontaneous run, reading offline) can be logged via `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` capture mail. Optional body fields: `Duration: <minutes>` for zeitbudget goals; `Note: <text>` to annotate.

## 3. Set up the mail rule for DOPS-Capture

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

## 4. Set up Outlook categories for category-intake (NEW in 2.1)

The category-intake channel lets you tag any Outlook mail or calendar event with a category to feed it into Daily-Ops without composing a `DOPS-` capture mail. The next capture-drain or afternoon sync folds it in, then swaps the category to `DOPS-Intake-Done` so nothing gets re-processed.

Create six new Outlook categories — one-time setup. Click path:

1. Outlook → **Home** → **Categorize** → **All Categories** (at the bottom)
2. Click **New** and create each entry below in turn:

| Category name | Suggested color | What it does |
|---------------|-----------------|--------------|
| `DOPS-Intake` | Yellow | AUTO classify — Cowork decides task / customer / meeting note based on context |
| `DOPS-Intake-Task` | Blue | Force task creation |
| `DOPS-Intake-Customer` | Green | Customer/stakeholder maintenance only — no task |
| `DOPS-Intake-Highlight` | Purple | Propose as highlight (afternoon sync confirms) |
| `DOPS-Intake-Goal` | Orange | Log a goal hit (matches against `goals.json`) |
| `DOPS-Intake-Done` | Gray | Visual confirmation after successful processing — never set this manually |

**Override precedence**: any TYPED category overrides AUTO if both are present on the same item.

**Same security filter as the DOPS-mailto pipeline** — items from senders outside your allowlist (your work address, personal addresses, optional `<SAFE_SENDERS>`) are NOT processed and surface under "Needs review — not processed for security" in the next drain or sync report.

**Smoke test** — after the first capture-drain run that follows the v2.1.0 prompt update:
- Tag a self-addressed mail with `DOPS-Intake-Task` (or any AUTO/TYPED category)
- Wait for the next 10:00 / 12:00 / 14:00 drain (or trigger `DOPS drain` manually)
- Verify: a new `inbox` task appears in `tasks.json`; the mail's category is replaced with `DOPS-Intake-Done`; the mail is moved to `DOPS-Capture/_processed/YYYY-MM-DD/`

If processing fails, the original category stays in place — the next drain retries.

## 5. Set up the OneDrive mirror folder (NEW in 2.0)

The skill mirrors `data/` to a folder in your personal OneDrive after every heavy run (morning, weekend, afternoon). To prepare:

1. Open your personal OneDrive (web or app)
2. At the OneDrive root, create a folder named **`daily-ops-readonly/`**
3. Optional but recommended: share this folder read-only with your manager (or whoever needs visibility) — write stays in the skill, the OneDrive copy is purely a read mirror
4. The first scheduled run will populate the folder. If the folder is missing, the run auto-creates it via Graph API.

Mirror trigger points:
- Morning briefing (after the briefing mail is sent)
- Weekend briefing (after the briefing mail is sent)
- Afternoon sync (as STEP 7, before the daily snapshot)

The capture drain at 10:00 / 12:00 / 14:00 does **NOT** mirror by design — too frequent. Use the `DOPS-NEW Mirror-Now` capture for a one-off interim push (e.g. before a flight).

## 6. Set up scheduled prompts (four prompts)

**Morning briefing (Mon–Fri 07:00):**
1. Open your AI assistant
2. Say: "Set up the morning briefing from the Daily-Ops-Ext skill as a scheduled task at 07:00 Mon–Fri"
3. Paste the prompt text from `scheduled-prompts/morning-briefing.md`
4. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<INTERNAL_DOMAIN>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`

**Capture drain (Mon–Fri 10:00 / 12:00 / 14:00):**
1. Say: "Set up the capture drain from the Daily-Ops-Ext skill as a scheduled task at 10:00, 12:00, and 14:00 Mon–Fri"
2. Paste the prompt text from `scheduled-prompts/capture-drain.md`
3. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`; `<SAFE_SENDERS>` mirrors the afternoon sync's allowlist (leave empty if not used)
4. Three short runs are the default — drop to two slots (12:00, 14:00) if your captures cluster around lunch only.
5. The drain is silent unless something needs your attention (rejected sender, unrecognized status word, missing reschedule deadline, M-code parser error).

**Afternoon sync (Mon–Fri 15:30):**
1. Say: "Set up the afternoon sync from the Daily-Ops-Ext skill as a scheduled task at 15:30 Mon–Fri"
2. Paste the prompt text from `scheduled-prompts/afternoon-sync.md`
3. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`; `<SAFE_SENDERS>` is optional (leave empty if not applicable)
4. Adjust the sync time if your day ends differently — should be BEFORE your last meeting so updates from the last meeting can still flow into the next morning briefing

**Weekend briefing (Sat/Sun 07:00):**
1. Say: "Set up the weekend briefing from the Daily-Ops-Ext skill as a scheduled task at 07:00 Sat/Sun"
2. Paste the prompt text from `scheduled-prompts/weekend-briefing.md`
3. **Required**: replace `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`, `<FAMILY_NAMES>`
4. Strictly personal-only — no work topics, no customer mails, no review/numbers sections

**Smoke test of the capture pipeline:**
- After the first 07:00 run, send yourself a test mail: subject `DOPS-T-XXXX-01 done`, body `Status: done\nNote: Setup test`
- On the 15:30 run the test task should move to `status="done"` in `tasks.json` and show up in the sync report
- Verify the OneDrive folder `daily-ops-readonly/` contains `tasks.json` with the latest hash

What the routine does daily:
- reads your calendar (today + 14 days)
- reads new mails and Teams messages since yesterday
- classifies mass mails / newsletters / large group chats → key points in the briefing, full text archived
- matches tasks against people calendars → recommends pulling forward before meetings
- maintains customer stakeholders from mails/Teams (customer mapping via mail domain)
- detects watchlist hits and lead-time reminders
- writes the briefing as `data/Briefings/YYYY-MM-DD-morning.json` (with `content_md` wrapper)
- sends the briefing as an HTML mail with capture links
- mirrors changed JSONs to OneDrive
- on the afternoon sync: writes a daily snapshot to `data/_snapshots/YYYY-MM-DD/`

## 7. Data flow & responsibilities

### The skill maintains automatically:
- `tasks.json`: new tasks from mails/Teams (with `people_emails` / `people_names` filled)
- `stakeholders-customer.json`: new contacts from mail/Teams/meetings, with chronicle entries linking to event_ids
- `customers.json`: `last_contact` updates and chronicle entries on material changes
- `highlights.json`: highlights as `status="proposed"` (you confirm at the next review sync)
- `Bulk-Sources.json`: new bulk sources — after your confirmation in the briefing
- `Briefings/`: one JSON per briefing (morning / weekend / afternoon)
- `Meeting-Notes/`: one JSON per processed meeting note
- `_snapshots/`: daily folder copy of the 10 living JSONs (afternoon sync)
- `_meta.json`: hashes, timestamps, snapshot targets — refreshed every mirror run

### You maintain manually:
- Initial customer population
- Internal stakeholders on team changes
- For new customer contacts: add `role` and `decision_level` (the skill marks the gaps with `?`)
- Hours log: the skill suggests, you verify and book externally
- Review cycles when your manager changes
- Vacation entries: status transitions (`geplant` → `eingereicht` → `genehmigt`) when you actually submit/get approval at HR

## 8. Backup recommendation

- **Automatic**: daily snapshot at 15:30 in `data/_snapshots/YYYY-MM-DD/` — folder copy of the 10 living JSONs, no pruning
- **OneDrive mirror**: `daily-ops-readonly/` is a live mirror, not a versioned backup — but OneDrive has its own version history, so a recent overwrite can be rolled back via OneDrive UI
- **Manual**: just copy the `data/` folder

## 9. Common customizations

- **Different region**: fill `wichtige-daten.json` `ferien_frei` block with your local holidays
- **Different roles**: in `stakeholders-internal.json`, roles are free-form — no fixed list
- **Different language**: section headings inside the briefing/sync prompts are freely editable; JSON field names stay as-is
- **Internal domain**: in the scheduled-prompt text, replace `<INTERNAL_DOMAIN>` with your real domain
- **OneDrive mirror folder name**: `daily-ops-readonly/` is the default. Rename freely; just keep the four scheduled prompts in sync.
- **Snapshot retention**: no automatic pruning. Cost is ~60 MB/year — acceptable. If you want to prune (e.g. monthly compaction): add a manual `DOPS-NEW Snapshot-Compact` task that lists which dates to keep.

## 10. Troubleshooting

- **Mirror PUT fails** — Graph API errors are logged under "Needs review" in the sync report but do NOT block the briefing/sync send. Common causes: expired token (re-auth your Cowork session), folder permissions changed, OneDrive quota exceeded. The next successful run picks up the missed files automatically (hash diff).
- **`_meta.json` hashes drift** — if you hand-edit a JSON outside the skill, the next mirror run detects the hash mismatch and re-PUTs the file. No corruption risk.
- **`stale=true` flag stuck** — recompute happens at the next morning briefing. Force it earlier by triggering `DOPS-NEW Urlaub-Recalc` capture and waiting for the next capture-drain run.
- **Missing OneDrive folder** — the first run auto-creates `daily-ops-readonly/`. If the auto-create fails (permissions), create the folder manually at the OneDrive root and re-run.
- **JSON parse error** — if you hand-edit and break the syntax, the next run logs a parse error in the sync report and skips the affected file. Fix with any JSON validator (e.g. `jq . file.json`) and re-run.
- **Category-intake item not processed** — items from senders outside the trusted-sender allowlist (your work address, personal addresses, optional `<SAFE_SENDERS>`) are NOT processed; they surface under "Needs review — not processed for security" in the next drain or sync report and the original `DOPS-Intake*` category stays in place. Items where processing failed for other reasons (parser error, missing goal match, ambiguous AUTO classification) also keep their category — the next drain or sync retries automatically. To force a re-attempt, remove and re-apply the category in Outlook.
- **Goal not surfacing in "Goals this week"** — verify `active=true` on the goal entry in `goals.json` and that at least one calendar event in the active fenster matches `kalender_kategorien` or `title_keywords` (case-insensitive). Use `DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` to log a manual hit and confirm the goal appears.

## Feedback

Adapt this template freely for your own use. If you have improvements, feel free to share via the [GitHub repo](../../).
