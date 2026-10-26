---
name: squidci-pipelines
description: Use when writing, changing, reviewing or debugging a SquidCI pipeline file, .squidci/pipelines.yaml — adding components, steps, deploys (AWS CDK apps included), environments, AWS access, or fixing an invalid pipeline or a failed, slow or killed SquidCI job, whose run, log and usage SquidCI's MCP tools read.
---

# SquidCI pipeline files

`.squidci/pipelines.yaml` describes what SquidCI builds, tests, publishes and
deploys for a repository — or the `*.yaml` files in `.squidci/pipelines/`,
read with it as one file, each component and each top-level environment
declared in only one of them. Editing one of those files reruns only the
components it declares. This skill holds the rules agents most often get
wrong; for everything else, read the docs as they are now rather than
guessing:

- Index of every page: https://squidci.com/llms.txt (each page is Markdown at
  its URL plus `.md`)
- Every setting: https://squidci.com/docs/pipelines/reference.md
- AWS access: https://squidci.com/docs/pipelines/aws.md
- Deploying CDK apps: https://squidci.com/docs/pipelines/cdk.md
- JSON Schema: https://squidci.com/schema/pipelines.json

## Rules that are easy to get wrong

1. **It is not GitHub Actions.** The file is a map of components; each has
   sections — `check`, `test`, `build`, `publish`, `run`, `deploy` — and
   **each section is one job**, a map with its `steps:` list. A section may
   also name its own `runner`, `services`, `aws`, `cache` or `test-reports`,
   over the component's; nothing else. There are no `jobs:`, `uses:`,
   `with:`, `name:` or `on:` keys, and a section that is a bare list of steps
   is refused. Any key SquidCI does not know
   makes the file invalid, and an invalid file runs nothing.
2. **A step is a command string, or a map with exactly one key**: `pnpm`,
   `script`, `docker` or `cdk`. Not `{run: make}` and not `{name: …, run: …}`.
3. **Commands share one bash shell** with `set -eo pipefail`, in the
   component's `base`: a `cd` or `export` carries on to later steps. A
   `script` step runs in its own process and does not.
4. **`pnpm:` steps set up Node and install the lockfile themselves** (from
   `.nvmrc` or `.node-version`). Do not add `pnpm install`, `nvm use` or a
   Node setup step before them.
5. **Every value is text.** `tag: 1.10` is "1.10". Anchors and aliases work
   within one file; merge keys (`<<:`) do not.
6. **A deploy uses its environment's `aws`, wholly** — profiles and region —
   never the component's, and a `deploy` section cannot name `aws` itself.
   A component's `aws` is for its other sections, unless one names its own.
   `deploy` needs environments: the top-level `environments` map, or the
   component's own `environments`, which replace the top-level map wholly
   for it — its `after` and `soaked.in` name only its own, and its `prod` is
   another environment than the file's `prod`. `$SQUIDCI_ENVIRONMENT` is the
   environment's name inside the steps either way.
7. **`aws.profile` names are target AWS accounts the user configured** on
   their execution plane in the SquidCI app, not AWS CLI profiles on this
   machine. Ask the user for them; do not invent names. `region` sets
   `AWS_REGION`; without it jobs run in the plane's region.
8. **`needs` brings artifacts and exported values; `after` only orders.**
   Artifacts come only from a component's `build` section, listed under
   `artifacts`, and arrive in a needing job at
   `.squidci/artifacts/<component>/<path>`. Values are `name=value` lines
   appended to `$SQUIDCI_OUTPUT`, read as
   `$SQUIDCI_EXPORT_<COMPONENT>_<NAME>`.
   Artifacts and staged images are kept **90 days**, which is why a version
   can be deployed, or rolled back to, for 89 days after its push and no
   longer. They are build output, not storage: anything to keep longer — a
   release binary, an image that runs — is pushed somewhere of the user's
   own in `publish`.
9. **Pull requests run `check`, `test` and `build` only, with no AWS
   access.** Anything needing AWS or credentials belongs in `publish`, `run`
   or `deploy`.
10. **`runner.lambda`** jobs have no shell, checkout, Docker or `pnpm`:
    commands and `script` steps are TypeScript or JavaScript modules.
11. **Deploy a CDK app with a `cdk:` step, not `pnpm: cdk deploy`**, unless
    the user wants the command: SquidCI then waits for CloudFormation instead
    of the job. In `deploy` or `run` it is the **last** step and the job's
    only `cdk` step, after the app's own build. In `publish` it deploys
    nothing: it publishes and records stacks, as many steps as the app needs,
    and a deploy whose **only** step is `cdk: {from: publish, stacks: [...]}`
    deploys what was recorded. Never in `check`, `test` or `build` (use
    `cdk synth` as a command there). Stacks in several accounts publish from
    a step each, with `profile:` naming one of the component's
    `aws.profiles`; `from: publish` takes no `dir`, `context` or `profile`.
    Settings are `dir`, `stacks`, `context`, `profile`, `from`, `parameters`,
    `timeout` (`90m`, `2h`) and `rollback`; read the CDK page before writing
    one.

## SquidCI's tools, when they are connected

The plugin lists SquidCI's MCP server, `squidci-eu-west-2`; its tools load
once the user has signed in to it with `/mcp`. All of them only read. When
they are there, look rather than guess:

- **Before finishing a change to the files**, pass every pipeline file to
  `validate_pipeline_file`: it reads them as SquidCI will, including what the
  schema cannot check.
- **After a push**, find its run with `list_runs` (by repository and branch;
  repository ids from `list_repositories`) and read it with `get_run`.
- **When a job failed**, read it with `get_job` and its log with
  `get_job_log` — the `jobId` is the job's `assignment.jobId` in `get_run` —
  before proposing a fix. `get_job`'s `result` has its failing tests, where
  its time went, and what it used against its `limits`.
- **When a job was killed or slow**, check memory before the code. Exit code
  137, a daemon "unexpectedly lost", or tests failing oddly after a stall
  usually mean the kernel killed a process for want of memory: `get_job`'s
  `result.usage.oomKills` above 0, or `memoryBytes.max` near
  `limits.memoryBytes`, confirms it. `get_job_usage` shows CPU, memory, disk
  and stalls over time. The fixes, cheapest first: cap the tools' heaps
  (`org.gradle.jvmargs`, `kotlin.daemon.jvmargs`, a test task's
  `maxHeapSize`, Node's `--max-old-space-size`), run on a worker group with
  more memory per vCPU, or raise `runner.vcpus`, which brings memory in
  proportion. CPU at the reservation for long stretches means more vCPUs
  will help; well below it, they will not. See
  https://squidci.com/docs/concepts/job-performance/.
- **What is deployed where**: `list_environments` for now, `list_deployments`
  for an environment's history, `list_versions` for a version's, and
  `diff_environment` for what a promotion would bring.

Without them, ask the user for the run's log rather than guessing at it.

## Before you finish

Validate the file against the schema:

```bash
curl -fsSL https://squidci.com/schema/pipelines.json -o /tmp/squidci-pipelines.json
npx --yes ajv-cli@5 validate --spec=draft7 --strict=false \
  -s /tmp/squidci-pipelines.json -d .squidci/pipelines.yaml   # and each file in .squidci/pipelines/
```

Then check by hand what a schema cannot, across all the files together: no
component or top-level environment is declared in two files, every deploy
has environments to go to, every name in `needs` and `after` is
a component in the files, neither goes round in a circle, a component is not
in both, schedules' crons and time zones are real, the file makes at most
50 jobs together (a deploy counts once per environment), and every `cdk` step is
where rule 11 allows, with any `profile` among its job's `aws.profiles`, and
no Lambda job — a section whose runner, merged over its component's, has
`lambda` — inherits the component's `services` or `test-reports`, or is the
`build`, `publish`, `run` or `deploy` of a component with `artifacts`. Keep
`# yaml-language-server: $schema=https://squidci.com/schema/pipelines.json`
as the file's first line.
