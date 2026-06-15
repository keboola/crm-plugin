---
name: territory-management
description: Manage sales territories — list territories, view accounts per territory, assign owners, auto-assign accounts by country, and handle reassignment requests with approval workflow. Activates when users need to work with territories, account assignments, or territory planning.
allowed-tools: ['Bash']
---

# Territory Management Skill

You are a sales operations assistant helping manage territories and account assignments in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- List or view territories
- See which accounts belong to a territory
- Set or change a territory owner
- Auto-assign an account based on HQ country
- Request an account reassignment between territories
- Approve or reject a reassignment request

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which territory workflow the user needs:
- **List Territories** — View all territories and their owners
- **Show Detail** — View a territory with key metrics
- **View Accounts** — List accounts assigned to a territory
- **Set Owner** — Assign or change territory owner
- **Auto-Assign** — Automatically assign an account based on HQ country
- **Request Reassignment** — Request moving an account to a different territory
- **Approve/Reject** — Process reassignment requests

### Step 2: List Territories

```bash
crm territory list
```

Territories in the Keboola CRM:
- **NA** — United States, Canada
- **EMEA** — Europe, Middle East, Africa
- **APAC** — Asia-Pacific
- **LATAM** — Latin America, Mexico

For each territory, show:
- Owner (assigned sales rep or team)
- Number of accounts
- Total pipeline value
- Active opportunities count

### Step 3: Show Territory Detail

```bash
crm territory show <territory_name>
```

Territory detail includes:
- **Owner**: Primary rep and overlay resources
- **Account count**: Total accounts, named accounts, active accounts
- **Pipeline**: Total pipeline value by stage
- **Performance**: Quota, attainment, win rate
- **Top accounts**: Largest accounts by ACV

### Step 4: View Accounts in a Territory

```bash
crm territory accounts <territory_name>
```

Account list with:
- Account name, segment (Enterprise, Strategic, Core)
- ACV (current and potential)
- Active opportunities
- Last activity date
- Account health score

Identify:
- Named accounts without active opportunities (potential whitespace)
- Accounts with no recent activity (at risk of churn)
- Accounts with expansion potential

#### CSM portfolio sub-view (post-sales ownership)

CSM ownership is tracked separately from Sales ownership and from the geo territory itself. To carve out a CSM's book of business from within `crm accounts list`, use the dedicated CSM-owner filters:

```bash
# All accounts where a specific CSM owns the post-sales relationship
crm accounts list --csm-owner-id <csm_user_id> --format human

# Strategic-tagged subset of that CSM's portfolio
crm accounts list --csm-owner-id <csm_user_id> --tag strategic

# Accounts WITH any CSM assigned (post-handover state)
crm accounts list --has-csm-owner

# Unassigned CSM slot — surfaces accounts that should be on someone's book
crm accounts list --no-csm-owner

# Cross-cut: this territory + this CSM
crm accounts list --territory EMEA --csm-owner-id <csm_user_id>
```

Use `--no-csm-owner` weekly during territory reviews to spot accounts that have closed-won deals but no CSM yet — those are at handover-failure risk. The `--csm-owner-id` filter requires a user ID; capture yours via `crm auth me` and substitute it in.

### Step 5: Set Territory Owner

```bash
crm territory set-owner <territory_name>
```

When assigning a territory owner:
- Verify the rep has capacity (not already overloaded)
- Check for active deals that need handover
- Ensure proper segment alignment (Enterprise reps for Enterprise accounts)
- Document the effective date

### Step 6: Auto-Assign Account by Country

```bash
crm territory auto-assign <account_id>
```

Auto-assignment rules based on HQ country:
- **NA**: United States, Canada
- **EMEA**: All European countries, UK, Middle East, Africa
- **APAC**: Australia, New Zealand, Japan, China, India, Southeast Asia
- **LATAM**: Mexico, Central America, South America, Caribbean

If the account HQ country does not match any territory rule, flag for manual review.

After assignment:
- Verify the territory is correct
- Notify the territory owner of the new account
- Check if the account matches the territory's segment focus

### Step 7: Request Reassignment

```bash
crm territory request-reassignment <account_id>
```

Reassignment requests are needed when:
- An account's HQ has moved to a different region
- A strategic decision to consolidate accounts under one rep
- Customer relationship requires a specific rep
- Territory rebalancing

Capture:
- Account being reassigned
- Source territory and destination territory
- Reason for reassignment
- Any active deals that will transfer
- Proposed effective date

### Step 8: Approve or Reject Reassignment

```bash
crm territory approve-reassignment <reassignment_id>
```

Before approving, verify:
- The reassignment reason is legitimate
- Both territory owners are aware
- Active deals have a handover plan
- No commission disputes will arise
- The move makes strategic sense

If rejecting, provide a clear reason and alternative suggestion.

## Related RoE Rules

- **Territory assignment**: Based on account HQ country; overrides require VP approval
- **Named accounts**: Strategic accounts are explicitly assigned regardless of geography
- **Account handover**: When territory changes, all active opportunities transfer with the account
- **Commission protection**: Deals in Offer/Negotiation stage at time of reassignment stay with the original rep
- **Rebalancing**: Territory reviews happen quarterly; mid-quarter changes are exceptional
- **Overlap resolution**: If an account has subsidiaries in multiple territories, the parent account territory takes precedence

## Example Prompts

- "Show me all territories and their owners"
- "Which accounts are in the EMEA territory?"
- "Auto-assign the new account — their HQ is in Germany"
- "Move Acme Corp from NA to EMEA — they relocated"
- "Approve the reassignment request for account 15"
- "Who owns the APAC territory?"
- "/territory-management"
- "Show territory performance for NA"
