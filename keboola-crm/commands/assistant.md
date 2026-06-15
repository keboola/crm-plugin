---
description: 'Spawn the crm-assistant subagent — a conversational Keboola CRM operator that bootstraps identity (`crm doctor`, `crm auth me`), picks the matching role persona (rep→sales, manager/vp→manager, csm→cs), loads the relevant skill playbook(s) into its own context via Read, and drives the `crm` CLI end-to-end for the request. Read-mostly; confirms every state-changing command and never invents IDs. Use for one-shot CRM tasks ("prep me for the Acme call", "what should I work on today", "advance the Globex deal", "who is at renewal risk"). Post-close revenue is Orders, not the removed Contracts.'
allowed-tools: Task
argument-hint: [what you want done, e.g. "prep me for the Acme renewal call"]
---

# /keboola-crm:assistant — one-shot CRM operator

Spawn the `crm-assistant` subagent to carry a CRM request to a useful
result in a single autonomous pass. It runs in a fresh context window,
so the operating rules (bootstrap identity, ground IDs, confirm
state-changing commands, Orders-not-Contracts) survive regardless of
what the parent conversation was doing.

## Behavior

1. Pass `$ARGUMENTS` to the `crm-assistant` subagent as the task. If
   `$ARGUMENTS` is empty, the subagent bootstraps identity and proposes
   a role-appropriate starting point (e.g. today's pipeline for a rep,
   the approval queue for a manager, at-risk renewals for a CSM).
2. The subagent will:
   - run `crm doctor` + `crm auth me` to confirm config and learn the
     acting user's role;
   - select the persona skill (sales / manager / cs) and any task skills
     the request needs, and `Read` their `SKILL.md` playbooks;
   - resolve any record IDs by search before acting;
   - run read commands freely, but **stop and confirm** before any
     state-changing or notifying command.
3. Relay the subagent's summary: what it found, what it ran, what
   changed, and the recommended next step.

The subagent is read-mostly and never invents API URLs, tokens, or
record IDs. If the CLI is unconfigured it reports exactly what is
missing rather than guessing.
