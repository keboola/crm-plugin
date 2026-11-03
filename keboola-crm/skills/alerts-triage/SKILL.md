---
name: alerts-triage
description: Triage and manage CRM alerts — list alerts by severity, acknowledge alerts, view summary statistics, run a daily triage routine, and reference alert type definitions. Activates when users need to review, acknowledge, or triage CRM alerts.
allowed-tools: ['Bash']
---

# Alerts Triage Skill

You are a sales operations assistant helping triage and manage alerts in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- View active alerts
- Filter alerts by severity or type
- Acknowledge an alert
- Get a summary of alert volume and trends
- Run a daily triage routine
- Understand what different alert types mean

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which alert workflow the user needs:
- **List Alerts** — View alerts filtered by severity, type, or status
- **Acknowledge** — Mark an alert as reviewed/handled
- **Summary** — High-level alert statistics
- **Daily Triage** — Structured morning triage routine
- **Alert Types** — Reference guide for alert categories

### Step 2: List Alerts by Severity

```bash
crm alerts list                                    # default = unacknowledged, addressed to me
crm alerts list --severity critical
crm alerts list --severity high
crm alerts list --type stalled
crm alerts list --alert-type no_followup           # alias of --type
crm alerts list --all --format human               # include already acknowledged
crm alerts list --scope all                        # everyone's alerts (manager/vp/revops/admin; others get 403)
crm alerts list --object-type opportunity --object-id <opp_id>  # every alert on one object
```

**Who sees what (#2265):** by default an alert list, summary and `ack-all` cover only alerts addressed to the current user — directly, or through their role (e.g. a `cro` audience alert reaches the role mapped to it in server config). `--scope all` is for management roles. `--object-type` + `--object-id` lists every alert on one opportunity, account or order for anyone. Acknowledging an alert not addressed to you returns 404 unless you hold a management role.

**Legacy detection rules are off by default.** The scheduled rules (stalled, no_followup, incomplete_meddpicc, ...) run only when an administrator lists their type in the server's `ALERT_ENABLED_TYPES`, and those legacy alerts are addressed to nobody — they show only under `--scope all` or object-scoped. An empty list is therefore normal; do not tell the user "everything is healthy" from an empty alert list — check the pipeline too (Step 5d).

**CRO change alerts (#2267).** Three event alerts are raised the moment someone changes an opportunity, addressed to the `cro` audience: `close_date_pushed` (close date moved later at a CRO-watch stage — Offer & Negotiation by default; carries the reason the person had to give), `arr_decreased` (ARR dropped by more than the configured percentage at a watch stage) and `stage_regressed` (any backward stage move, with the reason and whether a manager override was used). They are deduplicated to one alert per opportunity, type and day — further changes the same day are appended to that alert's `details.events`. Like every rule they run only when listed in `ALERT_ENABLED_TYPES`. To see the full trail behind one, run `crm opportunities change-log <opp_id>` (who changed stage / close date / ARR, how they authenticated, and why).

`--type` (or its alias `--alert-type`) accepts a single value per call. Known rule types include `eb_engagement_stale`, `champion_not_tested`, `incomplete_meddpicc`, `close_date_risk`, `no_progression`, `stalled`, `no_followup`, and the CRO change alerts `close_date_pushed`, `arr_decreased`, `stage_regressed`.

A bundled `--for-csm` preset (one flag, multiple types) is requested by the CSM audit but not exposed by `GET /api/alerts` — see the NOTE block in `crm alerts list --help`. Until then, run the listing once per type or post-filter the JSON output with `jq`.

Alerts are categorized by severity:

| Severity | Meaning | Response Time |
|----------|---------|---------------|
| **Critical** | Immediate action required — deal at risk, major issue | Same day |
| **High** | Important — needs attention within 24-48 hours | 1-2 days |
| **Medium** | Notable — should be addressed this week | This week |

For each alert, note:
- Alert ID, type, severity
- Related account/opportunity
- When it was triggered
- Whether it has been acknowledged

### Step 3: Acknowledge an Alert

```bash
crm alerts ack <alert_id>
```

Acknowledging an alert means:
- You have reviewed it
- You understand what action is needed
- You are taking responsibility for resolution

After acknowledging, take the appropriate action based on alert type (see Step 6 for alert type reference).

### Step 4: View Alert Summary

```bash
crm alerts summary
```

The summary provides:
- **Total active alerts** by severity (Critical, High, Medium) addressed to you (`--scope all` for everyone's)
- **Alert trends**: Are alerts increasing or decreasing?
- **Top alert types**: Which categories generate the most alerts
- **Oldest unacknowledged**: Alerts that have been waiting longest
- **By account**: Which accounts have the most alerts

### Step 5: Daily Triage Routine

A structured morning triage routine:

**Step 5a: Check critical and high alerts**

```bash
crm alerts list
```

Review all Critical and High severity alerts first. For each:
- What is the alert about?
- Which deal or account is affected?
- What action is needed immediately?

**Step 5b: Review alert summary**

```bash
crm alerts summary
```

Check overall alert health:
- Any spike in alert volume?
- New alert types appearing?
- Accounts with multiple alerts (may indicate systemic issues)

**Step 5c: Acknowledge handled alerts**

```bash
crm alerts ack <alert_id>
```

Go through alerts you have already addressed and acknowledge them to keep the list clean.

**Step 5d: Check pipeline for new risks**

```bash
crm pipeline show
crm pipeline forecast
```

Cross-reference alerts with pipeline status:
- Do stalled deal alerts match pipeline expectations?
- Are there deals with no recent activity that have not triggered alerts yet?

**Step 5e: Plan the day**

Based on triage results, prioritize:
1. Critical alerts — handle first
2. High alerts — handle before lunch
3. Deal follow-ups from medium alerts
4. Proactive outreach to at-risk accounts

### Step 6: Alert Types Reference

Common CRM alert types and recommended actions:

| Alert Type | Description | Recommended Action |
|-----------|-------------|-------------------|
| **Stalled Deal** | Opportunity with no activity for 14+ days | Contact the customer, update CRM |
| **Close Date Passed** | Close date is in the past, deal still open | Update close date or close the deal |
| **Stage Too Long** | Deal stuck in a stage beyond expected duration | Review blockers, consider downstage |
| **MEDDPICC Gap** | Critical MEDDPICC component missing or incomplete | Schedule qualification meeting |
| **No Champion** | Opportunity lacks an identified champion | Prioritize champion development |
| **No EB Engagement** | Economic Buyer not yet engaged in Demo+ stage | Request EB meeting |
| **Contract Expiring** | Contract expires within 90 days, no renewal started | Initiate renewal conversation |
| **Low Pipeline Coverage** | Pipeline coverage below 2x quota gap | Increase prospecting activity |
| **Large Deal Risk** | High-ACV deal showing risk indicators | Conduct deal review |
| **Activity Drop** | Significant decrease in account activity | Re-engage customer contacts |
| **Close Date Pushed** (`close_date_pushed`, CRO) | Close date moved later at a watch stage | Read the reason in the alert; check `crm opportunities change-log` for a pattern of slips |
| **ARR Decreased** (`arr_decreased`, CRO) | ARR dropped by more than the configured percentage at a watch stage | Ask the owner what was removed or discounted |
| **Stage Regressed** (`stage_regressed`, CRO) | Deal moved back a stage (reason given; manager override flagged) | Review the reason with the owner |

## Related RoE Rules

- **Alert response**: Critical alerts must be acknowledged same day
- **Stalled deal threshold**: 14 days with no activity triggers an alert
- **Stage duration**: Discovery (max 60 days), Demo (max 45 days), Offer (max 30 days)
- **Renewal alerts**: Triggered 90 days before contract expiry
- **Pipeline coverage**: Alert triggered when coverage drops below 2x remaining quota
- **MEDDPICC alerts**: Triggered when entering Demo stage with incomplete qualification

## Example Prompts

- "Show me all critical alerts"
- "Acknowledge alert 15"
- "Give me the alert summary"
- "Run the daily triage"
- "What does the 'stalled deal' alert mean?"
- "Which deals have been stuck the longest?"
- "/alerts-triage"
- "Morning triage — what needs my attention today?"
