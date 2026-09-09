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
  or feature requests, it also opens one Linear issue per request in the
  "Product Feedback & Requests" team. Do NOT use it for scheduling a FUTURE
  call, logging a bare manual note with nothing recorded to write up, querying
  calls already logged,
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
Transcript confidence: <good | degraded | good, localized damage at [mm:ss]>

1. [REQUEST] <one sentence: the single thing they asked for>
   Why: <the business problem, in their words>
   Workaround: <what they do today, or "none mentioned">
   Criticality: <Deal Blocker | Renewal Risk | Important | Nice-to-Have>
   Timeline: <if stated, else "not stated on the call">
   Raised by: <name(s) + company>
   Evidence: "<short verbatim quote>" [mm:ss]
2. [BUG] <what is broken; goes to L1, never to Linear>
3. [SALES] <interest in something Keboola already has or is piloting>
4. [LOSS] <why they are leaving, downgrading, or rejected an option>
5. [DROP] <not product feedback: deal terms, close dates, champions, action
   points, or a question that was answered on the call>
(If none surfaced, write exactly: "None surfaced.")
```

Why English + fixed sections: the CRM is read by a mixed-language team and the
deal history must stay uniform. Be faithful to the transcript — don't invent
numbers; mark uncertain figures as approximate ("~$X, to confirm").

The last section feeds step 7. Run the confidence gate in **5a** before writing
it, and classify every item into exactly one of the five classes above.

**One numbered item = one distinct ask.** RLS *and* read-only user pricing in
one breath is two items. Bundled asks are the most common defect in this queue,
and a bundle cannot be prioritised, split or closed independently.

**Whose ask is it.** Only what the customer asked for or complained about.

- If a Keboola person raised the gap first, it is a `[REQUEST]` only if the
  customer then articulates the impact on their own operation. Agreement alone
  ("that'd be great", "sounds good") is `[SALES]` or `[DROP]`.
- Exception: a gap the customer names **in direct answer to a question Keboola
  asked** is a `[REQUEST]`. Asking "what are we missing?" and then discarding the
  answer is the inverse failure.
- A question answered on the call is `[DROP]`. If the answer was evasive or did
  not address the question, that is a gap. File it, and say so.

**The already-exists test runs first and wins.** If the thing exists, the next
step is to hand it over or sell it, so it is `[SALES]` even when the customer
goes on to describe their pain in detail. Only if it does not exist does the
attribution test above decide between `[REQUEST]` and `[DROP]`. Keboola offering
an existing MVP, app or partner tool is always `[SALES]`.

**Evidence or nothing.** Every `[REQUEST]` carries a verbatim quote in the
original language with its timestamp. If you cannot quote it, you cannot file it.

**Never attribute a specific the customer did not say.** A vendor, product,
number or arrangement a Keboola person offered as an example does not become the
customer's. PROF-146 attributed a "Gemini/GCP arrangement" to the customer that
the CSM had invented as an illustration.

**Pricing: deal versus product.** Terms for this deal, meaning discount, commit,
close date or early-renewal mechanics, are `[DROP]`. A missing pricing model,
user tier or SKU is a `[REQUEST]`: the customer wants something that does not
exist rather than a better price for what does.

**Loss signals.** On a churn, downgrade or "we evaluated X instead" call, capture
why as `[LOSS]` items with evidence: what they compared Keboola to, the cost or
capability delta they quote, and which Keboola option they rejected. A rejected
Keboola offer counts, so if the CSM proposes a cheaper model and the customer
says it still does not close the gap, that is a `[LOSS]` about packaging. These
are not requests; nobody asked for anything.

**Criticality** is a judgement from what was said on the call. If nothing
supports a level, use `Important` and say so.

### 5a. Transcript confidence gate

Run this before writing the CUSTOMER FEEDBACK section.

If step 2 left you without a transcript, an empty transcript is **not** a call
without feedback. Never write "None surfaced." for it: that is a silent false
negative, indistinguishable from a genuine zero. Say the transcript is missing
and stop.

Otherwise judge the damage. It is either pervasive, meaning the whole transcript
is untrustworthy, or localized, meaning one passage is. A single artifact must
not block a call whose other evidence is clean.

Mark `degraded` and file **nothing** if any of these hold:

- Looping or repeated filler: the same point restated three or more times in
  near-identical words, or sentences that clearly are not speech. Judge this by
  reading; slight rewording defeats exact-match detection.
- The header warns that speaker labels are not consistent, e.g. a multi-part
  recording diarized separately per part, so "Speaker 3" is not one person
  throughout. You cannot attribute an ask, so you cannot apply the attribution
  test.
- Two or more subtitle-credit hallucinations ("Titulky vytvořil …",
  "Subtitles by …", "Thanks for watching").
- Large stretches that do not parse as sentences.

Otherwise mark `good, localized damage at [mm:ss]` and continue. A single
subtitle-credit artifact truncates the speech around it, so treat that passage
as lost, but only that passage.

On `degraded`: still write the seven sections, marked low-confidence at the top.
File nothing to Linear and hand nothing to L1. List each candidate item with the
passage it rests on and ask the CSM to confirm or correct each one.

On `localized`: the gate is per item. An item is filable only if its own Evidence
quote is clean and parses. If it sits in damaged speech, tag it `[UNCLEAR]` and
hold that one item. Say which passage was lost.

Never smooth damaged ASR into confident prose. Quote the garbage verbatim and tag
it `[UNCLEAR]`: a quote the CSM can recognise as broken is useful, an invented
fluent sentence is not.

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

### 7. Route the feedback (one Linear issue per product request)

**This step is not optional.** If the CUSTOMER FEEDBACK section holds anything
other than "None surfaced.", step 7 must run, and your final report must name
the destination of every item in it. Writing the requests into the notes and then
not routing them is this skill's largest failure mode; it has happened, and those
requests were still missing from PROF a month later.

Stop if 5a marked the transcript `degraded`. Confirm the items with the CSM
first; nothing below runs on a degraded transcript.

**Resolve the customer once per call, before creating anything.**

1. `list_customers` with `query=<CRM account name>`
2. Confirm the returned customer's `externalIds` contains the activity's
   `account_id`. If it does not, or nothing comes back, stop and ask.

Take the account from the activity's `account_id`, never from the transcript:
some transcripts never name their account, and at least one recording is stored
against two different accounts.

**Dedup before creating anything.** For each `[REQUEST]`:

- Search PROF by account plus topic and match on the **ask**, not the product
  area. An existing issue in the same area that does not name this specific ask
  is `relatedTo`, not a duplicate.
- Search the eng teams too, not only PROF. PROF-153's Kai-scope ask duplicated
  AJDA-3052 in an eng backlog and was filed anyway.
- Check the account's previous call notes for the same ask. Weekly and biweekly
  accounts re-raise things constantly, and an ask that was answered verbally on
  the last call is not a gap. PROF-167 was filed four days after the answer was
  given on another call.
- If an open issue already names the same ask for the same account, `add_comment`
  on it with the call date, the activity id, and only what is new. Report the
  comment, not a new identifier.

**`[REQUEST]` items: one Linear issue each, never a bundle.** Via the Linear
MCP (`mcp__*__save_issue`, `mcp__*__save_customer_need`):

```
save_issue
  team:   "Product Feedback & Requests"
  title:  "<Account> - <the single ask>"
  state:  "Triage"
  labels: ["Customer Success", "Product Request",
           "<Product bucket/feature label>", "<Criticality label>"]
  priority: <derived from Criticality: Deal Blocker=1, Renewal Risk=2,
             Important=3, Nice-to-Have=4>
  description:
    **What does the customer want?**
    <the one ask, one sentence, then detail>

    **Why do they need it? / Impact**
    <the business problem, in their words>

    **Who raised it**
    <names, with company>

    **Current workaround**
    <what they do today, or "none mentioned">

    **Evidence**
    > "<verbatim quote>" - <speaker>, [mm:ss]

    **Timeline**
    <if stated, else omit this field entirely>

    **Source**
    <Account> · CRM account `<account_id>` · <opportunity `<id>` OR
    "account-level call"> · call <YYYY-MM-DD> · CRM activity `<activity_id>`
```

Pick the `Product bucket/feature` label from the existing label group. If no
label clearly fits, omit it and say so: a wrong label routes the triage agent to
the wrong owner.

Priority is derived from Criticality, not judged separately. One judgement fills
both fields.

Then attach the customer:

```
save_customer_need
  customer: <confirmed customer id>
  issue:    <new issue identifier>
  body:     <the one-sentence ask>
  source:   <CRM activity URL>
```

**Required fields.** Every field above except Timeline is required. If the
transcript does not support one, write the literal string "not stated on the
call". Never leave it blank and never invent content. Re-read the draft against
this list before calling `save_issue`.

**Source line.** Use only fields the activity actually carries. If
`opportunity_id` is null, write "account-level call"; never infer an opportunity
from the account or the topic. PROF-166 cites an opportunity its activity does
not have.

Do not copy ARR, customer tier or CS owner into the description. The Linear
customer record already carries them and a copy goes stale.

**`[BUG]` items: L1 support, no Linear issue.** Bugs do not belong in the
product-request flow. List them back under "Bugs - hand over to L1 support via
the usual route", one line each, and record them in the CRM activity notes so the
deal history shows the customer raised them. If L1 finds a product gap behind a
bug, the JSM sync creates the PROF issue itself, so there is nothing further to
do here.

**`[SALES]` items: CRM, not Linear.** Interest in something Keboola already has
or is piloting is an expansion signal. Record it in the CRM activity notes and
list it back under "Expansion signals - for the CRM, not Product". Filing these
into PROF inflates apparent demand for things that already exist.

**`[LOSS]` items: CRM notes, surfaced explicitly, no Linear issue.** Record them
under a "Why we lost / what they chose instead" heading in the CRM activity
notes, each with its evidence quote, and list them back under "Product
intelligence - needs a destination". Do not file them as requests: nobody asked
for anything, and filing them pollutes demand counts. There is no system of
record for this yet, so the job here is to make sure it is written down and
visibly handed to the CSM.

**`[DROP]` items** are recorded in the CRM activity notes for deal history.
Nothing is filed.

**Before writing anything**, show the user the full routed set as a numbered
list: per item its destination (new Linear issue, comment on an existing issue,
L1, expansion signal, loss intelligence, dropped), and for Linear issues the
title, criticality, priority and labels. Confirm before creating; these are
visible to the whole Product team.

After creating, report every identifier and URL, and add them to the CRM activity
notes.

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
| Empty transcript | Never "None surfaced." - say the transcript is missing and stop (5a) |
| Transcript looks damaged | Run the 5a gate; pervasive damage means file nothing |
| Re-processing a call with feedback | Dedup against PROF, the eng teams, and the account's previous call notes (step 7) |
| Feedback section written but nothing filed | Step 7 is mandatory whenever the section is non-empty |
| `crm` flag rejected | Run `<cmd> --help`, adapt (flags differ by version) |
