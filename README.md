# SquidCI for Claude Code

> **Deprecated.** SquidCI's MCP server now teaches an agent everything this
> plugin did — the pipeline file's rules, its tools, starting runs — and
> reaches every client, Claude Managed Agents included. Uninstall the plugin
> with `/plugin uninstall squidci@squidci`, then connect the server directly:
>
> ```bash
> claude mcp add --transport http squidci-eu-west-2 https://api.eu-west-2.squidci.cloud/mcp
> ```
>
> See https://squidci.com/docs/mcp/.

A Claude Code plugin marketplace with one plugin, `squidci`: a skill for
writing and changing SquidCI pipeline files, `.squidci/pipelines.yaml`, and
SquidCI's MCP server, whose read-only tools show an agent the organization's
runs, logs, job usage, environments, versions and deployments.

```text
/plugin marketplace add SquidCI/claude-plugin
/plugin install squidci@squidci
```

The skill holds only the rules an agent gets wrong without being told, and
reads the rest from SquidCI's docs and JSON Schema as they are now:
https://squidci.com/llms.txt and https://squidci.com/schema/pipelines.json.

## Layout

| path | what |
|---|---|
| `.claude-plugin/marketplace.json` | the marketplace, `squidci`, listing its one plugin |
| `squidci/.claude-plugin/plugin.json` | the plugin; bump `version` with every change, or installed copies do not update |
| `squidci/skills/squidci-pipelines/SKILL.md` | the skill |
| `squidci/.mcp.json` | the MCP servers: one per prod cell, named `squidci-<cell>`; a new cell is a new entry and a version bump |

```bash
claude plugin validate .            # the marketplace
claude plugin validate squidci      # the plugin
```

The MCP servers need nothing installed: each is an endpoint of a cell, and
the user signs in to the one their organization uses with `/mcp`. One nobody
signs in to loads no tools.

Point the skill at the docs rather than restating them: what it says about
the pipeline file must stay true as SquidCI changes, and the docs are what
SquidCI keeps true.
