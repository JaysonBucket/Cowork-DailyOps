# Changelog

All relevant changes to the Daily-Ops Ext skill. Format: [Semantic Versioning](https://semver.org/) — MAJOR.MINOR.PATCH.

- **MAJOR** — breaking changes (Excel schema breaks, incompatible briefing logic). Manual migration required.
- **MINOR** — new features, backwards-compatible. Existing data remains usable as-is.
- **PATCH** — bugfixes, documentation, cosmetic fixes.

> **Note on the version jump 1.0.0 → 1.4.1**: the ext variant is pulled up to the same version state as the internal `daily-ops-ms` (1.4.1) so both variants stay consistent. The features described under [1.1.0]–[1.4.1] were already implemented in `daily-ops-ms` and are ported to ext in one step (without the MS specifics).

---

## [2.1.0] — 2026-05-04

**MINOR — additive on top of 2.0.0.** Two new feature blocks bolted on, no breaking changes, no data migration.

### Why 2.1

Two adjacent gaps after the 2.0.0 stabilization:

1. **No goal tracking against the calendar.** Personal goals (sport, focus blocks, reading streak, meeting-load cap) lived in the user's head — the calendar already contained the truth, but nothing computed Soll/Ist or surfaced gaps. A simple Outlook-category-based matching layer closes this without imposing a new tracking workflow.
2. **DOPS-mailto is the only intake path.** Users in Outlook all day asked: "why do I need to compose a mail just to file a task?" — when categorizing the mail/event with one click would be enough. The DOPS-mailto pipeline stays primary (it carries richer state: status word, note block, ad-hoc subject), but Category-Intake adds a low-friction additive channel.

### New

- **`goals.json` data file** with four goal types (additive — no other JSONs touched):
  - `kadenz` — count hits per fenster (e.g. "3× sport per week"). Soll = `soll_pro_fenster`.
  - `zeitbudget` — sum durations per fenster (e.g. "240 min focus per week"). Soll = `soll_minuten_pro_fenster`.
  - `streak` — consecutive daily hits (e.g. "read every day"). Tracks `current_streak` + `longest_streak`.
  - `anti_goal` — max-cap per fenster (e.g. "≤25 meetings per week"). Status `ok` / `warnung` (last 20% of cap) / `überschritten`.
- **Fenster types**: `rolling-7d`, `calendar-week`, `rolling-30d`, `calendar-month`, `daily`.
- **Matching**: by Outlook category (`goal.match.kalender_kategorien`) or by case-insensitive title-keyword match (`goal.match.title_keywords`); plus optional `min_duration_min` and `show_as_filter` (default excludes `free`). Anti-goals support `exclude_categories` to carve out exceptions.
- **Goal Adherence Check (logic 6) in the morning briefing**:
  - Resolves the active fenster, counts/sums hits, computes Soll/Ist gap per goal.
  - Surfaces in new briefing section "**Goals this week**" with one line per active goal.
  - For unmet goals with a configured `preferred_slot`: suggests the next free 60-min window matching the slot's weekdays + earliest time + duration. **No auto-create — user manually books the suggested slot in Outlook**. Slot suggestion is information only.
  - Anti-goals: `warnung` shows inline; `überschritten` escalates to the Top-3 area as a P2 advisory line.
  - Chronicle hooks: `Target reached` flip, streak-break (`Streak broken after <N> days`), anti-goal `Limit exceeded`.
- **`DOPS-NEW Goal-Done <goal_id> <YYYY-MM-DD>` capture** — log a manual goal hit when the user did the activity outside the calendar (e.g. spontaneous run, reading offline). Body fields: optional `Duration: <minutes>`, optional `Note: <text>`. Capture-drain and afternoon-sync both handle this — appends a `manual-capture` history entry to `goals.json`, increments streak if applicable, chronicle_logs. No ad-hoc task created (the capture is fully actioned).
- **Outlook-category intake — additive intake channel** alongside the DOPS-mailto pipeline:
  - **AUTO category**: `DOPS-Intake` — Cowork classifies based on context. Mail from own/personal address with verb-like first line → Task; customer-domain sender with contact change → Customer-Update; customer-domain generic → Task; past calendar event with external attendees → Meeting Note + Stakeholder fold; future event with deliverable-implying title → Task with due_date=event.start.
  - **TYPED override categories**: `DOPS-Intake-Task` / `DOPS-Intake-Customer` / `DOPS-Intake-Highlight` / `DOPS-Intake-Goal`. TYPED overrides AUTO if both are present on the same item.
  - **Lifecycle**: after successful processing, the original DOPS-Intake* category is removed and replaced with `DOPS-Intake-Done` (visual confirmation in Outlook). Mails additionally archive to `DOPS-Capture/_processed/YYYY-MM-DD/`. Failed processing leaves the category untouched — the next drain retries.
  - **Same trusted-sender security filter** as the DOPS-mailto pipeline. External-sender items surface under "Needs review — not processed for security".
  - **STEP 6 in capture-drain** scans both mail and calendar (`today−1d…today+14d`) every 10:00/12:00/14:00 run. Drain defers full meeting-note archival and customer-md updates to the afternoon sync.
  - **STEP 6.5 in afternoon-sync** repeats the scan more thoroughly: full meeting-note archival on AUTO past calendar events (writes `Meeting-Notes/YYYY-MM-DD-<CODE>.json` with `<CODE>` = `M-MMDD-CAT` format, CAT = first 6 chars of subject uppercase), full Kunden/<slug>.json updates. Catches anything tagged after the last drain. Write-backs flush BEFORE the mirror step so the new state lands in the same diff.
- **New triggers** in the SKILL.md frontmatter description: "personal goals", "goal adherence", "Soll/Ist", "tag in Outlook" — natural-phrase entries; the explicit `DOPS` commands stay unchanged.

### Changed

- **Selective load**:
  - Morning briefing reads `goals.json` (filtered to `active=true`) alongside the existing 11 living JSONs.
  - Capture-drain reads `goals.json` only on demand (Goal-Done capture, `DOPS-Intake-Goal` classification).
  - Afternoon-sync reads `goals.json` (active=true) — needed for Goal-Done handler and Category-Intake Goal classification.
  - Weekend briefing: unchanged (deliberately — goals are evaluated during the work week; private streak goals can be re-evaluated Monday morning if needed).
- **Snapshot targets** — `_meta.json.snapshot_targets` now includes `goals.json`. Daily snapshot folder grows by ~5 KB/day (negligible).
- **OneDrive mirror** — `goals.json` mirrors automatically via the existing hash-based diff. Re-PUT happens only on flips (status transitions, streak updates, hit additions, manual logs).
- **Briefing layout** extended with section "**15. Goals this week**". Subsequent vacation/year-rollover sections renumbered (16+).

### Migration from 2.0.x

- **No breaking changes — fully additive.** Existing data remains usable as-is.
- **Required**:
  1. Replace the three weekday scheduled prompts (morning-briefing, capture-drain, afternoon-sync) with the v2.1.0 prompt texts. Placeholder substitutions are unchanged.
  2. Populate `data/goals.json` with your real goals (the shipped file ships with four example goals: sport-kadenz, focus-zeitbudget, read-streak, meeting-anti-goal — replace freely or set `active=false` to disable).
  3. Create the Outlook categories: `DOPS-Intake`, `DOPS-Intake-Task`, `DOPS-Intake-Customer`, `DOPS-Intake-Highlight`, `DOPS-Intake-Goal`, `DOPS-Intake-Done`. SETUP.md describes the click path.
- **Optional**: existing morning-briefing scheduled task can stay paused while you populate `goals.json` — once `active=true` is set on at least one goal, the next run surfaces it.
- **No mail-rule changes** — DOPS-mailto pipeline keeps working exactly as before.
- **Weekend briefing is unchanged** — no v2.1.0 prompt update needed there.

### Notes

- **Why no auto-create for slot suggestions?** Goals are personal; the user wanted Cowork to inform but not act. A 1-click "book this slot" flow can be added in 2.2 if usage shows the manual step is friction.
- **Why a separate `Goal-Done` capture instead of reusing `DOPS-NEW`?** The handler short-circuits ad-hoc task creation — the capture is fully actioned (history append + chronicle_log) without leaving an inbox task. Keeping a distinct subject keyword (`Goal-Done`) makes the special handling explicit and unambiguous.
- **Why both AUTO and TYPED categories?** AUTO covers the lazy path ("just tag it, figure it out"). TYPED covers the precise path ("I know what this is, don't guess"). Override precedence keeps both safe to combine.
- **Storage growth**: `goals.json` per-goal history grows ~1 entry per fenster per goal. At 4 goals × 1 hit/day × 250 working days ≈ 1000 entries/year ≈ 80 KB/year. Negligible.
- **Mirror cost**: goals.json hashes change frequently (every hit logged). Adds ~3-5 PUTs/day to the existing ~9 PUTs/day baseline. Still negligible vs. quotas.

---

## [2.0.0] — 2026-05-03

**MAJOR — breaking migration from xlsx to JSON.** Manual migration required (see SETUP.md).

### Why 2.0

Two pain points triggered the rewrite:

1. **No mobile/manager access.** Cowork has no mobile app, and the xlsx files only lived on disk inside the skill sandbox. There was no way for the user (or their manager) to get a read-only look at the data on the go.
2. **Excel-lock fragility.** xlsx-in-mantle was brittle: openpyxl could not read every workbook reliably from the sandbox; capture-drain runs frequently skipped slots because the workbook was open on the desktop. The Excel-lock-aware retry logic became a constant source of edge cases.

The fix: full migration to JSON, plus a daily push to a personal OneDrive folder so the data is reachable from any device.

### New

- **JSON data model** — all 4 xlsx workbooks (Cowork-Hub, Tasks-Inbox, wichtige-daten, highlights) decompose into 11 small JSON files in `data/`:
  - `_meta.json` — schema/skill version, MD5 hashes per file (for the mirror diff), `last_mirror` / `last_snapshot` timestamps, snapshot target list.
  - `customers.json` — single customers list (was: Cowork-Hub sheet 1).
  - `stakeholders-internal.json` / `stakeholders-customer.json` — split into two files (was: Cowork-Hub sheets 2–3).
  - `partners.json` (sheet 4), `projects.json` (sheet 5), `hours-log.json` (sheet 6), `watchlist.json` (sheet 7).
  - `tasks.json` — single file with `status` field (`inbox` / `active` / `blocked` / `done` / `backlog` / `cancelled`). Replaces the three Tasks-Inbox sheets.
  - `wichtige-daten.json` — `personen_anlaesse` / `ferien_frei` / `urlaub` (bilanz + config + eintraege) in one file.
  - `highlights.json` — `highlights` / `review_cycles` / `numbers_tracker` in one file.
  - `Bulk-Sources.json` — sources + heuristics + allowlist (was: Bulk-Sources.md, free-form).
- **Customer files as JSON** — `data/Kunden/<slug>.json` with a `content_md` wrapper holds the human-readable narrative; structured fields cover stakeholder cross-links and chronicle.
- **Meeting-Notes as JSON** — `data/Meeting-Notes/YYYY-MM-DD-<CODE>.json` with `kind`, `event_id`, `event_subject`, `attendees`, `content_md`, `new_stakeholders`, `follow_up_task_ids`.
- **Briefing files as JSON** — `data/Briefings/YYYY-MM-DD-{morning,afternoon,weekend}.json` with `content_md` wrapper plus structured fields (`codes_map`, `stats`, `vacation_state_snapshot`).
- **Universal Notes + Chronicle pattern** — every record (customer, stakeholder, project, partner, task, watchlist item, holiday, vacation entry, highlight, customer file) carries `notes` (free text, single field) and `chronicle` (timestamped array). Operations vocabulary available across the briefing/sync logic:
  - `note_replace(record_id, text)`, `note_append(record_id, text)`, `note_clear(record_id)`
  - `chronicle_log(record_id, entry, topic?, event_id?, event_subject?, outcome?, follow_up_task_ids?)` — append a timestamped entry with optional calendar cross-reference
  - `chronicle_edit(record_id, ts, fields)`, `chronicle_link(record_id, ts, event_id, event_subject)`
  - Stakeholder contact events log automatically with the linked calendar event.
- **OneDrive mirror** — `data/` is mirrored to a personal OneDrive folder `daily-ops-readonly/` after each scheduled run that does heavy synthesis (morning briefing, weekend briefing, afternoon sync). Hash-based diff: only files with a changed MD5 are pushed via Graph API. `_meta.json` is updated last after all hashes settle, then re-PUT. Mirror failures log under "Needs review" but do NOT block the briefing/sync send.
- **Daily snapshots** — at 15:30 the afternoon sync copies the 10 living JSONs (everything in `_meta.json.snapshot_targets`) to `data/_snapshots/YYYY-MM-DD/`. No ZIP, no pruning. Briefings, customer JSONs, and meeting-notes are date-keyed and self-historical, so they need no extra snapshot. ~250 KB per day, ~60 MB after a full year — acceptable; user can prune manually if needed.
- **Event-driven vacation recalculation** — `urlaub.bilanz[year]` carries a `stale` flag and a `trigger_recalc_on` list (`["calendar-change", "manual", "year-rollover"]`). The afternoon sync sets `stale=true` when calendar events with category "Urlaub" change; the user can also flag it via `DOPS-NEW Urlaub-Recalc` capture. The morning briefing performs the actual recalculation when `stale=true` or `last_calculated` is older than 7 days, then resets `stale=false` and stamps `last_calculated`.
- **`DOPS-NEW Mirror-Now` capture** — ad-hoc trigger to force an interim OneDrive mirror in the next capture-drain run (for travel-day exceptions where the user needs the latest state pushed before 15:30).
- **`DOPS-NEW Urlaub-Recalc` capture** — flag `urlaub.bilanz[current_year].stale=true` so the next morning briefing recomputes the balance from scratch.

### Changed

- **Selective load model** — each scheduled run reads only the JSON subset it needs, keeping per-run context under 100 KB:
  - Morning briefing: all 11 living JSONs + `Bulk-Sources.json` (chronicle filtered to last 90d on stakeholders, last 90d on highlights).
  - Capture drain: `_meta` + `tasks.json` (open only) + `stakeholders-customer.json` (chronicle last 30d) + state file. `customers.json` only on demand.
  - Afternoon sync: `_meta` + `tasks.json` + customer/stakeholder/project JSONs + today's morning briefing codes_map + capture mails + `wichtige-daten.json` (urlaub block).
  - Weekend briefing: `_meta` + `tasks.json` (filtered `bereich="Privat"`) + `wichtige-daten.json`. `customers.json` on demand for the sanity check.
- **Mirror placement** — capture-drain runs do NOT mirror (too frequent, would burn Graph API calls without benefit). Mirror happens only in the morning briefing, weekend briefing, and afternoon sync. Three mirrors per day is the sweet spot.
- **Excel-lock-aware retry logic — REMOVED.** No longer applicable. JSON reads/writes are unconditional; capture-drain runs always succeed unless the morning briefing file is missing.
- **Monday-only Excel backup — REMOVED.** Replaced by the daily snapshot folder. Monday no longer carries extra work in the morning briefing.
- **Customer markdown maintenance** — moved from per-file `.md` edits to `Kunden/<slug>.json` with `content_md` updates. Same human-readable content, cleaner programmatic access.
- **Meeting-Notes archival** — afternoon sync writes `.json` files (with `content_md` wrapper) instead of plain `.md`. Captures `attendees`, `new_stakeholders`, `follow_up_task_ids` as structured fields.
- **Capture-drain M-code handling** — drained M-codes are marked with mail category "DOPS-drained" so the afternoon sync knows the inbox-fold step (stakeholders + follow-up tasks) has already happened, but the full meeting-notes archival to `Meeting-Notes/<date>-<code>.json` still runs in the afternoon sync as before.

### Migration from 1.7.x

The data format is incompatible. There is no auto-upgrade path. To migrate manually:

1. **Back up the old `data/` folder** (xlsx + Bulk-Sources.md + Kunden/*.md) outside the skill so you can reference your real content during the migration.
2. **Re-import the v2.0.0 skill** — overwrites `data/` with the new JSON template and the empty `Briefings/`, `Meeting-Notes/`, `_snapshots/` folders.
3. **Move your real data into the JSONs**:
   - Cowork-Hub sheet 1 → `customers.json` `customers` array.
   - Sheets 2/3 → `stakeholders-internal.json` / `stakeholders-customer.json`.
   - Sheets 4–7 → `partners.json` / `projects.json` / `hours-log.json` / `watchlist.json`.
   - Tasks-Inbox sheets → `tasks.json` (preserve `status` field per row).
   - `wichtige-daten.xlsx` → `wichtige-daten.json` (Personen & Anlässe → `personen_anlaesse`; Ferien & Frei → `ferien_frei`; Urlaub → `urlaub.bilanz` + `urlaub.eintraege`; copy `Default-Anspruch` / `Max-Rollover-Tage` into `urlaub.config`).
   - `highlights.xlsx` → `highlights.json` (3 arrays).
   - `Bulk-Sources.md` → `Bulk-Sources.json` (`sources` array; `heuristics` and `allowlist` blocks).
   - One `Kunden/<slug>.json` per customer, copying the old `.md` content into `content_md`.
4. **Update all four scheduled prompts** with the v2.0.0 prompt texts from `scheduled-prompts/`. Replace placeholders as before.
5. **Create the OneDrive mirror folder** `daily-ops-readonly/` in your personal OneDrive (or let the first run auto-create it).
6. **First run** — manually trigger a morning briefing. Verify the mirror runs and `_meta.json` `files.<filename>.hash` fills in for every JSON. Check the OneDrive folder shows all files.
7. **Optional**: trigger a `DOPS-NEW Urlaub-Recalc` capture once to validate the stale-flag recalculation path.

### Notes

- **Storage growth**: chronicle entries accumulate over time. Year-1 estimate at ~5 chronicle entries per stakeholder per quarter on a portfolio of ~50 stakeholders is ~4 MB total — small enough that no compaction is needed for several years. If individual JSONs grow past ~500 KB, consider archiving completed customers to a separate `customers-archive.json`.
- **Mirror cost**: Graph API PUT calls per day at the steady state (3 mirror runs × ~3 changed files each) ≈ 9 calls/day. Negligible vs. quotas.
- **Snapshot cost**: ~60 MB/year on disk. Tiny.
- **Why not SQLite?** JSON keeps the data trivially diff-able, hand-editable, and grep-friendly. The whole skill operates on file reads/writes; introducing a query engine would make the OneDrive mirror harder and gain nothing for portfolios under ~1000 records.

---

## [1.7.0] — 2026-05-03

### New

- **Capture drain — fourth scheduled prompt** (`scheduled-prompts/capture-drain.md`):
  - Mon–Fri at 10:00 / 12:00 / 14:00 — three lightweight runs between the morning briefing and the afternoon sync.
  - Folds new captures (T- / P- task updates and `DOPS-NEW` ad-hoc tasks) into `Tasks-Inbox.xlsx` so the inbox stays current during the day.
  - **Silent by default** — only sends a "Needs review" mail if a capture was rejected by the security filter, an unrecognized status word appeared, a `reschedule` was missing a deadline, or an M-code mail had a parser error.
  - **Excel-lock-aware** — checks for `~$Tasks-Inbox.xlsx` / `~$Cowork-Hub.xlsx` before writing. Up to 3 retries with 60s wait. If still locked, the run is skipped and logged in the state file (next slot picks up).
  - **Restricted scope** — no highlight detection, no end-of-day proposal, no full meeting-notes archival. Heavy synthesis stays in the afternoon sync. M-code mails are only "drained" (stakeholders + follow-up tasks folded in) and left in `DOPS-Capture` for the afternoon sync to archive properly.
  - State file at `<SKILL_PATH>/data/.drain-state.json` tracks last-run timestamp, processed count, and any skipped reasons.

- **Vacation tracking — new `Urlaub` sheet in `wichtige-daten.xlsx`**:
  - **Vacation balance table per year** — one row per year: `Jahr | Anspruch | Übertrag aus Vorjahr | Verbraucht | Verbleibend | carryover deadline`. `verbraucht` is a `SUMIFS` formula over the entries below filtered by year (excluding `storniert`); `verbleibend = anspruch + uebertrag_vorjahr − verbraucht`.
  - **Config** — `Default-Anspruch (Tage)` (default 30) and `Max-Rollover-Tage` (default 5). Defaults for new bilanz rows; override per row as needed.
  - **Einträge-Tabelle** — `Von | Bis | Tage | Status | Kommentar`. Status enum: `geplant` (booked externally, not yet officially submitted) | `eingereicht` (submitted, awaiting approval) | `genehmigt` (approved) | `storniert` (cancelled — does not count toward `Verbraucht`).
  - Real Excel date types in `Von` / `Bis` columns so the `SUMIFS(...DATE(year,1,1)...DATE(year,12,31)...)` filter works without helper columns.

- **Vacation logic in the morning briefing (logic 5)**:
  - **5a — Year rollover** (only on the first weekday of a new calendar year, or when the current year has no bilanz row): adds a row with `anspruch = default_anspruch`, `uebertrag_vorjahr = MIN(verbleibend(prev_year), max_rollover_tage)`, `carryover deadline = "30.06.<year>"`. Surfaces a confirmation line for 3 consecutive days or until the user touches the row.
  - **5b — Daily alerts**: status `geplant` is the high-alarm path — surfaces in section "Vacation requests pending submission" at ≤60d, escalates at ≤30d, becomes a P1 in the Top-3 at ≤14d, daily until status changes. Status `eingereicht` surfaces in "Approval pending" at ≤14d (no escalation).
  - **5c — Carryover deadline**: when a bilanz row has unconsumed carryover, an alert escalates 60d → 30d → 14d (P1) before the carryover deadline. After the deadline, `verbleibend` is clamped and a one-time "carryover days expired" line is shown.

- **Vacation logic in the weekend briefing**: vacation entries within the next 30 days appear with markers (`⚠️ Submission missing` for `geplant`, `⏳ Approval pending` for `eingereicht`, `✓ approved` for `genehmigt`, `storniert` omitted). One-line carryover reminder when carryover deadline is within 60 days.

- **New triggers**: `DOPS drain` (manual capture-drain run), `DOPS urlaub` (vacation balance + open submissions).

### Changed

- **Day cycle diagram** in `SKILL.md` updated to four scheduled prompts (morning + drain ×3 + afternoon + weekend).
- **Morning briefing structure** extended with sections 15 (Year rollover), 16 (Vacation requests pending submission), 17 (Approval pending), 18 (Vacation balance). All hidden when empty so quiet days stay quiet.
- **Top-3 selection** now also considers escalated vacation alerts at ≤14d as P1 candidates.

### Migration from 1.6.x

- **No Excel schema breaks**. The new `Urlaub` sheet is added to `wichtige-daten.xlsx`. Existing rows in `Personen & Anlässe` and `Ferien & Frei` are untouched.
- **Required** when upgrading an existing `wichtige-daten.xlsx` manually:
  - Either copy the v1.7.0 template's `Urlaub` sheet over (Excel: right-click sheet → "Move or Copy" → target workbook), or re-import the workbook from the v1.7.0 release.
  - Fill in your current year's bilanz row (`Anspruch`, `Übertrag aus Vorjahr`). The `Verbraucht` formula and `Verbleibend` fill automatically as you add entries.
  - Optionally adjust `Default-Anspruch` and `Max-Rollover-Tage` in the config block to match your contract.
- **Required**: add a fourth scheduled task with the `capture-drain.md` prompt (Mon–Fri 10:00/12:00/14:00, inline). The `<SKILL_PATH>`, `<YOUR_MAIL>`, `<PERSONAL_MAIL_1..3>`, `<SAFE_SENDERS>` placeholders are the same as in the afternoon sync — copy them across once.
- **Required for the security loop**: the drain shares the afternoon sync's allowlist. If you have customized `<SAFE_SENDERS>` there, mirror the change in the drain prompt.
- **Optional**: rename your existing morning-briefing scheduled task to make room for the new `DOPS urlaub` trigger if you want a dedicated balance check; otherwise the morning briefing already surfaces the relevant vacation lines automatically.

### Notes

- The drain's three default slots (10/12/14) are tuned for desk-bound days. If you mostly capture during one window (e.g. lunch), drop to two slots or one — the run is cheap but should not run more than every 90 min to avoid hammering an already-locked workbook.
- The vacation `geplant` status is intentionally separate from `eingereicht`: only "extern booked, internally still open" deserves an alarm. Once you submit to HR, change the status to `eingereicht` and the daily alarm goes silent.

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
  - Captures from senders outside the allowlist are NOT processed but moved to `DOPS-Capture/_rejected/YYYY-MM-DD/` and reported under "Needs review" in the sync report.
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
