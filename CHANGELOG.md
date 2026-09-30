# Changelog

## Unreleased

- Bundle the 19 skills published at `https://accounts.myclickfunnels.com/skill.md`, including page markup, pages, blogs, courses, contacts, opportunities, the store, funnels, workflows, emails, the SDK, and stats. Relative links now point at `https://accounts.myclickfunnels.com`.
- The `clickfunnels` skill now lists the bundled skills.
- Add `scripts/sync-skills.mjs` and a workflow that checks for drift on every pull request and opens a sync pull request every week.

## 1.0.0

- Connect Cursor and Grok Bot to the ClickFunnels MCP server at `https://agents.myclickfunnels.com/mcp`.
- Add a skill for workspace selection, category discovery, and the page hosting check.
