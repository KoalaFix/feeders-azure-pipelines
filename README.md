# KoalaFix feeder step template for Azure Pipelines

Write a fact to KoalaFix memory from an Azure Pipelines run with one template reference: the release a tag run
builds, the version npm or PyPI serves as latest, the digest an image tag points to, or an HTTP probe of a service
only your network can reach. Agents then read a fact that says who checked it and when. The template runs the feeders from the `@koalafix/feeders` npm package,
the same ones the KoalaFix GitHub Action runs.

## Before the first run

1. A space with the key registered in it, for example `release.latest`, with a freshness window longer than the gap
   between runs.
2. A feeder token for that space whose allowed keys cover the key. Never a person's token.
3. The token saved as a secret variable, in one of two ways:
   - A variable group (Pipelines, Library) named `koalafix-feeder` with `KOALAFIX_FEEDER_TOKEN` as a variable and
     the padlock closed, so it is secret.
   - Or the same group linked to Azure Key Vault: in the group, turn on "Link secrets from an Azure key vault as
     variables", pick the service connection and the vault, and add the secret that holds the token. Key Vault
     names allow hyphens but not underscores, so the secret is named `koalafix-feeder-token`; set the template's
     `token` parameter to that name. The service connection needs Get and List on the vault's secrets.
4. The pipeline allowed to use the group (Library, the group's Pipeline permissions).
5. The template repository declared as a resource in your pipeline file (see the example). The repository is public,
   so a GitHub service connection to it only needs to read.

The `token` parameter takes the variable's name, never the token. A secret variable reaches a step only when the
step maps it, so the template copies it into the feeder's environment as `KALIBRATE_FEEDER_TOKEN` and nowhere else:
it is not in the pipeline file, a script line or a command line, and Azure Pipelines masks it in the log. Do not
echo it, and do not turn on `system.debug` for a pipeline that carries it.

When the group is missing, not linked or not allowed for the pipeline, Azure Pipelines leaves the reference as the
text `$(KOALAFIX_FEEDER_TOKEN)`. The step sees that, stops with exit 2 and says to link the group, rather than sending
the text and getting a 401. The registry password for image-tag gets the same check.

Before the first real run, check that the pipeline compiles: Validate in the pipeline editor's menu, or the Runs
Preview call of the Azure DevOps REST API (`previewRun: true`), which expands the templates without running
anything. To check a copy of this folder inside your own repository, reference it as `steps/feeder.yml@self`.

## Use it

```yaml
resources:
  repositories:
    - repository: koalafix
      type: github
      name: koalafix/feeders-azure-pipelines
      ref: refs/tags/v1
      endpoint: your-github-service-connection

variables:
  - group: koalafix-feeder

steps:
  - template: steps/feeder.yml@koalafix
    parameters:
      feeder: npm-version
      space: your-space
      key: web.widgets.version
      package: "@your-org/widgets"
```

`azure-pipelines.yml` in this folder is a full example: a release job on a tag and a probe job on a schedule.

## Parameters

| Parameter | Default | Meaning |
|---|---|---|
| `feeder` | required | `azure-release`, `npm-version`, `pypi-version`, `http-probe` or `image-tag` |
| `space` | required | the space the key belongs to |
| `key` | required | the key to write, lower-case `subject.fact` |
| `token` | `KOALAFIX_FEEDER_TOKEN` | the name of the secret variable that holds the feeder token |
| `serverUrl` | `https://mcp.koalafix.com` | the KoalaFix server's address |
| `package` | | npm-version: the npm package; pypi-version: the PyPI project |
| `url` | | http-probe: the address to request with GET |
| `expectStatus` | `200` | http-probe: the status that counts as up |
| `expectBody` | | http-probe: a regular expression the body must match |
| `interval` | `60` | http-probe: seconds between runs; set it to the schedule's gap |
| `image` | | image-tag: the image tag whose digest is written, `registry/name:tag` |
| `registryUsername` | | image-tag: the registry user, for a private image |
| `registryPassword` | | image-tag: the name of the secret variable that holds the registry password |
| `feedersVersion` | the version the template tag was cut with | the `@koalafix/feeders` version npx runs |
| `installNode` | `true` | install Node 24 with UseNode; turn it off on an agent that already has Node 24 |

These match the Action's inputs (`server-url` is `serverUrl`, and so on), with `azure-release` in place of
`github-release`, and `feedersVersion` and `installNode` added.

The steps run with bash: Linux and macOS agents, and Windows agents with Git Bash installed. The feeder needs
Node 24 with npx.

## Write the release of a tag

Run it in a pipeline for a tag (`trigger: tags: include: ["v*"]`). Azure Pipelines gives a step no release event, so
the template first writes a release from the run's variables for the feeder to read: the tag from
`Build.SourceBranch`, the run's results page as evidence, and `prerelease: true` for a SemVer pre-release tag such
as `v2.0.0-rc.1`. The fact holds the version (the tag without a leading `v`), the tag, the page and the flag.
Azure Pipelines gives a run no release time, so the pipeline's start (`System.PipelineStartTime`) is the time the
fact became true. The event key names the commit and the tag, so running the pipeline again for the same tag is a
replay, which changes nothing. The key carries a `github:` prefix because it is made by the GitHub release feeder; it is only a name.
A run that is not for a tag exits 2 and sends nothing.

```yaml
- template: steps/feeder.yml@koalafix
  parameters:
    feeder: azure-release
    space: your-space
    key: release.latest
```

## Write the version npm serves as latest

```yaml
- template: steps/feeder.yml@koalafix
  parameters:
    feeder: npm-version
    space: your-space
    key: web.widgets.version
    package: "@your-org/widgets"
```

It reads the public registry; no npm credential is needed. Running it again for the same version is a check: the answer is `unchanged` and the fact's window restarts.

## Probe a service only your network can reach

Run it from a scheduled pipeline on a self-hosted agent inside your network, in a job whose `pool` names that agent
pool. KoalaFix's hosted probes and Microsoft-hosted agents cannot see inside your network. Schedules in YAML are
`cron` entries; set `interval` to the gap in seconds.

```yaml
schedules:
  - cron: "*/15 * * * *"
    branches:
      include: [main]
    always: true

jobs:
  - job: probe
    condition: eq(variables['Build.Reason'], 'Schedule')
    pool:
      name: internal
    steps:
      - template: steps/feeder.yml@koalafix
        parameters:
          feeder: http-probe
          space: your-space
          key: billing.health
          url: http://billing.internal/healthz
          expectStatus: "200"
          interval: "900"
```

It writes `{ status, ok, matched }`. A service that does not answer is written as `ok: false`, never skipped. The
step fails only when KoalaFix did not take the fact; whether the service is up is in the fact itself. Put
`checkout: none` first in the job: the probe needs no source.

A self-hosted agent with no route to the Node download (UseNode fetches from the internet) needs Node 24 installed
and `installNode: false`. It also needs a route to the npm registry, or a mirror that carries `@koalafix/feeders`, on
every run: npx fetches the feeders each time.

## Exit codes

| Code | Means |
|---|---|
| 0 | the fact landed, or KoalaFix already held it |
| 1 | KoalaFix refused the request or could not be reached, or the feeder could not read its source |
| 2 | an input is missing or wrong, a secret's variable group is not linked, or the release step ran outside a tag run; nothing was sent |
| 3 | KoalaFix took the request but refused the fact; the log says which key and why |
