---
name: clickfunnels-funnels
description: >
  Update funnel settings and build funnel workflows programmatically - paths,
  favicons, tags, split tests, conditional splits, branch positioning,
  multi-page branches and stored-filter wiring. Use this skill when you need to
  change a funnel's settings, routing, or lifecycle through the public API;
  duplicate, archive, restore, or remove a funnel; or manage branded short links
  and read their click counts.
version: "1.2"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/pages/skill.md
    description: Companion skill for building/editing pages programmatically via the Page Markup API.
  - path: https://accounts.myclickfunnels.com/.well-known/page-markup/skill.md
    description: PML reference - the DSL used to author the `markup` field on pages and blog posts.
  - path: https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md
    description: Companion skill for creating reusable Refine filters consumed by conditional splits and broadcasts.
---

# ClickFunnels Funnels Skill

A guide to funnel settings, workflows, and lifecycle: edit a funnel's path, favicon,
domain, tags and code, then build split tests and conditional splits around its pages.
Duplicate, archive, restore, or delete unused funnels. Also covers branded short
links and their click statistics.

## When to use this skill

- Change an existing funnel's URL path, favicon, domain, tags, or custom code.
- Create branded campaign links, check path availability, and read click counts.
- Wrap an existing page in an A/B split test (variant page swap, weight tuning).
- Insert a conditional split that routes contacts down a "matched" or
  "unmatched" branch based on a stored filter.
- Grow a branch with multiple sequential pages (Welcome -> Onboarding -> CTA).
- Position a split between specific steps of a funnel using `show_page_step_id`.
- Read back the resulting tree (with the filter inline via `expand[]=filter`).
- Archive or restore a funnel and its attached pages, or delete an unused funnel shell.
- Check page content and the funnel's payment mode without confusing the two.
- Make an independent same-workspace copy of a funnel and poll its progress.

## Authentication

OAuth2 password grant using an OAuth application's credentials. See the
[Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md#authentication) for
details - the same access token is used here.

The token must be authorized for the workspace. Category-restricted tokens need
`funnels:read` for reads (including `/structure` and clone-action polling) and
`funnels:write` for lifecycle calls and starting a duplication. Restoring archived
products also requires Store write access. Deleting a funnel that has attached
workflow triggers also requires Workflows write access.

## Payment mode and page content

The funnel's `live_mode` controls payment processing: `false` is test mode,
while `true` enables live payments. Changing it does not publish pages. Page
content is saved live through the existing Pages API, one page write at a
time. See [Saving live page content](https://accounts.myclickfunnels.com/.well-known/pages/skill.md#saving-live-page-content)
for the internal, custom HTML, and external SDK hosting paths and the approval
guardrail for replacing editor content.

Use `GET /api/v2/funnels/:id` to inspect payment mode and
`PUT /api/v2/funnels/:id` or `PATCH /api/v2/funnels/:id` to change it. The API
reference calls these HTTP operations
[`getFunnels`](https://developers.myclickfunnels.com/reference/getfunnels)
and [`updateFunnels`](https://developers.myclickfunnels.com/reference/updatefunnels);
these names are OpenAPI operation IDs.

```http
GET /api/v2/funnels/5
Authorization: Bearer <access_token>

PUT /api/v2/funnels/5
Authorization: Bearer <access_token>
Content-Type: application/json

{"funnel":{"live_mode":false}}

GET /api/v2/funnels/5
Authorization: Bearer <access_token>
```

`PUT` and `PATCH` use the existing update operation with the `funnel` JSON
root. The path accepts a numeric `id` or its opaque `public_id`. The unwrapped
Funnel response returns `live_mode`, not publication status. Boolean values
and the strings `"true"` and `"false"` assign that mode; null, empty, and
other values leave it unchanged. Enable live payments only when the operator
asks for them. A valid repeated assignment is still only a mode update.

Read page resources back with `GET /api/v2/pages/:id` to see their computed
URLs, and use the existing funnel structure endpoint to inspect the step tree. Do not
construct URLs by joining paths or treat a returned URL or a successful write
as proof of public delivery. There is currently no atomic whole-funnel release
or pollable publish receipt. For restricted tokens, reads need funnels read
scope and writes need funnels write scope, alongside workspace and record
access. MCP clients use `describe_category("funnels")`, then `fetch` or
`update` on `funnel` and the matching page resource with a granted
`workspace_id`, record `id`, and flat `attributes` on updates.

## Endpoint reference

| Action                    | Method | Path                                                                |
|---------------------------|--------|---------------------------------------------------------------------|
| Fetch funnel              | GET    | `/api/v2/funnels/:id`                                               |
| Update funnel settings    | PUT/PATCH | `/api/v2/funnels/:id`                                            |
| Archive funnel            | POST   | `/api/v2/funnels/:id/archive`                                       |
| Restore funnel            | POST   | `/api/v2/funnels/:id/unarchive`                                     |
| Delete empty funnel       | DELETE | `/api/v2/funnels/:id`                                               |
| List split tests          | GET    | `/api/v2/funnels/:funnel_id/split_test_steps`                        |
| Create split test         | POST   | `/api/v2/funnels/:funnel_id/split_test_steps`                        |
| Show split test           | GET    | `/api/v2/funnels/:funnel_id/split_test_steps/:id`                    |
| Update split test         | PATCH  | `/api/v2/funnels/:funnel_id/split_test_steps/:id`                    |
| Destroy split test        | DELETE | `/api/v2/funnels/:funnel_id/split_test_steps/:id?branch=...`         |
| List conditional splits   | GET    | `/api/v2/funnels/:funnel_id/conditional_split_steps`                 |
| Create conditional split  | POST   | `/api/v2/funnels/:funnel_id/conditional_split_steps`                 |
| Show conditional split    | GET    | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id`             |
| Update conditional split  | PATCH  | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id`             |
| Destroy conditional split | DELETE | `/api/v2/funnels/:funnel_id/conditional_split_steps/:id?branch=...`   |
| Duplicate funnel          | POST   | `/api/v2/funnels/:funnel_id/clone_actions`                           |
| Poll funnel duplication   | GET    | `/api/v2/funnels/clone_actions/:id`                                  |
| List funnel duplications  | GET    | `/api/v2/funnels/:funnel_id/clone_actions`                           |

## Funnel lifecycle

Archive a funnel when you want to retire a populated funnel without removing its
workflow. `POST /api/v2/funnels/:id/archive` has no request body. It archives the
funnel and its attached pages, changes the funnel's `current_path` to a unique
path, and unlinks redirect steps targeting it. A shared funnel cannot be archived.
Archived funnels remain available through funnel list and fetch calls.

```http
POST /api/v2/funnels/123/archive
Authorization: Bearer <access_token>
```

Restore it with a bodyless `POST /api/v2/funnels/:id/unarchive`. The funnel and
attached pages become active again, but the old path and redirect links do not
return. If the funnel has attached pages, archived products are queued for
restoration after the call commits; check their state separately if your next
step depends on them. When archived products will be submitted for restoration,
the token needs Store write access as well as Funnels write access. Funnels write
alone suffices when there are no archived products to restore. A workspace funnel
limit can prevent restoration.

```http
POST /api/v2/funnels/123/unarchive
Authorization: Bearer <access_token>
```

Both calls return HTTP 200 with the full funnel representation. You can pass
the numeric `id` or opaque `public_id`, and request the existing optional
`expand[]` fields. A repeated archive or restore returns 409; validation
failure returns 422.

To remove an unused funnel shell, call `DELETE /api/v2/funnels/:id`. This works
for active or archived funnels only when they are unshared and their workflow
has no steps beyond its generated root. It returns 204 with no body. A populated
funnel returns 409; archive it instead. The workflow and generated root step
remain after deletion. The funnel's attached workflow triggers are deleted by
the existing destroy cascade, so a trigger-bearing funnel requires both Funnels
write and Workflows write access. Funnels write alone is sufficient when the
funnel has no attached triggers.

```http
DELETE /api/v2/funnels/123
Authorization: Bearer <access_token>
```

## Funnel settings

Use `PUT` or `PATCH /api/v2/funnels/:id` with a `funnel` object. The `:id` can
be the numeric id or public id returned by the API. Funnel creation already
accepts `current_path`; this update also accepts it and adds
`favicon_image_id`. Send `current_path`, not `path`.
The update returns the saved funnel, and `GET /api/v2/funnels/:id` reads it back.

```http
PUT /api/v2/funnels/123?expand[]=head_code
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "funnel": {
    "name": "Autumn offer",
    "current_path": "/autumn-offer",
    "favicon_image_id": 73,
    "domain_id": 42,
    "tag_ids": [9],
    "head_code": "<meta name=\"campaign\" content=\"autumn\">",
    "head_code_mode": "replace"
  }
}
```

The path is normalized to lowercase with a leading slash and no trailing slash.
Blank, root, reserved and occupied paths return 422, including a path held by
another resource's historical redirect. You can reuse this funnel's own former
path. A path edit is unavailable while a domain change is propagating. A domain
change may still be propagating after the update returns 200; read the funnel
again before relying on the final URL.

The favicon must be an image id from this funnel's workspace. Send `null` to
clear it. `tag_ids` replaces the tag set; `[]` clears it, while `null` leaves it
alone. Image, tag and domain ids must belong to the funnel's workspace. Code
fields append by default, so use `head_code_mode: "replace"` or
`footer_code_mode: "replace"` for repeatable updates; `null` clears a code field.
Add `expand[]=head_code` or `expand[]=footer_code` when reading code, since those
fields are otherwise omitted from the response.

With MCP, call `describe_category` for `funnels`, then the generic `update` tool
with `resource: "funnel"`, the granted workspace's public id, the funnel id and
flat `attributes` such as `current_path` and `favicon_image_id`.

## Split test steps

A split test step wraps an existing page in a funnel into a 2-branch A/B test.
Each variant carries the `show_page_step` + `page` attached to that branch
(both `null` when the branch is empty), plus the branch's `weight` (0..100;
the two weights must sum to 100).

The first variant's `page_id` must already be attached to a step in this
funnel's workflow (that page gets wrapped in place - no cloning). The second
variant's `page_id` must be a page with **no existing show-page step** - an
internal page not yet attached to any funnel, or a standalone external (SDK)
page (see
[Placing an external (SDK) page in a branch](#placing-an-external-sdk-page-in-a-branch)).
Variant entries accept only `page_id` and `weight`; any other key is a 422.

### Positioning the split

By default (no `show_page_step_id`), the split lands at the funnel entry -
`sort_order 0` under the workflow root - with all surviving steps moved into
a `Root Step after Sequence End` convergence sequence positioned right after
the split.

When `show_page_step_id` references an existing show-page step in this funnel,
the split is inserted **after the workflow step that wraps it**, and steps
N+1...end are migrated under the convergence sequence.

`show_page_step_id` accepts a numeric or obfuscated
`Workflows::Steps::ShowPageStep` id - pass `show_page_step.id` (or
`show_page_step.public_id`) from any page response.

**Positioning caution (applies to split tests AND conditional splits):**
creating a split MOVES the wrapped step inside a branch sequence, so a step id
you fetched earlier may no longer sit where you think. Read
[Funnel Structure](#funnel-structure) immediately before creating a split to
pick the anchor from the current tree, and re-read it immediately after to
confirm placement. Anchoring on a step that sits inside another split's branch
nests the new split inside that branch - if you want a split to apply to ALL
traffic after an A/B test, create it before the A/B test (or anchor it on a
step outside the branches), not on a page that is already a variant.

### Example - wrap step N+1 as the test variant

```http
POST /api/v2/funnels/123/split_test_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "split_test_step": {
    "show_page_step_id": "aBcDeF",
    "variants": [
      {"page_id": 500, "weight": 60},
      {"page_id": 501, "weight": 40}
    ]
  }
}
```

The response is the new `SplitTestStep` resource with both variants resolved.

### Destroying a split test

```http
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=left
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=right
DELETE /api/v2/funnels/123/split_test_steps/9876?branch=both
```

`branch` defaults to `both`. **A 1-branch split is not a supported state**, so
removing either single branch (`left` or `right`) collapses the entire split:
the surviving branch's pages are promoted up under the split's parent (so they
remain attached to the funnel's main workflow) and the split test wrapper is
destroyed. `branch=both` is the same end-state with both sides discarded.

## Conditional split steps

A conditional split routes contacts down one of two branches based on whether
they match a stored Refine filter:

- `branches[0]` - **matched** branch (filter matches the contact)
- `branches[1]` - **unmatched** branch (filter does not match)

Branches can be empty, hold a single page, or grow into multi-page sequences
(see below).

### Empty create

```http
POST /api/v2/funnels/123/conditional_split_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{}
```

Creates an empty unconfigured split that the caller can fill in via PATCH.

### Multi-page branches via `page_ids`

```http
POST /api/v2/funnels/123/conditional_split_steps
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "branches": [
      {"page_ids": [600, 601, 602]},
      {"page_ids": [603]}
    ],
    "show_page_step_id": 100
  }
}
```

`page_id` (single) and `page_ids` (array) are mutually exclusive per branch,
and branch entries accept **only** those two keys - anything else (e.g.
`external_url`, `show_page_step_id`) is a 422. Each referenced page must be a
workspace-owned page with **no existing show-page step**: an internal page not
yet attached to any funnel, or a standalone external (SDK) page.

### Placing an external (SDK) page in a branch

Branch entries never take a URL - they take a page id. To route a branch to a
page you host yourself:

1. Create a **standalone** external page (no `funnel` block) - see
   [External (SDK) Pages](https://accounts.myclickfunnels.com/.well-known/pages/skill.md#standalone-external-pages):

   ```http
   POST /api/v2/workspaces/42/pages/external
   Authorization: Bearer <access_token>
   Content-Type: application/json

   {"page": {"name": "External OTO", "external_url": "https://acme.example.com/oto"}}
   ```

2. Pass the returned page id in the branch (or split-test variant) slot:

   ```http
   POST /api/v2/funnels/123/conditional_split_steps
   Authorization: Bearer <access_token>
   Content-Type: application/json

   {
     "conditional_split_step": {
       "branches": [{}, {"page_id": "<page_public_id>"}]
     }
   }
   ```

This works the same for a split test's second variant (`variants[1].page_id`).
The same rule applies to internal pages: create the page first (unattached to
any funnel), then reference it by id.

### Attaching a filter

Once a stored filter exists (see the
[Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md#applying-filters-to-conditional-split-steps)),
attach it via `condition.filter_id`:

```http
PATCH /api/v2/funnels/123/conditional_split_steps/9876
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "conditional_split_step": {
    "condition": {"filter_id": "AbCdEf"}
  }
}
```

`filter_id` is the public id of a stored filter in the same workspace. Raw
numeric ids are rejected for cross-tenant safety. Filters that were created
under the standard public API are subject to the
[safe-condition whitelist](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md#safe-condition-whitelist) -
once persisted, they can be referenced freely from this surface (no extra gating
on the consumer side).

### Reading back with the inline filter

```http
GET /api/v2/funnels/123/conditional_split_steps/9876
GET /api/v2/funnels/123/conditional_split_steps?expand[]=filter
```

On the show endpoint, `condition.filter` is always included as the full
`RefineFilter` object. On the list endpoint it is included only when
`expand[]=filter` is passed.

### Destroying a conditional split

```http
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=matched
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=unmatched
DELETE /api/v2/funnels/123/conditional_split_steps/9876?branch=both
```

`branch` defaults to `both`. **A 1-branch conditional split is not a supported
state**, so removing either single branch (`matched` or `unmatched`) collapses
the entire split: the surviving branch's pages are promoted up under the
split's parent (so they remain attached to the funnel's main workflow) and the
conditional split wrapper is destroyed. `branch=both` is the same end-state
with both sides discarded.

## Funnel structure

```http
GET /api/v2/funnels/:id/structure
```

Returns a lean, flattened tree of the funnel's **visible** steps - show-page,
conditional-split, and split-test steps, in order. The workflow's internal
container steps (the root container, split-branch containers, and the
post-split convergence) are spliced out, so `steps` is a flat ordered list you
can walk directly. Useful for rendering an outline, finding a step, or picking a
`sort_order` before inserting a page.

! **FORBIDDEN - a funnel step's path is its own standalone slug, never the funnel
path + the step slug.** The `url` returned per step is composed **automatically**
by ClickFunnels from the funnel/step hierarchy - do not reconstruct or guess it.
When you create the pages that become these steps (via the
[Pages skill](https://accounts.myclickfunnels.com/.well-known/pages/skill.md#setting-a-pages-path-current_path)),
each step's `current_path` must be its **own** slug only - never a parent path
concatenated with the child's slug. e.g. for funnel path `mangia-mangia-catering`,
set the step's `current_path` to its own path (`mangia-mangia-catering`),
**never** the nested `mangia-mangia-catering/catering`.

A worked example - `Entry -> Conditional Split (matched: VIP Welcome / unmatched:
Guest Welcome) -> Split Test (Premium Offer A 60% / Premium Offer B 40%)`. Note the
flattening: split children are inline under `branches`/`variants`, internal sequence
containers are gone, `sort_order` is container-local (it resets inside each branch),
and split nodes carry a `convergence_step_id` (the step where every path rejoins):

```json
{
  "funnel": { "id": 98, "public_id": "mYLmMN", "name": "Coaching Upsell Funnel" },
  "steps": [
    {
      "id": 1178, "public_id": "jxGkbJ", "step_type": "show_page_step",
      "name": "Entry", "sort_order": 0, "show_page_step_id": "WJmQqJ",
      "page": { "id": 1413, "public_id": "joXkgY", "name": "Entry",
        "url": "https://acme.myclickfunnels.com/entry", "external": false, "sdk_token": null }
    },
    {
      "id": 1184, "public_id": "JEDdre", "step_type": "conditional_split_step",
      "name": "Conditional Split Step", "sort_order": 1, "convergence_step_id": 1195,
      "branches": [
        { "branch": "matched", "steps": [
          { "id": 1189, "public_id": "vbaRZj", "step_type": "show_page_step",
            "name": "VIP Welcome", "sort_order": 0, "show_page_step_id": "neonkv",
            "page": { "id": 1418, "public_id": "YGBWDj", "name": "VIP Welcome",
              "url": "https://acme.myclickfunnels.com/vip-welcome", "external": false, "sdk_token": null } }
        ] },
        { "branch": "unmatched", "steps": [
          { "id": 1190, "public_id": "vyggrJ", "step_type": "show_page_step",
            "name": "Guest Welcome", "sort_order": 0, "show_page_step_id": "RJgLRj",
            "page": { "id": 1419, "public_id": "ewMRAj", "name": "Guest Welcome",
              "url": "https://acme.myclickfunnels.com/guest-welcome", "external": false, "sdk_token": null } }
        ] }
      ]
    },
    {
      "id": 1195, "public_id": "vXkkrj", "step_type": "split_test_step",
      "name": "Split Test Step", "sort_order": 1, "convergence_step_id": null,
      "variants": [
        { "weight": 60, "steps": [
          { "id": 1191, "public_id": "vpzzzJ", "step_type": "show_page_step",
            "name": "Premium Offer A", "sort_order": 0, "show_page_step_id": "MJnnnJ",
            "page": { "id": 1422, "public_id": "JygoNY", "name": "Premium Offer A",
              "url": "https://acme.myclickfunnels.com/premium-offer-a", "external": false, "sdk_token": null } }
        ] },
        { "weight": 40, "steps": [
          { "id": 1193, "public_id": "jQwwqe", "step_type": "show_page_step",
            "name": "Premium Offer B", "sort_order": 0, "show_page_step_id": "QeOPbJ",
            "page": { "id": 1420, "public_id": "eVNQme", "name": "Premium Offer B",
              "url": "https://acme.myclickfunnels.com/premium-offer-b", "external": false, "sdk_token": null } }
        ] }
      ]
    }
  ]
}
```

Common node fields: `id`, `public_id`, `step_type` (short token, e.g.
`show_page_step`), `name`, `sort_order`. Fields by `step_type`:

| `step_type` | Extra fields |
|--------|--------------|
| `show_page_step` | `show_page_step_id` (pass this to `funnel.show_page_step_id` on Create/Update Page to swap), `page: { id, public_id, name, url, external, sdk_token }` |
| `conditional_split_step` | `convergence_step_id` (where the branches rejoin), `branches: [{ branch: "matched"\|"unmatched", steps: [...] }]` |
| `split_test_step` | `convergence_step_id`, `variants: [{ weight, steps: [...] }]` |

Branch and variant `steps` arrays are flattened the same way (no nested
container wrappers). `page.external` and `page.sdk_token` are always present
on show-page nodes; `sdk_token` is `null` unless `external: true`.

## Duplicate a funnel

Start a same-workspace copy with a bodyless POST. The response is a durable
receipt, not the new funnel:

```http
POST /api/v2/funnels/123/clone_actions
Authorization: Bearer <access_token>
```

```json
{
  "id": 456,
  "public_id": "LmNpQr",
  "workspace_id": 12,
  "funnel_id": 123,
  "status": "queued",
  "cloned_funnel_id": null,
  "error_code": null
}
```

Send no JSON attributes. Name, path, site, domain, mode, and scheduling cannot
be overridden. Every POST is an explicit new copy attempt and returns a new
receipt.

Poll the receipt by numeric or public ID:

```http
GET /api/v2/funnels/clone_actions/LmNpQr
Authorization: Bearer <access_token>
```

Continue while `status` is `queued` or `processing`. On `completed`, fetch
`cloned_funnel_id` through `GET /api/v2/funnels/:id`. On `failed`, inspect the
`error_code`, which is `clone_failed` after background retries are exhausted.
Polling never restarts work. To recover earlier receipts, list them with
`GET /api/v2/funnels/:funnel_id/clone_actions`; follow `Pagination-Next` using
the `after` query parameter when there are more results.

The new funnel stays on the source site and domain, receives a generated
localized name and path, starts in test mode, and has a disabled workflow.
Pages and steps are independent copies, including a fresh token for every
custom HTML page. Product and automation links remain attached. Tags, history,
and share links are not copied. External SDK pages are unsupported.

MCP clients use the same flow:

```text
take_action(resource: "funnel", action: "clone", id: "123", workspace_id: "12")
list(resource: "funnel_clone_action", parent_id: "123", workspace_id: "12")
fetch(resource: "funnel_clone_action", id: "LmNpQr", workspace_id: "12")
fetch(resource: "funnel", id: "789", workspace_id: "12")
```

A 409 means the source has a domain change in progress. Archived funnels,
missing workflows, external pages, or a missing source-team membership return
422 before the action is accepted. A read-only token can poll but cannot start
a copy. A token without access to the source workspace cannot create or read
its receipts.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 422 "condition.filter_id must be a stable identifier, not a raw numeric id" | Sent the database PK | Use the filter's public id (e.g. `AbCdEf`). |
| 422 "condition.filter_id not found in this workspace" | Filter belongs to another workspace | Use a filter created in this workspace. |
| 400 "conditional_split_step.branches[i].page_id not found in this workspace" | Page id from another workspace | Use a page in this workspace. |
| 422 "conditional_split_step.branches[i].page_id is already used by another funnel step" | Page already wrapped in a `ShowPageStep` | Use a fresh page. |
| 400 "conditional_split_step.branches[i] must provide either page_id or page_ids, not both" | Mixed shapes per entry | Pick one. |
| 404 "Not found" | Funnel / step id does not belong to this workspace's accessible scope | Verify the funnel id and access token. |
| 403 "write access to Store" | Restoring the funnel would also submit archived products for restoration | Reconnect or use a token with Store write access. |
| 403 "write access to Workflows" | Deleting the funnel would also delete attached workflow triggers | Reconnect or use a token with Workflows write access. |
| 401 "API key missing or invalid" | Missing, revoked, or wrong token | Re-authenticate. |

## Branded short links

Manage redirect links independently from funnel pages. Use the existing generic
MCP tools with `resource: "short_link"` in category `funnels`: `list`, `fetch`,
`create`, `update`, and `destroy`. Use `read_action` with `action: "availability"`
or `action: "stats"`; both actions are read-only. A selected workspace, its grant,
and the corresponding `funnels:read` or `funnels:write` scope are required for
scoped tokens. The same authorization rules apply through HTTP.

| Action | Method | Path |
|--------|--------|------|
| List | GET | `/api/v2/workspaces/:workspace_id/url_redirects` |
| Create | POST | `/api/v2/workspaces/:workspace_id/url_redirects` |
| Fetch | GET | `/api/v2/url_redirects/:id` |
| Update | PATCH | `/api/v2/url_redirects/:id` |
| Delete | DELETE | `/api/v2/url_redirects/:id` |
| Check availability | GET | `/api/v2/workspaces/:workspace_id/url_redirects/availability` |
| Click statistics | GET | `/api/v2/url_redirects/:url_redirect_id/stats` |

### Create and maintain a campaign link

1. Choose a ready-for-use domain belonging to the workspace. `domain_id` accepts
   its numeric or public ID; `null` or `0` selects the default-site domain.
2. Check availability with `current_path=/launch` and the chosen `domain_id`.
   A `200` response contains `status` (`available` or `unavailable`), normalized
   `current_path`, and the effective numeric `domain_id`. This reserves nothing.
3. Create with a `url_redirect` body. `original_url` is required on create.

```http
POST /api/v2/workspaces/123/url_redirects
Authorization: Bearer <access_token>
Content-Type: application/json

{"url_redirect":{"original_url":"https://example.com/offer","current_path":"/launch","domain_id":7,"utm_source":"newsletter"}}
```

A successful create returns `201`, a `Location` header and the flat resource.
Repeated POST requests create separate records. Lists use the standard V2 page
size (20). Use `sort_property`, `sort_order` and `after`; follow the `Pagination-Next` or `Link`
response header for the next cursor. Fetch the resource before deciding to create
another link; there is no idempotency key on this endpoint.

Writes accept only `name`, `original_url`, `second_url`, `split_percentage`,
`pixel_code`, `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`,
`current_path`, `domain_id`, and `redirect_type`. `split_percentage` is 0-100.
`redirect_type` is `temporary` (302, default) or `permanent` (301). Omitted update
fields retain their values. Omitted `pixel_code` on create, or an explicit `null`,
becomes an empty string. `workspace_id`, `url_id`, and `site_id` cannot be written.

An explicit nonblank path must be available after normalization. Another
resource's current path or historical alias, or an occupied custom-domain root,
returns `422` without modifying either resource. The dashboard permits some alias
reuse and homepage takeover; the API deliberately rejects these cases. A free
custom-domain root is allowed; the default-site root is invalid. The current path
or a historical alias owned by this same link is allowed. When checking an
existing link, pass its `url_redirect_id` to availability and explicitly choose
its domain; omitted `domain_id` checks the default-site domain.

Omit or leave `current_path` blank on create to generate a path. Omit it on update
to retain the path, or send it blank to generate a new one. Generated paths retain
automatic disambiguation. Path changes retain historical aliases. Deleting a link
removes all its URLs, returns `204`, and a repeated delete returns `404`.

The response contains `id`, `public_id`, `workspace_id`, `domain_id`, every writable
field, `full_url`, `created_at`, and `updated_at`. Share the returned canonical
`full_url`. Domain changes propagate asynchronously, so it can still show the
previous domain until that propagation completes; fetch again to observe it.
When changing `domain_id` with `current_path` omitted, the retained path must be
free or owned by this link on the new domain; otherwise the update returns `422`
without scheduling propagation. An explicitly blank or null `current_path` still
generates a path.
Ownership is checked again during propagation. If another resource claims the
destination before it runs, the previous canonical URL remains in place; choose
a free `current_path` instead of taking that resource's URL.
CRUD responses do not include click analytics.

### Read campaign clicks

```http
GET /api/v2/url_redirects/aBcDef/stats?timerange_start=2026-10-01T00:00:00Z&timerange_end=2026-10-05T00:00:00Z
Authorization: Bearer <access_token>
```

```json
{
  "url_redirect": {"id": 42, "public_id": "aBcDef", "name": "Launch"},
  "timerange": {"from": "2026-10-01T00:00:00Z", "to": "2026-10-05T00:00:00Z"},
  "click_count": 12
}
```

`click_count` counts tracked clicks, including repeated clicks from one visitor,
using the existing short-link bot exclusion. It is zero when there are no clicks.
The default window is the last 30 days and the maximum is 90 days. Future end
values clamp to now; reversed windows reset to the default window ending at the
applied end. Read back `timerange` for the window actually used. Invalid timestamps
return `422`.

Missing required bodies or availability `current_path` return `400`. Missing or
invalid credentials return `401`; insufficient scopes or workspace grants return
`403`. Missing/inaccessible links (including an availability editing target from
another workspace) return `404`. Invalid or foreign domains, invalid redirect
types, model validation failures and explicit path conflicts return `422`. An
unavailable availability check still returns `200` and never identifies its owner.
