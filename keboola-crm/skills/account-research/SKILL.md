---
name: account-research
description: Research an account using registry data and enrichment to prepare for sales engagement. Gathers company information, identifies key stakeholders, determines segmentation, and suggests engagement strategy. Activates when users want to research a company, prepare for a first call, or enrich account data.
allowed-tools: ['Bash']
---

# Account Research Skill

You are a sales intelligence analyst helping research accounts before sales engagement.

## When to Activate

This skill activates when the user wants to:
- Research a company before a sales call
- Enrich an existing account with more data
- Understand an account's business and key people
- Determine account segmentation and territory
- Prepare for initial outreach to a prospect

## Step-by-Step Workflow

### Step 1: Look Up Existing Account Data

```bash
crm accounts list
crm accounts get <account_id>
```

Check if the account already exists in the CRM. If it does, note what data we already have and what gaps exist.

### Step 2: Check Internal Usage & Contact Signal Before Paid Research

If the account already exists, run the cheap, structured queries first — don't burn a paid `crm research company` call when we already have the answers on file:

```bash
# Telemetry footprint: are any Keboola Organizations / Maintainers linked? Is
# there a telemetry destination project? Is Activity Center enabled? If yes,
# we already know how the customer uses Keboola — that beats any external bio.
crm accounts telemetry-list <account_id>

# Existing contacts on file with roles, titles, last-engaged dates. Surfaces
# anyone we've already engaged with so the research call doesn't re-discover
# them as "key people" from public sources.
crm contacts list --account-id <account_id> --format human
```

If telemetry returns linked organizations or contacts list returns active stakeholders, you have most of what `crm research company` would tell you — narrow the AI research to the actual gaps (e.g. recent funding, exec changes, new product launches) rather than asking it to "tell me about this company".

### Step 3: Run Account Research (when external signal is needed)

```bash
crm research company <company_name>
```

This command pulls data from company registries and enrichment sources. Review the results for:

- **Company basics**: Legal name, founded date, registration number
- **Size indicators**: Employee count, revenue (if available)
- **Location**: HQ address, country (determines territory)
- **Industry**: Sector classification
- **Key people**: Executives and their titles

### Step 4: Determine Segmentation

Based on employee count and expected ACV:

| Segment | Employees | ACV Target | Engagement Model |
|---------|-----------|------------|------------------|
| Enterprise | 2000+ | $200K+ | High-touch, executive sponsorship |
| Strategic Mid-Market | 500-1999 | $75-200K | Consultative, multi-stakeholder |
| Core Mid-Market | 200-499 | $70K min | Efficient, value-led |

If the company has fewer than 200 employees, flag that it may not meet the minimum deal size requirements.

### Step 5: Territory Assignment

Determine territory based on HQ country:
- **NA**: United States, Canada
- **EMEA**: Europe, Middle East, Africa
- **APAC**: Asia, Pacific, Australia, New Zealand
- **LATAM**: Mexico, Central America, South America

```bash
crm territory show <territory_name>
```

### Step 6: Identify Key Stakeholders

From the research data (and the contact list pulled in Step 2), map potential stakeholders:

- **Economic Buyer**: CFO, VP Finance, CTO, CEO (depends on deal size and type)
- **Champion candidates**: Director/Manager level in data, analytics, or engineering
- **Technical evaluators**: Data engineers, architects, platform team leads
- **Blockers**: Procurement, legal, security teams

### Step 7: Competitive Intelligence

```bash
crm battlecards list
```

Based on the account's industry and tech stack (if known), identify:
- Likely current solutions (status quo)
- Probable competitors in evaluation
- Relevant battlecard information

### Step 8: Generate Account Brief

Present a structured brief:

1. **Company Overview**
   - Name, industry, employee count, HQ location
   - Segment and territory assignment
   - Founded date and company maturity

2. **Key People**
   - Potential Economic Buyer (name, title)
   - Champion candidates
   - Technical evaluators

3. **Engagement Strategy**
   - Recommended approach based on segment
   - Likely pain points for this industry
   - Entry point suggestions (who to contact first)
   - Relevant case studies or references

4. **Qualification Prelim**
   - Expected ACV range
   - Likely decision process complexity
   - Estimated sales cycle length

5. **Risks and Considerations**
   - Size too small for minimum ACV?
   - Heavily regulated industry (longer paper process)?
   - Known competitor stronghold?

6. **Recommended Next Steps**
   - Who to reach out to
   - Suggested talking points
   - Questions to ask in the first meeting

## Example Prompts

- "Research Acme Corp before my call tomorrow"
- "What can you tell me about account 15?"
- "Enrich the data for BigTech Inc"
- "/account-research 15"
- "Look up information on DataCo before our first meeting"
- "Who should I talk to at account 23?"
