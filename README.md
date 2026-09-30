# ClickFunnels

Cursor plugin that connects an agent to [ClickFunnels](https://www.clickfunnels.com) through the app-native [MCP](https://modelcontextprotocol.io/) server. The same listing appears in the Grok Bot Marketplace, which installs Cursor marketplace plugins.

Build funnels and pages, manage contacts, products, orders, and subscriptions, and run email and automations in the signed-in workspace.

## Install

After the plugin is listed:

1. Open **Marketplace** in Cursor or Grok Bot.
2. Search for **ClickFunnels**.
3. Choose **Add**.
4. Sign in when the browser prompt opens.

The connecting ClickFunnels user has to administer at least one team or workspace. No API key is required.

To try the plugin before it is listed, symlink this repo and restart the app:

```bash
mkdir -p ~/.cursor/plugins/local
ln -sfn "$(pwd)" ~/.cursor/plugins/local/clickfunnels
```

A team admin on a Cursor Teams or Enterprise plan can distribute it without a public listing. In **Dashboard → Plugins & MCPs**, choose **Import from Repo** and paste this repository URL. `.cursor-plugin/marketplace.json` points at this directory, so the import finds the one plugin.

## Authentication

The server is `https://agents.myclickfunnels.com/mcp` (Streamable HTTP). Clients discover OAuth 2.1 with PKCE from that host, including dynamic client registration, so this repo has no client id or client secret. The first tool call opens the ClickFunnels consent screen.

Tokens are issued for the signed-in user. Disconnecting the plugin revokes them.

## MCP

```json
{
  "mcpServers": {
    "clickfunnels": {
      "type": "http",
      "url": "https://agents.myclickfunnels.com/mcp"
    }
  }
}
```

## What the agent can do

Tools are generic: `describe_category`, then `list`, `fetch`, `create`, `update`, `destroy`, `read_action`, and `take_action`. Categories cover contacts, funnels, pages, courses, emails, automations, the store, and the rest of the public API.

The developer hub is [developers.myclickfunnels.com](https://developers.myclickfunnels.com).

## Skills

`skills/clickfunnels/` is written in this repo. It tells the agent to pick a workspace first, to confirm a page's hosting model before writing one, and to get approval before overwriting a page that already has content. It also lists the other skills.

Every other directory under `skills/` is a copy of a skill published at [accounts.myclickfunnels.com/skill.md](https://accounts.myclickfunnels.com/skill.md). Don't edit those copies. Change the published skill, then re-sync:

```bash
node scripts/sync-skills.mjs          # fetch the published skills and rewrite skills/
node scripts/sync-skills.mjs --check  # exit 1 if skills/ has drifted
```

The script reads the [Agent Skills index](https://accounts.myclickfunnels.com/.well-known/agent-skills/index.json) and downloads each skill. It checks each file against the digest in the index, then rewrites relative links to absolute `https://accounts.myclickfunnels.com` URLs. It records the published digests in `skills.lock.json` and regenerates the skill list in `skills/clickfunnels/SKILL.md`. It needs Node 18 or later and has no dependencies.

`.github/workflows/sync-skills.yml` runs the check on every pull request and push to `main`. Every Monday, and on manual dispatch, it runs the sync and opens a pull request if anything changed.

## Logo

`assets/logo.svg` is the current ClickFunnels mark on a white tile, so it stays readable in the marketplace grid.

## License

MIT
