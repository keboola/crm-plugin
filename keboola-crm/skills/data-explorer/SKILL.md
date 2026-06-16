---
name: data-explorer
description: Build ad-hoc data-exploration views and dashboards over CRM data using natural language. Activates when the user asks to filter, slice, highlight, chart, aggregate, or compose multiple views into a dashboard (e.g. "show opportunities owned by X with stage Demo and highlight Activity Center", "give me a pipeline-by-stage chart", "build a CRO dashboard with KPIs and charts"). Translates the request into typed JSON and calls `crm views` / `crm dashboards`.
allowed-tools: ['Bash', 'Read']
---

# Data Explorer Skill

You build typed JSON specs over the CRM saved-views and dashboards APIs
on the user's behalf. The user describes what they want to see in
natural language; you fetch the catalog, draft the spec, hand it to the
``crm views`` / ``crm dashboards`` CLI, and return URLs.

## When to activate

The user wants to:

- See a custom slice of the data ("opportunities owned by X")
- Highlight rows matching a condition ("flag Activity Center customers")
- Chart something ("pipeline by stage", "ARR by quarter") — bar / line /
  donut / KPI
- Pin a view or a dashboard to their menu
- Compose multiple views into a single dashboard ("daily standup view")
- Iterate on an existing artefact ("make the highlight rose", "add a
  KPI for total ARR to that dashboard")
- Delete an artefact

If the user asks something that does not require persisting an artefact
("what is the average deal size right now?") — answer directly without
this skill.

## Inputs you must read (Phase 4 — three sources of truth)

Always re-fetch these at the start of every turn; do not cache between
turns. Together they answer: *what fields exist, what real values
look like, and how big the dataset is.*

### 1. Catalog — what's queryable

```bash
crm views catalog --format json | jq
```

Returns:
* **entities** — read them from the catalog; do not assume a fixed set
  (it widens server-side). Today it serves ``opportunity``, ``account``,
  ``contact``, ``order``, and ``order_item``. Each field carries
  ``type``, ``operators``, ``groupable``, ``aggregatable`` flags, and for
  enums an ``enum_values``
  list so you don't have to guess "is it ``offer_negotiation`` or
  ``Offer/Negotiation``?". A few fields worth knowing:
  * ``owner.full_name`` — the owner's display name (on a post-sale
    ``account`` that's the CSM). Use it for display columns and chart
    group-by to get readable axes instead of raw SFIDs. *Filtering* by it
    is post-fetch, so to count exactly on a large table filter by
    ``owner_id`` instead.
  * ``account`` ``id`` with ``in`` — filtering an account view by a set
    of SFIDs runs at the SQL level (before pagination), so a ``table``
    total matches the identical KPI/chart regardless of sort or limit.
  * ``order_item`` ``arr`` — the per-line-item entity backs per-product /
    per-client ARR roll-ups (e.g. a Snowflake reseller spans an
    ``SNFLK-01`` + ``SNFLK-02`` line). ``arr`` is the annualized figure
    (recurring × 12, discounts negative, one-off 0 — Order ARR semantics,
    #924), ``aggregatable`` and sortable. *Filtering* ``arr`` is post-fetch
    (``eq``/``neq`` only) — same gotcha as ``owner.full_name`` above — so
    for an exact per-client total, narrow at the SQL level first
    (``order_status`` ``eq`` ``activated``, ``product_code`` ``in``
    ``[SNFLK-01, SNFLK-02]``, or ``account_id``), then ``sum`` ``arr``
    grouped by ``account_name``.
* **relationships** — explicit FK edges the catalog declares per entity
  (e.g. ``opportunity.account_id → account``,
  ``opportunity.owner_id → user``, ``account.owner_id → user``). Use
  ``<relationship>.<field>`` paths (e.g. ``account.industry``,
  ``owner.full_name``) to read across an edge in filters / columns /
  group_by. Read the live catalog for the full set per entity; don't
  hardcode it.
* **highlight_palette** — the 5 keys you may pass to ``highlight.style``.
* **placeholders** — currently just ``__current_user__``.

### 2. Samples — what real values look like

```bash
curl -s "$BASE_URL/api/views/samples?entity=opportunity&n=10" | jq
```

Returns 10 RLS-checked recent rows with the analytics-relevant fields
filled in (no PII, no MEDDPICC narratives). Use this to ground enum
values: if the catalog says ``account.industry`` is a free-text
string, samples will tell you the actual industry strings in the data
("Internet", "Manufacturing", …) so your spec doesn't filter for
``"saas"`` when the data has ``"SaaS"``.

### 3. Stats — how big is the haystack

```bash
curl -s "$BASE_URL/api/views/stats?entity=opportunity" | jq
```

Returns ``total``, counts by stage, counts by opportunity_type, top-10
owners, and an ``amount`` distribution (``count``, ``sum``, ``median``,
``p25``, ``p75``, ``p90``, ``max``). Use this to:

* Avoid proposing a chart that would have one bucket. ("group by
  ``account.industry``" — stats will tell you if there are 30
  industries or 3.)
* Set realistic ``limit`` values. (3,485 opps total → ``limit: 100``
  is fine for a list view, ``limit: 500`` for a chart.)
* Tell the user useful framing in your reply: "you've got 991 closed
  won and 2,270 closed lost — your win rate is ~30%. Want me to chart
  win rate by stage or by owner?"

## Workflow A — single saved view

### Step 1 — Restate and ground

Repeat the user's request back, mapping each phrase to a catalog field.
If a concept does not exist in the catalog (e.g. "lead source" is not in
Phase 1), **ask before substituting**. Silent substitution erodes trust.

### Step 2 — Build a JSON file

Compose a ``SavedViewCreateRequest`` matching the schema below.

```json
{
  "title": "LinkedIn opps · Balu (Activity Center)",
  "description": "Demo deals owned by Balu, highlighting accounts with Activity Center on.",
  "pin_mode": "literal",
  "pinned": false,
  "spec": {
    "entity": "opportunity",
    "view": "table",
    "filter": {
      "op": "and",
      "clauses": [
        {"field": "owner_id", "op": "eq", "value": "005KBSALES1234567"},
        {"field": "stage",    "op": "eq", "value": "demo"}
      ]
    },
    "columns": ["name", "stage", "amount", "account.name", "account.activity_center_enabled"],
    "sort": [{"field": "amount", "direction": "desc"}],
    "highlight": [
      {
        "when": {"field": "account.activity_center_enabled", "op": "eq", "value": true},
        "style": "row-amber"
      }
    ],
    "limit": 100
  }
}
```

For chart / KPI views, use ``view`` of ``bar_chart`` / ``line_chart`` /
``donut`` / ``kpi`` and add ``group_by`` + ``aggregations``:

```json
{
  "title": "Pipeline · Deals by stage",
  "spec": {
    "entity": "opportunity",
    "view": "bar_chart",
    "filter": {"op": "and", "clauses": []},
    "columns": [], "sort": [], "highlight": [],
    "group_by": ["stage"],
    "aggregations": [{"op": "count", "alias": "Deals"}],
    "limit": 500
  }
}
```

KPI rules: zero ``group_by`` + exactly one aggregation. Charts: exactly
one ``group_by`` + one or more aggregations. Tables ignore both.

#### Date windows, time-series, and week-over-week

The catalog (`crm views catalog`) is the source of truth — read it for the
exact `date_presets`, `time_grains`, and which fields are `range_filterable`
(can back a window) / `time_groupable` (can be a time axis). Do not hardcode
those lists; they can widen server-side.

* **`date_range`** — a server-resolved window on one date field, pushed to
  SQL so KPI/chart totals cover the *whole* match set (not one page). Use a
  preset (resolved at render time, so a pinned tile stays current) or a custom
  `start`/`end`:

  ```json
  "date_range": {"field": "created_at", "preset": "this_quarter"}
  "date_range": {"field": "close_date", "preset": "custom", "start": "2026-04-01", "end": "2026-06-30"}
  ```

* **`time_grain`** (`day`/`week`/`month`/`quarter`) — bucket a date
  `group_by` into a time series. "Opportunities created per week":

  ```json
  {"entity": "opportunity", "view": "line_chart",
   "date_range": {"field": "created_at", "preset": "last_90_days"},
   "group_by": ["created_at"], "time_grain": "week",
   "aggregations": [{"op": "count", "alias": "Created"}],
   "columns": [], "sort": [], "highlight": [], "limit": 500}
  ```

* **`view: "kpi_comparison"`** — one metric over a current window vs the
  immediately-preceding one, with delta + trend (week-over-week, QoQ). Requires
  a `date_range` and exactly one aggregation, no `group_by`:

  ```json
  {"entity": "opportunity", "view": "kpi_comparison",
   "date_range": {"field": "created_at", "preset": "last_7_days"},
   "aggregations": [{"op": "count", "alias": "New deals"}],
   "columns": [], "sort": [], "highlight": [], "limit": 100}
  ```

  The previous window is derived per preset: rolling presets (`last_7_days`)
  compare against the trailing equal span; calendar presets (`this_quarter`)
  compare against the prior calendar unit. One metric per card — compose two
  tiles for count + amount side by side.

Dashboards can carry a board-wide `date_range` (a `DateWindowSpec` — `preset`
or custom `start`/`end`, no field). Pass it as the `date_range` key in the
`crm dashboards create` / `compose` payload, or PATCH it via
`crm dashboards update` (`{"date_range": {"preset": "this_quarter"}}`, or
`{"clear_date_range": true}` to drop it). At render it re-windows every tile
declaring its own `date_range`: the board sets the *when*, each tile keeps its
*field*. Flip a whole board from one quarter to the next in one call.

### Step 3 — Create

```bash
cat <<'JSON' | crm views create --from-file -
{ ... your payload ... }
JSON
```

Or save to a file and ``crm views create --from-file my_view.json
--pin --pin-mode dynamic``.

### Step 4 — Iterate

```bash
crm views show $ID --format json | jq .spec > current_spec.json
# Mutate the file, then:
crm views show $ID --format json | jq '.spec = '"$(cat current_spec.json)" \
  | jq '{spec: .spec, change_summary: "tweak"}' \
  | curl -s -X PATCH "$BASE_URL/api/views/$ID" -H "Content-Type: application/json" -d @-
```

(``crm views update`` is Phase 2c+ — until it ships, PATCH stays via curl
inside the docker network.)

### Step 5 — Pin / Unpin / Export

```bash
crm views pin $ID
crm views export $ID --export-format csv -o my_view.csv
```

## Workflow B — composed dashboard (Phase 3)

When the user asks for "a dashboard" or "an overview" or "a daily
stand-up screen", build N views + 1 dashboard in one atomic call:

```bash
cat <<'JSON' | crm dashboards compose --from-file - --pin
{
  "title": "CRO daily standup",
  "description": "KPIs + stage chart + late-stage table.",
  "pinned": true,
  "views": [
    {
      "title": "KPI · Total open ARR",
      "spec": {"entity": "opportunity", "view": "kpi",
               "filter": {"op": "and", "clauses": []},
               "columns": [], "sort": [], "highlight": [],
               "group_by": [],
               "aggregations": [{"op": "sum", "field": "amount", "alias": "Open ARR"}],
               "limit": 500}
    },
    {
      "title": "Bar · Deals by stage",
      "spec": {"entity": "opportunity", "view": "bar_chart",
               "filter": {"op": "and", "clauses": []},
               "columns": [], "sort": [], "highlight": [],
               "group_by": ["stage"],
               "aggregations": [{"op": "count", "alias": "Deals"}],
               "limit": 500}
    },
    {
      "title": "Late-stage table",
      "spec": {"entity": "opportunity", "view": "table",
               "filter": {"op": "and", "clauses": [
                 {"field": "stage", "op": "eq", "value": "offer_negotiation"}]},
               "columns": ["name", "amount", "owner_id", "close_date"],
               "sort": [{"field": "close_date", "direction": "asc"}],
               "highlight": [], "limit": 50}
    }
  ],
  "items": [
    {"view_index": 0, "position": 0, "col_span": 4},
    {"view_index": 1, "position": 1, "col_span": 8},
    {"view_index": 2, "position": 2, "col_span": 12}
  ]
}
JSON
```

Layout hints: ``col_span`` is on a 12-col grid. KPIs typically use 3-4,
charts 6-8, full-width tables 12. The compose endpoint runs all
inserts inside one transaction — if any view fails validation
(catalog mismatch, unsupported operator, exceeded clause limit), the
whole dashboard rolls back and the user sees a single clear error
instead of half-built artefacts.

The response includes the ``url`` of the new dashboard
(``/dashboards/{id}``); send that back to the user. They can pin it,
delete it, or refine individual views from there.

## Failure modes

| Symptom                                   | Likely cause                                 | What to tell the user |
|-------------------------------------------|----------------------------------------------|------------------------|
| 422 "Unknown filter field"                | LLM hallucinated a field                     | Re-fetch catalog; ask which real field they meant |
| 422 "not aggregatable"                    | `sum`/`avg`/`min`/`max` on non-numeric field | Use `count` or pick the numeric field (`amount`) |
| 422 "not groupable"                       | `group_by` on a field without the flag       | The catalog tells you which are groupable — read it again |
| 422 "exactly one group_by"                | Chart with 0 or 2+ group_by                  | Charts are 1-D in Phase 2; pick one |
| 422 "view_index out of range"             | Compose payload references a non-existent view | Renumber items or add the missing view |
| 404 on render                             | View deleted or owner mismatch               | Check `crm auth me` matches the creator |

## Phase-2c+ hooks

- Drill-down (chart-point click → child view with the bucket value as a
  filter) is Phase 3b. Today the dashboard tile shows an "Open ↗" link
  to the underlying saved view so the user can drill manually.
- Cross-view date windows shipped: a dashboard's board-wide `date_range`
  re-windows every tile that declares its own `date_range` (see above).
  Cross-view *filter* overlay (a shared FilterGroup ANDed into each tile)
  also ships via the dashboard `filter` field.
- Sharing a view / dashboard with a teammate is Phase 4. Today every
  artefact is owner-scoped.
