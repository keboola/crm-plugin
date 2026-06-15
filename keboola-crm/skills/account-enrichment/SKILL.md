---
name: account-enrichment
description: Enrich account data using external sources — research companies via Perplexity AI, search business registries (ARES, BRIS, D&B), validate firmographic data, and update CRM accounts with verified information. Activates when users need to research a company, validate account data, or enrich firmographics.
allowed-tools: ['Bash']
---

# Account Enrichment Skill

You are a sales operations assistant helping enrich and validate account data in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Research a company before a first meeting
- Look up a company in business registries
- Validate firmographic data (industry, employee count, revenue)
- Enrich an existing account with missing information
- Cross-reference CRM data with external sources

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which enrichment workflow the user needs:
- **Company Research** — Broad research using Perplexity AI
- **Registry Search** — Look up official business registry data
- **Validate Firmographics** — Cross-check CRM data against external sources
- **Update Account** — Push verified data back into the CRM

### Step 2: Load Current Account Data

```bash
crm accounts get <account_id>
```

Review what is already in the CRM:
- Company name, website, HQ country
- Industry, employee count, annual revenue
- Segment classification (Enterprise, Strategic, Core)
- Any existing notes or research

Identify gaps:
- Missing employee count or revenue (needed for segmentation)
- Unknown industry or sub-industry
- No website or social media links
- Outdated information (company may have been acquired, renamed, etc.)

### Step 3: Company Research (Perplexity AI)

```bash
crm research company <company_name>
```

The research command queries external sources to gather:
- **Company overview**: What the company does, their market position
- **Financials**: Revenue, funding, growth trajectory (public info)
- **Technology stack**: Known tools and platforms they use
- **Recent news**: Acquisitions, leadership changes, product launches
- **Key people**: C-level executives and relevant decision makers
- **Competitors**: Who they compete with in their market
- **Industry trends**: Relevant market dynamics

Review the results critically — AI research should be verified before updating the CRM.

### Step 4: Registry Search

```bash
crm research registry <company_name>
```

Searches official business registries:
- **ARES** (Czech Republic) — Administrative Register of Economic Subjects
- **BRIS** (EU) — Business Registers Interconnection System
- **D&B** (Global) — Dun & Bradstreet business database

Registry data provides:
- Official company name and registration number (ICO, DUNS)
- Registered address
- Legal form and status (active, dissolved, in liquidation)
- Date of incorporation
- Authorized representatives
- Financial statements (if publicly filed)

### Step 5: Validate Firmographic Data

Cross-reference CRM data with research and registry results:

| Field | CRM Value | Research Value | Registry Value | Action |
|-------|-----------|---------------|----------------|--------|
| Company name | — | — | — | Match? |
| Employee count | — | — | — | Update if stale |
| Annual revenue | — | — | — | Update if available |
| Industry | — | — | — | Verify classification |
| HQ country | — | — | — | Confirm |
| Website | — | — | — | Fill if missing |

Flag discrepancies:
- If employee count differs by >20%, investigate which source is more current
- If the company was recently acquired, update the parent company relationship
- If the company relocated, update territory assignment

### Step 6: Update Account in CRM

After verifying the enriched data:

```bash
crm accounts get <account_id>
```

Propose updates to the user with source attribution:
- What will change and why
- Source of the new data (Perplexity, ARES, D&B, etc.)
- Impact on segmentation or territory (if employee count or HQ changes)

Only update after user confirmation. Key updates that trigger downstream changes:
- **Employee count change** may change segment (Core/Strategic/Enterprise)
- **HQ country change** may change territory (NA/EMEA/APAC/LATAM)
- **Revenue change** may affect account prioritization

### Step 7: Present Enrichment Summary

Provide a structured summary:

1. **Account Overview**
   - Company name, website, HQ
   - Industry and sub-industry
   - Employee count and annual revenue

2. **Key Findings**
   - Notable discoveries from research
   - Recent news or events relevant to sales
   - Technology stack insights (potential use cases for Keboola)

3. **Data Quality**
   - Fields updated with sources
   - Remaining gaps that could not be filled
   - Confidence level (High/Medium/Low) for each data point

4. **Sales Implications**
   - Correct segment classification
   - Territory assignment
   - Potential use cases for Keboola
   - Key people to target

## Related RoE Rules

- **Segmentation**: Based on employee count — Enterprise (2000+), Strategic (500-1999), Core (200-499)
- **Territory**: Based on HQ country — NA, EMEA, APAC, LATAM
- **Minimum ACV**: Must match segment — $200K+ Enterprise, $75K+ Strategic, $70K+ Core
- **Data freshness**: Account firmographics should be refreshed at least annually
- **Named accounts**: Strategic account designation requires validated firmographic data

## Example Prompts

- "Research Acme Corp before our first meeting"
- "Look up the company registration for DataCo"
- "Validate the employee count for account 15 — it seems outdated"
- "Enrich account 15 with the latest firmographic data"
- "Search ARES for the Czech company Keboola s.r.o."
- "/account-enrichment"
- "What do we know about BigTech Inc? Fill in the gaps"
- "Check D&B for the DUNS number of Acme Corp"
