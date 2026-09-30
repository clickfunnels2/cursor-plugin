---
name: clickfunnels-opportunities
description: >
  Track deals, applications, and any staged process against contacts using
  Sales Pipelines, Stages, Opportunities and their Notes - create pipelines
  with ordered stages, open one opportunity per process instance, move it
  through stages, and attach a note timeline. Includes guidance on when a
  pipeline is the right model versus plain contact tags and custom attributes.
  Reads require Opportunities read access; creating, updating and deleting
  require Opportunities write access.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/contacts/skill.md
    description: The contact resource opportunities attach to - tags, custom attributes, bulk actions, list traps.
  - path: https://accounts.myclickfunnels.com/.well-known/workflows/skill.md
    description: Automation - workflows have a create-opportunity action step.
---

# ClickFunnels Opportunities Skill

Opportunities are the process layer of the CRM. A contact records **who someone
is**; an opportunity records **one instance of a process you are running with
them** - a deal, an application, an outreach attempt - positioned in exactly one
stage of a pipeline. This skill covers pipelines, their stages, opportunities,
and opportunity notes.

## When to use a pipeline versus the contact itself

The contact surface (tags, custom attributes) and the pipeline surface
(opportunities in stages) answer different questions. Choosing wrong is not
fatal, but it produces either unqueryable tag soup or a pipeline nobody moves.

**Model it on the contact when it is a lasting property of the person:**

- Segment membership, source, interests -> **tags** (`newsletter-subscriber`,
  `src-linkedin`)
- Facts and free text about the person -> **custom attributes**
  (`signup_date`, `conversation_notes`)

**Model it as an opportunity when it is a process with a direction:**

- It moves through **stages** toward an end state (won, lost, done)
- It can happen **more than once per person** - a second deal, a repeat
  application. Tags cannot count instances; opportunities are rows
- It has a **value**, an owner (`assignee_id`), or a **deadline**
- People act on it from a board - opportunities are what the CRM's kanban
  view renders

Rules of thumb:

- If you would name it with a past-tense verb (`replied`, `purchased`), it is
  probably a stage, not a tag. If you find yourself creating tag *sequences*
  (`lead-new`, `lead-contacted`, `lead-qualified`) and removing one to add the
  next, you are hand-rolling a pipeline - use one.
- If it never progresses (a source, a cohort, a preference), it is a tag. A
  pipeline whose opportunities never move is a tag with extra steps.
- Use **both** together: the opportunity carries the process state; the contact
  keeps the durable evidence (tags for segmentation, custom attributes for
  facts). Broadcasts and workflows segment on the contact surface, so anything
  an email audience needs must live there, not in the pipeline.

## Authentication

All endpoints require an authorized token.

```
Authorization: Bearer <access_token>
```

Reads need Opportunities read access; creating, updating and deleting need
Opportunities write access.

## Route shape

Same rule as contacts, and just as easy to get wrong: **collection routes are
workspace-scoped, member routes are not**. A member route called with the
workspace prefix returns **404**, which reads like "record missing" rather than
"wrong route".

| Action | Method | Path |
|--------|--------|------|
| List pipelines | GET | `/api/v2/workspaces/{workspace_id}/sales/pipelines` |
| Create pipeline | POST | `/api/v2/workspaces/{workspace_id}/sales/pipelines` |
| Fetch / update / delete pipeline | GET / PATCH / DELETE | `/api/v2/sales/pipelines/{id}` |
| List / create stages | GET / POST | `/api/v2/sales/pipelines/{pipeline_id}/stages` |
| Fetch / update / delete stage | GET / PATCH / DELETE | `/api/v2/sales/pipelines/stages/{id}` |
| List opportunities | GET | `/api/v2/workspaces/{workspace_id}/sales/opportunities` |
| Create opportunity | POST | `/api/v2/workspaces/{workspace_id}/sales/opportunities` |
| Fetch / update / delete opportunity | GET / PATCH / DELETE | `/api/v2/sales/opportunities/{id}` |
| List / create notes | GET / POST | `/api/v2/sales/opportunities/{opportunity_id}/notes` |
| Fetch / update / delete note | GET / PATCH / DELETE | `/api/v2/sales/opportunities/notes/{id}` |

Both the numeric `id` and the obfuscated `public_id` work wherever `{id}`
appears. List endpoints paginate with the same cursor mechanism as contacts:
follow the `Pagination-Next` header, never a page counter.

## Pipelines and stages

A pipeline is created **with its stages in one call**. `stages_attributes` is
required and must contain at least one stage - omitting it returns **422
`Pipeline must have at least one stage`**. Stages are created in the order
given.

```http
POST /api/v2/workspaces/{workspace_id}/sales/pipelines
Content-Type: application/json

{
  "sales_pipeline": {
    "name": "Enterprise deals",
    "stages_attributes": [
      {"name": "Qualified",  "close_probability": 20},
      {"name": "Proposal",   "close_probability": 60},
      {"name": "Closed Won", "close_probability": 100},
      {"name": "Closed Lost", "close_probability": 0}
    ]
  }
}
```

The response includes the stages with their generated ids - keep them, every
opportunity write needs a `pipelines_stage_id`. `close_probability` is written
as an integer percentage and comes back as a float (`20` -> `20.0`).

`close_probability` is not cosmetic: each stage's `weighted_value` is the sum
of its opportunities' values multiplied by it, and **a probability of `100`
marks the stage as closing** (see the `closed_at` behavior below). Model
"lost" as a `0` stage, "won" as `100`.

Stages can be added to an existing pipeline later; `sort_order` inserts at a
position and shifts what follows. A new workspace ships with a `Default`
pipeline (New Leads -> ... -> Closed Won / Closed Lost) - list before creating,
you may not need a new one.

**Only empty pipelines can be deleted.** `DELETE /api/v2/sales/pipelines/{id}`
removes the pipeline and its stages (204), but if any opportunities remain in
its stages the request is rejected with **422 `Pipeline must be empty before it
can be deleted`**. Delete the opportunities
(`DELETE /api/v2/sales/opportunities/{id}`) or move them to another pipeline
(`PATCH` with a `pipelines_stage_id` from the target pipeline) first - the API
never cascades over live opportunities. The dashboard's delete, by contrast,
removes a pipeline together with everything in it.

Creating a pipeline or an opportunity **installs the Opportunities app** in the
workspace, so the board appears in the dashboard navigation - the same thing
the dashboard's "Add App" button does. On workspaces where API-created
pipelines predate this behavior, install the app once from the workspace's
Apps page.

## Opportunities

One opportunity = one instance of the process for one contact. `name`,
`pipelines_stage_id` and `primary_contact_id` are required; `value` (in the
workspace currency), `assignee_id` and `closed_at` are optional.

```http
POST /api/v2/workspaces/{workspace_id}/sales/opportunities
Content-Type: application/json

{
  "sales_opportunity": {
    "name": "Acme Corp - annual plan",
    "pipelines_stage_id": 2414675,
    "primary_contact_id": 1506855354,
    "value": 12000
  }
}
```

Moving an opportunity through the pipeline is a PATCH of its stage id:

```http
PATCH /api/v2/sales/opportunities/{id}
Content-Type: application/json

{"sales_opportunity": {"pipelines_stage_id": 2414679}}
```

> **`closed_at` is managed for you - one way.** Creating an opportunity in, or
> moving it into, a stage with `close_probability: 100` sets `closed_at`
> automatically. Moving it **back** to an earlier stage does **not** clear
> `closed_at` - a reopened deal still reads as closed until you null the field
> explicitly in the same or a following PATCH. If you compute "open deals" as
> `closed_at == null`, reopened opportunities silently vanish from the count.

`assignee_id` takes a Team **Membership** id, not a user id. Enumerate them via
`GET /api/v2/teams/{team_id}/memberships` - the users list only returns the
API's own platform-agent user.

### Listing and filtering

`filter[...]` supports `id`, `pipeline_id`, `pipelines_stage_id` and
`primary_contact_id`, each as a comma-separated list; `sort_property` accepts
`id` or `sort_order`. As with contacts, pass `-g` to curl so the brackets are
not globbed:

```bash
# Everything in one pipeline
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/sales/opportunities?filter[pipeline_id]=481917"

# All of one contact's opportunities across pipelines
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/sales/opportunities?filter[primary_contact_id]=1506855354"
```

For stage-by-stage counts, fetching the pipeline is cheaper than listing: each
stage in the response carries its `opportunity_ids`, `total_value` and
`weighted_value`.

## Notes

Notes are the running commentary on one opportunity - calls, decisions, next
steps. The parameter is `content` (not `body` - that returns **422
`Content can't be blank`**):

```http
POST /api/v2/sales/opportunities/{opportunity_id}/notes
Content-Type: application/json

{"sales_opportunities_note": {"content": "Demo done, sending proposal Friday."}}
```

Notes hang off the opportunity and die with it. Durable facts about the person
belong in the contact's custom attributes, where they survive the process
ending and remain visible from every other opportunity.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 404 on an opportunity or stage you just created | Member route called with the `/workspaces/{id}` prefix | Drop the workspace prefix on member routes. |
| 422 `Pipeline must have at least one stage` | `stages_attributes` missing or empty on pipeline create | Send at least one stage in `stages_attributes`. |
| 422 `Content can't be blank` on a note | Parameter named `body` instead of `content` | Use `content`. |
| `curl: (3) bad range in URL` | Shell/curl globbing on `filter[...]` | Add `-g`, or percent-encode the brackets. |
| "Open" counts drop after a deal is reopened | `closed_at` survives moving back to an earlier stage | Null `closed_at` explicitly when reopening. |
| Assignment fails or targets the wrong person | `assignee_id` given a user id | Pass a Team Membership id from `/api/v2/teams/{team_id}/memberships`. |
| Board not visible in the dashboard nav | Opportunities app not installed (pipeline predates auto-install) | Create any pipeline/opportunity via API, or install once from the workspace Apps page. |
| 422 `Pipeline must be empty before it can be deleted` | Pipeline DELETE with opportunities still in its stages | Delete the opportunities, or PATCH them to another pipeline's stage, then retry. |
