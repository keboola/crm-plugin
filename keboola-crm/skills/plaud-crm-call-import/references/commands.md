# Commands & parsing reference

Concrete, verified invocations for the Plaud→CRM pipeline. Flags can drift
between `crm` versions; if one is rejected, run `<command> --help` and adapt.

## Plaud MCP

| Tool | Purpose | Key args |
|---|---|---|
| `mcp__plaud__get_current_user` | confirm auth | — |
| `mcp__plaud__list_files` | find recordings | `query` (name substring), `date_from`/`date_to` (`YYYY-MM-DD`), `page`, `page_size` |
| `mcp__plaud__get_file` | recording detail / transcription status | `file_id` |
| `mcp__plaud__get_transcript` | full diarized transcript | `file_id` |

`get_transcript` returns a JSON array of up to 3 blocks keyed by `data_type`:
- `transaction` — verbatim diarized transcript (the one to parse). `data_content`
  is a JSON **string** → an array of `{start_time(ms), end_time, content, speaker}`.
- `transaction_polish` — cleaned version, often empty.
- `outline` — array of `{start_time, end_time, topic}`; use it to make the
  KEY FACTS section exhaustive.

Large transcripts may be saved to a file by the tool harness instead of returned
inline — read/parse that file.

### jq recipe: JSON transcript → readable `[mm:ss] Speaker: text`

```bash
F=<path to raw get_transcript json>
# Write outside the repo — transcripts carry customer PII and must never be committed.
OUT="$HOME/.crm-imports/<account>_<date>_transcript.txt"
mkdir -p "$HOME/.crm-imports"
jq -r '.[] | select(.data_type=="transaction") | .data_content' "$F" \
 | jq -r '.[] | "[\(.start_time/1000 | floor | (./60|floor|tostring) + ":" + (.%60|floor|if .<10 then "0"+tostring else tostring end))] \(.speaker): \(.content)"' \
 > "$OUT"
```

Once you know who Speaker 1 / Speaker 2 are (from the calendar), it's nice to
relabel them with real names in the saved transcript.

### Timezone gotcha

Recording `name` uses local time; `start_at` is UTC. In summer (CEST = UTC+2) a
recording named `11:01` has `start_at` ≈ `09:01`. Convert when matching against
the calendar.

## crm CLI

```bash
crm auth me                                        # identity
crm accounts search <name>                         # → account_id
crm opportunities list --account-id <account_id>   # → opportunity_id
crm contacts search --email <email>                # existing contact?
crm contacts list --account-id <account_id>        # all account contacts

# create a missing attendee as a contact
crm accounts add-contact <account_id> \
  --first-name <F> --last-name <L> --email <email> [--title <t>] [--role champion|economic_buyer|technical|...]

# log the call (first positional arg is ACCOUNT id; opportunity is a flag)
crm activities log <account_id> \
  --type call --opportunity-id <opp_id> \
  --subject "<title> - <ACCOUNT>" \
  --notes "$(cat ~/.crm-imports/<summary_en>.txt)" \
  --at "YYYY-MM-DD HH:MM" \
  --duration-minutes <n> \
  --transcript-file ~/.crm-imports/<transcript>.txt \
  --source plaud --external-id <plaud_file_id> \
  --language <cs|en> \
  --met "Name <email>:met"

# edit in place (replaces notes; preserves id/performed_at)
crm activities update <activity_id> --notes "$(cat ~/.crm-imports/<summary_en>.txt)"
crm activities update <activity_id> --met "Name <email>:met"
crm activities update <activity_id> --append-notes "Follow-up: ..."

# link raw mentions to existing contacts (e.g. after creating a contact)
crm activities promote-mentions --activity <activity_id>
crm activities get <activity_id> --full              # verify
```

### `--met` role suffixes

`met` (in the room) · `mentioned` (referenced, not present) · `introduced_by` · `cc`.
Format: `"Name <email>:role"`, `"Name:role"`, or `"email:role"`. Idempotent on
email; an email matching an existing same-account contact auto-links.

## Linear MCP (product feedback)

Create a product-feedback issue with `mcp__*__save_issue`:

```
save_issue(
  team: "Product Feedback & Requests",   # the target team for customer feedback
  title: "<Account> — <short request summary>",
  description: "<markdown: bulleted requests + a Source line back to the CRM account/opp/activity>",
  labels: [...optional]
)
```

- `team` is required on create; the team name `"Product Feedback & Requests"`
  resolves without needing the ID (id `c83c0fbb-be46-4fc1-8a46-69ee86720b8a` —
  verify via `mcp__*__list_teams` if the name ever stops resolving, e.g. the
  team is renamed or the workspace is migrated).
- Use `mcp__*__list_issues` / search first to avoid filing a duplicate for a
  request that's already tracked.

## Google Calendar MCP

`mcp__*__list_events` with `startTime`/`endTime` (ISO 8601), `timeZone`
(e.g. `Europe/Prague`). Match the event by time window around the recording and
by a title/account that fits. External attendees (non-`@keboola.com`) are the
"people met"; the `@keboola.com` attendee is the rep/owner.
