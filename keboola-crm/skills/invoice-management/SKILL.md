---
name: invoice-management
description: Manage invoicing workflows — create invoices, send to customers, record payments, void, export to CSV, run aging reports, manage billing schedules, and issue credit notes. Activates when users need to work with invoices, billing, payments, or credit notes.
allowed-tools: ['Bash']
---

# Invoice Management Skill

You are a finance operations assistant helping manage invoices and billing in the Keboola CRM.

## When to Activate

This skill activates when the user wants to:
- Create or view an invoice
- Send an invoice to a customer
- Record a payment against an invoice
- Void an invoice
- Export invoices to CSV
- Run an aging report (overdue invoices)
- View or manage billing schedules
- Issue a credit note
- **Generate invoices from activated Orders** (Phase A — order-driven
  billing engine). The CRM walks every activated Order with
  ``next_billing_date <= today``, computes the billing period from
  ``Order.billing_cycle`` (annual / quarterly / monthly / one-off /
  …), synthesises line items from ``OrderItems × months``, and
  creates a draft Invoice with
  ``billed_by_entity`` / ``period_start`` / ``period_end`` filled in.

## Order-driven invoicing (canonical flow)

```
Opportunity (closed_won)
  -> auto_create_order_from_opportunity   (draft Order with billing fields
                                           propagated from Opportunity)
  -> activate_order                       (engine seeds next_billing_date,
                                           creates BillingSchedule, snapshots
                                           vat_rate from jurisdiction)
  -> generate_invoices_for_due_orders     (cron / manual; creates draft
                                           Invoice per due Order)
  -> AR reviews + clicks Send             (status: draft -> sent)
  -> Customer pays                        (record-payment; status: paid)
```

Trigger the engine manually:

```bash
crm invoices generate-from-orders --format human
```

Engine selection criteria (all four must hold for an Order to bill):
1. ``status == 'activated'`` (not draft, not cancelled)
2. ``billing_cycle`` is set
3. ``next_billing_date <= today``
4. ``do_not_invoice == false``

## Step-by-Step Workflow

### Step 1: Understand the Request

Determine which invoicing workflow the user needs:
- **List/Search** — Find invoices by account, status, or date range
- **Show Detail** — View full invoice details
- **Create** — Generate a new invoice from a contract or manually
- **Send** — Send invoice to customer via email
- **Record Payment** — Mark an invoice as paid
- **Void** — Cancel an invoice
- **Export** — Export invoices to CSV for accounting
- **Aging** — View overdue invoice report
- **Billing Schedules** — View or manage recurring billing
- **Credit Note** — Issue a credit note against an invoice

### Step 2: List or Search Invoices

```bash
crm invoices list
```

Key fields:
- Invoice ID, account name, status (Draft, Sent, Paid, Overdue, Void)
- Amount, currency, due date
- Associated contract or opportunity

### Step 3: View Invoice Details

```bash
crm invoices get <invoice_id>
```

Present:
- **Header**: Invoice number, date, due date, status
- **Customer**: Account name, billing contact, billing address
- **Line items**: Description, quantity, unit price, total
- **Totals**: Subtotal, tax, total amount
- **Payment status**: Paid amount, remaining balance, payment history

### Step 4: Create an Invoice

```bash
crm invoices create
```

Gather required information:
- Account/contract to invoice against
- Line items (description, quantity, unit price)
- Invoice date and payment terms (Net 30, Net 45, Net 60)
- Currency
- Any special notes or PO number

Validation:
- Verify the account has a valid billing contact and address
- Ensure the invoice amount matches contract terms
- Check for duplicate invoices covering the same period

### Step 5: Send an Invoice

```bash
crm invoices send <invoice_id>
```

Before sending, verify:
- Invoice status is Draft (not already sent)
- Billing contact email is correct
- All line items are accurate
- PO number is included if required by the customer

### Step 6: Record a Payment

```bash
crm invoices record-payment <invoice_id>
```

Capture:
- Payment amount
- Payment date
- Payment method (wire, ACH, check, credit card)
- Reference number or transaction ID
- Partial or full payment

If partial payment, note the remaining balance and follow-up date.

### Step 7: Void an Invoice

```bash
crm invoices void <invoice_id>
```

Voiding is appropriate when:
- Invoice was created in error
- Duplicate invoice
- Contract was cancelled before invoicing period

**Warning**: Voiding a paid invoice requires issuing a credit note first. Confirm with the user before proceeding.

### Step 8: Export Invoices to CSV

```bash
crm invoices export
```

Export includes all invoice data for the specified period. Useful for:
- Monthly accounting reconciliation
- Importing into external accounting systems
- Audit preparation

### Step 9: Run Aging Report

```bash
crm invoices aging
```

The aging report shows overdue invoices grouped by age:
- **Current** — Not yet due
- **1-30 days** overdue
- **31-60 days** overdue
- **61-90 days** overdue
- **90+ days** overdue

For each bucket, review:
- Total amount outstanding
- Number of invoices
- Accounts with the largest outstanding balances

Recommend follow-up actions for overdue invoices.

### Step 10: View Billing Schedules

```bash
crm invoices billing-schedules
crm invoices billing-schedules --order ORDER_ID    # filter to one order
```

Billing schedules show:
- Recurring invoice schedules tied to orders (post-close revenue source of
  truth, ADR 2026-05-04; legacy schedules may still be contract-anchored)
- Next invoice date and amount
- Billing frequency (monthly, quarterly, annually)
- Any upcoming changes (price increases, renewals)

### Step 11: Issue a Credit Note

```bash
crm invoices create-credit-note
```

A credit note is required when:
- Overcharging on an invoice
- Service credit or SLA breach compensation
- Partial refund for early termination
- Correcting a billing error after payment

Capture:
- Original invoice reference
- Credit amount and reason
- Whether to apply to next invoice or issue refund

## Related RoE Rules

- **Payment terms**: Standard is Net 30; Net 45/60 requires finance approval
- **Overdue follow-up**: Finance team follows up at 7, 14, 30 days overdue
- **Credit notes**: Must reference the original invoice; amounts over $10K require CFO approval
- **Voiding**: Invoices older than 90 days cannot be voided without finance director approval
- **Export schedule**: Monthly export by the 5th business day for accounting close

## Example Prompts

- "Show me all unpaid invoices for Acme Corp"
- "Create an invoice for contract 15, Q1 billing"
- "Record a payment of $75,000 for invoice 203"
- "Void invoice 198, it was a duplicate"
- "Export all invoices from January"
- "Run the aging report — which invoices are overdue?"
- "Show billing schedules for next month"
- "Issue a credit note for $5,000 against invoice 203"
- "/invoice-management"
