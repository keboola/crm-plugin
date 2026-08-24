---
name: forecast-attainment
description: Track attainment against assigned revenue targets and check pipeline coverage — view personal and team forecast, assign targets, monitor progress, and assess coverage ratios. Activates when users want to check whether they are on track to hit their number, set targets, or assess pipeline health against a target.
allowed-tools: ['Bash']
---

# Forecast & Attainment Skill

You are a sales operations assistant helping track attainment against assigned
revenue targets in the Keboola CRM.

**Renamed from `quota-attainment` in #450 PR 6.** The `quotas` module it drove was
deleted: no surface could ever write a quota, so every command returned an empty
result. Targets are now assigned through `crm targets` and read through
`crm forecast show`.

## When to Activate

This skill activates when the user wants to:
- Check whether they are on track to hit their number
- View team attainment (managers)
- Assign or update revenue targets
- Assess pipeline coverage against a target
- Understand why a forecast number looks wrong

## The one rule that matters most

**A missing target is `—` / `null`, and it is NOT zero.** It means nobody assigned a
target for that period. Never report "0% attainment" or "missed the target" for such
a cell — the correct statement is *"no target is set for this period"*, and the
action is to set one. Reading null as zero turns "we have not decided the goal" into
"we failed the goal", which is a different and much worse conversation to have with
a rep.

Equally: **`unknown_baseline_count` above zero means the numbers UNDERSTATE
reality.** Those deals have no determinable net-new ARR and are excluded from every
total. Exactly two causes, both fixed on the opportunity: no `opportunity_type` is
set, or nothing prices the deal. Say so when you summarise such a cell.

**And a cell can OVERSTATE without this column moving at all.** Since 2026-08-22 a
renewal with no predecessor order linked is counted at its FULL contract value
rather than excluded — a flat 200k renewal of a 200k contract adds 200k of "new"
ARR. Those deals are *not* in `unknown_baseline_count` and carry
`baseline_unknown: false`, so `crm forecast show` cannot distinguish them from
healthy ones. A zero Unknown column is therefore **not** evidence that a cell is
correct. To check, run `crm forecast opportunities` and read the **Flag** column:
`over: no pred` marks exactly these rows, and the fix is to link the predecessor
order. The three `out:` markers are the understating cases above.

## Step-by-Step Workflow

### Step 1: Understand the request

- **Am I on track?** — personal forecast row
- **Team view** — every owner's row plus a company total
- **Set a target** — assign for a scope and period
- **Coverage** — pipeline against the remaining target
- **Company vs individual** — the over-assignment delta

### Step 2: Read the forecast

```bash
crm forecast show --year 2026 --format human
```

One table per owner plus a Company footer, one row per fiscal period, with:

- **Closed Won** — booked net-new ARR
- **Commit** — won, plus high-probability deals that are **not stalled**
- **Best Case** — probability-weighted; stalled deals contribute 0
- **Pipeline** — every non-lost deal, unweighted, stalled **included**
- **Target** — the assigned number, or `—` when none is set
- **Att %** — Closed Won ÷ Target
- **Gap** — Target − Commit (negative means committed over target)
- **Cov** — Pipeline ÷ remaining target; `—` once the target is met
- **Unknown** — deals whose net-new ARR could not be determined

A single owner:

```bash
crm forecast show --year 2026 --owner <user_id> --format human
```

Monthly instead of quarterly:

```bash
crm forecast show --year 2026 --period-type month --format human
```

Without the `forecast.read` permission you see only your own row, and
`scoped_to_self` is `true` in the JSON output. That is not an error — say so rather
than implying the org has one owner.

### Step 3: Assign a target

```bash
crm targets set --scope-id <user_id> --amount 500000 --year 2026 --period-index 1
```

Company-wide, for the same period:

```bash
crm targets set --scope-type company --amount 4000000 --year 2026 --period-index 1
```

`period-index` is 1-4 for quarters, 1-12 for months, and 0 for a full year.
Re-running for the same cell **updates** it — the write is an upsert, so a second
call is an edit, not a duplicate.

Requires `targets.manage` (admin / VP / RevOps). A manager can read targets but not
assign them.

### Step 4: Check company vs individual assignment

```bash
crm targets coverage --year 2026 --format human
```

`over_assignment` is how far the sum of individual targets exceeds the company
number. **Over-assigning is normal practice** — the company target is assigned
independently, not derived as the sum — so do not present the sum as the company
target, and do not call a positive delta an error.

`company_target` and `over_assignment` come back `—` when no company target exists.

### Step 5: Add historical context

```bash
crm analytics arr-history
crm pipeline forecast
```

Use these for trend and for the weighted close-date view; they are unchanged by the
quotas removal.

## Interpreting attainment

Judge pace, not just the raw percentage: attainment of 45% halfway through a quarter
is on track; the same number with two weeks left is not. When you have both, state
the comparison rather than the bare number.

A stalled deal sitting in negotiation is deliberately **not** in Commit — that is the
CRO's rule, not a data problem. If a rep asks why their Commit dropped, a newly
stalled deal is the first thing to check.

## Related skills

- `pipeline-review` — deal-by-deal inspection behind these totals
- `revenue-analytics` — ARR / NRR / bookings reporting
- `commission-management` — payouts (a separate money basis; see epic #1270)
