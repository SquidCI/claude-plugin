# SquidCI for Claude Code

A Claude Code plugin marketplace with one plugin, `squidci`: a skill for
writing and changing SquidCI pipeline files, `.squidci/pipelines.yaml`.

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

```bash
claude plugin validate .            # the marketplace
claude plugin validate squidci      # the plugin
```

Point the skill at the docs rather than restating them: what it says about
the pipeline file must stay true as SquidCI changes, and the docs are what
SquidCI keeps true.
