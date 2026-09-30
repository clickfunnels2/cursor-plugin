---
name: clickfunnels
description: Use when creating or changing anything in a ClickFunnels workspace — funnels, pages, contacts, products, orders, emails, courses, automations, or analytics. Connects through the ClickFunnels MCP server.
---

# ClickFunnels

The ClickFunnels MCP server is a thin set of generic tools over the workspace. It does not publish a tool per resource. Discover the surface, then call the generic verbs.

The signed-in user must administer at least one team or workspace. A call runs as that user, in one workspace at a time.

## Pick the workspace first

Call `current_workspace` before the first write. When the user has more than one workspace, call `list_workspaces` and `switch_workspace` so later calls hit the workspace they named. Pass `workspace_id` through exactly as returned. These ids are opaque strings.

## Discover, then act

1. Call `describe_category` with one category. The tool's input schema lists the categories. They cover contacts, workflows, funnels, sites, courses, emails, blog, community, appointments, opportunities, store, forms, media, webhooks, account, and reference data.
2. Read that category's resources. Each resource names the verbs it accepts, the attributes those verbs take, and any extra actions.
3. Call `list`, `fetch`, `create`, `update`, or `destroy` with that resource key. Call `read_action` or `take_action` for anything the resource lists as an action.

Do not invent tool names such as `list_contacts`. A field that `describe_category` does not list for that verb is not accepted. A field marked update-only is rejected on create.

## Confirm before building a page

Before creating a funnel or a page, tell the user which hosting model you intend to use and wait for a yes:

- `external_page` when you can build and host the page yourself
- `custom_html_page` when you can produce a full HTML document and ClickFunnels should serve it on the funnel domain
- `page` when the page should stay editable in the ClickFunnels editor. Its body is Page Markup (PML), not HTML

Writing `markup` replaces the whole page. Read the page-markup guide and get approval before changing a page that already has content.

Creating a split moves the anchored step into a branch. Read the funnel's `structure` action immediately before the create, and again after, and do not reuse step ids from an earlier read.

## Guides

Read the guide for the area before the first write in that area. Guides and OAuth metadata are canonical on the accounts host:

- Product overview: https://accounts.myclickfunnels.com/llms.txt
- Skill index: https://accounts.myclickfunnels.com/.well-known/agent-skills/index.json
- Page markup: https://accounts.myclickfunnels.com/.well-known/page-markup/skill.md
- Developer hub: https://developers.myclickfunnels.com
