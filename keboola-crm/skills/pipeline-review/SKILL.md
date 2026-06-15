---
name: pipeline-review
description: Review pipeline health, forecast accuracy, and identify risks across the sales pipeline. Analyzes stage distribution, deal velocity, stalled opportunities, and forecast gaps. Activates when users want a pipeline summary, forecast review, pipeline health check, or quarterly business review preparation.
allowed-tools: ['Bash']
---

# Pipeline Review Skill

You are a sales operations analyst helping review pipeline health and forecast accuracy.

## When to Activate

This skill activates when the user wants to:
- Review their pipeline
- Check forecast numbers
- Identify pipeline risks or gaps
- Prepare for a pipeline review or QBR
- Understand deal velocity and conversion rates

## Step-by-Step Workflow

### Step 1: Load Pipeline Data

```bash
crm pipeline show
crm pipeline forecast
crm opportunities list
crm analytics dashboard
crm analytics arr
```

`crm pipeline show` accepts `--owner <user_id>` and `--territory <code>` to scope the view to a specific rep or region. Deeper slices requested by the CRO audit (`--quarter`, `--type`, `--view`, `--csm-owner-id`, `--ending-within`, `--segment`) are not yet exposed — see the NOTE block in `crm pipeline show --help`. For now, fall back to `crm opportunities list` with `--stage`, `--owner`, `--account`, `--amount-min/--amount-max`, `--close-date-from/--close-date-to` to approximate them.

Common scoped views that already work today:

```bash
# Single rep's pipeline
crm pipeline show --owner user_123 --format human

# Regional rollup
crm pipeline show --territory EMEA --format human

# Approximation of "this quarter's deals" until --quarter lands
crm opportunities list --close-date-from 2026-04-01 --close-date-to 2026-06-30
```

### Step 2: Analyze Stage Distribution

Review the pipeline shape. A healthy pipeline should have:
- **Discovery**: Largest number of deals (wide top of funnel)
- **Demo**: Fewer than Discovery but healthy volume
- **Offer/Negotiation**: Fewer still, but with high conversion likelihood
- **Weighted pipeline**: Should be 3-4x the quota target

Flag issues:
- **Inverted funnel**: More deals in later stages than earlier (pipeline generation problem)
- **Bloated middle**: Too many deals stuck in Demo (qualification or follow-up problem)
- **Thin top**: Few Discovery deals (prospecting problem, future quarter risk)

### Step 3: Identify Stalled Deals

Look for opportunities that have been in the same stage for too long:
- **Discovery**: > 30 days without progression
- **Demo**: > 45 days without progression
- **Offer/Negotiation**: > 30 days without progression

For each stalled deal:
1. How long has it been in the current stage?
2. What are the blocking MEDDPICC gaps?
3. When was the last customer interaction?
4. Recommendation: push, nurture, or disqualify?

### Step 4: Forecast Analysis

Analyze the forecast by category:

- **Commit**: Deals the rep is confident will close this period (should be in Offer stage or later, all MEDDPICC strong)
- **Best Case**: Deals likely to close with favorable conditions (Demo or Offer stage, most MEDDPICC validated)
- **Pipeline**: Deals in earlier stages contributing to future quarters
- **Upside**: Deals that could close but are not yet committed

Check:
- Is the commit number realistic given the stage and MEDDPICC quality?
- Are close dates being pushed repeatedly? (sandbagging or wishful thinking)
- Is the weighted pipeline sufficient to cover the gap between commit and target?

### Step 5: Territory Coverage

```bash
crm territory list
crm reports forecast
```

Review by territory:
- Which territories are above/below quota pace?
- Are there territory gaps in pipeline generation?
- Any concentration risk (too much revenue from one account)?

### Step 6: Deal Quality Audit

For the top 5-10 deals by ACV, quick-check:

```bash
crm opportunities get <id>
crm opportunities qualify <id>
```

- Are MEDDPICC components adequate for the current stage?
- Are close dates realistic?
- Is the ACV within the account segment range?

### Step 7: Check Alerts

```bash
crm alerts list
```

Review any system-generated alerts:
- Stale opportunities
- Missing MEDDPICC components
- Upcoming close dates with incomplete qualification
- Contract renewals approaching

### Step 8: Generate Pipeline Report

Present a structured report:

1. **Pipeline Summary**
   - Total pipeline value (by stage)
   - Weighted pipeline value
   - Number of deals by stage
   - Average deal size

2. **Forecast**
   - Commit vs. target gap
   - Best case scenario
   - Pipeline coverage ratio
   - Quarter-over-quarter trend

3. **Health Indicators**
   - Pipeline shape (healthy, inverted, thin)
   - Stalled deal count and value
   - Average days in stage
   - Win rate trend

4. **Top Risks**
   - Deals most likely to slip
   - Missing MEDDPICC components on large deals
   - Close dates that appear unrealistic

5. **Recommended Actions**
   - Top 3 deals to focus on this week
   - Deals to disqualify or push to next quarter
   - Pipeline generation targets needed

## Example Prompts

- "Show me my pipeline"
- "How does the Q2 forecast look?"
- "Do I have enough pipeline to hit quota?"
- "/pipeline-review"
- "What deals are at risk of slipping?"
- "Pipeline health check"
- "Help me prepare for my pipeline review"
