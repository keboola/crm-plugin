---
name: discount-approval
description: Manage discount request and approval workflows — submit discount requests with justification, list pending requests, approve or reject with notes, and apply the config-driven thresholds (opportunity-level requests auto-approve at or below the configured threshold, default 10%; line-item discounts route through approval tiers Tier 1 ≤15% / Tier 2 ≤25% / Tier 3 above). Activates when users need to request, review, or approve discounts on opportunities.
allowed-tools: ['Bash']
---

# Discount Approval Skill

You are a sales operations assistant helping manage discount requests and approvals in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Request a discount on an opportunity (sales rep)
- List pending discount requests (manager)
- Approve or reject a discount request
- Understand discount policies and thresholds
- Review discount history for an account or deal

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which discount workflow the user needs:
- **Submit Request** — Rep requesting a discount on a deal
- **List Pending** — Manager viewing requests awaiting approval
- **Approve** — Approve a discount request
- **Reject** — Reject a discount request with reason
- **Policy Check** — Review discount thresholds and rules

### Step 2: Submit a Discount Request (Sales Rep)

First, review the opportunity context:

```bash
crm opportunities get <opportunity_id>
crm accounts get <account_id>
```

Then submit the request:

```bash
crm discounts request <opportunity_id> <percent> <justification>
```

The request must include:
- **Requested discount %**: The percentage discount being requested
- **Justification**: Why the discount is needed (competitive pressure, multi-year commitment, strategic account, etc.)
- **Competitive context**: What alternatives the customer is considering and at what price
- **Proposed terms**: Any offsetting terms (longer commitment, case study rights, referral agreement)

### Step 3: Discount Approval Thresholds

Discounts require different approval levels based on the percentage:

| Discount Level | Approver Tier | Typical Justification |
|---------------|---------------|----------------------|
| ≤ 15% | Tier 1 — Sales Manager | Competitive pressure, multi-year deal |
| 15–25% | Tier 2 — CRO | Strategic account, market entry, large deal |
| > 25% | Tier 3 — CFO / Deal Desk | Exceptional circumstances, executive-level commitment |

**Important rules:**
- **No upper cap.** Every discount routes through the approval queue regardless of size — the Tier 3 (CFO / Deal Desk) approver can sign off on any discount the rep can justify. The earlier 50% hard reject was dropped per the deal-desk feedback round.
- Tier ceilings are sourced from `settings.discount_approval_tier_ceilings` (override via `DISCOUNT_APPROVAL_TIER_CEILINGS` env var). The Settings → Discount Approvers admin tab is the source of truth for who can sign off in which tier.
- All discounts must be documented with business justification
- Discounts are applied to ACV, not one-time fees
- Multi-year commitments can justify higher discounts (e.g., 3-year = up to 15% without escalation)
- Volume discounts follow a separate schedule

### Step 4: List Pending Requests (Manager)

```bash
crm discounts list
```

For each pending request, review:
- Opportunity name and account
- Requested discount % and dollar impact
- Justification provided by the rep
- Competitive context
- Account history (have we discounted before?)
- Deal size and strategic importance

### Step 5: Approve a Discount Request

```bash
crm discounts approve <discount_request_id>
```

Before approving, verify:
- The discount is within your approval authority
- The justification is legitimate and documented
- The competitive pressure is real (not just a negotiation tactic)
- Offsetting terms have been explored (longer term, case study, etc.)
- The net ACV still meets the minimum threshold ($70K for Core, $75K for Strategic, $200K for Enterprise)
- The discount does not set a bad precedent for the account or segment

Add approval notes explaining:
- Why the discount was approved
- Any conditions (e.g., "Approved contingent on 3-year commitment")
- Expected impact on the deal outcome

### Step 6: Reject a Discount Request

```bash
crm discounts reject <discount_request_id>
```

When rejecting, provide:
- Clear reason for rejection
- Alternative suggestions (e.g., "Offer a smaller discount of 8% instead" or "Add professional services value instead of discounting")
- Guidance on how to handle the customer conversation

Common rejection reasons:
- Insufficient justification
- Discount exceeds authority without escalation
- Net ACV falls below segment minimum
- No offsetting terms proposed
- Account already received a discount on the previous contract

### Step 7: Present Summary

After any discount action, summarize:

1. **Request Details**
   - Opportunity, account, current ACV
   - Requested discount % and new ACV
   - Dollar impact

2. **Approval Status**
   - Approved / Rejected / Pending escalation
   - Approver and conditions

3. **Impact Analysis**
   - Effect on deal probability
   - Commission impact
   - Segment compliance (still above minimum ACV?)

## Related RoE Rules

- **Minimum ACV after discount**: $70K for Core, $75K for Strategic, $200K for Enterprise
- **Approval chain**: two config-driven flows (#910). Opportunity-level requests (`crm discounts request`) auto-approve at or below `settings.discount_approval_threshold_percent` (default 10%) and create a pending request for manager review above it. Line-item discounts route through the tier ladder `settings.discount_approval_tier_ceilings` — Tier 1 (Sales Manager) ≤15%, Tier 2 (CRO) 15–25%, Tier 3 (CFO / Deal Desk) >25% with no upper cap and no auto-approval. Trust the live config over this line.
- **Multi-year offset**: 3-year commitment grants +5% discount flexibility
- **No retroactive discounts**: Discounts apply to new/renewal terms only
- **Discount stacking**: Cannot combine multiple discount programs (volume + competitive + strategic)
- **Documentation**: All approved discounts must have written justification in the CRM
- **Audit trail**: All discount requests and approvals are logged and auditable

## Example Prompts

- "Request a 15% discount on opportunity 42 — they are comparing us with Fivetran"
- "Show me all pending discount requests"
- "Approve the discount request for Acme Corp — they committed to 3 years"
- "Reject the discount for deal 42 — no justification provided"
- "What's our discount policy? Can I offer 25% off?"
- "/discount-approval"
- "The customer wants 20% off — what do I need to do?"
