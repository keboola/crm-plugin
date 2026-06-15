---
name: stage-transition
description: Advance a sales opportunity through the sales stages (Discovery -> Demo -> Offer/Negotiation -> Closed Won). Validates all stage exit criteria, checks MEDDPICC completeness, and guides the rep through any gaps before allowing progression. Activates when users want to move a deal to the next stage, check stage readiness, or advance a deal.
allowed-tools: ['Bash']
---

# Stage Transition Skill

You are a sales process gatekeeper helping advance opportunities through the Keboola sales stages while enforcing stage exit criteria.

## When to Activate

This skill activates when the user wants to:
- Move a deal to the next stage
- Check if a deal is ready for stage advancement
- Understand what is needed for the next stage
- Advance or progress an opportunity

## Sales Stage Flow

```
Discovery (20%) --> Demo (30%) --> POC (60%, optional) --> Offer/Negotiation (90%) --> Closed Won (100%)
                                                                                   \-> Closed Lost (0%)
```

Probabilities are config-driven (`stage_probabilities` via `/api/config/public`);
the numbers above are the defaults at the time of writing (#910).

## Step-by-Step Workflow

### Step 1: Load Current State

```bash
crm opportunities get <opportunity_id>
crm opportunities qualify <opportunity_id>
crm opportunities stage-requirements <opportunity_id>
```

Identify the current stage and target stage.

### Step 2: Validate Stage Exit Criteria

#### Discovery -> Demo

All of the following must be met:

- [ ] **MEDDPICC components started** — All 8 components have at least initial entries
- [ ] **Minimum ARR**: $70K+ identified and documented
- [ ] **Compelling event**: Identified with a specific timeline (e.g., "Contract with current vendor expires Sept 30")
- [ ] **Champion identified**: Named person with title, initial engagement documented
- [ ] **Technical fit confirmed**: Customer's use case aligns with Keboola capabilities
- [ ] **Pain identified**: Specific problem documented in customer's language (250+ chars — the configured `stage_pain_statement_min_chars` gate)
- [ ] **Next steps clear**: Demo agenda agreed with customer

#### Demo -> Offer/Negotiation

All of the following must be met:

- [ ] **All MEDDPICC validated**: Each component validated with customer stakeholders (not just internal assumptions)
- [ ] **Economic Buyer engaged**: EB has been in a meeting or has scheduled one
- [ ] **Champion tested**: Champion was asked to do something and delivered (e.g., arranged EB meeting, shared internal docs)
- [ ] **Decision criteria confirmed**: 3+ criteria documented and Keboola positioned favorably on top priorities
- [ ] **Decision process stakeholders assigned**: Each decision process step should have stakeholders assigned. Use `crm opportunities add-decision-step <opp_id> --stakeholders <contact_id1,contact_id2>`
- [ ] **Metrics validation date**: Must not be older than 30 days. Stale validation dates block the transition.
- [ ] **Technical validation completed**: POC, demo, or technical deep-dive completed successfully
- [ ] **Competition mapped**: All alternatives documented with our positioning
- [ ] **Commercial discussion started**: Customer aware of general pricing range

#### Offer/Negotiation -> Closed Won

All of the following must be met:

- [ ] **All MEDDPICC confirmed with EB**: Economic Buyer has confirmed decision criteria, timeline, and budget
- [ ] **Paper process mapped**: Legal, procurement, security steps documented with timeline
- [ ] **Legal/security review**: Completed or timeline agreed with specific dates
- [ ] **Commercial terms agreed**: Pricing, discount (if any), contract length agreed
- [ ] **Contract signed**: Final contract executed

```bash
# For Offer -> Closed Won, also run:
crm opportunities checklist <opportunity_id>
```

### Step 3: Identify Gaps

For each unmet criterion:
1. Explain what is missing
2. Suggest specific actions to address it
3. Provide recommended questions for the next customer interaction
4. Estimate effort/time to close the gap

### Step 4: Gate Decision

Based on the review, recommend one of:

- **READY** — All criteria met, proceed with stage transition
- **CONDITIONAL** — Minor gaps that can be addressed quickly (1-2 items), proceed with a plan
- **NOT READY** — Significant gaps remain, do not advance

If READY or CONDITIONAL:

```bash
crm opportunities update-stage <opportunity_id> <stage>
```

If NOT READY, provide a prioritized action plan with timeline.

### Step 5: Post-Transition Verification

```bash
crm opportunities get <opportunity_id>
```

Confirm the stage and probability have been updated correctly (defaults of
the config-driven `stage_probabilities`, #910):
- Discovery -> Demo: probability should be 30%
- Demo -> POC: probability should be 60%
- POC (or Demo) -> Offer: probability should be 90%
- Offer -> Closed Won: probability should be 100%

Provide next steps for the new stage.

## Special Cases

### Regression (Moving Backward)

If the user wants to move a deal backward (e.g., Demo back to Discovery):
- Ask for the reason
- Document the regression reason
- This is acceptable but should be rare and well-justified

### Closed Lost

If moving to Closed Lost at any stage:
- Trigger the loss-analysis skill workflow
- Ensure loss reason is captured
- Document lessons learned

### Discount Requests

If commercial terms require a discount:

```bash
crm discounts request <opportunity_id> <percent> <justification>
```

Document justification and get approval before finalizing offer.

## Example Prompts

- "Move opportunity 42 from Discovery to Demo"
- "Is deal 42 ready for the next stage?"
- "What do I need to advance the Acme Corp deal?"
- "/stage-transition 42"
- "Advance opportunity 42 to Offer stage"
- "Check stage readiness for deal 42"
