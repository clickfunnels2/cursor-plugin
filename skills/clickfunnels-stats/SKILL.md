---
name: clickfunnels-stats
description: >
  Read aggregated analytics of both kinds: performance metrics for a funnel or a
  single page - views, opt-ins, sales, earnings over a configurable timerange -
  and workspace record stats over a filtered set of records - counts, breakdowns by a
  property, date ranges and numeric totals for contacts and orders. Use this
  skill any time you need a number rather than the records themselves: surfacing
  funnel or page performance, recurring agent jobs that audit funnels daily, or
  answering "how many match this" and "what is the breakdown" without paging a
  collection.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/funnels/skill.md
    description: Funnel workflow APIs (split tests, conditional splits) - the structure these stats describe.
  - path: https://accounts.myclickfunnels.com/.well-known/pages/skill.md
    description: Page authoring API - useful follow-up after stats surface a low-performing page.
  - path: https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md
    description: Filter authoring API - used to retarget audiences when stats surface an audience-fit problem.
---

# ClickFunnels Stats Skill

Read aggregated analytics. There are two different kinds, answering two different
questions, so start by picking the one you want:

| You want | Use | Shape |
|----------|-----|-------|
| How a funnel or page is PERFORMING over time - views, opt-ins, sales, earnings | Funnel stats / Page stats, below | One summary per funnel or page, over a timerange |
| How an EMAIL performed - opens, clicks, bounces, unsubscribes, attributed revenue | Email stats, below | One summary per broadcast or workflow step |
| How many RECORDS match something, or how they break down - "how many contacts have this tag", "revenue by billing status" | Record stats, below | One number (or one number per group) over a filtered set of records |
| The matching RECORDS themselves, not a statistic | [Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md#reading-a-filters-matches) | Projected rows for a filter |

Performance stats are time-series metrics about traffic. Record stats count rows
in the database. Asking the wrong one is the most common mistake here: "how many
contacts have tag X" is record stats, not funnel stats.

## Email stats: broadcasts and workflow steps

Email engagement shares one vocabulary across broadcasts and workflow steps: the counts
`opened`, `unique_opens`, `clicked`, `unique_clicks`, `bounced`, `unsubscribed`,
`spam_complaints`, the whole-percentage rates `open_rate`, `click_rate`,
`click_to_open_rate`, `bounce_rate`, `unsubscribe_rate`, and an attributed `revenue` block
(`total`, `orders_count`, `per_recipient`, `per_unique_open`, `per_unique_click`). One
parser reads those from either endpoint. The delivery counters and where the blocks live
differ, because a broadcast has a fixed audience and a workflow step does not:

| | Broadcast | Workflow step |
|---|---|---|
| Endpoint | `GET /api/v2/emails/broadcasts/{broadcast_id}/stats` | `GET /api/v2/workflows/steps/{step_id}/stats` (Closed Beta - `403` for a workspace without access; request it at https://developers.myclickfunnels.com/page/code-support) |
| Where the blocks live | `performance`, `revenue` | `email.performance`, `email.revenue` (`email` is `null` for a step that sends no email) |
| Delivery counters | `progress.sent`, `performance.delivered` | `email.performance.sends`, `recipients`, `failed` |
| Rate denominator | the broadcast's recorded audience | `recipients` (distinct contacts sent to inside the window) |
| Timerange | None. Reports on its own send window, returned as `timerange`. Passing `timerange_start`/`timerange_end` is a `422`. | `timerange_start` / `timerange_end`, defaulting to 30 days and capped at 90. |

There is no `sends` on a broadcast and no `delivered` on a step. Read the delivery counter
the table names for the endpoint you called; the engagement and revenue keys are the same.

The timerange difference is deliberate, and it is the thing to get right: a broadcast is
sent once, so its window is a fact about the broadcast; a workflow step keeps sending for
as long as the workflow runs, so the window is a question you have to ask. Both echo
`timerange` in the response - read it back rather than assuming.

Details, including the full payload and how revenue is attributed, are in the
[Emails Skill](https://accounts.myclickfunnels.com/.well-known/emails/skill.md) and the
[Workflows Skill](https://accounts.myclickfunnels.com/.well-known/workflows/skill.md#step-stats).

Note that these rates are whole percentages (`open_rate: 51` means 51%), while the funnel
and page metrics below use fractions. Do not feed them to the same formatter. A workflow
step's rates compare events inside the window, so they can exceed 100 when contacts open
an email they were sent before the window began; the Workflows skill explains when.

## Performance stats: funnels and pages

Two resources:

- **Funnel stats** - one funnel, one summary, optional per-step breakdown.
- **Page stats** - one page, single-step analytics, with funnel context when the page is reached via a funnel step.

Both return the same metric vocabulary (`views_all`, `views_unique`, `optins`,
`optin_rate`, `sales_count`, `sales_rate`, `sales_value`, `earnings_per_view`, etc.)
so an agent can write one analyzer that consumes either payload.

## When to use this skill

- Daily / weekly automated audits that flag underperforming funnels or pages.
- Generating a report for a workspace owner ("which funnel made the most this week").
- Detecting regressions: a step's `optin_rate` dropping week-over-week.
- Picking which page to A/B test next (lowest `optin_rate` in a multi-step funnel).
- Picking which step to retarget with a conditional split (lowest `sales_rate` with high `views_all`).
- Sizing an audience before acting on it ("how many contacts match this filter").
- Segmenting a record set by a property ("contacts by active status", "orders by billing status").
- Totalling a numeric column over a filtered set ("revenue from these orders").

This skill is **read-only**. If the analysis suggests a fix, hand off to the [Funnels Skill](https://accounts.myclickfunnels.com/.well-known/funnels/skill.md) (split tests / conditional splits) or the [Pages Skill](https://accounts.myclickfunnels.com/.well-known/pages/skill.md) (page rewrite via PML).

## Authentication

OAuth2 password grant. See the [Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md#authentication) for the full flow - the same access token works here. No additional feature gates required for read access.

Performance stats need read access to the funnel or page. Record stats follows the
resource being analyzed: `contact` needs Contacts read access, `order` needs Store read
access. A token scoped to one but not the other gets a 403 naming the category it is
missing.

## Endpoint reference

| Action            | Method | Path                                              |
|-------------------|--------|---------------------------------------------------|
| Funnel stats      | GET    | `/api/v2/funnels/:funnel_id/stats`                |
| Page stats        | GET    | `/api/v2/pages/:page_id/stats`                    |
| Record stats      | GET    | `/api/v2/workspaces/:workspace_id/stats`          |

Record stats computes and persists nothing, so a read-only token may call it.

## Timerange parameters (performance stats only)

| Param              | Type    | Default                              | Notes                                                              |
|--------------------|---------|--------------------------------------|--------------------------------------------------------------------|
| `timerange_start`  | ISO8601 | now - 30 days                        | Clamped: if start > end, window resets to 30 days ending at `end`. |
| `timerange_end`    | ISO8601 | now                                  | Clamped: future end dates are clamped to now.                      |

The window is also clamped to a **90-day maximum**. Requests for longer windows are silently truncated to 90 days. For multi-quarter analysis, page through 90-day windows and aggregate client-side.

## Funnel stats

`GET /api/v2/funnels/:funnel_id/stats`

### Query parameters

| Param         | Type   | Notes                                                                          |
|---------------|--------|--------------------------------------------------------------------------------|
| `expand[]`    | string | Pass `steps` to receive a `steps` array with per-step metrics instead of the default `page_public_ids` array. |

### Response (default - without `expand[]=steps`)

```json
{
  "funnel": { "id": 1, "public_id": "gESyMv", "name": "My Sales Funnel" },
  "currency": "USD",
  "timerange": { "from": "2026-03-03T00:00:00Z", "to": "2026-04-02T23:59:59Z" },
  "summary": {
    "earnings_per_click": "1.24",
    "upfront_sales": "4250.00",
    "upfront_sales_count": 17,
    "recurring_sales": "850.00",
    "average_cart_value": "250.00",
    "pageviews": 3422
  },
  "page_public_ids": ["abc123", "def456", "ghi789"]
}
```

### Response (with `expand[]=steps`)

`page_public_ids` is replaced by `steps`. Each step has the same shape as the `Pages::Stats` `step` block:

```json
{
  "funnel": { ... },
  "currency": "USD",
  "timerange": { ... },
  "summary": { ... },
  "steps": [
    {
      "page_id": 100,
      "name": "Opt-in",
      "current_path": "/optin",
      "views_all": 1500,
      "views_unique": 1200,
      "optins": 360,
      "optin_rate": 0.3,
      "sales_count": 0,
      "sales_rate": 0,
      "sales_value": "0.00",
      "recurring_sales_count": 0,
      "recurring_sales_value": "0.00",
      "earnings_per_view": "0.00",
      "earnings_per_unique_view": "0.00"
    }
  ]
}
```

## Page stats

`GET /api/v2/pages/:page_id/stats`

Stats are computed from the page's associated funnel step. If the page is not part of a funnel (e.g. a standalone site page), `funnel` and `step` are both `null` - there is no other analytics surface for these pages yet.

### Response

```json
{
  "currency": "USD",
  "timerange": { "from": "2026-03-03T00:00:00Z", "to": "2026-04-02T23:59:59Z" },
  "page": {
    "id": 100,
    "public_id": "abc123",
    "name": "Order Page",
    "current_path": "/order-canonical",
    "type": "funnel_page"
  },
  "funnel": { "id": 3, "public_id": "fnlXyz", "name": "Q2 Promo" },
  "step": {
    "page_id": 100,
    "name": "Order",
    "current_path": "/order-canonical",
    "views_all": 800,
    "views_unique": 720,
    "optins": 0,
    "optin_rate": 0,
    "sales_count": 24,
    "sales_rate": 0.033,
    "sales_value": "5400.00",
    "recurring_sales_count": 4,
    "recurring_sales_value": "120.00",
    "earnings_per_view": "6.75",
    "earnings_per_unique_view": "7.50"
  }
}
```

## Record stats: counts, breakdowns and totals

**Closed Beta** - not yet enabled for all workspaces. Request access at
https://developers.myclickfunnels.com/page/code-support. Performance stats for funnels and
pages (above) are generally available and unaffected.

One endpoint answers "how many", "what is the breakdown" and "what is the total" for
every analyzable resource. Name the resource, then request the statistics you want as
flags.

```http
GET /api/v2/workspaces/{workspace_id}/stats?resource=contact&count=true&count_by[property]=is_active&filter[tag_ids]=39
Authorization: Bearer <access_token>
```

```json
{
  "resource": "contact",
  "workspace_id": 5,
  "stats": {
    "count": 468,
    "count_by": {
      "property": "is_active",
      "values": { "true": 218, "false": 250 },
      "capped": false
    }
  }
}
```

A flag of `true` computes the statistic with its defaults. A flag given as an object
supplies its parameters. Omitting a statistic, or passing `false`, skips it. Asking
for several in one call is cheaper than several calls, because they share the same
underlying scan.

### Statistics by resource

| Statistic | Parameters | Returns | `contact` | `order` |
|-----------|------------|---------|-----------|---------|
| `count` | none | Matching records | yes | yes |
| `count_by` | `property` | One count per distinct value of that property | yes | yes |
| `created_at_range` | none | Earliest and latest `created_at` in the set | yes | yes |
| `sum` | `property` | Total of one numeric property | no | yes |

`count_by` and `sum` take their `property` from a per-resource allow-list, because
grouping or totalling an unindexed column on a large table is expensive:

| Resource | `count_by` properties | `sum` properties |
|----------|-----------------------|------------------|
| `contact` | `is_active`, `anonymous`, `email_suppression_reason`, `time_zone` | - |
| `order` | `billing_status`, `service_status`, `order_type`, `live_mode` | `total_amount`, `tax_amount`, `total_collected` |

Call the endpoint with a resource but no statistics and the 422 lists exactly what
that resource supports, so the endpoint documents itself:

```json
{ "error": "no stats requested. Supported for contact: count, count_by, created_at_range" }
```

### Narrowing the set

Three selectors, all optional, composed with AND. Omit all three to report on every
record of that resource in the workspace.

| Selector | Use for |
|----------|---------|
| `filter` | Simple field matches, using the same keys the resource's own list accepts |
| `stable_id` | A one-off filter token, e.g. from the contact-filter generator |
| `stored_filter_id` | A saved filter in this workspace |

| Resource | `filter` keys |
|----------|---------------|
| `contact` | `id`, `email_address`, `is_active`, `tag_ids` |
| `order` | `id`, `contact_id`, `order_type`, `billing_status`, `service_status`, `live_mode` |

Comma-separated values match any of them, so `{"tag_ids": "39,44"}` counts contacts
carrying either tag. An unknown key is a 422 rather than being ignored, because a
silently dropped filter returns a confidently wrong number.

To build a `stable_id` or `stored_filter_id` from criteria or from plain English, see
the [Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md).

```http
GET /api/v2/workspaces/{workspace_id}/stats?resource=order&count=true&sum[property]=total_amount&count_by[property]=billing_status
Authorization: Bearer <access_token>
```

```json
{
  "resource": "order",
  "workspace_id": 5,
  "stats": {
    "count": 96,
    "sum": { "property": "total_amount", "value": "10843.26" },
    "count_by": {
      "property": "billing_status",
      "values": { "paid": 17, "pending": 77, "refunded": 2 },
      "capped": false
    }
  }
}
```

Decimal totals such as money are strings so their precision survives JSON serialization,
and they carry the same scale the resource's own records do - a total of exactly one hundred
dollars is `"100.00"`, not `"100.0"`, so the value can be formatted or compared without
re-deriving the scale.

`sum.value` is `null` when, and only when, the matching set is EMPTY. A non-empty set whose
rows all have no value in that column totals `"0.00"`, matching what those records themselves
report, so `null` never has to be told apart from a real zero. `created_at_range` follows the
same rule: `{"min": null, "max": null}` means nothing matched. `count` is the cheapest way to
confirm it - ask for it in the same call.

### Bounds on very large sets

Counting scans rows, so a single call is bounded rather than allowed to become a full
table scan.

- Past the bound, `count` returns the bound and a `count_capped_at` appears alongside
  it. The real total is higher - narrow the filter and ask again.
- `count_by` reports `capped: true` when its breakdown covers only the bounded slice.
- Exact aggregates (`created_at_range`, `sum`) are **not computed at all** past the
  bound. They are named in an `unavailable` array with an `unavailable_reason`, rather
  than returned as a confidently wrong number.

```json
{
  "resource": "contact",
  "workspace_id": 5,
  "stats": { "count": 100000, "count_capped_at": 100000 },
  "unavailable": ["created_at_range"],
  "unavailable_reason": "More than 100000 matching records, so exact aggregates were not computed. Narrow the filter and retry."
}
```

A filter that takes too long to evaluate returns a 422 saying so. That is
deterministic rather than transient: retrying it unchanged fails identically. Narrow
it, or ask for a `count` rather than records.

Because record stats are in Closed Beta, a workspace that does not have access answers
`403` saying so. That is not retryable and not a scope problem: the message names the
alternatives that need no access, and the Refine Filters skill covers them (ask a filter for
`results` with `count: true`, or page the resource's own list). Performance stats for a funnel
or page are unaffected.

The 503 that mentions the request budget is the opposite case, and the two must not be
confused. It means the REQUEST's wall clock ran out before the filter got a fair slice,
so nothing was read and the filter itself was never shown to be too broad. Retrying is
worth doing there.

### Two calls at a time

Record stats are database-heavy, so this endpoint allows two in flight at once per access
token and workspace and answers a third with a `429`:

```json
{
  "error": "Too many filtered requests are already running for this workspace and access token. Try again in 2 seconds.",
  "hint": null,
  "retry_after": 2
}
```

Wait `retry_after` seconds and retry - it is in the body as well as in the `Retry-After`
header, so the delay is readable however your client reaches the API. This is transient: the
same request succeeds once one of the running reads finishes.

The limit counts every stats call, whether or not you passed a filter, so a dashboard that
fans four tiles out in parallel will have two of them refused. Ask for several statistics in
ONE call instead - they share the scan, so it is cheaper as well as inside the limit - and run
at most two calls concurrently.

## Metric vocabulary (performance stats)

| Field                          | Meaning                                                              |
|--------------------------------|----------------------------------------------------------------------|
| `views_all`                    | Total pageviews in the window (includes repeats).                    |
| `views_unique`                 | Distinct visitor sessions in the window.                             |
| `optins`                       | Lead form submissions on this step.                                  |
| `optin_rate`                   | `optins / views_unique` as a 0..1 float.                             |
| `sales_count`                  | Distinct purchases attributed to this step.                          |
| `sales_rate`                   | `sales_count / views_unique` as a 0..1 float.                        |
| `sales_value`                  | Upfront sales total (`Money.to_s`, e.g. `"5400.00"`).                |
| `recurring_sales_count`        | Subscriptions / installments started.                                |
| `recurring_sales_value`        | Recurring revenue captured in the window.                            |
| `earnings_per_view`            | `sales_value / views_all`.                                           |
| `earnings_per_unique_view`     | `sales_value / views_unique`.                                        |
| `summary.earnings_per_click`   | Funnel-level EPC across the whole workflow.                          |
| `summary.upfront_sales`        | Funnel-level one-time revenue.                                       |
| `summary.recurring_sales`      | Funnel-level recurring revenue.                                      |
| `summary.average_cart_value`   | `summary.upfront_sales / summary.upfront_sales_count`.               |

Money fields are strings (`"5400.00"`) to preserve precision. Rates are floats in `[0, 1]`.

## Example use case - recurring "funnel auditor" agent

A nightly agent job that pulls every funnel's stats for the trailing 7 days, scores them, and writes a digest to Slack with concrete next-step suggestions.

### Job shape

```
schedule:    cron, daily 06:00 UTC
inputs:      workspace_id, oauth access_token, list of funnel_ids (or "all")
outputs:     ranked findings + suggested actions, posted to Slack
```

### Pseudocode

```python
WINDOW_DAYS = 7
now = datetime.utcnow()
window = {
    "timerange_start": (now - timedelta(days=WINDOW_DAYS)).isoformat() + "Z",
    "timerange_end":   now.isoformat() + "Z",
}

findings = []
for funnel_id in funnel_ids:
    # 1. Pull funnel-level summary + per-step breakdown.
    funnel = GET(f"/api/v2/funnels/{funnel_id}/stats",
                 params={**window, "expand[]": "steps"})

    summary = funnel["summary"]
    steps   = funnel["steps"]

    # 2. Heuristic: low EPC overall.
    if float(summary["earnings_per_click"]) < 0.10 and summary["pageviews"] > 500:
        findings.append({
            "funnel_id":  funnel["funnel"]["public_id"],
            "severity":   "high",
            "kind":       "low_epc",
            "message":    f"EPC ${summary['earnings_per_click']} on "
                          f"{summary['pageviews']} views - funnel is monetising poorly.",
            "suggestion": "Add a conditional split on a high-intent filter "
                          "(see /.well-known/refine-filters/skill.md) or "
                          "A/B test the order page (see /.well-known/funnels/skill.md "
                          "#split-test-steps).",
        })

    # 3. Heuristic: optin step with low rate but plenty of traffic.
    for step in steps:
        if step["views_unique"] > 200 and 0 < step["optin_rate"] < 0.15:
            findings.append({
                "funnel_id":  funnel["funnel"]["public_id"],
                "step":       step["current_path"],
                "severity":   "medium",
                "kind":       "weak_optin",
                "message":    f"Step `{step['current_path']}`: optin rate "
                              f"{round(step['optin_rate']*100,1)}% on "
                              f"{step['views_unique']} unique views.",
                "suggestion": "Build a NEW page with a stronger hero + CTA in PML "
                              "(see /.well-known/page-markup), then wrap the "
                              "original page in a split test against it at weight "
                              "50/50. Do not overwrite the live page: that needs "
                              "explicit approval first, see the approval guardrail "
                              "in the Page Markup skill.",
            })

        # 4. Heuristic: order step with traffic but no sales.
        if step["views_unique"] > 200 and step["sales_count"] == 0 \
           and step["name"].lower().startswith(("order", "checkout")):
            findings.append({
                "funnel_id":  funnel["funnel"]["public_id"],
                "step":       step["current_path"],
                "severity":   "high",
                "kind":       "zero_sales",
                "message":    f"Order step `{step['current_path']}` had "
                              f"{step['views_unique']} unique views and zero sales.",
                "suggestion": "Confirm pricing, payment processor connection, and "
                              "fulfillment readiness before iterating on copy.",
            })

# 5. Rank and post.
findings.sort(key=lambda f: ({"high":0,"medium":1,"low":2}[f["severity"]],
                              -float(f.get("views", 0))))
slack_post(format_digest(findings))
```

### Notes for the agent author

- **Cache the auth token.** It is long-lived; refresh once per run, not per call.
- **One funnel <-> one HTTP call.** Always pass `expand[]=steps` so you can score steps without a second roundtrip.
- **Money is a string.** Cast with `float()` before comparing thresholds.
- **`optin_rate` and `sales_rate` are 0..1 floats**, not percentages. Multiply by 100 only when rendering.
- **Skip empty windows.** If `summary.pageviews` < 50, the noise floor is too high - emit "insufficient traffic" instead of a finding.
- **Don't repeat suggestions.** Persist the previous run's findings keyed by `(funnel_id, kind, step)` and skip findings that are still open from the prior run unless severity escalates.
- **Page stats are useful for landing-page audits.** When the agent is given a `page_id` instead of a funnel (e.g. for a standalone landing page hooked into ads), fetch `/api/v2/pages/:page_id/stats` directly - the response carries the same per-step block under `step`.
- **Suggest, do not overwrite.** An auditor proposes changes; it does not rewrite live pages on its own. Writing `markup` to an existing page replaces the whole page with a PML subset of what the editor can build, so it requires explicit approval from the person first: see [the approval guardrail](https://accounts.myclickfunnels.com/.well-known/page-markup/skill.md#stop-get-approval-before-overwriting-an-existing-page). Prefer the additive route the heuristics above suggest: build a new page and split-test it against the original.

### Failure modes to handle

- **404** - the funnel/page id is unknown to this workspace, or the page is not a `user_page`.
- **429 on record stats** - more than two stats calls in flight for this token and workspace.
  Wait the body's `retry_after` seconds and retry; batch statistics into one call rather than
  fanning out one call per tile.
- **`funnel: null` / `step: null` on a page** - the page isn't wired into a funnel; emit a "no analytics available" finding rather than a numeric one.
- **Clamped window** - if you ask for >90 days, the response silently uses 90. Re-read `timerange.from` / `timerange.to` from the response, not from your request, before quoting the window in the digest.

## Related skills

- [Funnels Skill](https://accounts.myclickfunnels.com/.well-known/funnels/skill.md) - the structure these stats describe; primary destination for "fix it" actions.
- [Pages Skill](https://accounts.myclickfunnels.com/.well-known/pages/skill.md) - build a replacement for a low-performing page in PML (overwriting an existing page needs approval first).
- [Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md) - author a stored filter for retargeting via a
  conditional split, and read a filter's matching records when you need the records rather than a statistic.
