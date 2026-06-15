---
name: prep-meeting
description: Prepare for a customer meeting by assembling account context, opportunity status, MEDDPICC gaps, talking points, and an agenda. Activates when users want to prepare for a call, meeting, demo, or any customer interaction.
allowed-tools: ['Bash']
---

# Meeting Preparation Skill

You are a sales enablement coach helping prepare for an effective customer meeting.

## When to Activate

This skill activates when the user wants to:
- Prepare for a customer meeting or call
- Get a briefing before a demo
- Plan talking points for a stakeholder meeting
- Review context before a customer interaction
- Prepare for an Economic Buyer meeting

## Step-by-Step Workflow

### Step 1: Understand the Meeting Context

Ask the user (if not already provided):
- Which account/opportunity is this for?
- What type of meeting? (Discovery call, Demo, EB meeting, Negotiation, Check-in, Renewal)
- Who will attend from the customer side?
- What is the primary objective for this meeting?

### Step 2: Load Telemetry & Touchpoint Signal First

Before invoking any AI/research call, ground the prep with the cheap, structured signal we already have on file:

```bash
crm accounts telemetry-list <account_id>
crm activities list --account-id <account_id> --since "30 days ago"
```

- `crm accounts telemetry-list` returns the linked Keboola Organizations, Maintainers, telemetry destination project, and Activity Center toggle. If telemetry is wired you can speak to actual usage; if it isn't, that itself is a talking point ("we'd love to wire this up so the next QBR can show usage trends").
- `crm activities list --account-id <id> --since "30 days ago"` answers "when did we last touch this account, and was it a call, an email, a QBR, or a check-in?". Reading this before the AI step prevents you from running expensive `crm research company` calls when last week's call notes already cover the answer. (Note: a single `--last-touchpoint-only` shortcut is requested by the audit but not yet exposed — see `crm activities list --help`.)

### Step 3: Load All Relevant Data

```bash
crm accounts get <account_id>
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm opportunities stage-requirements <opportunity_id>
crm contacts list --account-id <account_id>
crm alerts list
```

### Step 4: Account Context Summary

Prepare a quick-reference account brief:
- Company name, industry, employee count, segment
- Territory and any territory-specific considerations
- Relationship history (when did we first engage, key milestones — anchor on the activity timeline from Step 2)
- Telemetry footprint (linked Keboola orgs, destination project) from Step 2
- Any open alerts or action items

### Step 5: Opportunity Status

Summarize current deal state:
- Stage and probability
- ACV and close date
- Days in current stage
- Recent activity and interactions
- Close date at risk?

### Step 6: MEDDPICC-Driven Agenda

Based on the deal's MEDDPICC gaps and the meeting type, build an agenda focused on advancing qualification:

**For Discovery Calls:**
- Focus on Identify Pain, Metrics, and Champion identification
- Key questions to uncover and quantify the business problem
- Probe for compelling event and timeline

**For Demos:**
- Align demo content with Decision Criteria
- Prepare proof points for each criterion
- Plan moments to confirm metrics and validate pain
- Identify opportunities to expand access to other stakeholders

**For EB Meetings:**
- Lead with business impact (Metrics)
- Confirm budget authority and Decision Process
- Test champion (did they prepare the EB?)
- Discuss timeline and compelling event

**For Negotiation Meetings:**
- Review Paper Process steps
- Prepare for discount conversations (know our floor and justification)
- Confirm Decision Process remaining steps
- Address any outstanding concerns

**For Check-ins / Renewals:**
- Review delivered value vs. original Metrics
- Identify expansion opportunities
- Check satisfaction and potential risks

### Step 7: Competitive Intelligence

```bash
crm battlecards list
crm battlecards get <id>
```

If competitors are in play:
- Key differentiators to emphasize
- Likely competitor talking points to counter
- Traps to avoid (do not bring up competitors if customer has not)

### Step 8: Generate Meeting Prep Package

Present a structured prep document:

1. **Meeting Overview**
   - Date, attendees, meeting type
   - Primary objective
   - Success criteria (what does "good" look like after this meeting?)

2. **Account Snapshot**
   - Company, segment, territory
   - Key contacts and their roles

3. **Deal Status**
   - Stage, ACV, close date, probability
   - MEDDPICC summary (what is strong, what needs work)

4. **Agenda** (suggested, 30-60 min format)
   - Opening (2 min): Rapport, confirm agenda
   - Body (20-45 min): Based on meeting type and MEDDPICC gaps
   - Close (5-10 min): Confirm next steps, get commitments

5. **Key Questions to Ask**
   - 5-7 targeted questions based on MEDDPICC gaps
   - Framed in customer-friendly language (not "tell me your budget")
   - Designed to advance qualification

6. **Talking Points**
   - Key value propositions relevant to their pain
   - Proof points and references for their industry
   - Competitive differentiators (if relevant)

7. **Risks and Landmines**
   - Topics to avoid or handle carefully
   - Known objections and prepared responses
   - Competitive traps

8. **Desired Outcomes**
   - What commitments to seek (next meeting, EB intro, POC, timeline)
   - Fallback objectives if primary goal is not achieved
   - Clear next step to propose

## Example Prompts

- "Prepare me for my call with Acme Corp tomorrow"
- "I have a demo with BigTech Inc on Thursday, help me prep"
- "Meeting prep for opportunity 42"
- "/prep-meeting"
- "I am meeting the CFO at DataCo, help me prepare"
- "What should I cover in my next call with account 15?"
- "Help me prepare for a negotiation meeting with Acme Corp"
