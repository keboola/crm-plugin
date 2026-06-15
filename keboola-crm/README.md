# Keboola CRM Plugin for Claude Code

A Claude Code plugin that helps sales reps, managers, and CS teams manage their CRM workflow using the Keboola CRM CLI. Provides role-based and task-based skills covering the full sales cycle from account research through deal close, invoicing, commissions, and revenue analytics. (A few contributor-only skills used when developing the CRM monorepo itself — e.g. `/architecture-brief` — are excluded from the public marketplace build.)

## Available Skills

### Role-Based Skills

| Skill | Command | Description |
|-------|---------|-------------|
| `sales` | `/sales` | Sales rep workflow — meetings, pipeline, MEDDPICC, deal closing |
| `cs` | `/cs` | Customer success — renewals, health monitoring, QBR prep |
| `manager` | `/manager` | Sales manager — team pipeline, quotas, approvals, commissions |

### Task-Based Skills

| Skill | Command | Description |
|-------|---------|-------------|
| `new-opportunity` | `/new-opportunity` | Create and qualify a new opportunity with MEDDPICC |
| `meddpicc-review` | `/meddpicc-review` | Review and improve MEDDPICC qualification scores |
| `stage-transition` | `/stage-transition` | Advance opportunity through sales stages |
| `pipeline-review` | `/pipeline-review` | Review pipeline health and forecast accuracy |
| `account-research` | `/account-research` | Research account using registry + enrichment |
| `deal-review` | `/deal-review` | Comprehensive deal review before stage gate |
| `loss-analysis` | `/loss-analysis` | Capture structured loss analysis for lost deals |
| `prep-meeting` | `/prep-meeting` | Prepare for customer meeting with full context |
| `contract-management` | `/contract-management` | Manage contracts — create, renew, amend, history chain, bookings |
| `invoice-management` | `/invoice-management` | Manage invoices — create, send, payments, aging, credit notes |
| `commission-management` | `/commission-management` | Manage commissions — plans, earnings, statements, clawbacks |
| `quota-attainment` | `/quota-attainment` | Track quota attainment and pipeline coverage |
| `territory-management` | `/territory-management` | Manage territories — assignments, reassignment, auto-assign |
| `competitive-intel` | `/competitive-intel` | Competitive intelligence — battlecards, pre-call briefs |
| `discount-approval` | `/discount-approval` | Discount request and approval workflow with thresholds |
| `account-enrichment` | `/account-enrichment` | Enrich accounts — Perplexity research, registry search, firmographics |
| `revenue-analytics` | `/revenue-analytics` | Revenue analytics — ARR, waterfall, NRR, cohorts, dashboard |
| `alerts-triage` | `/alerts-triage` | Triage alerts — list, acknowledge, summary, daily routine |
| `log-activity` | `/log-activity` | Capture a great activity record — log/edit a meeting or call, record people met (`--met`, auto-links to contacts), set the real date (`--at`), correct in place, link attendees |
| `gong-transcript-import` | `/gong-transcript-import` | Import call transcripts — Gong / Plaud / manual, bulk import, MEDDPICC extraction |
| `architecture-brief` | `/architecture-brief` | Engineering, **monorepo only** (excluded from the public marketplace build): produce a structured brief (affected modules, files to open, cross-cutting concerns, related ADRs, testing strategy) for a CRM coding task. Pass an issue number or module name. |

## Prerequisites

- [Claude Code](https://claude.ai/code) installed
- Access to a Keboola CRM instance

## Installation

### 1. Install the CRM CLI

```bash
uv tool install <crm-app-url>/downloads/keboola_crm_cli-<VERSION>-py3-none-any.whl
```

### 2. Configure the CLI

```bash
crm config init --api-url <your-crm-api-url> --token <your-pat-token>
```

To obtain a PAT token, ask your CRM admin or generate one from the CRM web UI under Settings > API Tokens.

Verify the connection:

```bash
crm doctor
```

### 3. Install the Plugin

**Recommended: Install via Claude Code marketplace**

In Claude Code, run:
```
/plugin marketplace add keboola/crm-plugin
/plugin install keboola-crm
```

`keboola/crm-plugin` is the **public** distribution repo, auto-synced from
this monorepo on every CRM release — so customers without access to the
private `keboola/crm` repo can install it. Pick up a newer version with:
```
/plugin marketplace update
/plugin update keboola-crm
```

**Alternative: Test locally without install** (contributors, from a monorepo checkout)

```bash
curl -sL <crm-app-url>/downloads/keboola-crm-plugin.tar.gz | tar xz
claude --plugin-dir ./plugins/keboola-crm
```

### 4. Verify

In Claude Code, type `/keboola-crm:new-opportunity` or `/keboola-crm:pipeline-review`.

All skills are namespaced: `/keboola-crm:<skill-name>`.

## Usage Examples

```
# Create a new opportunity
/new-opportunity Acme Corp wants to replace their legacy ETL with Keboola, ~$150K ACV

# Review MEDDPICC for a deal
/meddpicc-review Check the qualification for opportunity 42

# Advance a deal stage
/stage-transition Move opportunity 42 from Discovery to Demo

# Pipeline review
/pipeline-review Show me the Q2 forecast and pipeline gaps

# Research an account
/account-research Research Acme Corp before our first meeting

# Deal review before stage gate
/deal-review Full review of opportunity 42 before Demo stage

# Capture loss analysis
/loss-analysis Opportunity 42 was lost to Fivetran

# Prepare for meeting
/prep-meeting I have a call with Acme Corp tomorrow at 2pm

# Contract management
/contract-management Renew contract 15 with a 10% price increase

# Invoice management
/invoice-management Show unpaid invoices for Acme Corp

# Commission tracking
/commission-management How much have I earned this quarter?

# Quota attainment
/quota-attainment Am I on track to hit my number?

# Territory management
/territory-management Auto-assign the new account — HQ is in Germany

# Competitive intelligence
/competitive-intel Show me the Fivetran battlecard

# Discount approval
/discount-approval Request 15% discount on opportunity 42

# Account enrichment
/account-enrichment Research Acme Corp before our first meeting

# Revenue analytics
/revenue-analytics Show me current ARR by segment

# Alert triage
/alerts-triage Morning triage — what needs my attention today?
```

## Sales Methodology Reference

This plugin enforces Keboola's Rules of Engagement:

- **Stages**: Discovery (20%) -> Demo (30%) -> POC (60%, optional) -> Offer/Negotiation (90%) -> Closed Won (100%) / Closed Lost (0%) — probabilities are config-driven (`stage_probabilities`)
- **Qualification**: Full MEDDPICC framework required for stage progression
- **Minimum ACV**: $70K for Core Mid-Market, $75K for Strategic, $200K for Enterprise
- **Territories**: NA, EMEA, APAC, LATAM (assigned by HQ country)

## License

MIT
