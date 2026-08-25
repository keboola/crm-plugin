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
- **Team view** — a row for every owner **holding a target**, a `Total` row over
  those rows, and a separate org-wide `company` figure
- **Set a target** — assign for a scope and period
- **Coverage** — pipeline against the remaining target
- **Company vs individual** — the over-assignment delta

### Step 2: Read the forecast

```bash
crm forecast show --year 2026 --format human
```

One table per owner plus a **`Total`** footer table, one row per fiscal period,
with:

- **Closed Won** — booked net-new ARR
- **Commit** — won, plus high-probability deals that are **not stalled**
- **Best Case** — probability-weighted; stalled deals contribute 0
- **Pipeline** — every non-lost deal, unweighted, stalled **included**
- **Target** — the assigned number, or `—` when none is set
- **Att %** — Closed Won ÷ Target
- **Gap** — Target − Commit (negative means committed over target)
- **Cov** — Pipeline ÷ remaining target; `—` once the target is met
- **Unknown** — deals whose net-new ARR could not be determined

### Three things about this grid that will mislead you if you skip them

Changed 2026-08-24 (design spec §4.1/§4.2/§4.6). All three are load-bearing when you
summarise a forecast for a human.

**1. A row exists because someone assigned that person a TARGET — not because they
own a deal.** The grid lists exactly the owners holding a target for the selected year
and quarters. So:

- **An owner missing from the grid may still own plenty of pipeline.** Never say "X
  has no deals" or "X is not forecasting" from an absent row. It means nobody assigned
  them a number.
- `--owner <id>` is **intersected** with that set, so naming an untargeted owner
  returns *no row*, not an empty one. `available_owners` in the JSON lists the owners
  that can be rows; read it before asserting who exists.
- Your own row always appears, target or not.

**2. `rows: []` is an ordinary answer, and `empty_reason` says which one.**
Production held **zero** targets when this rule shipped, so an empty grid is the
day-one state, not a broken endpoint. Read `empty_reason` from `--format json` and
report the one you were given — they have three different fixes:

| `empty_reason` | What it means | What to tell the user |
|---|---|---|
| `no_targets_in_fiscal_year` | Nobody holds a target for this metric and year | Assign targets in Settings → Targets |
| `no_targets_in_selected_quarters` | The year has targets, the selected quarters do not | Widen `--quarter`, or target those quarters |
| `owner_filter_matches_no_targeted_owner` | Targets exist, but not for the owners in `--owner` | Drop the filter to see who does hold one |

**Never present an empty grid as "the company has no pipeline."** Check `company`
first — it is almost certainly non-zero.

**3. `company` is the organisation; the `Total` table is not.** The JSON's `company`
object covers **every** opportunity in range — whoever owns it, whether or not anyone
targeted them, and regardless of `--owner`. The `Total` table sums only the rows above
it. The two are *expected* to differ whenever a deal belongs to an untargeted owner or
a filter is active; that difference is a fact about the book, so do not report it as an
inconsistency and do not reconcile one against the other.

- `company.aggregate` — the org's Closed Won / Commit / Best Case / Pipeline, plus the
  separately-assigned **company** target and its attainment.
- `company.over_assignment` — every assigned owner target minus the company target,
  paired per period. Positive means the individuals were deliberately given more than
  the company number, which is normal practice.
- `company` is **`null` when, and only when, the caller lacks `forecast.read`.** Then
  you have no org figure at all — say so, rather than summing the visible rows and
  calling the result the company.

**`crm forecast show --format human` does not print the company figure.** It prints the
per-owner tables and the `Total` footer only. Use `--format json` when the question is
about the organisation.

**When there are no rows, `--format human` prints the `empty_reason` explanation on
STDOUT and no tables at all.** That is deliberate: every figure in them would be
zero, and a zeroed `Total` footer reads as "the org has no pipeline" while `company`
is typically non-zero at the same moment. So the output is a sentence rather than a
table — read it and report the fix it names, and do not treat a table-less result as
a failed command.

An earlier version of this paragraph said "an empty human output is the explanation".
That was true only by accident of where the text went: the explanation was on
**stderr**, so an agent piping stdout got nothing at all and the sentence telling it
why was on a stream it was not reading. The message now goes to stdout, so there is
no "empty output" case to interpret.

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
number, **per period** — this command returns one row per fiscal period, and each
row's delta compares that period's owner targets against that period's company
target. Do not add the column up and present the result as an annual
over-assignment: a period with no company target has no delta to contribute, so a
total would silently mix covered and uncovered periods. (`crm forecast show`'s
`company.over_assignment` is the same rule already summed across the selection, over
only the periods that carry a company target.)

**Over-assigning is normal practice** — the company target is assigned
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
