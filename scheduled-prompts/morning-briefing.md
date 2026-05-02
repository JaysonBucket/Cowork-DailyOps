# Scheduled Prompt — Morning Briefing (07:00)

**Recommended configuration:**
- Frequency: weekly
- Days: Monday–Friday
- Time: 07:00
- Mode: `inline` (same session — Cowork learns your style over time)

**Important before setup:**

1. In the prompt text below, replace the placeholder `<SKILL_PATH>` with the real skill path of your environment (see SETUP.md for common values).
2. Replace `<YOUR_MAIL>` with your own mail address (all `mailto:` links go to yourself).
3. Replace `<INTERNAL_DOMAIN>` with your internal mail domain (e.g. `company.com`) — otherwise internal colleagues will be detected as external customer contacts.
4. Replace `<PERSONAL_MAIL_1..3>` and `<FAMILY_NAMES>` with your personal addresses/names (NEW in 1.6.0 — drives work/personal detection). Leave fields empty if not applicable.
5. **Mail rule** for the DOPS-Capture folder is set up (instructions in SETUP.md).

---

```text
Create my morning briefing as the central daily preparation and send it as an HTML mail to me (recipients=['me']) with subject "Morning briefing DD.MM. — Top 3: <top task title>".

Read from <SKILL_PATH>/data/: Cowork-Hub.xlsx (Customers, Internal Stakeholders, Customer Stakeholders, Watchlist), Tasks-Inbox.xlsx (Inbox sheet incl. capture-code and area columns), wichtige-daten.xlsx (People & Occasions, Holidays & Time Off), highlights.xlsx (Highlights Log, Review Cycles), Bulk-Sources.md (mass-mail classification).

Collect in parallel:
- Today + next 14 days of calendar (events + status, attendee mails, flag categorized vs. uncategorized events)
- New mails since yesterday evening — read From + all CC addresses. IMPORTANT: exclude mails with `DOPS-` in the subject (those are your own captures from the day — they are processed by the afternoon sync, not by the morning briefing)
- New Teams messages in 1:1 and group chats since yesterday — read participants
- Watchlist hits in mail/Teams (people, buzzwords from Cowork-Hub.xlsx Watchlist sheet)
- Lead-time reminders from wichtige-daten.xlsx (today's due lead-time days)
- School holidays in your region(s) starting today or in the next 7 days

BULK FILTERING (logic 0 — before everything else):
Before feeding mails/Teams into the other logics, classify each message:
1. Match against Bulk-Sources.md (sources list): exact sender / domain wildcard / subject pattern / chat name
2. If no match: check heuristics (>20 recipients, DL sender like noreply/news/digest/lists, typical bulk subjects like "Newsletter"/"Weekly"/"Digest"/"[FYI]"/"Reminder:"/"Survey"/"Webinar", HTML with unsubscribe link, group chat with >15 participants without @-mention to me, no personal greeting). At least 2 hits → bulk candidate.
3. Allowlist from Bulk-Sources.md ALWAYS takes precedence — never treat manager / customer domains as bulk, even on DL sends.

Treatment per message:
- "Ignore" → not in the briefing at all, move directly to mail folder "Archiv/Bulk" (create the folder if missing)
- "KeyPoints" → 1–2 sentences in the section "Bulk key points", then move to "Archiv/Bulk". Do NOT quote full text — only on explicit follow-up.
- "FullText" → feed into the main logics like a normal mail (task detection, watchlist, highlight detection)
- New bulk candidates without an entry in Bulk-Sources.md → list under "New bulk sources detected" with a treatment proposal, do NOT archive until I confirm.

IMPORTANT: bulk mails MUST NOT feed into people-meeting matching, customer stakeholder maintenance, or task detection. They are explicitly excluded from those logics.

PERSONAL FILTER (logic 0b — NEW in 1.6.0 — directly after bulk filtering, before all work logics):
Classify every meeting and every task into one of two buckets: work or personal.

Personal detection — ANY of these criteria is enough:
1. Calendar event has at least one attendee from: <PERSONAL_MAIL_1>, <PERSONAL_MAIL_2>, <PERSONAL_MAIL_3>
2. Calendar event has an attendee with a display name from: <FAMILY_NAMES> (also without a visible mail)
3. Calendar event has mail category/label "Privat" or "Familie"
4. Entry in wichtige-daten.xlsx → automatically personal (birthdays, anniversaries, holidays)
5. Tasks-Inbox.xlsx column "Bereich" exactly = "Privat"
6. Event title contains personal indicators without work context: "Geburtstag", "Hochzeitstag", "Jahrestag", "Familie", "Schule", one of the family first names

Treatment of personal events: NO capture code, NO mailto link. Own briefing section at the end. NOT in people-meeting matching with work tasks. NOT in customer stakeholder maintenance.

Treatment of personal tasks (Bereich="Privat"): capture code with P-prefix instead of T (P-MMDD-NN). Own briefing section "Personal tasks (open)" — NOT in top 3. Mailto link works the same as for work tasks.

Treatment of new tasks from mails/Teams: default Bereich="Beruf". If the mail is from or to one of the personal addresses: Bereich="Privat".

Weekday layout:
- Mon–Fri: personal sections at the end of the briefing, compact. Top 3 is work-only.
- Sat/Sun: this prompt does not fire. Instead the separate weekend briefing runs (see scheduled-prompts/weekend-briefing.md).

PEOPLE-MEETING MATCHING (logic 1 — work only):
For every open task in Tasks-Inbox.xlsx with the "People (Mail)" column populated:
1. Search the calendar of the next 14 days for events with at least one of these people attending
2. If hit: write the next shared event into the column "Next shared meeting" (format: "DD.MM. HH:MM Event title") and the date in "Meeting date"
3. If the shared event is BEFORE the deadline AND status="open": recommend in the briefing "Handle before meeting on DD.MM. with X" — own section "Pull forward before meeting"
4. If the shared event is AFTER the deadline: just note as info ("Follow-up conversation on DD.MM. with X")

For new tasks from mails/Teams: ALWAYS enter all From + CC addresses into "People (Mail)" (comma-separated) and the display names into "People (Names)". Source type: "Mail", "Teams" or "Meeting". Source link: mail ID / chat link / event ID.

CUSTOMER STAKEHOLDER MAINTENANCE (logic 2):
When external people appear in mails/Teams/calendar events (mail domain is NOT @<INTERNAL_DOMAIN>), match them against Cowork-Hub.xlsx sheet "Kunden-Stakeholder":
1. Customer mapping: mail domain → customer sheet (column "Customer name") match. If unclear, derive customer from mail/event context (subject, other attendees).
2. YES → "Last contact" set to today, "# Contacts" +1.
   NO → new row with customer, name, role ("(add role)"), decision-maker level ("(add level)"), mail, first contact date=today, channel, occasion, last contact=today, # contacts=1.
3. Also add to <SKILL_PATH>/data/Kunden/<CustomerName>.md under "## People / Stakeholders".

CAPTURE CODE ASSIGNMENT (logic 3 — extended in 1.6.0):
Assign every entry in the briefing a unique capture code per schema:
- Work tasks: T-MMDD-NN (NN = sequential number per day, starting at 01)
- Personal tasks (Bereich="Privat"): P-MMDD-NN (separate sequential numbering per day, starting at 01 — NEW in 1.6.0)
- Meetings (work): M-MMDD-XX (XX = short tag from event title, max 6 chars, uppercase, no special chars — e.g. "Project Review Sync" → "PR" or "PRSYNC")
- Personal events: NO code (see logic 0b)
- For tasks: write the code into the column "Capture-Code" of Tasks-Inbox.xlsx (do not overwrite if already filled — existing codes stay stable across multiple days!)
- For meetings: code is generated only for the briefing, not persisted (meetings have event IDs)

MAILTO LINK GENERATION (logic 4 — NEW in 1.4):
For every capture code in the briefing, generate a mailto: link with URL-encoded subject and body. Recipient is always <YOUR_MAIL>.

Schema for tasks:
  Subject: "DOPS-<CAPTURE-CODE> <task title>"
  Body:
    Status: [done | progress | paused | reschedule]

    Note:

    ---
    Context: <task title>, due <DD.MM.>, people: <people names>, source: <source type>

Schema for meetings:
  Subject: "DOPS-<CAPTURE-CODE> <event title>"
  Body:
    Notes:

    New stakeholders: (Name <Mail>, Role)
    Follow-up tasks: (description — Cowork creates an inbox entry automatically)

    ---
    Context: <event title>, <DD.MM. HH:MM>-<HH:MM>, attendees: <attendee list>

Schema for ad-hoc task:
  Subject: "DOPS-NEW <short-description>"
  Body:
    Description:
    Due:
    People:
    Bereich: [Beruf | Privat]
    Priority: [P1 | P2 | P3]

URL encoding: space → %20, newline → %0A, colon → %3A, pipe → %7C, square brackets → %5B/%5D, comma → %2C, hyphen stays as - (do not escape).

LENGTH LIMIT (critical — otherwise mail clients/browsers will not open the link):
Mail clients reliably accept mailto URLs up to ~2000 characters. Compute total URL length per entry BEFORE inserting into the HTML.

Tier 1 (default): full body as defined above. Verify URL length.

IMPORTANT: notes ALWAYS go AFTER the `---` separator, never before — the afternoon sync parser ignores everything after `---` as a context block. If notes appear before `---`, they would be interpreted as user input and break the status parser.

Tier 2 (URL > 1800 chars): trim the context block after `---` — keep only title + date, drop the people/attendee list and source. Directly underneath (still after `---`) add a hint line: "Full context in the daily briefing: <SKILL_PATH>/data/Briefings/YYYY-MM-DD-morning.md".

Tier 3 (still > 1900 chars): drop the context block entirely. Body reduces to:
- Tasks: "Status:\n\nNote:\n\n---\nContext trimmed — see daily briefing YYYY-MM-DD-morning.md"
- Meetings: "Notes:\n\nNew stakeholders:\nFollow-up tasks:\n\n---\nContext trimmed — see daily briefing YYYY-MM-DD-morning.md"
Also display the symbol "📎 context trimmed" next to the entry in the briefing HTML — so when reading the briefing you already know the capture mail does not contain everything.

In the NORMAL CASE (>95% of tasks/meetings) tier 1 is enough.

BRIEFING MAIL LAYOUT (HTML, mobile-friendly):
The mail must be readable on mobile — short paragraphs, no wide tables, clear buttons.

Structure:
1. Greeting + date
2. Top 3 for today (P1 tasks from inbox) — per entry: title, code in parentheses, mailto link "Send update"
3. **Pull forward before meeting** — per entry: "T-XXXX-NN — task XYZ before meeting DD.MM. with person" + mailto link
4. Today's events — per entry: "M-XXXX-XX — time title, attendees" + mailto link "Send notes"
5. Lead-time alerts (personal dates from wichtige-daten.xlsx)
6. Watchlist hits since yesterday
7. New tasks since yesterday (identified from mails/Teams) — already entered in inbox WITH people columns and capture code
8. **New customer contacts** (newly created today — role/level open)
9. Highlight detection (positive customer mails, completed milestones yesterday)
10. **Bulk key points** (mass mails / newsletters / large group chats — 1–2 sentences each, archived to "Archiv/Bulk")
11. **New bulk sources detected** (candidates for inclusion in Bulk-Sources.md with treatment proposal — awaiting confirmation)
12. Upcoming holidays / time off in your region
13. **Personal events today & next 7 days** (compact, at the end — NEW in 1.6.0)
14. **Personal tasks (open)** (Bereich="Privat" from Tasks-Inbox.xlsx, with P codes — NEW in 1.6.0)

At the end of the mail a **cheatsheet block**:

  Capture cheatsheet:
  - Click "Send update" / "Send notes" — mail opens pre-filled
  - Status word in the 1st line (done | progress | paused | reschedule)
  - Note underneath, leave the rest unchanged
  - Ad-hoc task: new mail, subject "DOPS-NEW short-description"
  - Folded in by the daily sync at 15:30

Also write the briefing file <SKILL_PATH>/data/Briefings/YYYY-MM-DD-morning.md (same content as the mail, without mailto links). Codes map at the end of the file: a table "Capture code → Task ID / Event ID" MUST exist for the afternoon sync.

Enter new tasks into the Inbox sheet. Track detected highlights as "proposed". Update the customers sheet "Last contact". For confirmed bulk sources add an entry to Bulk-Sources.md.

ADDITIONALLY ON MONDAY (backup step before the briefing): copy all 4 xlsx files plus the Kunden/ folder to the cloud backup folder "daily-ops-ext/_backup/YYYY-MM-DD/". Keep the last 8 weekly backups, delete older ones.

Sprache: Deutsch (or adjust). Pragmatic, no sugarcoating. On missing data/stakeholders flag actively rather than papering over gaps.
```

---

## Customizations

- **`<SKILL_PATH>`**: replace fully before the first run (typically: your real skill path in the Cowork environment)
- **`<YOUR_MAIL>`**: your own mail address for the mailto links
- **`<INTERNAL_DOMAIN>`**: your internal mail domain (e.g. `company.com`) — otherwise internal colleagues will be detected as external customers
- **Language, region, backup target**: as in previous versions
