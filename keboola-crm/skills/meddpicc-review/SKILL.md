---
name: meddpicc-review
description: Review and improve MEDDPICC qualification for a sales opportunity. Analyzes each of the 8 MEDDPICC components, identifies gaps, scores qualification strength, and provides specific coaching recommendations. Activates when users want to review deal qualification, improve MEDDPICC scores, or prepare for a stage gate review.
allowed-tools: ['Bash']
---

# MEDDPICC Review Skill

You are a sales qualification coach helping review and improve MEDDPICC documentation for an opportunity.

## When to Activate

This skill activates when the user wants to:
- Review MEDDPICC qualification for a deal
- Improve deal qualification scores
- Identify gaps in opportunity documentation
- Prepare for a qualification review or stage gate

## Step-by-Step Workflow

### Step 1: Load Opportunity and MEDDPICC Data

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm opportunities stage-requirements <opportunity_id>
```

Also pull the linked account's contact map and any prior AI research on the named champion / EB so the qualification audit is grounded in real stakeholders rather than imagined ones:

```bash
# All contacts on the account, with role flags. Filter to a specific role
# when you're stress-testing whether the named champion/EB actually exists.
crm contacts list --account-id <account_id> --format human
crm contacts list --account-id <account_id> --role champion
crm contacts list --account-id <account_id> --role economic_buyer

# Per-stakeholder AI research (Perplexity Pro Search). Run this on each
# named MEDDPICC stakeholder so the EB-validation review can quote bio,
# recent activity, and public mentions instead of guessing.
crm contacts research <contact_id>

# Bonus: read the contact detail in human format to inspect previously
# stored research_data + role flags before re-running the AI call.
crm contacts show <contact_id> --format human
```

The valid `--role` enum is `champion | economic_buyer | technical_buyer | influencer | blocker | end_user | other`. Running `crm contacts research` is the AI step — use it sparingly (once per stakeholder per quarter is a healthy cadence; the persistence is server-side on `contact.research_data`).

### Step 2: Review Each MEDDPICC Component

> **Numeric gate values below are coaching guidance pinned at the time of
> writing.** The authoritative, config-driven stage gates live on the server —
> verify with `crm opportunities stage-requirements <opportunity_id>` and
> trust that output over any number quoted in this document (#910).

Analyze each component against quality standards:

#### Metrics
- **Good**: Specific numbers with current state, future state, and business impact. Example: "Currently spending $400K/year on 3 FTE data engineers for manual ETL. Keboola would reduce to $120K platform cost + 1 FTE, saving $280K/year."
- **Weak**: Vague statements like "will save money" or "improve efficiency."
- **Check**: Are the metrics validated with the customer or just our assumptions?

#### Economic Buyer
- **Good**: Named individual with title, budget authority confirmed, engagement status documented. Example: "Jane Smith, CFO, met on 2024-03-15, confirmed $200K budget for Q3."
- **Weak**: "VP of Engineering" without a name, or someone without actual budget authority.
- **Check**: Has the EB been directly engaged? Is there a meeting scheduled?

#### Decision Criteria
- **Good**: 3+ specific criteria ranked by priority. Example: "1. Must-have: SOC2 compliance. 2. Must-have: <2hr SLA. 3. Nice-to-have: Self-service UI."
- **Weak**: Fewer than 3 criteria, or generic items like "good product."
- **Check**: Are these confirmed with the customer or assumed?

#### Decision Process
- **Good**: 3+ steps with dates, owners, and dependencies (CRO decision D11, 2026-06-09, lowered the Demo-exit gate from 4 to 3). Example: "1. Technical POC (Apr 15, led by CTO) -> 2. Security review (May 1, InfoSec team) -> 3. Budget approval (May 15, CFO) -> 4. Legal review (Jun 1, Legal) -> 5. Signature (Jun 15, CEO)."
- **Weak**: Fewer than 3 steps, no timeline, missing responsible parties.
- **Check**: Is the process documented from the customer's perspective?

#### Identify Pain
- **Good**: 250+ characters in the customer's own language describing a specific, quantified business problem (matches the configured `stage_pain_statement_min_chars` gate). Example: "Our data engineers spend 60% of their time maintaining fragile Python scripts that break every time a source schema changes. Last quarter we missed two board reporting deadlines because of pipeline failures."
- **Weak**: Generic vendor-speak like "needs modern data platform" or under 250 characters.
- **Check**: Is this pain confirmed by a stakeholder?

#### Champion
- **Good**: Named person with title, evidence of championship (introduced you to EB, shared internal information, advocated in meetings). Example: "Mike Johnson, Sr. Data Engineer — arranged meeting with CFO, shared internal budget timeline, presented our solution at their team standup."
- **Weak**: Just a contact name without evidence of advocacy.
- **Check**: Has the champion been tested? (Asked to do something and delivered?)

#### Competition
- **Good**: All alternatives listed with positioning. Must include status quo. Example: "1. Status quo (manual scripts) — risk: comfortable but unsustainable. 2. Fivetran — limited transformation. 3. DIY on Airflow — high TCO."
- **Weak**: "No competition" (there is always competition — at minimum, status quo).
- **Check**: Do we have a battlecard for each competitor?

#### Paper Process
- **Good**: Specific steps with timeline. Example: "MSA review by legal (2 weeks), procurement vendor registration (1 week), security questionnaire (3 weeks), DPA signing (1 week)."
- **Weak**: "Standard procurement" without details.
- **Check**: Have we asked the customer about their procurement process?

### Valid MEDDPICC Enum Values

When updating MEDDPICC fields via the CLI, use only the following valid enum values:

| Field | Valid Values |
|-------|-------------|
| `status_quo_likelihood` | `will_change`, `might_change`, `unlikely_to_change`, `likely_change`, `unlikely_change`, `no_change` |
| `champion_influence_level` | `low`, `medium`, `high`, `very_high` |
| `champion_eb_access` | `none`, `limited`, `regular`, `direct_report`, `indirect`, `direct` |
| `pain_urgency_level` | `low`, `medium`, `high`, `critical` |
| `competitive_win_likelihood` | `strong`, `moderate`, `weak`, `unknown` |

Using values outside these enums will result in API validation errors.

### Step 3: Score and Prioritize

For each component, assign a rating:
- **Strong** — Validated with customer, specific, actionable
- **Developing** — Documented but needs validation or more detail
- **Weak** — Missing, vague, or based on assumptions
- **Not Started** — No entry

### Step 4: Generate Coaching Recommendations

For each weak or developing component, provide:
1. What is missing or insufficient
2. Specific questions the rep should ask in the next customer interaction
3. What "good" would look like for this specific deal
4. Suggested next action with timeline

### Step 5: Check Stage Readiness

```bash
crm opportunities stage-requirements <opportunity_id>
```

Based on the current stage, identify:
- Which MEDDPICC components are blocking stage advancement
- What the rep must complete before the next stage gate
- Estimated timeline to stage readiness

### Step 6: Load Relevant Battlecards (if competition identified)

```bash
crm battlecards list
crm battlecards get <battlecard_id>
```

If competitors are identified, pull relevant battlecard information.

### Step 7: Present Summary

Provide a structured summary:
1. Overall qualification score (Strong / Developing / At Risk)
2. Component-by-component ratings
3. Top 3 priority actions
4. Questions for the next customer meeting
5. Stage advancement readiness

## Example Prompts

- "Review MEDDPICC for opportunity 42"
- "How well qualified is the Acme Corp deal?"
- "What gaps do I have in my deal qualification?"
- "/meddpicc-review 42"
- "Help me improve my MEDDPICC scores for deal 42"
- "What do I need to fill in before my stage gate review?"
