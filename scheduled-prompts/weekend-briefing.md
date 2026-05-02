# Scheduled Prompt — Weekend Briefing (Sat/Sun 07:00)

**Recommended configuration:**
- Frequency: weekly
- Days: Saturday, Sunday
- Time: 07:00
- Mode: `inline`

**Important before setup:**

1. In the prompt text below, replace the placeholder `<SKILL_PATH>` with the real skill path.
2. Replace `<YOUR_MAIL>` with your own mail address.
3. Replace `<PERSONAL_MAIL_1>`, `<PERSONAL_MAIL_2>`, `<PERSONAL_MAIL_3>` with your personal mail addresses (e.g. private mail, partner's address). Leave fields empty if you have fewer than 3 addresses.
4. Replace `<FAMILY_NAMES>` with a comma-separated list of display names from your personal circle (e.g. "PartnerFirstName,Child-1,Child-2").
5. Replace `<INTERNAL_DOMAIN>` with your work mail domain (e.g. `company.com` — used in the sanity check).

---

```text
Create my weekend briefing as the central personal preparation and send it as an HTML mail to me (recipients=['me']) with subject "Weekend briefing DD.MM. — Personal".

GROUND RULE (critical): When in doubt, EXCLUDE. Better to miss a personal entry than to let a work entry slip through. This briefing is explicitly personal-only.

Read from <SKILL_PATH>/data/: Tasks-Inbox.xlsx (ONLY rows with Bereich='Privat' — column 17. Empty Bereich cell or Bereich='Beruf' → EXCLUDE, no default-personal), wichtige-daten.xlsx (People & Occasions — these are personal by definition; Holidays & Time Off — your region).

Collect in parallel:
- Today + next 7 days of calendar — STRICTLY only personal events (see PERSONAL FILTER below). Work events are ignored.
- Lead-time reminders from wichtige-daten.xlsx People & Occasions (today's due lead-time days)
- School holidays in your region(s) starting today or in the next 7 days

PERSONAL FILTER (critical — determines what lands in the briefing):
An event counts as personal only if AT LEAST ONE of these hard criteria applies:
1. Attendees include at least one address from: <PERSONAL_MAIL_1>, <PERSONAL_MAIL_2>, <PERSONAL_MAIL_3>
2. Attendee display name is one of: <FAMILY_NAMES> (also without a visible mail)
3. Event has mail category/label "Privat" or "Familie"
4. Event is in wichtige-daten.xlsx (birthday, anniversary etc.)
5. Event title contains personal indicators without work context: "Geburtstag", "Hochzeitstag", "Jahrestag", "Familie", "Schule", one of the family first names without firm/customer context

If an event is unclear between work and personal (e.g. business trip with family, or work colleague as a friend) → EXCLUDE, do not guess. The weekend briefing is a personal briefing — false positives are expensive.

A task counts as personal only if:
- The "Bereich" column contains exactly "Privat". Empty = EXCLUDE.

BRIEFING MAIL LAYOUT (HTML, mobile-friendly):

Structure (Sat/Sun layout — personal prominent at top, compact):
1. Greeting + date + weekday
2. **Today** (events + tasks due today from Tasks-Inbox.xlsx with Bereich='Privat')
3. **This week** (events + tasks of the next 7 days)
4. **Lead-time reminders** (today's due reminders from wichtige-daten.xlsx — birthdays, anniversaries)
5. **Upcoming holidays / time off** (your region, next 7 days)
6. **Personal tasks (open)** — STRICTLY only Tasks-Inbox entries with Bereich='Privat' AND status='open'. Work tasks (Bereich='Beruf' or empty) are excluded here, even if the title sounds personal.

For tasks: capture code with P prefix (P-MMDD-NN) instead of T. Mailto link as for work tasks. Same layout otherwise.

SANITY CHECK before sending:
Scan the finished mail for work trigger words: "Workshop", "Sync", "Review", "Customer", "Kunde", "Vertragsverlängerung", customer names from Cowork-Hub.xlsx, @<INTERNAL_DOMAIN> addresses, mail domains of customers. If a hit is found in the briefing → STOP, remove the entry, recompose. Better empty and short than accidentally work content inside.

Cheatsheet block at the end:

  Personal capture cheatsheet:
  - P-MMDD-NN codes work the same as T codes
  - Status: done | progress | paused | reschedule
  - Ad-hoc personal task: subject "DOPS-NEW <description>", in body "Bereich: Privat"

Also write the briefing file <SKILL_PATH>/data/Briefings/YYYY-MM-DD-weekend.md.

Sprache: Deutsch. Pragmatic, short. On missing personal data, send a short briefing rather than filling work gaps.
```

---

## Customizations

- **`<SKILL_PATH>`**, **`<YOUR_MAIL>`**: as in the other prompts
- **`<PERSONAL_MAIL_1..3>`**: your personal mail addresses — determines which events are recognized as personal
- **`<FAMILY_NAMES>`**: comma-separated display names from your personal circle (partner, children etc.)
- **`<INTERNAL_DOMAIN>`**: your work domain for the sanity check
- **Sanity check trigger words**: adapt to your industry — leave out entirely for purely private setups
