---
name: clickfunnels-forms
description: >
  Read complete ClickFunnels survey responses and current questions, map them to Form
  fields and choices, correct stored answers, and receive each completed response by webhook.
  Use when exporting survey answers or configuring an integration's field mappings.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Platform overview and authentication entry points.
  - path: https://accounts.myclickfunnels.com/.well-known/contacts/skill.md
    description: Read contact identity separately when the integration needs it.
---

# ClickFunnels Forms and Surveys Skill

Discover a survey's current questions before submissions, retrieve full saved responses,
and receive the full response when a respondent completes a survey. Use the linked Form resources for
supported labels, choice labels and stored-answer corrections.

## When to use this skill

- Export every stored answer, including repeated-field answers and anonymous respondents.
- Configure field mappings before the first survey submission.
- Correct a saved answer or rename a supported label through the existing Form API.
- Receive each completed response by webhook.

## Authentication

Use an authorized API/OAuth bearer token granted the target workspace. Survey reads use
`forms:read`; existing Form corrections require `forms:write`. Webhook endpoint management
requires `webhooks:write`. Read contact identity separately with Contacts permission when
needed. Do not assume that an anonymous survey response has supplied identifying details.

For MCP, call `describe_category` with `forms`. The `survey_workflow` and
`survey_workflow_run` resources expose only `list` and `fetch`. List runs with the survey
ID as `parent_id`. Form edits use the existing `form_field_set`, `form_field`,
`form_field_option`, and `form_answer` resources discovered in the same category.

## Endpoint reference

| Action | Method | Path |
| --- | --- | --- |
| List surveys with their current questions | GET | `/api/v2/workspaces/:workspace_id/survey_workflows` |
| Read current questions and choices | GET | `/api/v2/survey_workflows/:id` |
| List full saved responses | GET | `/api/v2/survey_workflows/:survey_workflow_id/runs` |
| Fetch a full saved response | GET | `/api/v2/survey_workflows/runs/:id` |
| Change a field label | PATCH | `/api/v2/forms/fields/:id` |
| Change a contact-info title | PATCH | `/api/v2/forms/field_sets/:id` |
| Add a choice label | POST | `/api/v2/forms/fields/:field_id/options` |
| Rename a choice label | PATCH | `/api/v2/forms/fields/options/:id` |
| Correct a saved answer | PATCH | `/api/v2/forms/submissions/answers/:id` |
| Manage outgoing subscriptions | POST | `/api/v2/workspaces/:workspace_id/webhooks/outgoing/endpoints` |

GETs have no request body. Numeric IDs and public IDs work in paths. Lists are bare arrays;
follow Link / Pagination-Next with `after` until no next page remains. `per_page` defaults
to 20 and is capped at 100. Disabled and archived surveys remain readable. Runs from the
survey builder's preview have `preview: true` and send no webhook.

## Read the survey catalog

```http
GET /api/v2/workspaces/3/survey_workflows
Authorization: Bearer YOUR_TOKEN
```

Each listed survey already includes its `questions`. Fetch one survey by ID:

```http
GET /api/v2/survey_workflows/235
Authorization: Bearer YOUR_TOKEN
```

A survey contains its details (`id`, `public_id`, `name`, `site_id`, `form_id`,
`enabled`, `archived`, timestamps) and `questions`. Questions include every branch in ID
order, not a respondent's traversal path. A question has its own ID, `type`, `title`,
`field_set_id`, and `fields`. A contact-info question contains multiple fields. Fields
include their IDs, key, label, stored field type, contact attribute, multiple-choice flag,
and current presentable options with label and image information. Selected hidden Other
choices can still appear in saved answers even when absent from this available catalog.

## Read entire responses

```http
GET /api/v2/survey_workflows/235/runs?filter[status]=completed
Authorization: Bearer YOUR_TOKEN
```

The list returns every run of the survey. `filter[status]` takes one of active, paused,
completed and canceled; `filter[contact_id]` takes comma-separated contact IDs and combines
with the status filter. A paused run commonly waits at
a question; browser abandonment does not record cancellation. Use status, since
`completed_at` is also populated for canceled runs.

```http
GET /api/v2/survey_workflows/runs/136
Authorization: Bearer YOUR_TOKEN
```

Each run includes `id`, `public_id`, `survey_workflow_id`, nullable `contact_id`, status,
completion/cancellation times, timestamps, and all associated `form_submissions`. Each
submission has IDs, `form_id`, `contact_id` and every currently stored answer. Answer rows
include IDs, a `field` object with its ID, key and current label, nullable text `entry`,
selected option IDs/current labels, and creation/update times. Choices merge single-choice
and multi-choice storage.

Answers are ordered by creation time descending and then ID descending within each
submission. Repeated answers are retained. Editing an older row does not move it above a
newer-created row. Consumers can choose the latest row per field, or use updated_at to
apply their own correction policy. No nested answer request or truncation is involved.

Submissions can be associated with several runs. An answer has no originating run, so
never infer ownership from its timestamp. Labels are current; deleted rows and overwritten
old values are absent. This is a saved-data export, not an immutable audit history.

## Supported Form corrections

Choose IDs from the returned catalog and run, then update the appropriate existing Form
resource. The question ID, field ID and answer ID are different identifiers.

```http
PATCH /api/v2/forms/fields/331
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json
```

```json
{"forms_field":{"label":"What would you like to grow?"}}
```

```http
PATCH /api/v2/forms/submissions/answers/151
Authorization: Bearer YOUR_TOKEN
Content-Type: application/json
```

```json
{"forms_submissions_answer":{"entry":"Grow my consulting business"}}
```

For a contact-info question title use `forms_field_set.title`; use individual field labels
for its subfields. Option-label writes use the `forms_fields_option` root. Multi-choice
answer corrections use `fields_option_ids` (empty array clears them); legacy single-choice
uses `fields_option_id` (null clears it). Clear obsolete single-choice storage when replacing
it with a multi-choice correction, or vice versa.

Creating/reordering Form fields does not create/reorder survey question steps. Use the
survey builder for graph, branching, order and type changes. Generic field-type writes do
not perform a supported survey-type conversion. Form naming does not rename the survey;
question descriptions, contact-capture settings, choice images/scores/status and survey
lifecycle are not available through these generic Form writes.

Saved-answer corrections send no webhook and do not progress, identify, opt in or complete
a respondent. A new generic Form submission is not automatically linked
to a survey run. These structured submissions are distinct from page-submission logs.
Deleting fields, choices or submissions can remove stored answer data or selections.

## Completion webhook

Discover exact subscription keys at `GET /api/v2/webhooks/outgoing/event_types`.

| Event | Trigger |
| --- | --- |
| `survey_workflow_run.completed` | A respondent completes the survey. Not sent for cancellation or builder previews. |

The webhook's `data` is the full run returned by the single-run endpoint, captured when the
run completes. Anonymous respondents are included. Later answer corrections send no webhook;
re-fetch the run when you need the current saved answers. Deduplicate deliveries by
event_id; retries keep their stored payload.

## Worked example

1. List surveys in the granted workspace and choose the desired ID.
2. Fetch its current questions. Map each question's fields/options to your destination,
   retaining their stable IDs and the Form/field-set relationships.
3. List the survey's runs with `filter[status]=completed`, following pagination; retain all
   answer rows or choose your own latest policy.
4. Subscribe to `survey_workflow_run.completed`, using the existing outgoing endpoint API.
5. Correct a saved answer through Form resources when needed, then re-fetch its run to
   verify the saved value. Use event_id to make your receiver idempotent.

The [Forms and surveys guide](https://developers.myclickfunnels.com/docs/forms)
contains complete curl examples, response payloads and relationship/sequence diagrams.

## Common errors

| Status | Cause | Fix |
| --- | --- | --- |
| 401 | Missing/invalid bearer | Authenticate with a valid token. |
| 403 | Missing Forms access or workspace grant | Follow the response hint and request the narrow permission/grant. |
| 404 | Missing or inaccessible survey or run | Use the returned survey/run IDs from the granted workspace. |
| 422 | Invalid status | Use one of active, paused, completed or canceled. |
