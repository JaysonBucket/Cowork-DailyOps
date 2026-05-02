# Bulk-Sources — mass mails & group chats

Cowork maintains this list itself over time. You can adjust it any time.

## How it works

During the morning briefing, Cowork identifies mails / Teams messages that look like mass communication (see heuristics below). Hits are matched against this list:

- **Treatment "KeyPoints"** → only 1–2 sentences in the briefing covering what is relevant. Full text only on follow-up. The mail is moved to the folder `Archiv/Bulk` after the briefing.
- **Treatment "FullText"** → treat like a normal mail (exception to the rule, e.g. important manager updates)
- **Treatment "Ignore"** → not mentioned in the briefing at all, archived directly

New sources not yet in the list are proposed under "New bulk sources detected" in the briefing for confirmation — you give them a treatment.

## Detection heuristics

Cowork flags a mail/message as a bulk candidate if **at least two** apply:

- > 20 recipients in To/CC
- Sender is a distribution list (`*@lists.*`, `*-dl@*`, `noreply@*`, `donotreply@*`, `news@*`, `digest@*`)
- Subject contains: "Newsletter", "Weekly", "Daily", "Digest", "[FYI]", "Update —", "Announcement", "Reminder:", "Heads up:", "Survey", "Webinar"
- HTML mail with banner image and/or unsubscribe link
- Teams group chat with > 15 participants and you are not directly @-mentioned
- No personal greeting addressed to you (no "Hi {{FirstName}}", no "@{{FirstName}}")

---

## Sources list

| Type | Pattern (sender / domain / subject / chat name) | Treatment | Reason | Created on |
|------|--------------------------------------------------|-----------|--------|------------|
| Sender | _(example)_ `noreply@company.com` | Ignore | System notifications | _(date)_ |
| Domain | _(example)_ `*@lists.company.com` | KeyPoints | Distribution lists — mostly FYI | _(date)_ |
| Subject | _(example)_ `Weekly Digest*` | KeyPoints | Newsletter — rarely needs action | _(date)_ |
| Chat | _(example)_ `Team All-Hands` | KeyPoints | Large channel, only relevant on @mention | _(date)_ |

---

## Exceptions / allowlist

Whatever sits here is NEVER treated as bulk — even when the heuristics apply.

| Pattern | Reason |
|---------|--------|
| _(example)_ manager mail address | Even on DL mails: always FullText |
| _(example)_ customer domain | Never archive external mails as bulk |

---

## Archive target

Default: Outlook folder `Archiv/Bulk` (created automatically if missing).

Adjustable in the scheduled-prompt text — change the target folder if you use a different structure.

---

## Maintenance

- **Weekly** Cowork proposes under "New bulk sources detected" what was added this week
- **Quarterly** check whether the treatments still fit (what used to be "KeyPoints" might now be "Ignore")
- If an archived bulk mail bothers you ("I would have liked to see that one"), set the treatment in the table to "FullText"
