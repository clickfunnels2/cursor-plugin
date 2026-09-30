---
name: clickfunnels-refine-filters
description: >
  Generate contact filters from plain English, define and manage reusable
  contact filters for conditional splits and broadcasts, apply existing
  order-filter tokens to the orders list, and read a filter's matches or
  statistics without paging the whole set. Use this skill when a CF2 surface
  takes a `filter_id`, when listing orders through a filter already built in
  ClickFunnels, when you need to know how many records a filter matches, or
  when you need the matching records cheaply enough to act on them. Reading,
  one-time generation, counting and projecting matches all require only
  Contacts read access; persisting or modifying a contact filter requires
  Contacts write access.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/funnels/skill.md
    description: Funnel workflow APIs (split tests, conditional splits) - primary consumer of stored filters.
  - path: https://accounts.myclickfunnels.com/.well-known/pages/skill.md
    description: Companion skill for building/editing pages programmatically via the Page Markup API.
  - path: https://accounts.myclickfunnels.com/.well-known/page-markup/skill.md
    description: PML reference - the DSL used to author the `markup` field on pages and blog posts.
---

# ClickFunnels Refine Filters Skill

Build, store, and reuse audience filters that any CF2 surface accepting a
`filter_id` can attach. The two main consumers are:

- **Conditional Split Steps** - branch contacts down a workflow path based on a stored filter.
- **Email Broadcasts** - scope which contacts a broadcast is sent to.

Other surfaces (segments, workflow branches, store upsells, shipping zones, etc.)
also accept stored filter ids when applicable.

## When to use this skill

- Creating a reusable filter once and attaching it to multiple workflow
  branches or conditional splits
- Routing contacts down a conditional split based on tag membership, opt-in
  history, or other allow-listed contact attributes
- Scoping an email broadcast audience without re-listing recipients on every
  send
- Listing orders through a filter already built or saved in the ClickFunnels
  Orders UI
- Updating the audience for an already-attached filter without re-touching the
  consumer (the broadcast or split will continue to use the same `filter_id`,
  but the criteria are now different)
- Answering "how many records match this?" without fetching the records
- Pulling the ids or email addresses of a matching audience cheaply enough to
  act on them in a bulk tag, enroll, workflow run, unsubscribe or export
- Breaking a filtered set down by a property, or totalling a numeric column
  over it

A filter created via this API is a saved filter record scoped to a workspace -
it is **shared across every consumer** that takes a `filter_id` in that same
workspace.

## Authentication

OAuth2 password grant using an OAuth application's credentials.

All endpoints require an authorized token. Category-scoped tokens need Contacts
read access for reads and one-time generation; writes, including generation with
`save: true`, need Contacts write access.

```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
client_id=<app_uid>
client_secret=<app_secret>
username=<app_uid>
password=<app_secret>
scope=read write delete
```

Store the `access_token` from the response. It is long-lived.

## Endpoints

| Action  | Method | Path                                                        |
|---------|--------|-------------------------------------------------------------|
| List    | GET    | `/api/v2/workspaces/:workspace_id/refine_filters`           |
| Show    | GET    | `/api/v2/refine_filters/:id`                                |
| Create  | POST   | `/api/v2/workspaces/:workspace_id/refine_filters`           |
| Update  | PATCH  | `/api/v2/refine_filters/:id`                                |
| Delete  | DELETE | `/api/v2/refine_filters/:id`                                |
| Generate from text | POST | `/api/v2/workspaces/:workspace_id/contacts/filters`   |
| Show generated filter | GET | `/api/v2/contacts/filters/:id`                      |
| Stats for a filtered set | GET  | `/api/v2/workspaces/:workspace_id/stats`          |

The manual CRUD endpoints use the wrapped `refine_filter` body documented
below. The natural-language generation endpoint uses a separate flat body.

List has two modes, and `after` is the switch. Omit it and you get the original
full-registry response: every saved filter in one payload, no `Pagination-Next`
header. Send it and you get bounded pages of 20.

On the first paged request you have no cursor yet, so send `after=0`. It is the
opt-in itself rather than a row id (real ids start at 1), and it means "start at
the beginning" in whichever direction `sort_order` asks for, so
`after=0&sort_order=desc` walks the registry newest-first. From there, send each
`Pagination-Next` value back as the next `after` until that response header is
absent, which is how you know you read the last page.

List also supports `filter[name]=...` for an exact-match lookup, so a saved
filter can be resolved by name instead of id. A legacy row whose stored state can
no longer be decoded is returned with `filter_class: null` and empty `criteria`
instead of failing the list; it can still be deleted.

The list returns **every** filter class saved in the workspace, not only the
ones this API authors. Alongside `ContactsFilter` and `OrdersFilter` you will
see classes saved by other ClickFunnels surfaces - `ContactsSegmentsFilter`,
`ProductsFilter`, `FunnelsFilter`, `ContactUpsellsFilter` and others. Treat
`filter_class` as an open set: select the class you want rather than assuming
the rest are absent.

## Generate a contact filter from plain English

Use the generation endpoint when the audience is easier to describe than to
construct manually, or when it needs a condition outside the manual API's safe
whitelist. It selects from the full ContactsFilter catalog, while still
validating condition names, clauses, structure, and workspace-owned ids.

The request body is flat (there is no `refine_filter` wrapper):

```http
POST /api/v2/workspaces/5/contacts/filters
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "text": "contacts tagged VIP who purchased the Cool Shirt",
  "save": false
}
```

- `text` is required.
- `save` defaults to `false`. The endpoint performs a read-grade computation,
  persists nothing, and returns a one-time `stable_id` token.
- `save: true` persists the filter and requires Contacts write access. `name` is
  optional and only applies when the filter is saved.

An unsaved response has `null` identity fields because there is no database
record to fetch later:

```json
{
  "id": null,
  "public_id": null,
  "workspace_id": null,
  "name": null,
  "stable_id": "H4sI...",
  "filter": {
    "conjunction": "and",
    "criteria": [
      {"attribute": "tags.id", "clause": "in", "value": ["12"]}
    ]
  }
}
```

Pass `stable_id` as the `stable_id` query parameter on the contacts list. Treat
it as opaque and let the HTTP client encode the query parameter. With curl, use
`-G --data-urlencode "stable_id=$STABLE_ID"`; do not paste the token directly
into a URL or decode it first.

A `stable_id` copied from the URL of a filtered Contacts page also works, even
when the UI wrapped the criteria in a group, as long as every conjunction in
the token is the same word (all `and` or all `or`). Tokens mixing `and`/`or`
return 422, grouped or not - a flat filter cannot express the mix, so it is
refused rather than resolved to one of the two words.

When `save: true`, the response also includes `id`, `public_id`, `workspace_id`,
and `name`. That saved result can be fetched from
`GET /api/v2/contacts/filters/:id` and managed through the regular
`refine_filters` CRUD endpoints.

### Text matching that cannot use an index

On a text attribute, `eq` (equals) and `sw` (starts with) resolve against an
index. `cont` (contains) and `ew` (ends with) cannot: a leading wildcard forces
a scan of every value stored for that attribute. The cost of that scan tracks
how many values the attribute has, not how many contacts the finished filter
selects, so adding further criteria may not bring it down. On a workspace with
millions of contacts it is what trips the 10s evaluation limit under
[Common errors](#common-errors).

This matters most for free-text contact custom attributes (`utm_source`,
`company`, and so on), which only this generation endpoint can select on. When
the audience allows it, describe an exact value or a prefix instead of a
substring: "utm_source is hts_cc" or "utm_source starts with hts" rather than
"utm_source contains hts". When a substring genuinely is the requirement,
expect the evaluation to be slow on a large workspace, and prefer asking
`/stats` for a count over reading records.

## Safe condition whitelist

The public RefineFilter API accepts a deliberately small set of conditions and
clauses. The underlying Refine engine supports many more, but a number of those
either bypass indexes (text `contains`, regex), rely on cross-tenant tables
(segments, custom attributes), or scan event tables open-ended (broadcast or
opt-in conditions without a scoping id). To keep things safe for partners
running at scale, every public-API request is checked against the table below
before the criteria are persisted.

Requests that violate the policy return **422 Unprocessable Entity** with every
violation listed in the message and a pointer to the Developer Community for
new-condition requests:
[https://developers.myclickfunnels.com/page/code-support](https://developers.myclickfunnels.com/page/code-support).

If you need filter conditions that fall outside this whitelist, request them
via the [Developer Community](https://developers.myclickfunnels.com/page/code-support) -
the team adds approved conditions to the policy directly.

| Condition                                                         | Allowed clauses                                                                                                                  | Notes                                                              |
|-------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| `email_address`, `first_name`, `last_name`                        | `eq`, `sw`                                                                                                                       | **Text - equals & starts_with only.** No `contains`/regex/`dcont`. |
| `anonymous`                                                       | `eq`, `dne`                                                                                                                      | Boolean-style option.                                              |
| `created_at`, `unsubscribed_at`, `last_activity`                  | All clauses supported by the underlying date condition (`eq`, `gte`, `lte`, `btwn`, `nbtwn`, `st`, `nst`, etc.)                 | Indexed datetime columns.                                          |
| `tags.id`                                                         | `eq`, `dne`, `in`, `nin`                                                                                                         | Tag membership - must reference workspace-owned tag ids.           |
| `has_affiliate_attribution`, `has_active_affiliate_attribution`, `referred_by_affiliate` | All standard clauses for the underlying condition.                                                                  | Affiliate attribution lookups.                                     |
| `email_suppression`                                               | `st`, `nst`, `eq`, `in`                                                                                                          | Email suppression reason set/not-set or membership.                |
| `received_broadcast`, `opened_broadcast`, `clicked_broadcast`     | `eq`, `in`                                                                                                                       | **Must include a specific broadcast id** - no open-ended scans.    |
| `opted_in_funnel_step`, `opted_in_funnel_at`, `opted_in_on_standalone_page` | `eq`, `in`                                                                                                             | **Must include a specific funnel / step / page id** - no open-ended scans. |

**Conjunction:** `and` only. `or` is rejected. Multi-criterion filters are
joined with AND across the board.

**Implicitly rejected:** text `contains`/`dcont`/regex on name/email; segment
membership; viewed funnel/product; any `events.product_id`-style condition;
custom contact attributes; community membership/topic conditions;
product/variant/price/course conditions; bounced/unsubscribed/did-not-open
broadcast conditions; and any open-ended broadcast/opt-in scan that omits its
scoping id.

If the policy is too narrow for your use case, request the missing condition
in the [Developer Community](https://developers.myclickfunnels.com/page/code-support).

## Request body shape

```json
{
  "refine_filter": {
    "name": "VIP newsletter audience",
    "filter_class": "ContactsFilter",
    "conjunction": "and",
    "criteria": [
      { "attribute": "tags.id",     "clause": "in", "value": ["tag-pub-id-1", "tag-pub-id-2"] },
      { "attribute": "created_at",  "clause": "gte", "value": "2026-01-01T00:00:00Z" }
    ]
  }
}
```

- `name` is optional. When set, it must be unique within the workspace.
- `filter_class` defaults to `ContactsFilter` (the only class supported in v1).
- `conjunction` is fixed to `"and"` on the public API. Mixed/nested grouping is
  not supported.
- `criteria` must be a non-empty array. Each entry is `{attribute, clause, value}`.

## Response shape

```json
{
  "id": 42,
  "public_id": "AbCdEf",
  "workspace_id": 5,
  "name": "VIP newsletter audience",
  "filter_class": "ContactsFilter",
  "conjunction": "and",
  "criteria": [
    { "attribute": "tags.id",    "clause": "in",  "value": ["tag-pub-id-1", "tag-pub-id-2"] },
    { "attribute": "created_at", "clause": "gte", "value": "2026-01-01" }
  ],
  "created_at": "2026-04-01T12:00:00.000Z",
  "updated_at": "2026-04-01T12:00:00.000Z"
}
```

`value` is normalized by condition type. Option/select values come back as
arrays (even when you sent a scalar), absolute date values are ISO dates
(`YYYY-MM-DD`), and relative date values are objects with `days` and
`modifier`.

Create, show, and update return one filter object. List returns an envelope, not
a bare array:

```json
{
  "refine_filters": [
    {
      "id": 42,
      "public_id": "AbCdEf",
      "workspace_id": 5,
      "name": "VIP newsletter audience",
      "filter_class": "ContactsFilter",
      "conjunction": "and",
      "criteria": [
        {"attribute": "tags.id", "clause": "in", "value": ["12"]}
      ],
      "created_at": "2026-04-01T12:00:00.000Z",
      "updated_at": "2026-04-01T12:00:00.000Z"
    }
  ]
}
```

## Validation

- Unknown `attribute` -> 422
- Attribute or clause outside the safe whitelist (gate off) -> 422 with policy message
- Unknown or unsupported `clause` for the attribute -> 422
- `value` referencing IDs not in the current workspace (tags, products,
  variants, courses) -> 422
- Unparseable `value` for date attributes -> 422
- `filter_class` other than `ContactsFilter` -> 422
- `conjunction: "or"` (gate off) -> 422

## Clause cheatsheet

Refine uses short codes for clauses. The most common ones:

| Code   | Meaning                                |
|--------|----------------------------------------|
| `eq`   | equals (single value)                   |
| `dne`  | does not equal                          |
| `in`   | in (multi-value)                        |
| `nin`  | not in (multi-value)                    |
| `sw`   | starts with                             |
| `st`   | is set (column has a value)             |
| `nst`  | is not set                              |
| `gt`   | greater than                            |
| `gte`  | greater than or equal                   |
| `lt`   | less than                               |
| `lte`  | less than or equal                      |
| `btwn` | between (date range; value is `[d1, d2]`) |
| `nbtwn`| not between                             |
| `exct` | exactly (relative date with `days`)     |

For a date attribute using `gt`, `lt`, or `exct`, send `value` as
`{"days":"30","modifier":"ago"}`. `modifier` must be `ago` or
`from_now`. Absolute date clauses such as `gte` and `lte` continue to take an
ISO 8601 string.

## Worked examples

### 1. Contact has the "VIP" tag

```http
POST /api/v2/workspaces/5/refine_filters
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "refine_filter": {
    "name": "VIPs",
    "criteria": [
      {"attribute": "tags.id", "clause": "in", "value": ["12"]}
    ]
  }
}
```

### 2. Contact's email starts with `vip+`

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "VIP-prefixed",
    "criteria": [
      {"attribute": "email_address", "clause": "sw", "value": "vip+"}
    ]
  }
}
```

### 3. Contacts created in 2026

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Joined in 2026",
    "criteria": [
      {"attribute": "created_at", "clause": "gte", "value": "2026-01-01T00:00:00Z"}
    ]
  }
}
```

### 4. Contacts created in the last 30 days

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Joined in the last 30 days",
    "criteria": [
      {
        "attribute": "created_at",
        "clause": "lt",
        "value": {"days": "30", "modifier": "ago"}
      }
    ]
  }
}
```

### 5. Contacts that received a specific broadcast

```http
POST /api/v2/workspaces/5/refine_filters
Content-Type: application/json
Authorization: Bearer <access_token>

{
  "refine_filter": {
    "name": "Received Spring Promo",
    "criteria": [
      {"attribute": "received_broadcast", "clause": "in", "value": ["77"]}
    ]
  }
}
```

A `received_broadcast` criterion **must** include a specific broadcast id -
omitting it returns 422 (open-ended scans are not allowed).

## Applying existing order filters

The Refine Filter CRUD endpoints above create contact filters only. The orders
list can still apply an order filter that already exists in ClickFunnels:

- `stable_id` is the token from the URL of a filtered Orders page. Keep the
  token unchanged, but send it through your HTTP client's normal query-parameter
  encoding. Do not concatenate the raw token into a URL.
- `stored_filter_id` is the id or public id of a saved order filter in the same
  workspace. Use the List endpoint above and choose an entry whose
  `filter_class` is `OrdersFilter`.

Treat order-filter entries returned by List or Show as read-only in this API.
The Update endpoint authors contact filters only; edit order filters in the
ClickFunnels Orders UI.

If both are supplied, `stable_id` takes precedence.

```bash
ORDER_FILTER_TOKEN='<stable_id copied from the filtered Orders page>'

curl -skG \
  -H "Authorization: Bearer <access_token>" \
  --data-urlencode "stable_id=${ORDER_FILTER_TOKEN}" \
  "https://<workspace-subdomain>.myclickfunnels.com/api/v2/workspaces/<workspace_id>/orders"
```

```http
GET /api/v2/workspaces/{workspace_id}/orders?stored_filter_id={saved_order_filter_id}
Authorization: Bearer <access_token>
```

The API does not build a new order filter from raw criteria. Build it in the
Orders UI or use an existing saved order filter, then pass one of the selectors
above. Tokens copied from the filtered UI often wrap criteria in groups; a
grouped token is accepted as long as every conjunction in it is the same word
(all `and` or all `or` - the grouping is redundant and is flattened). A
malformed token, wrong filter type, cross-workspace id, or a token mixing
`and`/`or` returns 422 (grouped or not).

## Reading a filter's matches

A filter selects a set, so every filter endpoint can hand you that set in the same
response. Pass `results` and you get the matching records back alongside the filter
itself, projected to the fields you name. Without it the response is unchanged.

```http
POST /api/v2/workspaces/{workspace_id}/contacts/filters
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "text": "contacts tagged CC-webinar",
  "results": { "fields": ["id", "email_address"], "count": true }
}
```

```json
{
  "id": null,
  "public_id": null,
  "workspace_id": 5,
  "name": null,
  "stable_id": "eJyrVrJSMDRSsrKuVspNzMlMzytJzSuJT8xLL...",
  "filter": {
    "conjunction": "and",
    "criteria": [{ "attribute": "tags.id", "clause": "in", "value": ["39"] }]
  },
  "results": {
    "fields": ["id", "email_address"],
    "rows": [
      [35760, "ada@example.com"],
      [35761, "grace@example.com"]
    ],
    "count": 468,
    "truncated": true,
    "bytes": 3042,
    "bytes_per_record": 34,
    "max_bytes": 3072,
    "stop_reason": "byte_budget",
    "next_after": "35908",
    "suggested_max_bytes": 20656,
    "guidance": "Returned 90 of 468 matching records. Options: to get all 468 in ONE call, retry with results.max_bytes=20656 (and results.limit at least 468) ..."
  }
}
```

`results` works the same way on all four filter endpoints - the generation POST, the
generated-filter GET, and the saved-filter create/update/GET. Pass `results: true`
for the defaults, which project `id` only and do not compute a count. Pass
`results.count: true` only when you need the bounded total; on a truncated page that
requires one additional evaluation of the filter. For a number without rows, use the
workspace stats endpoint instead.

Saved contact and order filters support `results`. Other filter classes return 422
until the API defines an explicit projection allow-list for them.

### The row format

`fields` gives the column order once and `rows` carries bare value arrays, so zip
them to rebuild objects. An object per row would repeat every key on every record,
which roughly doubles the payload for a projection of two or three scalars - the
opposite of the point. A single projected field still returns one-element arrays, so
the shape never changes.

Projectable fields for a CONTACT filter, scalars only:

`id`, `public_id`, `workspace_id`, `email_address`, `first_name`, `last_name`,
`phone_number`, `uuid`, `time_zone`, `anonymous`, `is_active`, `unsubscribed_at`,
`email_suppression_reason`, `last_notification_email_sent_at`, `created_at`,
`updated_at`

Tags, custom attributes and visits are deliberately not projectable. They are the
expensive part of a contact, and leaving them out is what makes a large set
affordable. Ask for the full representation from the contacts list when you need
them.

Each value renders exactly as the resource's own record endpoint renders that column, so a
projected row can be compared with a record you read elsewhere. Money is a string at the
currency's scale (`"100.00"`), and `null` means the record has no value in that column.

### Sizing the response yourself

You decide how much comes back, because only you know what your client can hold:

| Field | Meaning |
|-------|---------|
| `results.max_bytes` | Byte budget for the rows. Defaults to 3072. Clamped to 4194304. |
| `results.limit` | Maximum records. Defaults to 500. No server maximum - the byte budget is the real bound. |
| `results.after` | Cursor from a previous `next_after`, to continue a truncated set. |
| `results.count` | Include the bounded total match count. Defaults to `false`; a truncated page needs a second filter evaluation when this is `true`. |

Whichever bound hits first stops the response, and `stop_reason` says which:
`row_limit` (raise your `limit`), `byte_budget` (raise `max_bytes`, or project fewer
fields), `record_exceeds_budget` (one record alone is over budget - project fewer
fields), or `time_budget` (the request reached the server's filtered-query wall-clock
budget). `time_budget` is the only one you cannot lift: page with `next_after` or
narrow the filter, because raising `max_bytes` or `limit` is not what stopped it.

Every response reports its estimated row cost - `bytes`, `bytes_per_record`, and the
`max_bytes` actually applied - so size the next call from measurement rather than
from a guess. The estimate is measured before final response encoding, which can be
slightly larger for values such as timestamps; leave headroom. When a response is
truncated and the whole set could still fit in one call, `suggested_max_bytes` is the
value to retry with. It is extrapolated from the widest row returned SO FAR rather
than the average, so it is an estimate: a later
record wider than anything on that page can still make the retry truncate. Check
`truncated` on the retry rather than assuming you got the whole set.

Three things worth knowing before you raise the budget:

- `max_bytes` bounds the ROWS, not the whole response. Over an MCP connection the
  payload is carried twice (structured plus text) alongside the filter definition,
  which measures at 2.0x-2.9x the rows on the wire. Divide by about 3 when sizing
  against a tool-result cap. The default is already sized that way.
- `max_bytes` is a row-cost budget, not an exact wire-size ceiling. Leave headroom
  for final response encoding.
- `count` is present only when you pass `results.count: true`. It is the filter's
  TOTAL match count, not the number of rows returned. On a very large audience it
  saturates at the server's scan cap and a `count_capped_at` comes back with it -- then
  `count` is a floor, not a total, and you narrow the filter to get an exact number. If
  only that extra count query reaches the time budget, the valid rows and cursor still
  return, `count` is omitted, and `count_unavailable_reason` explains how to recover.

### Counting instead of reading

If the question is "how many" rather than "which ones", do not fetch rows at all. The
workspace stats endpoint computes statistics over a filtered set without returning the
records, and it accepts the same selectors you built above.

```http
GET /api/v2/workspaces/{workspace_id}/stats?resource=contact&count=true&stored_filter_id=42
Authorization: Bearer <access_token>
```

```json
{ "resource": "contact", "workspace_id": 5, "stats": { "count": 468 } }
```

Counts, breakdowns by a property, date ranges and numeric totals, the statistics each
resource supports, and the bounds on very large sets are all documented in the
[Stats Skill](https://accounts.myclickfunnels.com/.well-known/stats/skill.md#record-stats-counts-breakdowns-and-totals).

### Listing contacts instead

When you need fields the projection does not offer, list contacts through the filter:

```http
GET /api/v2/workspaces/{workspace_id}/contacts?stable_id={token}
Authorization: Bearer <access_token>
```

Prefer `results` on the filter for a whole audience: the contacts list pages at 20
records, so reassembling a large set through it costs one request per 20 records.

## Applying filters to consumers

A filter is only useful once attached to a consumer. The two primary surfaces:

### Applying filters to conditional split steps

Conditional splits route contacts down one of two branches based on whether
they match a stored filter. Once you have an `id` (or `public_id`) for the
filter, hand it to any conditional split:

```http
PATCH /api/v2/funnels/{funnel_id}/conditional_split_steps/{split_id}
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "condition": {"filter_id": "AbCdEf"}
  }
}
```

Read it back with the inline filter resolved:

```http
GET /api/v2/funnels/{funnel_id}/conditional_split_steps/{split_id}
GET /api/v2/funnels/{funnel_id}/conditional_split_steps?expand[]=filter
```

For the full conditional split surface (creating splits, attaching pages to
branches, positioning), see the [Funnels Skill](https://accounts.myclickfunnels.com/.well-known/funnels/skill.md#conditional-split-steps).

### Applying filters to email broadcasts

Email broadcasts that are scoped via a stored filter accept the filter id
directly on the broadcast resource:

```http
POST /api/v2/workspaces/{workspace_id}/emails/broadcasts
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "emails_broadcast": {
    "name": "Spring Promo",
    "subject": "Spring is here",
    "from_email": "hello@example.com",
    "html_body": "<p>...</p>",
    "filter_id": 42,
    "send_immediately": false
  }
}
```

`filter_id` on a broadcast is a numeric saved-filter id (not the
public id). Create the filter first via this skill, then pass its `id` field
verbatim to the broadcasts endpoint. When `filter_id` is omitted the broadcast
falls back to the workspace's full contact list.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 422 "criteria[0].attribute '...' is not supported by the public API" | Attribute is not in the safe whitelist | Use an allowed attribute, or request the new condition at the dev community link. |
| 422 "criteria[0].clause '...' is not allowed for attribute '...'" | Disallowed clause for a text attribute (e.g. `cont` on `email_address`) | Use `eq` or `sw`. |
| 422 "criteria[0].value must include a specific id for attribute '...'" | Open-ended broadcast/opt-in scan | Include the broadcast / funnel / step / page id you want to scope to. |
| 422 "conjunction 'or' is not supported by the public API" | `conjunction: "or"` is not supported on the public API | Use `"and"`. |
| 422 "criteria[0].attribute '...' is not a known attribute" | Misspelled attribute | Compare against the table above. |
| 422 "criteria[0].value contains ids not found in this workspace" | Tag/product/course id is from another workspace | Use ids you fetched from this workspace's `/contacts/tags`, `/products`, etc. |
| 422 "criteria[0].value '...' must be an ISO8601 date or datetime" | Date attribute received a non-ISO string | Send `YYYY-MM-DD` or full ISO8601. |
| 422 "filter_class must be 'ContactsFilter' (only supported class in v1)" | Only contact filters can be authored here, whatever classes the list returns | Omit `filter_class` to default it; edit other classes in the UI surface that owns them. |
| 422 from the orders list for `stable_id` | Token is malformed, is not an order filter, contains unsupported grouping, or references another workspace | Copy the token from the Orders UI and send it with normal query-parameter encoding. |
| 422 from the orders list for `stored_filter_id` | Saved filter is missing, belongs to another workspace, or is not an order filter | Use an order filter saved in the same workspace. |
| 422 "The request is too large to translate into a contact filter" from generation | The description implies too many separate criteria (a list of individual contacts or emails), so the model's answer runs past the request budget | Deterministic, not transient - retrying the same text fails identically. Describe the audience by attributes (tags, dates, purchases), or split it into smaller requests. |
| 503 from contact-filter generation | The AI model timed out or another temporary dependency was unavailable | Retry the same request after a short delay. |
| 200 with an empty `refine_filters` array on the first paged request | `after` carried something that is neither `0` nor a `Pagination-Next` cursor; a value that is not a real row id pages nothing | Start the walk with `after=0`, then echo each `Pagination-Next` value. |
| 422 "This filter took longer than 10s to evaluate and was stopped" | The filter selects more than can be evaluated in one request | Deterministic, not transient - retrying it unchanged fails identically. Narrow it (add criteria, or bound it by `created_at`), or ask the workspace stats endpoint for a count instead of records. If a criterion uses `cont` or `ew` on a text attribute, narrowing may not help - see [Text matching that cannot use an index](#text-matching-that-cannot-use-an-index). |
| 503 "This request ran out of its 20s budget before the filter could be evaluated" | The REQUEST's wall clock ran out, not the filter's - the time went elsewhere in the same request (a slow generation call, or earlier pages of the same projection) | The opposite of the 422 above: the filter was not shown to be too broad, so a retry can succeed. Retry once; if it keeps happening, narrow the filter or lower `results.limit` / `results.max_bytes` so each call does less. |
| 403 "Workspace stats are in Closed Beta" | Record stats over a filtered set are in Closed Beta, and this workspace does not have access yet | Not retryable and not a scope problem. Get the number from a filter instead: pass `results` with `count: true`, which returns a bounded count alongside the matching records. |
| 429 "Too many filtered requests are already running" | Two database-heavy filtered reads are active for this access token and workspace | Wait the body's `retry_after` seconds (also the `Retry-After` header), then retry. Transient - it succeeds once one of the running reads finishes. Avoid launching the same expensive filter concurrently. |
| 400 "stored_filter_id must be a top-level parameter ... not nested under filters[]" | A selector was sent as `filters[stored_filter_id]` or `filters[stable_id]`, which nothing reads | Send it at the top level. Nested, it used to be ignored and the request returned every record with a 200. |
| 422 "Unknown results.fields for contact: ..." | A projected field is not a scalar on the allow-list (for example `tags`) | Use a field from the projectable list; request the full representation from the contacts list for association-backed data. |
| 422 "results.max_bytes must be a positive integer" | `max_bytes` or `limit` was zero or negative | Send a positive value, or omit it for the default. |
| 422 "no stats requested. Supported for contact: ..." | The stats call named a resource but no stats | Add a stat flag. The message lists the ones that resource supports, so an empty call is a cheap way to discover them. |
| 401 "API key missing or invalid" | Missing, revoked, expired, or wrong token | Re-authenticate. |
| 404 "Not found" | Filter id does not exist or belongs to another workspace | Verify the id is reachable for this workspace's token. |
