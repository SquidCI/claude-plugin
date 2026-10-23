---
name: squidci-pipelines
description: Use when writing, changing, reviewing or debugging a SquidCI pipeline file, .squidci/pipelines.yaml — adding components, steps, deploys, environments, AWS access, or fixing an invalid pipeline or a failed SquidCI job.
---

# SquidCI pipeline files

`.squidci/pipelines.yaml` describes what SquidCI builds, tests, publishes and
deploys for a repository. This skill holds the rules agents most often get
wrong; for everything else, read the docs as they are now rather than
guessing:

- Index of every page: https://squidci.com/llms.txt (each page is Markdown at
  its URL plus `.md`)
- Every setting: https://squidci.com/docs/pipelines/reference.md
- AWS access: https://squidci.com/docs/pipelines/aws.md
- JSON Schema: https://squidci.com/schema/pipelines.json

## Rules that are easy to get wrong

1. **It is not GitHub Actions.** The file is a map of components; each has
   steps under sections — `check`, `test`, `build`, `publish`, `run`,
   `deploy` — and **each section is one job**. There are no `jobs:`, `steps:`,
   `uses:`, `with:`, `name:` or `on:` keys. Any key SquidCI does not know
   makes the file invalid, and an invalid file runs nothing.
2. **A step is a command string, or a map with exactly one key**: `pnpm`,
   `script` or `docker`. Not `{run: make}` and not `{name: …, run: …}`.
3. **Commands share one bash shell** with `set -eo pipefail`, in the
   component's `base`: a `cd` or `export` carries on to later steps. A
   `script` step runs in its own process and does not.
4. **`pnpm:` steps set up Node and install the lockfile themselves** (from
   `.nvmrc` or `.node-version`). Do not add `pnpm install`, `nvm use` or a
   Node setup step before them.
5. **Every value is text.** `tag: 1.10` is "1.10". No YAML anchors or aliases.
6. **A deploy uses its environment's `aws`, wholly** — profiles and region —
   never the component's. A component's `aws` is for its other sections.
   `deploy` requires a top-level `environments` map; `$SQUIDCI_ENVIRONMENT`
   names the environment inside the steps.
7. **`aws.profile` names are target AWS accounts the user configured** on
   their execution plane in the SquidCI app, not AWS CLI profiles on this
   machine. Ask the user for them; do not invent names. `region` sets
   `AWS_REGION`; without it jobs run in the plane's region.
8. **`needs` brings artifacts and exported values; `after` only orders.**
   Artifacts come only from a component's `build` section, listed under
   `artifacts`, and arrive in a needing job at
   `.squidci/artifacts/<component>/<path>`. Values are `name=value` lines
   appended to `$SQUIDCI_OUTPUT`, read as `$SQUIDCI_<COMPONENT>_<NAME>`.
9. **Pull requests run `check`, `test` and `build` only, with no AWS
   access.** Anything needing AWS or credentials belongs in `publish`, `run`
   or `deploy`.
10. **`runner.lambda`** components have no shell, checkout, Docker or `pnpm`:
    commands and `script` steps are TypeScript or JavaScript modules.

## Before you finish

Validate the file against the schema:

```bash
curl -fsSL https://squidci.com/schema/pipelines.json -o /tmp/squidci-pipelines.json
npx --yes ajv-cli@5 validate --spec=draft7 --strict=false \
  -s /tmp/squidci-pipelines.json -d .squidci/pipelines.yaml
```

Then check by hand what a schema cannot: every name in `needs` and `after` is
a component in the file, neither goes round in a circle, a component is not
in both, schedules' crons and time zones are real, and the file makes at most
50 jobs (a deploy counts once per environment). Keep
`# yaml-language-server: $schema=https://squidci.com/schema/pipelines.json`
as the file's first line.
