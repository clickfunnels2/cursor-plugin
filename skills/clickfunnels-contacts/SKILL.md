---
name: clickfunnels-contacts
description: >
  Manage CF2 contacts and the CRM surface programmatically - create, fetch,
  update, upsert and redact contacts; create and apply tags; run bulk tag,
  unsubscribe and export actions; and list contacts with filters, sorting and
  cursor pagination. Includes a worked recipe for importing a contact list from
  another platform (email service provider, CRM, membership site) idempotently.
  Reads require Contacts read access; creating, updating, tagging, unsubscribing
  and exporting require Contacts write access.
version: "1.0"
author: ClickFunnels
tools:
  - http
references:
  - path: https://accounts.myclickfunnels.com/llms.txt
    description: Quick-reference index - product overview, login, funnels, broadcasts, API docs.
  - path: https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md
    description: Build reusable contact filters, and the `stable_id` / `stored_filter_id` tokens the bulk actions accept.
  - path: https://accounts.myclickfunnels.com/.well-known/emails/skill.md
    description: Broadcasts and templates - the main consumer of an imported contact list.
  - path: https://accounts.myclickfunnels.com/.well-known/workflows/skill.md
    description: Automation triggered by tags applied through this skill.
  - path: https://accounts.myclickfunnels.com/.well-known/opportunities/skill.md
    description: Pipelines, stages and opportunities - the staged-process layer on top of contacts.
---

# ClickFunnels Contacts Skill

Contacts are the CRM records in a workspace. Everything that emails, enrolls,
segments or sells to a person resolves to a contact. This skill covers the
contact resource itself, its tags, and the bulk actions that operate over a
selection of contacts.

## When to use this skill

- Importing a subscriber or customer list from another platform
- Creating or updating a single contact from an integration
- Tagging contacts for segmentation, then handing the tag to a broadcast or workflow
- Bulk-unsubscribing a set of contacts, or honoring an opt-out carried over from another system
- Exporting the workspace's contacts to CSV
- Listing or searching contacts by email, tag, or id

For building reusable audience filters, see the
[Refine Filters Skill](https://accounts.myclickfunnels.com/.well-known/refine-filters/skill.md). For sending to the
list once it exists, see the [Emails Skill](https://accounts.myclickfunnels.com/.well-known/emails/skill.md).

Tags and custom attributes describe **lasting properties of a person**. The
moment you are tracking a *process* - something that moves through stages
toward won/lost/done, can recur per person, or carries a value and an owner -
model it as an opportunity in a pipeline instead of a tag sequence. The
decision guide and the pipeline API live in the
[Opportunities Skill](https://accounts.myclickfunnels.com/.well-known/opportunities/skill.md).

## Authentication

All endpoints require an authorized token.

```
Authorization: Bearer <access_token>
```

Reads need Contacts read access. Creating, updating, upserting, tagging,
unsubscribing, exporting and redacting need Contacts write access.

## Route shape - read this before anything else

Contact routes follow a rule that is easy to get wrong:

- **Collection routes are workspace-scoped**: `/api/v2/workspaces/{workspace_id}/contacts`
- **Member routes are NOT workspace-scoped**: `/api/v2/contacts/{id}`

This holds for tags and for the bulk actions too. Calling a member route with
the workspace prefix returns **404**, which reads like "this record does not
exist" rather than "wrong route". If you get an unexpected 404 on a record you
just created, check the prefix before concluding the record is missing.

| Action | Method | Path |
|--------|--------|------|
| List contacts | GET | `/api/v2/workspaces/{workspace_id}/contacts` |
| Create contact | POST | `/api/v2/workspaces/{workspace_id}/contacts` |
| Upsert contact | POST | `/api/v2/workspaces/{workspace_id}/contacts/upsert` |
| Fetch contact | GET | `/api/v2/contacts/{id}` |
| Update contact | PATCH | `/api/v2/contacts/{id}` |
| Delete contact | DELETE | `/api/v2/contacts/{id}` |
| Redact contact (GDPR) | DELETE | `/api/v2/contacts/{id}/gdpr_destroy` |
| List tags | GET | `/api/v2/workspaces/{workspace_id}/contacts/tags` |
| Create tag | POST | `/api/v2/workspaces/{workspace_id}/contacts/tags` |
| Fetch / update / delete tag | GET / PATCH / DELETE | `/api/v2/contacts/tags/{id}` |
| Bulk tag action | POST | `/api/v2/workspaces/{workspace_id}/contacts/tag_actions` |
| Bulk unsubscribe action | POST | `/api/v2/workspaces/{workspace_id}/contacts/unsubscribe_actions` |
| Bulk export action | POST | `/api/v2/workspaces/{workspace_id}/contacts/export_actions` |
| Poll any action | GET | `/api/v2/contacts/{action_type}/{id}` |

Both the numeric `id` and the obfuscated `public_id` work wherever `{id}`
appears.

## Listing contacts

### Pagination is cursor-based

There is **no `page` or `per_page` parameter**. The list endpoint returns a
fixed page and a cursor:

- `Pagination-Next` response header - the id to pass as `after` on the next call
- `Link` response header - the full next-page URL, `rel="next"`

Stop when the response body is an empty array, or when `Pagination-Next` is
absent.

> **Trap:** unrecognized top-level query parameters are silently ignored rather
> than rejected. `?per_page=100` returns a normal 200 with the default page
> size, so an agent that assumes it worked will silently process a fraction of
> the list and report success. Always drive pagination from the
> `Pagination-Next` header, never from a page counter.

```bash
# Walk the whole list
AFTER=""
while :; do
  URL="https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts"
  [ -n "$AFTER" ] && URL="${URL}?after=${AFTER}"
  BODY=$(curl -sS -D /tmp/h.txt -H "Authorization: Bearer ${TOKEN}" "$URL")
  [ "$BODY" = "[]" ] && break
  echo "$BODY" | jq -c '.[]'
  AFTER=$(grep -i '^pagination-next:' /tmp/h.txt | tr -d '\r' | awk '{print $2}')
  [ -z "$AFTER" ] && break
done
```

### Filters and sorting

| Parameter | Meaning |
|-----------|---------|
| `filter[email_address]` | Comma-separated email list |
| `filter[id]` | Comma-separated contact id list |
| `filter[tag_ids]` | Comma-separated tag id list |
| `filter[is_active]` | Boolean |
| `stored_filter_id` | Id or public id of a saved ContactsFilter |
| `stable_id` | URL-encoded Refine token |
| `sort_property` | `id` (default) or `updated_at` |
| `sort_order` | `asc` (default) or `desc` |
| `expand[]=email_engagement` | Include email engagement timestamps |

Unknown keys **inside** `filter` are rejected with 400 (`found unpermitted
parameter`), unlike unknown top-level parameters. Do not read a successful
response as proof that your top-level parameter was understood.

> **The trap that costs you a whole audience.** `stored_filter_id` and
> `stable_id` are **top-level** parameters. Written as a nested key -
> `?filters[stored_filter_id]=...` - they are just an unknown top-level
> parameter, so they are silently ignored and you get back the **first page of
> every contact in the workspace**, 200 OK, no warning. Nested under the
> singular `filter[...]` they are rejected outright. Only the bare form filters:
>
> | Written as | Result |
> |------------|--------|
> | `?stored_filter_id=dWPznD` | Filtered correctly |
> | `?stable_id=<token>` | Filtered correctly |
> | `?filters[stored_filter_id]=dWPznD` | 200, silently **unfiltered** |
> | `?filter[stored_filter_id]=dWPznD` | 400 `found unpermitted parameter` |
>
> The dangerous row is the third. If you feed those ids into a bulk tag or
> unsubscribe action you act on the whole workspace instead of your segment.
> Check the returned count against what you expect before any bulk write.

> **curl gotcha:** `curl` treats `[` and `]` as glob ranges and fails with
> `bad range in URL`. Pass `-g` (`--globoff`), or percent-encode as `%5B`/`%5D`.

```bash
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts?filter[tag_ids]=448335"
```

## Upserting a contact

`POST /api/v2/workspaces/{workspace_id}/contacts/upsert` matches on
`email_address`: it creates when there is no match (**201**) and updates when
there is (**200**). This is the endpoint to use for imports and for any
integration that may deliver the same person twice.

```http
POST /api/v2/workspaces/{workspace_id}/contacts/upsert
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "contact": {
    "email_address": "jane@example.com",
    "first_name": "Jane",
    "last_name": "Doe",
    "phone_number": null,
    "tag_ids": [448334, 448337],
    "custom_attributes": {
      "source_platform": "acme-esp",
      "signup_date": "2026-06-07"
    }
  }
}
```

Behavior worth knowing before you rely on it:

- `tag_ids` **overwrites** the contact's tags. Passing a partial array drops the
  tags you left out. To add tags without disturbing existing ones, use a bulk
  tag action with `action_type: "add"` instead.
- `custom_attributes` merges: new keys are created, existing keys are updated.
- Empty values do not clear fields. Null and `[]` are ignored rather than
  applied. Use `PATCH /api/v2/contacts/{id}` to actually reset a property.
- `custom_attributes` values are stored and returned as strings. Send numbers as
  strings and parse on the way out.

## Tags

Tag names are unique per workspace. Creating a duplicate returns **422
`Name has already been taken`**, and the error body does **not** include the id
of the tag that already exists, so there is no find-or-create shortcut. List the
tags first and build a name-to-id map:

```bash
curl -sS -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts/tags" \
  | jq 'map({(.name): .id}) | add'
```

```http
POST /api/v2/workspaces/{workspace_id}/contacts/tags
Content-Type: application/json

{"contacts_tag": {"name": "newsletter-subscriber", "color": "#4F46E5"}}
```

## Bulk actions

Tag, unsubscribe and export all share one shape. Supply **exactly one** target
selector:

| Selector | Value |
|----------|-------|
| `target_ids` | Array of contact `public_id` strings |
| `stored_filter_id` | Id of a saved ContactsFilter |
| `stable_id` | URL-encoded Refine token |
| `target_all` | `true` for every contact in the workspace |

They run asynchronously. The POST returns **201** with `target_count: null` and
`performed_count: 0`; poll the member route until `completed_at` is set.

```http
POST /api/v2/workspaces/{workspace_id}/contacts/tag_actions
Content-Type: application/json

{
  "contacts_tag_action": {
    "tag_ids": ["eWqAym"],
    "action_type": "add",
    "target_ids": ["bZMZAQQ", "YomoAxl"]
  }
}
```

`action_type` is `add` (default) or `remove`. Note the asymmetry: `tag_ids` here
takes tag **public_id** strings, while `tag_ids` on the contact upsert takes
**numeric** ids.

```bash
# Poll until done
curl -sS -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/contacts/tag_actions/122414" \
  | jq '{target_count, performed_count, completed_at}'
```

Unsubscribe uses `contacts_unsubscribe_action` and takes no `tag_ids`. Export
uses `contacts_export_action` and adds a `file_url` to the record once complete.

> `fields` is an allow-list of columns and it accepts custom-attribute keys
> alongside the built-in ones. Asking for `["email_address", "first_name",
> "last_name", "source_platform"]` returns a CSV headed
> `Email address,First name,Last name,Source Platform` with the custom attribute
> populated. Anything you do not name is simply absent, so list every column you
> want - an export is only as complete as its `fields` array.

## Recipe - importing contacts from another platform

This is the general shape for moving a list off an email service provider, CRM,
membership site or course platform. It is idempotent, so a failed run can be
re-run without creating duplicates.

### 1. Extract from the source, and keep the source's own state

Pull the full list from the source API. Capture more than email and name: opt-out
status, engagement counts, signup date, and any labels or segments. Those are the
fields that let you rebuild segmentation on the ClickFunnels side, and they are
the fields people most often drop and regret.

### 2. Read the destination first

List the existing contacts in the target workspace and build a set of
lowercased email addresses. Diff the source against it so the import is a
deliberate set, not a blind replay. Report the counts before writing anything:
total in source, already present, to be created, deliberately skipped.

### 3. Map segments to tags before importing

Create the tags first and hold a name-to-id map. Three kinds are worth having:

- **A list tag** (`newsletter-subscriber`) - what this list *is*
- **A provenance tag** (`acme-migration-2026-08-02`) - where it came from and when, so a bad import can be found and reversed
- **Segment tags** - one per meaningful source label or segment

### 4. Upsert one contact at a time

There is no batch-create endpoint. Contacts are imported with one HTTP request
each. Pace them (roughly 100 ms apart is comfortable) and expect a few transient
failures on a list of any size.

Map source fields onto:

- `email_address`, `first_name`, `last_name`, `phone_number`
- `tag_ids` - the list tag, the provenance tag, and any segment tags
- `custom_attributes` - engagement counts, source id, signup date, country

### 5. Retry, because you will need to

Transient **502s** happen on longer runs and the body is an **HTML error page,
not JSON**. A client that assumes JSON will raise a parse error on a response it
should simply retry. Branch on the status code, not on parseability.

There are no `RateLimit-*` or `Retry-After` headers to guide backoff, so use a
fixed small delay plus exponential retry. Because upsert is idempotent, retrying
is always safe. Record per-row status and re-drive only the failures.

```python
def upsert(session, ws, token, contact):
    r = session.post(
        f"https://api.myclickfunnels.com/api/v2/workspaces/{ws}/contacts/upsert",
        headers={"Authorization": f"Bearer {token}",
                 "Content-Type": "application/json"},
        json={"contact": contact}, timeout=30)
    return r.status_code, r  # 201 created, 200 updated

for row in rows:
    for attempt in range(4):
        status, resp = upsert(session, ws, token, build(row))
        if status in (200, 201):
            break
        time.sleep(2 ** attempt)      # 502s are transient; upsert is idempotent
    else:
        failures.append(row["email"])
    time.sleep(0.12)
```

### 6. Carry opt-outs across

An import that silently re-subscribes people who had opted out on the old
platform is a deliverability and compliance problem. Collect the opted-out
addresses, import them like everyone else, then bulk-unsubscribe them in one
action:

```http
POST /api/v2/workspaces/{workspace_id}/contacts/unsubscribe_actions
Content-Type: application/json

{"contacts_unsubscribe_action": {"target_ids": ["OaEadgo"]}}
```

Verify afterwards that `unsubscribed_at` is set on those contacts.

### 7. Reconcile

Do not trust the write responses alone. Walk the list endpoint with cursor
pagination and assert:

- Total contact count equals the pre-import count plus the number created
- The provenance tag count equals the number of rows you intended to import
- Each segment tag count matches the source
- The opted-out contacts have `unsubscribed_at` set

A bulk export action is a useful independent cross-check on the total. Name the
provenance attribute in `fields` and the CSV carries it, so the export doubles as
a verification that the custom attributes landed.

## Recipe - cleaning contacts out of a CRM

Removing contacts is how a workspace stays useful: an imported list that never engaged, a
segment from a platform you no longer use, contacts who unsubscribed long ago. It is also
the most destructive thing in this API and none of it is reversible.

**The rule for an agent running this: never delete anything the person has not seen a count
for and said yes to.** Confirm at every step below, with numbers, and state plainly what is
about to be removed. Batch deletes are not a place to infer intent.

### Step 1 - decide the method BEFORE you touch anything

This is the first question, not the last, because it changes what happens to the records and
cannot be undone either way:

| | `DELETE /api/v2/contacts/{id}` | `DELETE /api/v2/contacts/{id}/gdpr_destroy` |
|---|---|---|
| What it does | **Soft delete.** Hides the contact from lists and filters and stamps `deleted_at`; the record and its personal data stay readable at `GET /api/v2/contacts/{id}` | Overwrites email with `redacted-<hash>@example.com` and names with `REDACTED` |
| Use when | Cleaning up junk, test data, an import you regret | A person exercised a right-to-be-forgotten request |
| Erases personal data | **No** | Yes, except tags and custom attributes |
| Suppression signal | Lost - a later re-import can re-add them as subscribed | Lost in the same way |

Ask which one applies and get an explicit answer. If the goal is only "stop emailing them",
the answer is usually **neither** - unsubscribe instead (see the end of this section).

### Step 2 - define the audience, and show its size next to the whole

Build one filter and reuse it. Then report the count *as a fraction of the workspace*, because
"96 contacts" and "96 of 126 contacts" are very different decisions:

```bash
# everything in the workspace
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts" | jq length

# just the audience you intend to remove
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts?filter[tag_ids]=422201" | jq length
```

Both are cursor-paginated, so walk every page before quoting a number (see Pagination). A
count taken from the first page is wrong and will be wrong in the direction that makes the
deletion look small.

Present it and stop:

```
Tag "readme forum import": 96 contacts, of 126 in the workspace (76%).
Of those: 3 unsubscribed, 3 inactive, 95 with no recorded visit.
Delete all 96, or a narrower slice?
```

### Step 3 - check what else the audience is attached to

A contact is not just an email address. Before removing them, look for anything that makes
the record worth keeping, and report what you find:

- **Orders.** `GET /api/v2/workspaces/{workspace_id}/orders` - a contact with order history
  is almost never safe to delete; deleting loses the customer record behind a real payment.
- **Other tags.** A contact carrying tags beyond the one you filtered on probably belongs to
  another segment too. List the distinct tag combinations in the audience.
- **Recent activity.** `visits.last_visit` being set means they came back.

Report the overlaps and let the person narrow the audience. "All 96 also carry the tag
`Import - 05/01/26`" is the kind of fact that changes a decision.

### Step 3b - check engagement before you believe the audience is dead

`expand[]=email_engagement` puts `last_email_sent_at`, `last_email_opened_at` and
`last_email_clicked_at` on every contact, and `visits.last_visit` shows whether they ever
came to a page:

```bash
curl -sS -g -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/workspaces/${WS}/contacts?filter[tag_ids]=422201&expand[]=email_engagement"
```

> **If `last_email_sent_at` is null across the audience, stop.** A list that has never been
> emailed is not an unengaged list, it is an untested one - there is no signal to clean on,
> because nothing was ever sent to generate one. Deleting here throws away leads on the
> basis of an absence of data you created. Send to them once, wait, and let the engagement
> data decide.

The same logic applies in reverse: an audience with sends but no opens over several
campaigns is a genuinely evidence-backed removal candidate, and you can say so with numbers.

### A caution on "delete everyone who isn't in our community"

A tempting rule is "keep the contacts who joined our Discord / Slack / forum, remove the
rest". It usually cannot be implemented. Those platforms do not expose member email
addresses, so the only thing to match on is a display name against a name or an email local
part, and display names are nicknames. Matching 18 community members against 96 contacts in
one real run produced **2** confident matches; a fuzzy pass over the other 16 returned noise
that a human still had to adjudicate one by one.

Treat community membership as something you **confirm by hand for a short list**, or collect
deliberately (ask for the email at join time, tag on arrival). Never let an automated
name-match drive an irreversible delete: the false positives are exactly your most engaged
people, because they are the ones in both places.

### Step 4 - narrow to the actual intent

Most cleanups are not "delete the whole tag". Common narrower slices, each one filter:

- unsubscribed only - the people who already told you to stop
- no recorded visit AND older than N months - imported and never engaged
- inactive only

Re-run Step 2's count for the narrowed audience and confirm the new number.

### Step 5 - dry run, on the list itself

Export the exact audience before deleting it, so there is a record of what was removed:

```http
POST /api/v2/workspaces/{workspace_id}/contacts/export_actions
{"contacts_export_action": {"stored_filter_id": "dWPznD",
  "fields": ["email_address", "first_name", "last_name", "unsubscribed_at"]}}
```

Poll it, download the CSV, and confirm `target_count` matches the number agreed in Step 4. If
those two numbers disagree, stop - the filter is not selecting what you think it is.

### Step 6 - delete in small batches, and re-count between them

There is no bulk delete endpoint. It is one call per contact:

```bash
curl -sS -X DELETE -H "Authorization: Bearer ${TOKEN}" \
  "https://api.myclickfunnels.com/api/v2/contacts/{id}"   # 204 No Content
```

Do the first 5, re-run the Step 2 count, and show it before continuing. Then work in batches
with the count reported after each. A 502 mid-run is transient (see Common errors) - retry
that id rather than restarting, and never re-run a whole batch blindly, because ids already
deleted will 404 and mask a real failure.

### Step 7 - report what actually happened

Deleted count, failed ids, the new workspace total, and where the CSV from Step 5 lives.

### Prefer unsubscribing when the goal is "stop emailing them"

Unsubscribing stops delivery, preserves history, and keeps the suppression signal so a later
import of the same list cannot re-add the person as subscribed. It is also a bulk action
rather than N calls:

```http
POST /api/v2/workspaces/{workspace_id}/contacts/unsubscribe_actions
{"contacts_unsubscribe_action": {"stored_filter_id": "dWPznD"}}
```

Deleting an unsubscribed contact throws that protection away. Say so before doing it.

## Deleting versus redacting

These are two different operations and the difference matters.

**`DELETE /api/v2/contacts/{id}` is a soft delete.** It returns `204`, and the
contact disappears from the list endpoint and from every filter. But the record
survives: `GET /api/v2/contacts/{id}` still returns **200 with the full contact**
- email address, name, tags, custom attributes.

Read `deleted_at` to tell a deleted contact from a live one. It is null until the
contact is deleted and carries the deletion timestamp afterwards, and it is the
only field that means only that. Do not use `is_active`: it also goes `false` for
an unsubscribe, an email suppression, or a missing email address, so a `false`
there does not tell you which of those happened.

> **Do not use `DELETE` to satisfy an erasure request.** Verifying through the
> list endpoint shows the contact gone while the personal data is still readable
> to anyone holding the id.

A delete can also be refused. `DELETE` returns **422** with the reason when the
contact is the workspace's default contact, has an active subscription order, or
has course enrollments. Cancel the subscription, or pick the redaction route
below, and retry. A refused delete leaves the contact exactly as it was, so
check the status rather than assuming the call worked.

**`DELETE /api/v2/contacts/{id}/gdpr_destroy` is the redaction.** The route needs
the full `gdpr_destroy` - a bare `/gdpr` returns 404 `No such route`. It
overwrites the email address with a hashed `redacted-<hash>@example.com`, sets
first and last name to `REDACTED`, and clears the phone number. Documented as
`200`; observed `204`. Accept either.

The contact row itself remains after redaction, and so do its **tags and custom
attributes**. If you park source-system data in `custom_attributes` during an
import, redaction will not clear it. Keep anything identifying out of custom
attributes for that reason.

Both operations are irreversible.

Prefer unsubscribing over deleting when the goal is to stop sending email.
Unsubscribing stops delivery, preserves history, and keeps the record available
if the decision was wrong. Deleting loses the suppression signal, which means a
later import of the same list can re-add the person as subscribed.

## Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| 404 on a contact or tag you just created | Member route called with the `/workspaces/{id}` prefix | Drop the workspace prefix on member routes. |
| `curl: (3) bad range in URL` | Shell/curl globbing on `filter[...]` | Add `-g`, or percent-encode the brackets. |
| 400 `found unpermitted parameter: :x` | Unknown key inside `filter` | Use an allow-listed filter key. |
| 422 `Name has already been taken` | Tag name already exists in the workspace | List tags and reuse the existing id. |
| 502 with an HTML body | Transient upstream error | Retry on status code; do not parse the body as JSON. |
| Only a fraction of contacts returned | Used `page` / `per_page`, which are ignored | Paginate with `after` + the `Pagination-Next` header. |
| Tags disappeared after an upsert | `tag_ids` overwrites | Use a bulk tag action with `action_type: "add"`. |
| A field would not clear | Upsert ignores empty values | Use `PATCH /api/v2/contacts/{id}`. |
| Deleted contact still returns 200 on fetch | `DELETE` is a soft delete | Expected. Check `deleted_at` to tell it apart, or use `gdpr_destroy` if the data must actually go. |
| 422 on `DELETE /api/v2/contacts/{id}` | Workspace default contact, an active subscription order, or course enrollments | Read the message. Cancel the subscription, or redact instead of deleting. |
| 404 `No such route` on redact | Path is `/gdpr_destroy`, not `/gdpr` | Use the full segment. |
