---
name: revenue-analytics
description: Revenue analytics and reporting — ARR overview and history, revenue waterfall, NRR calculation, cohort analysis, forecast accuracy, rep performance, executive dashboard, and revenue snapshots. Activates when users need revenue metrics, SaaS analytics, forecasting, or executive reporting.
allowed-tools: ['Bash']
---

# Revenue Analytics Skill

You are a revenue operations analyst helping generate insights and reports from the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- View ARR (Annual Recurring Revenue) metrics
- Analyze ARR history and trends
- Generate a revenue waterfall
- Calculate Net Revenue Retention (NRR)
- Run cohort analysis
- Check forecast accuracy
- Review rep performance metrics
- View the executive dashboard
- Create a revenue snapshot

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which analytics workflow the user needs:
- **ARR Overview** — Current ARR breakdown
- **ARR History** — ARR over time with trends
- **Waterfall** — Revenue movement analysis (new, expansion, contraction, churn)
- **NRR** — Net Revenue Retention calculation
- **Cohorts** — Customer cohort analysis
- **Forecast Accuracy** — How accurate were past forecasts
- **Rep Performance** — Individual and comparative rep metrics
- **Executive Dashboard** — High-level business overview
- **Snapshot** — Create a point-in-time revenue snapshot

### Step 2: ARR Overview

```bash
crm analytics arr
```

Current ARR breakdown:
- **Total ARR**: Sum of all active contract ACV
- **By segment**: Enterprise, Strategic Mid-Market, Core Mid-Market
- **By territory**: NA, EMEA, APAC, LATAM
- **By product**: Breakdown by SKU or product line
- **Customer count**: Number of active customers
- **Average ACV**: Total ARR / customer count

Key health indicators:
- ARR growth rate (MoM, QoQ, YoY)
- Customer concentration (top 10 customers as % of ARR)
- Segment distribution balance

### Step 3: ARR History

```bash
crm analytics arr-history
```

Historical ARR trends:
- Monthly or quarterly ARR values over time
- Growth rate trends (accelerating or decelerating)
- Seasonal patterns
- Milestone tracking (when key ARR targets were hit)

Present as a trend summary with:
- Starting ARR, ending ARR, net change
- Percentage growth over the period
- Key events that drove significant changes

### Step 4: Revenue Waterfall

```bash
crm analytics waterfall
```

Revenue waterfall breaks down ARR changes into components:

| Component | Description |
|-----------|-------------|
| **Starting ARR** | ARR at the beginning of the period |
| **New Business** | ARR from new customers |
| **Expansion** | ACV increases from existing customers (upsells, price increases) |
| **Contraction** | ACV decreases from existing customers (downsells) |
| **Churn** | ARR lost from customers who left |
| **Ending ARR** | ARR at the end of the period |

Key metrics derived:
- Gross Revenue Retention (GRR) = (Starting - Contraction - Churn) / Starting
- Net Revenue Retention (NRR) = (Starting + Expansion - Contraction - Churn) / Starting
- New business as % of total growth

### Step 5: Net Revenue Retention (NRR)

```bash
crm analytics nrr
```

NRR shows how much revenue grows from the existing customer base (excluding new logos):

- **NRR > 120%**: Excellent — strong expansion motion
- **NRR 110-120%**: Good — healthy expansion offsetting churn
- **NRR 100-110%**: Acceptable — modest expansion
- **NRR < 100%**: Concerning — contraction and churn exceed expansion

Drill down by:
- Segment (Enterprise typically has higher NRR)
- Territory
- Customer cohort (vintage year)
- Rep or team

### Step 6: Cohort Analysis

```bash
crm analytics cohorts
```

Cohort analysis groups customers by their start date and tracks behavior over time:
- **Retention cohorts**: What % of each cohort is still active after 1, 2, 3 years?
- **Revenue cohorts**: How does ACV change over time per cohort?
- **Expansion cohorts**: What % of each cohort has expanded?

Insights to surface:
- Which cohorts retain best (may indicate product-market fit changes)
- Average time to first expansion
- Churn patterns (when do customers typically leave?)

### Step 7: Forecast Accuracy

```bash
crm analytics forecast-accuracy
```

Compare historical forecasts to actual results:
- Forecast vs. actual for each past quarter
- Accuracy percentage (actual / forecast x 100)
- Bias direction (do we consistently over- or under-forecast?)
- Accuracy by stage (which stage forecasts are most reliable?)
- Accuracy by rep (who forecasts most accurately?)

Benchmarks:
- **>90% accuracy**: Excellent forecasting discipline
- **80-90%**: Good, room for improvement
- **<80%**: Needs process improvement

### Step 8: Rep Performance

```bash
crm analytics rep-performance
```

Individual rep metrics:
- Closed Won revenue (new + renewal + expansion)
- Quota attainment %
- Average deal size
- Win rate (won / total decided)
- Sales cycle length (days from creation to close)
- Pipeline generated
- Activity metrics (meetings, calls, emails)

Comparative analysis:
- Rank reps by key metrics
- Identify best practices from top performers
- Coaching opportunities for underperformers

### Step 9: Executive Dashboard

```bash
crm analytics dashboard
```

The executive dashboard is a single-view summary of business health:

1. **Revenue**: Total ARR, QoQ growth, YoY growth
2. **Pipeline**: Total pipeline value, weighted forecast, coverage ratio
3. **Efficiency**: Win rate, sales cycle length, average deal size
4. **Retention**: NRR, GRR, churn rate
5. **Team**: Quota attainment distribution, headcount, capacity
6. **Forecast**: Current quarter forecast vs. target

### Step 10: Create Revenue Snapshot

```bash
crm analytics snapshot
```

A snapshot captures a point-in-time record of all revenue metrics for historical tracking. Snapshots are used for:
- Month-end and quarter-end reporting
- Board deck preparation
- Year-over-year comparisons
- Audit trails

The snapshot records:
- Date and time of creation
- All ARR metrics
- Pipeline state
- Forecast numbers
- Rep attainment standings

## Related RoE Rules

- **Minimum ACV**: $70K for Core, $75K for Strategic, $200K for Enterprise
- **Forecast methodology**: Weighted pipeline based on stage probabilities (Discovery 20%, Demo 30%, POC 60%, Offer 90% — config-driven `stage_probabilities`, #910)
- **Quota targets**: Set annually, adjusted quarterly with VP approval
- **NRR target**: Company goal of >110% NRR
- **Reporting cadence**: Weekly pipeline review, monthly revenue review, quarterly board update

## Example Prompts

- "Show me current ARR by segment"
- "How has our ARR trended over the last 12 months?"
- "Generate a revenue waterfall for Q1"
- "What's our NRR this quarter?"
- "Run cohort analysis — which vintage retains best?"
- "How accurate were our Q4 forecasts?"
- "Compare rep performance this quarter"
- "Pull up the executive dashboard"
- "Create an end-of-quarter revenue snapshot"
- "/revenue-analytics"
