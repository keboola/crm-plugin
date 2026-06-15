---
name: deal-review
description: Comprehensive deal review before a stage gate, QBR, or management review. Combines opportunity analysis, MEDDPICC audit, risk assessment, competitive positioning, and action planning into a single structured review. Activates when users want a full deal review, stage gate preparation, or management-ready deal summary.
allowed-tools: ['Bash']
---

# Deal Review Skill

You are a senior sales manager conducting a comprehensive deal review to assess readiness and risk.

## When to Activate

This skill activates when the user wants to:
- Perform a full deal review
- Prepare for a stage gate meeting
- Get a management-ready deal summary
- Assess overall deal health before a critical milestone
- Prepare for a QBR with deal-level detail

## Step-by-Step Workflow

### Step 1: Load All Deal Data

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm opportunities stage-requirements <opportunity_id>
crm accounts get <account_id>
crm contacts list --account-id <account_id> --format human
```

For a portfolio-wide triage scan ahead of the deal review (find the deals that need this treatment in the first place), use the new amount/close-date filters:

```bash
# Big deals closing this quarter
crm opportunities list --amount-min 100000 --close-date-from 2026-04-01 --close-date-to 2026-06-30 --format human

# Bounded ACV bucket
crm opportunities list --amount-min 50000 --amount-max 250000 --format human

# Renewal book of business
crm opportunities list --type existing_business_renewal --format human
```

Note: `--meddpicc-score-below`, `--has-champion / --no-champion`, `--has-eb`, `--stale-days`, and `--competitor` are requested by the audit but not yet exposed by `GET /api/opportunities` (see the NOTE block in `crm opportunities list --help`). When they land you'll be able to one-shot "MEDDPICC under 50 and stale 30+ days" — until then, list by amount/close-date and inspect `crm opportunities qualify` per deal.

### Step 2: Deal Overview Assessment

Review and summarize:
- **Deal basics**: Name, account, ACV, stage, probability, close date
- **Timeline**: Days in pipeline, days in current stage, original vs. current close date
- **Segment fit**: Does ACV match account segment? (Enterprise $200K+, Strategic $75-200K, Core $70K min)
- **Close date realism**: Has the close date been pushed? How many times?

### Step 3: Full MEDDPICC Audit

For each of the 8 components, assess quality on a 3-point scale:

| Rating | Meaning |
|--------|---------|
| Green | Validated with customer, specific, complete |
| Yellow | Documented but needs validation or more detail |
| Red | Missing, vague, or assumption-based |

Provide the rating and a one-line justification for each:
- Metrics
- Economic Buyer
- Decision Criteria
- Decision Process
- Identify Pain
- Champion
- Competition
- Paper Process

### Step 4: Risk Assessment

Evaluate deal risks across categories:

**Execution Risks:**
- Is the champion still engaged?
- Has the EB been directly engaged?
- Are we single-threaded (only one contact)?
- Have we met the technical team?

**Competitive Risks:**
- Is status quo the real competitor?
- Are we positioned well on the top decision criteria?
- Has the customer evaluated competitors recently?

**Timeline Risks:**
- Is there a compelling event driving the timeline?
- Has the close date been pushed?
- Are there procurement bottlenecks (budget cycles, security reviews)?

**Commercial Risks:**
- Is the ACV realistic for this account size?
- Are there discount expectations?
- Is multi-year vs. annual discussed?

### Step 5: Competitive Positioning

```bash
crm battlecards list
crm battlecards get <id>   # For each identified competitor
```

For each competitor identified in MEDDPICC:
- Our strengths vs. theirs
- Their likely attack points
- Our counter-arguments
- Customer perception (if known)

### Step 6: Stage Gate Readiness

```bash
crm opportunities stage-requirements <opportunity_id>
```

If the deal is approaching a stage gate:
- List all exit criteria with pass/fail status
- Identify the critical blockers
- Estimate timeline to resolve each blocker

If the deal is in Offer/Negotiation:
```bash
crm opportunities checklist <opportunity_id>
```

### Step 7: Generate Deal Review Report

Present a structured report suitable for management review:

1. **Executive Summary** (2-3 sentences)
   - Deal name, account, ACV, expected close date
   - Overall health: Healthy / At Risk / Critical
   - One key insight or concern

2. **MEDDPICC Scorecard**
   - 8 components with Green/Yellow/Red ratings
   - Overall qualification: Strong / Developing / Weak

3. **Risk Matrix**
   - Top 3 risks ranked by severity
   - Mitigation plan for each

4. **Competitive Landscape**
   - Primary competitors and our positioning
   - Win/loss probability assessment

5. **Timeline and Milestones**
   - Key dates (next meeting, POC, EB meeting, procurement start)
   - Critical path to close

6. **Action Plan**
   - Top 5 actions required, ordered by priority
   - Owner and due date for each
   - What help is needed from management

7. **Go/No-Go Recommendation**
   - Continue investing / Reassess / Disengage
   - Justification

## Example Prompts

- "Full review of opportunity 42"
- "Prepare deal review for the Acme Corp opportunity before our stage gate"
- "Give me a management-ready summary of deal 42"
- "/deal-review 42"
- "Is deal 42 healthy? Do a full assessment"
- "I have a QBR tomorrow, review my top deals"
