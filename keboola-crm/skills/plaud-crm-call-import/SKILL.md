---
name: plaud-crm-call-import
description: >-
  The canonical way to get a past meeting or call into the Keboola CRM: import
  its recording (from Plaud), download the transcript, and log it as a `call`
  activity with a structured English write-up. Use this WHENEVER the user wants
  to log/record a call or meeting to the CRM or write one up — e.g. "log
  activities from meetings to CRM", "log my call with <customer>", "write up the
  Bonami renewal call and put it in the CRM", "import today's Plaud recording",
  "pull my latest recording into the CRM as a call activity", "turn my recorded
  meeting into a CRM activity", or naming a recording by date. It finds the
  recording in Plaud, downloads the transcript, resolves the CRM account +
  opportunity, pulls the real attendees from Google Calendar, creates any
  missing contacts, and writes notes in a fixed 7-section English template (When
  / Participants / Key facts / Action points / Next steps / Customer sentiment /
  Customer feedback & product requests). When the call surfaces product feedback
  or feature requests, it also opens a Linear issue in the "Product Feedback &
  Requests" team. Do NOT use it for scheduling a FUTURE call, logging a bare
  manual note with nothing recorded to write up, querying calls already logged,
  prepping for an upcoming meeting, setting up/connecting the Plaud integration,
  or just downloading a recording's audio — those belong to other tools.
allowed-tools: ['Bash']
---

# Plaud → CRM Call Import

Turn a Plaud voice recording into a clean CRM `call` activity: download the
transcript, attach it to the right account and opportunity, resolve who was
actually in the room from the calendar, and write a standardized English
summary that a manager or CS colleague can read in 30 seconds.

The point of the fixed write-up template is consistency: every imported call
reads the same way, so the renewal/deal history stays scannable no matter who
ran the call or what language it was recorded in.

## Prerequisites (check once, fail early)

This pipeline depends on three integrations. If one is missing, say so plainly
rather than guessing:

- **Plaud MCP** — tools `mcp__plaud__list_files`, `get_file`, `get_transcript`,
  `get_current_user`. If these aren't loaded, the Plaud connector isn't
  connected; point the user at it before continuing.
- **`crm` CLI** — run `crm auth me` to confirm identity. All CRM reads/writes
  go through this.
- **Google Calendar MCP** — `mcp__*__list_events` (used to resolve attendees).
  Optional but strongly preferred; without it you must ask the user who
  attended.

## The pipeline

Work through these in order. Confirm with the user at the two decision points
(which recording, who attended) — everything else can run straight through.

### Preview / dry-run mode (read this before running any step)

If the user asks to "preview", "dry-run", "show me what you'd do", or "don't
write to CRM yet", run every **read-only** step (find recording, download
transcript, resolve account/opportunity, resolve attendees from the calendar,
draft the English write-up) but do **not** execute any state-changing action —
no `crm` writes (`activities log`, `accounts add-contact`, `activities update`,
`promote-mentions`) and no Linear issue creation. Instead, output the finished
7-section write-up, the exact `crm` commands you would run, and the draft of any
Linear issue you'd open, so the user can review and approve first. Only proceed
to the write steps once they confirm.

### 1. Find the recording

Translate what the user said into a Plaud lookup:

- **By customer/topic** → `mcp__plaud__list_files` with `query` (case-insensitive
  substring on the recording name, e.g. `"bonami"`). Plaud auto-names recordings
  from the transcript once processing finishes, so a name search is usually the
  most reliable.
- **By date** → `list_files` with `date_from`/`date_to` (`YYYY-MM-DD`).
- Recording names embed local time; `start_at` is UTC. A "today 11:00" recording
  will show `start_at` around `09:01` UTC in summer (CEST = UTC+2). Don't let the
  offset fool you into picking the wrong file.

If more than one plausibly matches, list the candidates (name, start time,
duration) and let the user pick. Never silently guess between two meetings.

### 2. Download the transcript

`mcp__plaud__get_transcript` with the `file_id`. Notes:

- A freshly-stopped recording may return `[]` — Plaud hasn't transcribed it yet.
  Confirm with `get_file`: if the name is still a bare timestamp and
  `transcript`/`note_list` are empty, transcription is pending. Tell the user to
  generate the transcript in Plaud, then retry. Don't fabricate content from the
  audio.
- The transcript is a JSON array of three blocks: `transaction` (the verbatim
  diarized transcript — this is the one you want), `transaction_polish` (often
  empty), and `outline` (a timestamped topic list — handy for the Key facts
  section). Parse the `transaction` block; each segment has `start_time` (ms),
  `speaker` (e.g. "Speaker 1"), and `content`.
- Plaud diarizes but does **not** name speakers. You'll map speakers to people
  in step 4.

Save a clean, readable transcript **outside the repo** — to `~/.crm-imports/`
— so there's a durable artifact `crm` can attach without risking a customer
transcript (PII) being committed to git. Never write these files inside the
CRM working tree. See `references/commands.md` for the exact jq recipe that
turns the JSON into `[mm:ss] Speaker N: text` lines.

### 3. Resolve account + opportunity

```bash
crm accounts search <name>                       # → account_id (acc/0011…)
crm opportunities list --account-id <account_id> # → opportunity_id (006…)
```

- One open opportunity → use it. Multiple → pick by date overlap with the call,
  or ask. None → ask the user, or log it as an account-level activity (the
  `--opportunity-id` flag is optional).

### 4. Resolve attendees from the calendar

This is what makes the write-up trustworthy — Plaud only gives you "Speaker 1 /
Speaker 2", but the calendar knows the real names and emails.

- Look up the event with `list_events` over a window around the recording's
  start time (e.g. ±90 min), in the user's timezone. Match by time and by a
  title/account that fits ("Renewal", the customer name, etc.).
- The external attendees (non-`@keboola.com` emails) are your "people met". The
  internal Keboola attendee is almost always the rep/owner = Speaker who is
  presenting.
- For each external attendee, check whether they're already a CRM contact:
  `crm contacts search --email <email>`.
  - **Exists** → it'll auto-link when you log with `--met`.
  - **Missing** → create it: `crm accounts add-contact <account_id>
    --first-name … --last-name … --email …`, then it links.
- People merely *mentioned* in the call (not on the invite) are not attendees.
  Capture them as `--met "Name <email>:mentioned"` only if the user wants them.

If the calendar lookup is ambiguous or empty, ask the user who was on the call
rather than guessing — a wrong contact link is worse than none.

### 5. Write the summary (FIXED 7-section English template)

Always produce the notes in **English**, regardless of the call language, using
exactly these seven sections and headers, in this order:

```
WHEN
<date, local start–end time + tz, duration; in person / video; recorded via Plaud>

PARTICIPANTS
- <Name (Company) — role on the call>
- ...
- Referenced, not present: <names + why they matter>

KEY FACTS DISCUSSED
- <the substance: commercials, usage, options, pricing, technical context — the
  things a colleague needs to understand the deal. Use the Plaud `outline` block
  to make sure you cover every topic.>

ACTION POINTS
- <Owner>: <what they committed to do>

NEXT STEPS
- <sequencing, dependencies, and the target date / decision>

CUSTOMER SENTIMENT
<2–4 sentences: how positive, what they value, competitive signals, risks>

CUSTOMER FEEDBACK & PRODUCT REQUESTS
- <Each distinct piece of product feedback, feature request, gap, complaint,
  or wish the customer voiced — with a short paraphrase of the context/why.>
- <If none surfaced, write exactly: "None surfaced.">
```

Why English + fixed sections: the CRM is read by a mixed-language team and the
deal history must stay uniform. Be faithful to the transcript — don't invent
numbers; mark uncertain figures as approximate ("~$X, to confirm").

The final section is special: it feeds the Linear step below. Only list things
the customer actually asked for or complained about — not general deal facts.
Examples that qualify: "wants Kai usage visible as a line item in telemetry",
"asked for a native Excel export", "frustrated that Snowflake consumption is
hard to understand". Examples that do NOT: pricing terms, close dates,
who the champion is.

### 6. Log the activity

Write the verbatim transcript file as the attachment and the English write-up as
the notes:

```bash
crm activities log <account_id> \
  --type call \
  --opportunity-id <opportunity_id> \
  --subject "<short title> - <ACCOUNT>" \
  --notes "$(cat ~/.crm-imports/<summary_en>.txt)" \
  --at "<YYYY-MM-DD HH:MM local>" \
  --duration-minutes <round(duration_ms/60000)> \
  --transcript-file ~/.crm-imports/<transcript>.txt \
  --source plaud \
  --external-id <plaud file_id> \
  --language <cs|en> \
  --met "<Name <email>:met>"   # one per external attendee
```

- `--external-id <plaud file_id>` is the dedup key: re-running the same recording
  returns 409 and is safely skipped.
- `--language` describes the **attached transcript** (cs if you kept the original
  Czech verbatim). The notes are always English regardless.
- If an attendee was created as a contact *after* logging, link them with
  `crm activities promote-mentions --activity <activity_id>`.

### 7. Create a Linear product-feedback issue (only if there's feedback)

If — and only if — the CUSTOMER FEEDBACK & PRODUCT REQUESTS section has real
items (not "None surfaced."), route them to Product so they don't get lost in
the CRM. Create one Linear issue in the **"Product Feedback & Requests"** team
(`mcp__*__save_issue` with `team: "Product Feedback & Requests"`):

- **Title**: `<Account> — <short summary of the request(s)>`, e.g.
  `Bonami — Kai usage visibility in telemetry`.
- **Description** (Markdown): the bulleted feedback/requests, plus a
  **Source** line linking back — account name + CRM opportunity + the call
  date and CRM activity id — so Product can trace it to the conversation.
- If there are several unrelated requests, prefer one issue per distinct theme
  over a single grab-bag issue; use judgement.
- Before creating, show the user the draft (title + description) and confirm —
  a Linear issue is visible to the whole Product team. After creating, report
  the issue identifier/URL and consider adding it to the CRM activity notes.

Don't create duplicates: if you're re-processing a call, check whether an issue
for the same request already exists before opening a new one.

### 8. Confirm

Report the activity id, account, opportunity, attendees linked (and any contact
created), and show the user the rendered write-up. Offer the optional follow-ons:
populate MEDDPICC from the transcript, or translate the verbatim transcript to
English and re-attach (`--transcript-file … --language en`).

## Command & parsing reference

Exact CLI flags drift between `crm` versions — when a flag is rejected, run the
relevant `--help` and adapt. The discovered commands, the jq transcript recipe,
and the speaker/timezone gotchas are collected in
[references/commands.md](references/commands.md). Read it when you need the
precise invocation.

## Common failure modes

| Situation | What to do |
|---|---|
| Transcript returns `[]` | Not transcribed yet — ask user to generate it in Plaud, retry |
| Two recordings match | List candidates, let user choose |
| No calendar event found | Ask the user who attended |
| Attendee not a CRM contact | `crm accounts add-contact`, then `--met` links it |
| Re-importing same recording | 409 on `--external-id` = already imported, skip |
| No product feedback in call | Write "None surfaced." and skip the Linear step |
| Re-processing a call with feedback | Check Linear for an existing issue before filing a new one |
| `crm` flag rejected | Run `<cmd> --help`, adapt (flags differ by version) |
