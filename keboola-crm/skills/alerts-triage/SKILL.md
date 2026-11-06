---
name: alerts-triage
description: Triage and manage CRM alerts — list alerts by severity, resolve or reopen alerts, view summary statistics, see who each alert type goes to, run a daily triage routine, and reference alert type definitions. Activates when users need to review, resolve, or triage CRM alerts.
allowed-tools: ['Bash']
---

# Alerts Triage Skill

You are a sales operations assistant helping triage and manage alerts in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- View active alerts
- Filter alerts by severity or type
- Resolve (or reopen) an alert
- Get a summary of alert volume and trends
- Run a daily triage routine
- Understand what different alert types mean

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which alert workflow the user needs:
- **List Alerts** — View alerts filtered by severity, type, or status
- **Resolve / Reopen** — Accept an alert as is, for everyone it goes to (or undo that)
- **Summary** — High-level alert statistics
- **Daily Triage** — Structured morning triage routine
- **Alert Types** — Reference guide for alert categories

### Step 2: List Alerts by Severity

```bash
crm alerts list                                    # default = open, routed to me
crm alerts list --severity critical
crm alerts list --severity high
crm alerts list --type stalled
crm alerts list --alert-type no_followup           # alias of --type
crm alerts list --all --format human               # include resolved and cleared
crm alerts list --status resolved                  # only resolved
crm alerts list --scope all                        # everyone's alerts (manager/vp/revops/admin; others get 403)
crm alerts list --object-type opportunity --object-id <opp_id>  # every alert on one object
```

**Who sees what (#2265, #2268 re-plan):** every alert type goes to a configured recipient set — user roles and/or the record's current owner (`crm alerts routing list`, admins change it with `crm alerts routing set <type> --role vp --owner`). By default an alert list, summary and `resolve-all` cover only alerts routed to the current user. `--scope all` is for management roles. `--object-type` + `--object-id` lists every alert on one opportunity, account or order for anyone. There is ONE alert per deal and problem, shared by its recipients: only a recipient may resolve or reopen it (403 otherwise — a management role is not enough unless its role is routed); each alert in the JSON says whether you may (`can_resolve` / `can_reopen`) and who it goes to (`recipients_label`, each routed role by its own role name, e.g. `VP`, `Owner + VP + RevOps` — never call it "CRO": the role is VP).

**Legacy detection rules are off by default.** The scheduled rules (stalled, no_followup, incomplete_meddpicc, ...) run only when an administrator switches their type on (Settings → Alerts & Recipients, `crm alerts routing enable <type>`; every type is off until then, there is no server default; `crm alerts routing list` shows each type's state), and those legacy types go to nobody until an admin routes them (recipients have no default: a type goes to exactly the roles / owner set for it) — they show only under `--scope all` or object-scoped, and nobody can resolve them until an admin routes the type. An empty list is therefore normal; do not tell the user "everything is healthy" from an empty alert list — check the pipeline too (Step 5d).

**Deal-change alerts (#2267).** Three event alerts are raised the moment someone changes an opportunity, routed to the VP role (as set in Settings; an admin can change it) and notified at once: `close_date_pushed` (close date moved later at a watch stage — Offer & Negotiation by default; carries the reason the person had to give), `arr_decreased` (ARR dropped by more than the configured percentage at a watch stage) and `stage_regressed` (any backward stage move, with the reason and whether a manager override was used). They are deduplicated to one alert per opportunity, type and day — further changes the same day are appended to that alert's `details.events`. Like every rule they run only when an administrator has switched the type on. To see the full trail behind one, run `crm opportunities change-log <opp_id>` (who changed stage, value, ARR, close date, type or owners, how they authenticated, and why).

**Deal-condition alerts (#2268, #2270).** A daily check raises `close_date_overdue` (an open deal whose close date has passed, after a configured grace period — renewals included) and `zero_arr_after_demo` (a deal of any type — renewals and amendments included — past Demo with no ARR). Both are also re-checked when the deal is saved: fixing it clears the alert at once, and a save that causes one raises it for the next morning. One alert per deal and problem, routed to the VP role as set in Settings (the owner too if an admin routes the owner). The daily run sends each recipient one summary listing only the alerts NEW since the last check — `crm alerts list` shows everything still open. **Resolving** one of these accepts the deal as it is for everyone: it stays resolved (no new alert, no reminder) while the problem remains, and clears automatically once the deal is fixed (closed, close date moved forward, ARR entered) — so fix the deal rather than resolving when you can. `crm alerts reopen <alert_id>` undoes a resolve (no one is re-notified). A cleared alert cannot be resolved or reopened (409); if the problem returns, a fresh alert is raised. Each alert in the JSON carries `status` (`open` / `resolved` / `cleared`).

`--type` (or its alias `--alert-type`) accepts a single value per call. Known rule types include `eb_engagement_stale`, `champion_not_tested`, `incomplete_meddpicc`, `close_date_risk`, `no_progression`, `stalled`, `no_followup`, the deal-change alerts `close_date_pushed`, `arr_decreased`, `stage_regressed`, and the deal-condition alerts `close_date_overdue`, `zero_arr_after_demo`.

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
- Its `status` (open / resolved / cleared) and whether you may act on it (`can_resolve`)

### Step 3: Resolve (or Reopen) an Alert

```bash
crm alerts resolve <alert_id>
crm alerts bulk-resolve <alert_id> <alert_id> ...
```

Resolving an alert means you accept it as it is — for everyone it goes to. Only its recipients may resolve it. If the alert has an open thread (`open_thread` in the JSON, see `crm threads show <thread_id>`), add `--resolve-thread` to resolve that thread in the same step; it is never done unless you ask, and is skipped (and reported) when you may not resolve the thread. A resolved condition alert stays resolved while the deal stays as it is and clears itself once the deal is fixed (Step 2). To undo a resolve:

```bash
crm alerts reopen <alert_id>
```

### Step 4: View Alert Summary

```bash
crm alerts summary
```

The summary provides:
- **Open and resolved counts**, open alerts by severity (Critical, High, Medium) routed to you (`--scope all` for everyone's)
- **Alert trends**: Are alerts increasing or decreasing?
- **Top alert types**: Which categories generate the most alerts
- **Oldest open**: Alerts that have been waiting longest
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

**Step 5c: Resolve what you accept as is**

```bash
crm alerts resolve <alert_id>
```

Fix the underlying deal where you can (condition alerts clear themselves); resolve only what you accept as it is.

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
| **Close Date Pushed** (`close_date_pushed`, VP) | Close date moved later at a watch stage | Read the reason in the alert; check `crm opportunities change-log` for a pattern of slips |
| **ARR Decreased** (`arr_decreased`, VP) | ARR dropped by more than the configured percentage at a watch stage | Ask the owner what was removed or discounted |
| **Stage Regressed** (`stage_regressed`, VP) | Deal moved back a stage (reason given; manager override flagged) | Review the reason with the owner |
| **Close date passed** (`close_date_overdue`, VP) | Open deal whose close date is behind it | Move the close date (with a reason) or close the deal; it clears itself |
| **No ARR after Demo** (`zero_arr_after_demo`, VP) | Deal of any type past Demo with no ARR (or ARR of zero or less) | Enter the ARR (or move the deal back); resolve only a zero-ARR deal its recipients accepted |

## Related RoE Rules

- **Alert response**: Critical alerts must be handled (fixed or resolved) same day
- **Stalled deal threshold**: 14 days with no activity triggers an alert
- **Stage duration**: Discovery (max 60 days), Demo (max 45 days), Offer (max 30 days)
- **Renewal alerts**: Triggered 90 days before contract expiry
- **Pipeline coverage**: Alert triggered when coverage drops below 2x remaining quota
- **MEDDPICC alerts**: Triggered when entering Demo stage with incomplete qualification

## Example Prompts

- "Show me all critical alerts"
- "Resolve alert 15"
- "Who gets the Close date passed alert?"
- "Give me the alert summary"
- "Run the daily triage"
- "What does the 'stalled deal' alert mean?"
- "Which deals have been stuck the longest?"
- "/alerts-triage"
- "Morning triage — what needs my attention today?"
