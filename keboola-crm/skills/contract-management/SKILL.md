---
name: contract-management
description: Manage the full contract lifecycle — create contracts, view details, renew, amend, inspect the renewal history chain, and generate bookings reports. Activates when users need to work with contracts, renewals, amendments, or bookings.
allowed-tools: ['Bash']
---

# Contract Management Skill

You are a sales operations assistant helping manage contracts in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Create or view a contract
- Renew an existing contract
- Amend contract terms (upsell, downsell, add products)
- View the full renewal/amendment history chain
- Generate a bookings report

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which contract workflow the user needs:
- **List/Search** — Find contracts by account, status, or date range
- **Show Detail** — View full contract details
- **Create** — Create a new contract from a Closed Won opportunity
- **Renew** — Initiate a renewal for an expiring contract
- **Amend** — Modify an active contract mid-term
- **History** — View the full chain of renewals and amendments
- **Bookings** — Generate a bookings report

### Step 2: List or Search Contracts

```bash
crm orders list
```

Review the list and help the user find the relevant contract. Key fields to note:
- Contract ID, account name, status (Active, Expired, Pending)
- Start date, end date, ACV, total contract value
- Renewal status (Auto-renew, Manual, Pending Renewal)

### Step 3: View Contract Details

```bash
crm orders get <order_id>
```

Present key information:
- **Account and opportunity** link
- **Term**: Start date, end date, duration
- **Financials**: ACV, TCV, payment terms, billing frequency
- **Status**: Active, Pending, Expired
- **Products/SKUs** included
- **Special terms** or discount details

### Step 4: Create a New Contract

A contract is created from a Closed Won opportunity. Verify the opportunity is in the correct state:

```bash
crm opportunities get <opportunity_id>
crm opportunities checklist <opportunity_id>
```

Ensure:
- Opportunity is Closed Won
- All paper process steps are complete
- Commercial terms are finalized
- Contract start date and duration are agreed

### Step 5: Renew a Contract

```bash
crm orders renew <order_id> --total <total_value> --end-date <YYYY-MM-DD>
```

This creates a **draft** renewal Order (status=draft) linked to the
original via `parent_order_id`. It is **not** active until you activate
it:

```bash
crm orders activate <new_order_id>
```

Note: unlike ARR/term-based renewals, `crm orders renew` takes the total
order value via `--total` and explicit dates via `--end-date`. The
effective date defaults to the parent order's end date + 1 day if not
overridden.

Before initiating renewal, review:
- Current order terms and total value
- Account health and satisfaction
- Any open issues or support tickets
- Proposed renewal terms (same, upsell, price increase)

Renewal best practices per RoE:
- Start renewal conversations 90 days before expiry
- Document any changes from original terms
- If total value changes by more than 10%, flag for manager review

### Step 6: Amend a Contract

```bash
crm orders amend <order_id> --total <total_value> --effective <YYYY-MM-DD> --end-date <YYYY-MM-DD> --description "..."
```

This creates a **draft** amendment Order (status=draft) linked to the
original via `parent_order_id`. As with renewals, the amendment is **not**
active until you activate it:

```bash
crm orders activate <new_order_id>
```

Common amendment scenarios:
- **Upsell** — Adding products or increasing usage tier
- **Downsell** — Reducing scope (requires manager approval)
- **Term change** — Extending or shortening the contract period
- **Price adjustment** — Mid-term pricing changes (requires VP approval for >10%)

For each amendment, capture:
- Reason for amendment
- Financial impact (delta ACV)
- Effective date
- Whether a new SOW or addendum is needed

### Step 7: View History Chain

```bash
crm orders history <order_id>
```

The history chain shows:
- Original contract and all subsequent renewals/amendments
- ACV changes over time
- Key dates and term changes
- Net expansion or contraction over the customer lifecycle

### Step 8: Generate Bookings Report

```bash
crm analytics bookings
```

Optionally scope the report by quarter and year, e.g. `crm analytics
bookings --quarter Q1 --year 2026`.

The bookings report shows:
- New business bookings (new logos)
- Renewal bookings
- Expansion bookings (upsells, amendments)
- Contraction (downsells)
- Churn (non-renewals)
- Net bookings total

## Related RoE Rules

- **Minimum ACV**: $70K for Core Mid-Market, $75K for Strategic, $200K for Enterprise
- **Renewal timeline**: Start 90 days before expiry
- **Discount thresholds**: config-driven (#910) — opportunity-level requests auto-approve at or below `settings.discount_approval_threshold_percent` (default 10%); line-item tiers: ≤15% Sales Manager, 15–25% CRO, >25% CFO / Deal Desk (no cap)
- **Amendment approvals**: Downsells require manager approval; material changes need legal review
- **Bookings recognition**: Bookings are recognized on contract signature date

## Example Prompts

- "Show me all active orders for Acme Corp"
- "Renew order ord_15 with a 10% price increase"
- "Amend order ord_15 to add the data apps SKU"
- "Show the full renewal history for order ord_15"
- "Generate a bookings report for Q1"
- "/contract-management"
- "What orders are expiring in the next 90 days?"
- "Create an order from opportunity 42"
