---
name: gong-transcript-import
description: Import call transcripts from Gong, Plaud, or manual entry into CRM. Activates when users want to import call recordings, match call participants to accounts, bulk import Gong data, process transcripts for MEDDPICC insights, or log a call with a transcript.
allowed-tools: ['Bash']
---

# Call Transcript Import & Processing

You are a sales operations assistant helping import and process call transcripts in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Import Gong or Plaud call transcripts into CRM
- Bulk import calls from a Gong JSON export
- Log a call with a full transcript
- Match call participants to CRM contacts/accounts
- Process call transcripts for MEDDPICC insights
- Review imported call data

## Data Source: Gong Export

Gong export lives in `data/gong_export/`:
- `calls_combined.json` — metadata + transcripts per call (~2,626 calls)
- `calls_summary.csv` — flat summary for quick analysis

Each call object structure:
```json
{
  "metaData": {
    "id": "gong-call-id",
    "url": "https://app.gong.io/call?id=...",
    "title": "Discovery Call",
    "started": "2025-09-15T14:00:00Z",
    "duration": 1260,
    "language": "en",
    "parties": [
      {"emailAddress": "rep@keboola.com", "name": "Sales Rep"},
      {"emailAddress": "jan@customer.cz", "name": "Jan Novak"}
    ]
  },
  "transcript": [
    {"speaker": "Sales Rep", "text": "..."},
    {"speaker": "Jan Novak", "text": "..."}
  ]
}
```

## Step-by-Step Workflow: Single Call Import

### Step 1: Identify the Call

Ask the user (if not already provided):
- Which call to import? (Gong URL, call ID, or file path)
- Which opportunity does it belong to?

### Step 2: Match Participants

This step resolves the **account** (and, later, links attendees to contacts).
You also carry each external party (`name` + `emailAddress` from
`metaData.parties`) forward to Step 4, where they become `--met` "people met"
that auto-link to contacts.

Extract external participant emails (skip @keboola.com addresses):

```bash
crm contacts search --email <participant_email>
```

If no exact match, fall back to domain:
```bash
crm contacts search --domain <email_domain>
```

If still no match, check Account.domain:
```bash
crm accounts search <domain>
```

### Step 3: Find the Opportunity

```bash
crm opportunities list --account <account_id>
```

- One open opportunity: use it
- Multiple open: pick by date overlap (call date within opportunity timeline)
- No open opportunity: ask user which opportunity, or create one

### Step 4: Import the Call

The first positional argument is the **account id** (`acc_…`, resolved in
Step 2); attach the opportunity with `--opportunity-id`. Capture each
external attendee with `--met` (one flag per person, `"Name <email>:met"`)
— this records first-class "people met" that **auto-link** to any existing
CRM contact on the account, so you don't run a separate promote step for
known contacts. Set `--at` from the call's `started` timestamp so the
activity is dated when the call happened, not when you imported it.

```bash
crm activities log <account_id> \
  --type call \
  --opportunity-id <opportunity_id> \
  --subject "<call_title> - <account_name>" \
  --notes "<brief summary of transcript>" \
  --at "<started_iso, e.g. 2025-09-15 14:00>" \
  --transcript-file <path_to_transcript_file> \
  --duration <duration_seconds> \
  --source gong \
  --external-id <gong_call_id> \
  --external-url <gong_url> \
  --language <language> \
  --met "Jan Novák <jan@customer.cz>:met" \
  --met "<each external party from metaData.parties — skip @keboola.com>"
```

If 409 Conflict: call already imported (dedup via external_id), skip.

> Use `--met` (not `--participants`) for attendees — `--met` people show
> under "People met", are clickable, and auto-link to contacts;
> `--participants` only stores bare emails on the call record. See the
> [`log-activity`](../log-activity/SKILL.md) skill for the full activity-
> capture playbook.

### Step 5: Link any remaining attendees, then confirm

Attendees whose email already matched a contact were auto-linked at import.
For anyone who became a contact *after* the call (or to resolve them all in
one shot), run the bulk promote:

```bash
crm activities promote-mentions --activity <activity_id>
crm activities get <activity_id> --full
```

Verify the call record was created with transcript, duration, attendees, and
the correct date.

## Step-by-Step Workflow: Bulk Import

### Step 1: Load Gong Export

Read `data/gong_export/calls_combined.json`. For each call:

### Step 2: Match and Import Loop

```
For each call:
1. Extract external participant emails
2. crm contacts search --email <email>
   ├── Found → have account_id
   └── Not found → crm contacts search --domain <domain>
       ├── Found → have account_id
       └── Not found → LOG AS UNMATCHED, skip
3. crm opportunities list --account <account_id>
   ├── Found open opp → use it
   └── No open opp → LOG AS NO-OPP, skip
4. crm activities log <account_id> --type call --opportunity-id <opp_id> \
       --at "<call started>" --met "<each external party>" ...
   ├── 201 Created → imported (known contacts auto-linked via --met email)
   └── 409 Conflict → already imported, skip
5. Track: imported / skipped / unmatched / no-opp
```

### Step 3: Report Results

After completion, report:
```
Import complete:
  Total calls: X
  Imported: Y (Z%)
  Skipped (dedup): N
  Unmatched (no contact): N
  No opportunity: N

Top unmatched domains:
  gmail.com: N calls
  outlook.com: N calls
```

## Step-by-Step Workflow: MEDDPICC Extraction from Transcripts

### Step 1: Read the Transcript

```bash
crm activities get <activity_id> --full
```

### Step 2: Analyze for MEDDPICC Signals

Read the transcript and identify evidence of:

| Signal | What to look for | MEDDPICC field |
|--------|-----------------|----------------|
| Pain | "frustrated with", "costs us", "losing", "wasting time" | pain_statement |
| Metrics | Dollar amounts, percentages, KPIs, "currently we" | metrics_current_state |
| EB | "CFO", "VP", "budget", "approval", "sign off" | economic_buyer notes |
| Champion | Enthusiasm, "I'll push for this", "let me talk to" | champion_advocacy |
| Competition | Competitor names, "also looking at", "compared to" | competition notes |
| Timeline | "need by Q3", "deadline", "before end of year" | compelling_event |
| Risk | "budget cut", "freeze", "postpone", "re-evaluate" | deal risk signal |
| Objection | "concern is", "not sure about", "what about" | track for follow-up |
| Next steps | "let's schedule", "I'll send", "follow up on" | activity notes |

### Step 3: Update MEDDPICC Fields

```bash
crm opportunities update <opp_id> \
  --pain-statement "Customer described X problem costing $Y/month" \
  --metrics-current-state "Currently processing Z per day"
```

### Step 4: Update Activity Summary

```bash
crm activities update <activity_id> \
  --notes "Summary: Discussed X, Y, Z. Key insight: ..."
```

### Step 5: Review Updated Qualification

```bash
crm opportunities qualify <opp_id>
```

## Batch MEDDPICC Processing for an Account

```bash
# 1. List all call activities for account
crm activities list --account <account_id> --type call

# 2. For each call with source=gong that hasn't been processed:
crm activities get <activity_id> --full
# Read transcript, extract insights, update MEDDPICC

# 3. After all calls processed, review status:
crm meetings prep --account <account_id>
```

## Matching Priority

1. **Exact email** — contact + account (strongest signal)
2. **Domain match** — all contacts with `*@domain` (fallback)
3. **Account.domain** — account without specific contact
4. **Skip** — log as unmatched for manual review

## Error Handling

| Situation | Action |
|-----------|--------|
| Email not in CRM | Try domain fallback, then skip |
| No open opportunity | Skip, log for manual assignment |
| 409 Conflict (dedup) | Skip, already imported |
| Transcript empty | Import metadata only |
| Multiple accounts match domain | Pick most likely (most contacts, most recent activity) |
| Only internal participants | Skip (internal meeting) |

## Example Prompts

- "Import all Gong calls for Q1 2026"
- "Import this Gong call: https://app.gong.io/call?id=123456"
- "Log a call transcript for opportunity 42"
- "Process the transcript on activity act_123 for MEDDPICC insights"
- "What MEDDPICC signals are in the last 5 calls with Acme Corp?"
- "Bulk import Gong data from data/gong_export/"
- "/gong-transcript-import"
- "Match Gong call participants to CRM contacts"
