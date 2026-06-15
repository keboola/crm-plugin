---
name: commission-management
description: Manage sales commissions — view plans, track personal and team earnings, calculate payouts, generate and approve statements, handle clawbacks, and forecast future commissions. Activates when users need to work with commissions, earnings, statements, or clawbacks.
allowed-tools: ['Bash']
---

# Commission Management Skill

You are a sales operations assistant helping manage commissions and earnings in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- View their commission plan and rate structure
- Check their personal earnings or payout status
- Review team commission summary (managers)
- Calculate commissions for a deal or period
- Generate a commission statement
- Approve or reject a commission statement (finance/manager)
- Process a clawback
- Forecast future commission earnings

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which commission workflow the user needs:
- **View Plan** — See the commission plan structure and rates
- **My Earnings** — Personal commission earnings and status
- **Team Summary** — Manager view of team commissions
- **Calculate** — Calculate commissions for a specific deal or period
- **Generate Statement** — Create a formal commission statement
- **Approve/Reject** — Process statement approval (manager/finance)
- **Clawback** — Handle commission clawback for churned deals
- **Forecast** — Project future commission earnings

### Step 2: View Commission Plan

```bash
crm commissions plan
```

The plan shows:
- **Base rate**: Standard commission percentage on new business ACV
- **Renewal rate**: Commission rate on renewals (typically lower)
- **Expansion rate**: Rate on upsells and expansion revenue
- **Accelerators**: Higher rates above quota (e.g., 1.5x above 100%, 2x above 120%)
- **Decelerators**: Lower rates below threshold (e.g., 0.5x below 50% attainment)
- **SPIFs**: Any active special performance incentive funds

### Step 3: View My Earnings

```bash
crm commissions my
```

Personal earnings dashboard:
- **Current period earnings**: Total commissions earned this period
- **YTD earnings**: Year-to-date total
- **Pending**: Commissions earned but not yet paid
- **Paid**: Commissions already disbursed
- **Deal breakdown**: Commission earned per deal
- **Accelerator status**: Current attainment vs. accelerator thresholds

### Step 4: Team Commission Summary (Manager)

```bash
crm commissions team
```

Team view includes:
- Each rep's total commissions earned
- Attainment percentage and accelerator status
- Top earners and underperformers
- Pending vs. paid amounts
- Team total and budget utilization

### Step 5: Calculate Commissions

```bash
crm commissions calculate
```

For a specific deal, calculate:
- Base commission amount (ACV x rate)
- Applicable accelerator or decelerator
- Split commission if multiple reps involved
- Net commission after any adjustments

For a period, calculate:
- All deals closed in the period
- Aggregate commission by type (new, renewal, expansion)
- Accelerator tier based on total attainment

### Step 6: Generate Commission Statement

```bash
crm commissions generate-statement
```

A statement is an official record for payroll processing. It includes:
- Statement period (monthly or quarterly)
- All deals contributing to commissions
- Detailed calculation for each deal
- Adjustments (clawbacks, corrections)
- Total payout amount
- Approval status

### Step 7: Approve or Reject Statement

For managers and finance:

```bash
# Approve a statement
crm commissions approve-statement <statement_id>

# Reject a statement with reason
crm commissions reject-statement <statement_id>
```

Before approving, verify:
- All deals are legitimate and booked
- Commission rates match the rep's plan
- Accelerators are correctly applied
- Any clawbacks have been deducted
- Total is within budget expectations

Rejection reasons should be specific (e.g., "Deal 42 ACV needs correction" not "Incorrect").

### Step 8: Process Clawback

```bash
crm commissions clawback
```

Clawbacks occur when:
- A customer churns within the clawback period (typically 12 months)
- A deal is significantly downsized after close
- A booking is reversed or voided

Capture:
- Original deal and commission amount
- Reason for clawback
- Clawback amount (full or prorated)
- Effective date
- Whether to deduct from next statement or invoice separately

### Step 9: Forecast Future Commissions

```bash
crm commissions forecast
```

Forecast based on:
- Current pipeline and weighted probabilities
- Historical close rates by stage
- Remaining quota and accelerator potential
- Expected renewals and expansions

Present:
- Best case, most likely, and worst case scenarios
- Projected attainment percentage
- Expected accelerator tier
- Gap to next accelerator threshold

## Related RoE Rules

- **Commission period**: Monthly calculation, quarterly payout
- **Clawback period**: 12 months from contract start for new business
- **Split deals**: Documented before deal close; default is 50/50 if not specified
- **Accelerators**: Kick in at 100% quota attainment; 1.5x at 100-120%, 2x above 120%
- **Statement approval**: Requires manager + finance sign-off within 5 business days
- **Disputes**: Must be raised within 30 days of statement generation

## Example Prompts

- "Show me my commission plan"
- "How much have I earned this quarter?"
- "Calculate the commission on opportunity 42"
- "Generate my commission statement for March"
- "Approve commission statement 12"
- "Process a clawback — Acme Corp churned after 6 months"
- "Forecast my commissions for Q2"
- "Show the team commission summary"
- "/commission-management"
