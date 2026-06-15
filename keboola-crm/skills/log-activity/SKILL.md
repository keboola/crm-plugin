---
name: log-activity
description: The canonical playbook for capturing a GREAT activity record in the Keboola CRM — log a meeting/call/note, record who you met (people met / attendees), set when it actually happened (including backdating a meeting logged late), enrich or correct an already-logged activity in place, and link attendees to CRM contacts. Activates when the user wants to log/record/capture a meeting, call, demo, QBR, check-in or note; mentions "people met", "who attended", "attendees", "who was in the room"; wants to fix/correct/backdate an activity's date; or wants to add/remove people on an existing activity.
allowed-tools: ['Bash']
---

# Capture a Great Activity Record

You help reps and CSMs turn a customer interaction into a high-signal CRM
activity. The activity log is the source of truth for pipeline freshness,
account-health scoring, "who we've met" relationship graphs, and every
review Sales / CS / management run. A sloppy record (no attendees, wrong
date, one-line notes) quietly degrades all of those.

A great activity record answers four questions:

| | Question | How you capture it |
|---|---|---|
| **WHO** | Who did we meet? | `--met "Name <email>:role"` (repeatable) — first-class, clickable, auto-linked to contacts |
| **WHEN** | When did it actually happen? | `--at "YYYY-MM-DD HH:MM"` — set it whenever you log after the fact |
| **WHAT** | What was discussed / decided? | a structured `--subject` + `--notes` |
| **NEXT** | What happens next? | follow-ups captured in the notes (and MEDDPICC / next-step fields on the opp) |

## The golden rules

1. **Always capture the people you met with `--met`, not `--participants`.**
   `--met "Jan Novák <jan@firma.cz>:met"` records a first-class *person met*
   that shows under "People met", is clickable, and **auto-links to an
   existing CRM contact** when the email matches one on the account — no
   extra step. `--participants` only stores bare emails on the call record;
   they do **not** show as "who we met". Prefer `--met`. Use `--participants`
   only when you genuinely have nothing but a list of emails.
2. **If you log late, set `--at` to when the meeting actually happened.**
   Without it, the activity is stamped with *now* (the creation time), so a
   meeting from Tuesday morning logged on Thursday shows as Thursday. Pass the
   real local time: `--at "2026-07-08 10:00"`. (Date-only `--at 2026-07-08`
   is fine if you don't know the time.)
3. **Never delete-and-re-log to fix something.** Editing is in place
   (`crm activities update`) and preserves the activity's identity, date, and
   links. Add a forgotten attendee, remove a no-show, fix a name, append a
   follow-up, or correct the date — all without losing the record.
4. **Write notes a colleague could act on cold.** Context → what was
   discussed → decisions → follow-ups. Use the customer's own language for
   pain points.

## Log a new activity

First positional argument is the **account id** (`acc_…`). Associate an
opportunity with `--opportunity-id` when the touchpoint belongs to a deal.

```bash
crm activities log <account_id> \
  --type meeting \
  --subject "KB Agent CLI walkthrough — MIMOVRSTE" \
  --notes "In-person session with Pavel's team. Walked through the KB Agent CLI (install, usage, advantages over MCP). Very enthusiastic — they'd just spent a month building worse tooling. Follow-up: revisit in ~2 weeks." \
  --opportunity-id <opp_id> \
  --at "2026-07-08 10:00" \
  --duration-minutes 90 \
  --met "Pavel Brecík <pavel@mimovrste.cz>:met" \
  --met "Jan Karaffa <jan.karaffa@mimovrste.cz>:met" \
  --met "Jiří Sobotík <jiri@mimovrste.cz>:met"
```

- `--type`: `call | meeting | email | note | demo | proposal_sent |
  contract_sent | check_in | qbr | onboarding | handover_call |
  support_review`.
- `--met` grammar: `"Name <email>:role"`, or `"Name"`, or `"email"`. The
  optional `:role` is `met` (default), `mentioned`, `introduced_by`, or `cc`.
  Repeat `--met` once per person — a 14-person meeting gets 14 `--met` flags.
- `--at` accepts `YYYY-MM-DD` or `YYYY-MM-DD HH:MM` (local wall-clock; it is
  stored and shown without timezone conversion, so it reads back as typed).
- Account-level touchpoints (CS check-ins, QBRs) can omit `--opportunity-id`.

## Edit / enrich an activity in place

Use `crm activities update <activity_id>` — identity, date, and links are
preserved unless you explicitly change them.

```bash
# Add someone you forgot (auto-links if they're already a contact)
crm activities update act_123 --met "Jiří Pahorecký <jiri.p@firma.cz>:met"

# Remove a no-show (by email or by name)
crm activities update act_123 --unmet "someone@firma.cz"

# Append follow-up info without overwriting the original summary
crm activities update act_123 --append-notes "Pavel confirmed budget sign-off path via CFO."

# Fix a backdated date entered wrong
crm activities update act_123 --at "2026-07-08 10:00"
```

- Re-adding the same email is **idempotent** — it upserts (and can correct a
  misspelled name), never duplicates.
- `--append-notes` grows the notes; `--notes` replaces them.

## Link attendees to contacts (after a transcript import)

When a meeting was logged with raw `--met` people (e.g. from a transcript)
and some are already CRM contacts, resolve them all at once:

```bash
crm activities promote-mentions --activity <activity_id>
```

This links every unresolved attendee on that one activity to a matching
same-account contact in a single call. (`crm activities promote-mentions
<contact_id>` still exists for the reverse case: link one newly-created
contact across all historic activities.) Note: people met via `--met` whose
email already matches a contact are auto-linked at write time, so you only
need this for people who became contacts *after* the activity was logged.

## Attach files (voice memos, photos, whiteboards)

```bash
crm activities attach <activity_id> <file_path> --kind photo
```

`--kind`: `voice_memo | photo | whiteboard | screenshot | business_card |
other`. Stored in the Google Shared Drive against the activity.

## Confirm what you logged

```bash
crm activities get <activity_id> --full
```

Check the subject, date, notes, and that every person you met shows under the
mentions — resolved ones display their contact name.

## Quality checklist (run before you consider it logged)

- [ ] **Type** matches the interaction (`meeting`, `call`, `qbr`, …).
- [ ] **People met** captured with `--met` (one per attendee), not just emails.
- [ ] **Date** is the real meeting time — `--at` set if logged late.
- [ ] **Notes** give context, discussion, decisions, and next steps.
- [ ] **Opportunity** linked (`--opportunity-id`) when it belongs to a deal.
- [ ] After a transcript import: ran `promote-mentions --activity` so
      attendees who are contacts are clickable.

## Anti-patterns

- Logging attendees as `--participants` emails (invisible as "people met").
- Leaving `performed_at` as the creation date for a backdated meeting.
- `delete` + re-`log` to fix a typo or add a person — edit in place instead.
- One-line notes with no follow-up. Future-you and your colleagues can't act
  on "had a call, went well".

## Example prompts

- "Log yesterday's onsite with Acme — I met Jan and Petr, ~90 minutes."
- "I forgot to log Tuesday's meeting; it was 10am, here's who was there…"
- "Add Jiří to activity act_123, I forgot him."
- "Fix the date on act_123 — the meeting was July 8, not today."
- "Link the attendees on this imported call to their contacts."
