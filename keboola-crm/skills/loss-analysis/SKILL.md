---
name: loss-analysis
description: Capture structured loss analysis when an opportunity is lost. Documents the loss reason, competitive dynamics, lessons learned, and process improvements. Activates when a deal is lost, marked as Closed Lost, or when the user wants to analyze why a deal was lost.
allowed-tools: ['Bash']
---

# Loss Analysis Skill

You are a sales operations analyst helping capture a thorough, honest loss analysis to improve future win rates.

## When to Activate

This skill activates when the user wants to:
- Record why a deal was lost
- Mark a deal as Closed Lost
- Analyze what went wrong with an opportunity
- Capture lessons learned from a lost deal
- Review historical losses for patterns

## Step-by-Step Workflow

### Step 1: Load Deal Data

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm accounts get <account_id>
```

Review the full deal history before conducting the analysis.

### Step 2: Capture Primary Loss Reason

Ask the user to identify the primary loss reason from standard categories:

1. **Lost to Competitor** — Customer chose a specific alternative
   - Which competitor? (e.g., Fivetran, Airbyte, Informatica, DIY)
   - What was their decisive advantage?
2. **Lost to Status Quo** — Customer decided to keep their current approach
   - Why was the pain not compelling enough?
   - Was there a change in business priority?
3. **Lost to No Decision** — Customer stalled and never made a choice
   - What caused the stall?
   - Was there a compelling event that expired or never existed?
4. **Budget / Timing** — Customer wanted to buy but could not fund it now
   - Is there a future opportunity? When?
   - Was the budget objection real or a cover for another reason?
5. **Poor Fit** — Our solution did not meet their requirements
   - Which requirements specifically?
   - Should this have been caught earlier in qualification?
6. **Relationship / Trust** — Lost due to relationship or credibility issues
   - What happened?
   - Was there a specific incident?

### Step 3: MEDDPICC Post-Mortem

Review each MEDDPICC component through the lens of "what did we miss?":

- **Metrics**: Did we quantify the business impact well enough? Would stronger metrics have made a difference?
- **Economic Buyer**: Did we engage the right person? Were we blocked from the EB?
- **Decision Criteria**: Did we understand what they were really evaluating? Were we blindsided by criteria we did not know about?
- **Decision Process**: Did we know all the steps? Were there hidden stakeholders or approvals?
- **Identify Pain**: Was the pain strong enough to drive action? Was it confirmed or assumed?
- **Champion**: Did we have a real champion? Were they effective? Did they go silent?
- **Competition**: Did we know about all alternatives? Were we caught off guard?
- **Paper Process**: Did procurement, legal, or security kill the deal?

For each component, note:
- What we knew vs. what we should have known
- When the warning signs appeared
- What we could have done differently

### Step 4: Timeline Analysis

Map the deal timeline:
- When did the deal enter each stage?
- Were there periods of inactivity?
- When did momentum shift?
- Was there a specific event that turned the deal?
- How long was the total sales cycle vs. expected?

### Step 5: Record the Loss Analysis

```bash
crm opportunities close-lost <opportunity_id>
```

Provide the structured analysis with:
- Primary loss reason (category)
- Detailed loss narrative
- MEDDPICC gaps identified
- Timeline of key events
- Lessons learned
- Recommendations for future similar deals

### Step 6: Generate Win/Loss Pattern Check

```bash
crm reports win-loss
```

Compare this loss to broader patterns:
- Is this loss reason becoming more common?
- Are we losing to the same competitor repeatedly?
- Is there a pattern by segment, territory, or deal size?

### Step 7: Present Summary

Provide a structured loss analysis:

1. **Deal Summary**
   - Opportunity name, account, ACV, stage at loss
   - Time in pipeline, number of stage transitions
   - Primary loss reason

2. **What Happened**
   - Narrative of the deal progression and loss
   - Key turning point(s)

3. **MEDDPICC Gaps**
   - Top 3 qualification gaps that contributed to the loss
   - When these gaps became apparent

4. **Lessons Learned**
   - What we should do differently next time
   - Process or methodology improvements
   - Specific coaching points

5. **Follow-Up Actions**
   - Is there a future opportunity? If so, when to re-engage
   - Should we update battlecards based on this loss?
   - Are there product feedback items to log?

## Example Prompts

- "We lost the Acme Corp deal to Fivetran, let me capture the analysis"
- "Mark opportunity 42 as Closed Lost"
- "Why did we lose deal 42? Help me document it"
- "/loss-analysis 42"
- "Lost deal analysis for opportunity 42"
- "The customer chose to stay with their current solution, help me record this"
