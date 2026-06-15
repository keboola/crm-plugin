---
name: new-opportunity
description: Create and qualify a new sales opportunity in the Keboola CRM. Guides the user through account lookup or creation, opportunity creation with proper segmentation, initial MEDDPICC qualification, and stage setup. Activates when users want to create a new deal, log a new opportunity, or start tracking a prospect.
allowed-tools: ['Bash']
---

# New Opportunity Skill

You are a sales operations assistant helping create and qualify a new opportunity in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Create a new opportunity or deal
- Start tracking a new prospect
- Log a new sales engagement
- Add a deal to the pipeline

## Step-by-Step Workflow

### Step 1: Gather Opportunity Information

Ask the user for (if not already provided):
- **Account name** — Company the opportunity is with
- **Opportunity name** — Descriptive deal name
- **Estimated ACV** — Annual Contract Value estimate
- **Close date** — Expected close date
- **Description** — Brief description of the opportunity
- **Contact** — Primary contact at the account

### Step 2: Look Up or Create Account

```bash
# Search for existing account
crm accounts list

# If account exists, get details
crm accounts get <account_id>

# If account does not exist, create it
crm accounts create
```

If creating a new account, ensure:
- Company name is correct
- Industry and employee count are captured (needed for segmentation)
- HQ country is set (determines territory: NA, EMEA, APAC, LATAM)

### Step 3: Validate Account Segmentation

Based on employee count and ACV, verify the deal fits the segmentation:

| Segment | Employees | Minimum ACV |
|---------|-----------|-------------|
| Enterprise | 2000+ | $200K |
| Strategic Mid-Market | 500-1999 | $75K |
| Core Mid-Market | 200-499 | $70K |

**If ACV is below the minimum threshold, warn the user.** Deals below $70K ARR should not be pursued unless there is a documented exception.

### Step 4: Create the Opportunity

```bash
crm opportunities create
```

Set:
- Stage: **Discovery** (all new opportunities start here)
- Probability: **20%**
- Link to the correct account
- Set close date and ACV

### Step 5: Initialize MEDDPICC Qualification

```bash
crm opportunities qualify <opportunity_id>
```

Guide the user through initial MEDDPICC entries. At minimum for Discovery stage, capture:

1. **Identify Pain** — What problem is the customer trying to solve? Use the customer's own language. Aim for 250+ characters (the configured stage-gate minimum; verify with `crm opportunities stage-requirements`).
2. **Metrics** — What is the quantified business impact? Even rough estimates help (e.g., "spending $500K/year on manual ETL processes").
3. **Champion** — Who is the internal contact? Even if not yet validated as a true champion, record the primary contact.
4. **Competition** — What alternatives exist? Always include "status quo" as an option.

For the remaining components (Economic Buyer, Decision Criteria, Decision Process, Paper Process), create placeholder entries noting they need to be discovered.

### Step 6: Verify and Summarize

```bash
crm opportunities get <opportunity_id>
crm opportunities stage-requirements <opportunity_id>
```

Present a summary to the user:
- Opportunity details (name, account, ACV, stage, close date)
- MEDDPICC completion status
- What needs to be completed before advancing to Demo stage
- Any warnings (ACV below threshold, missing information)

## Validation Rules

- **ACV minimum**: $70K RoE guidance for any opportunity — flag if below. The API rejects below-floor amounts only when the configured floor (`thresholds.min_deal_size_arr` in `/api/config/public`) is greater than 0; the default is 0 (floor disabled), so never claim an unconditional hard block (#910).
- **Stage**: Always starts at Discovery (20%).
- **Territory**: Auto-assigned based on account HQ country.
- **MEDDPICC**: All 8 components should have at least placeholder entries.
- **Close date**: Should be realistic (typically 3-9 months from Discovery).

## Example Prompts

- "Create a new opportunity for Acme Corp, they want to replace their Informatica ETL, about $120K"
- "I just had a great call with BigTech Inc, they need a data platform, help me log this deal"
- "New prospect: DataCo, 800 employees, looking at $90K ACV for data integration"
- "/new-opportunity"
- "Start tracking a new deal"
