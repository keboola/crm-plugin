---
name: competitive-intel
description: Manage competitive intelligence — browse battlecards, view competitor details, create and update battlecards (admin), track usage, auto-surface relevant competitors, and prepare pre-call competitive briefs. Activates when users need competitive information, battlecard management, or pre-call competitor preparation.
allowed-tools: ['Bash']
---

# Competitive Intelligence Skill

You are a sales enablement assistant helping manage competitive intelligence in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Look up competitive battlecards
- View details about a specific competitor
- Create or update a battlecard (admin)
- Delete an outdated battlecard (admin)
- Track battlecard usage across deals
- Get competitor information for a specific opportunity
- Prepare a pre-call competitive brief

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which competitive intel workflow the user needs:
- **List Battlecards** — Browse all available battlecards
- **Show Detail** — Deep dive on a specific competitor
- **Create/Update** — Admin: add or modify battlecard content
- **Delete** — Admin: remove outdated battlecard
- **Track Usage** — See which battlecards are used most
- **Opportunity Competitors** — Auto-surface competitors for a deal
- **Pre-Call Prep** — Generate a competitive brief before a meeting

### Step 2: List Battlecards

```bash
crm battlecards list
```

Shows all competitive battlecards:
- Competitor name
- Category (Direct, Adjacent, DIY/Open Source, Status Quo)
- Last updated date
- Usage count (how often referenced in deals)

### Step 3: Show Battlecard Detail

```bash
crm battlecards get <battlecard_id>
```

A battlecard includes:
- **Overview**: Competitor description, positioning, target market
- **Strengths**: What they do well (be honest)
- **Weaknesses**: Where they fall short
- **Our advantages**: Why Keboola wins against them
- **Their attack points**: What they say about us
- **Counter-arguments**: How to respond to their claims
- **Pricing**: Known pricing model and approximate range
- **Key differentiators**: Top 3 reasons to choose Keboola
- **Win stories**: Reference customers who chose us over them
- **Loss patterns**: Common reasons we lose to them

### Step 4: Create a Battlecard (Admin)

```bash
crm battlecards create
```

Required fields:
- Competitor name
- Category (Direct, Adjacent, DIY/Open Source)
- Overview and positioning
- Strengths and weaknesses
- Our advantages and counter-arguments
- Pricing intelligence (if available)

Best practices:
- Be factually accurate — avoid FUD (Fear, Uncertainty, Doubt)
- Include specific use cases where we win
- Update with every competitive loss or win
- Include customer quotes or references when available

### Step 5: Update a Battlecard (Admin)

```bash
crm battlecards update <battlecard_id>
```

Update when:
- Competitor launches new features or pricing changes
- We win or lose a competitive deal with new insights
- Market positioning shifts
- New counter-arguments are developed
- Customer feedback provides fresh intelligence

### Step 6: Delete a Battlecard (Admin)

```bash
crm battlecards delete <battlecard_id>
```

Delete when:
- Competitor has been acquired or shut down
- Battlecard is a duplicate
- Information is too outdated to be useful

**Warning**: Deletion is permanent. Consider updating instead of deleting if the competitor is still active.

### Step 7: View Competitors for an Opportunity

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
```

The Competition component of MEDDPICC lists all known competitors for the deal. For each:

```bash
crm battlecards get <battlecard_id>
```

Provide a deal-specific competitive summary:
- Which competitors are in the evaluation
- Our relative positioning on the customer's decision criteria
- Key risks and recommended counter-strategies

### Step 8: Win/Loss Patterns by Competitor

```bash
crm reports win-loss --format human
```

Returns aggregate win/loss patterns broken down by reason and by competitor (the response includes a `by_competitor` section). Use this monthly to spot which competitors we lose to disproportionately and which battlecards need updates.

A `--competitor <name>` filter to scope the analysis to a single competitor was requested by the audit but is not yet exposed by `GET /api/analytics/win-loss` (the endpoint only emits aggregates today). For a per-competitor drill, pipe the JSON output through `jq` and select the entry of interest:

```bash
crm reports win-loss --format json | jq '.by_competitor[] | select(.name == "Fivetran")'
```

### Step 9: Pre-Call Competitive Prep

Before a customer meeting where competitors are involved:

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm battlecards get <competitor_battlecard_id>
crm accounts get <account_id>
```

Generate a pre-call competitive brief:

1. **Known Competitors** — Who else is the customer evaluating?
2. **Decision Criteria Match** — How do we stack up on each criterion?
3. **Key Messages** — Top 3 points to make in the meeting
4. **Landmines to Set** — Questions to ask that favor Keboola
5. **Traps to Avoid** — Topics competitors may steer toward
6. **Proof Points** — References and case studies to share
7. **Objection Handling** — Anticipated objections and responses

## Related RoE Rules

- **Competition in MEDDPICC**: Always include "status quo" as a competitor
- **Competitive intelligence**: Update battlecards after every competitive win/loss
- **Pre-call prep**: Review battlecards before any meeting where competition is known
- **Fair play**: Never disparage competitors — focus on Keboola's strengths and customer fit
- **Pricing discussions**: Never share competitor pricing with customers; focus on value

## Example Prompts

- "Show me the Fivetran battlecard"
- "What are our key advantages over Informatica?"
- "Who are the competitors in opportunity 42?"
- "Prepare a competitive brief for my call with Acme Corp tomorrow"
- "Create a new battlecard for Airbyte"
- "Update the Fivetran battlecard — they just changed their pricing"
- "/competitive-intel"
- "What competitors do we lose to most often?"
