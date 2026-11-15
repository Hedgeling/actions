# Hedgeling i18n check — GitHub Action

One step turns `hedgeling check` into a complete PR experience: inline
annotations on the findings **introduced by the PR**, a localization impact
table in the job summary, and a pass/fail status — with zero configuration.

## Quickstart

```yaml
name: Hedgeling i18n check
on: pull_request
permissions:
  contents: read
jobs:
  hedgeling-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Hedgeling/actions/check@v1
```

That is the whole workflow. On `pull_request` events the action defaults
`base` to the PR base SHA, so annotations and gates apply only to findings the
PR introduces (preexisting findings are counted, never annotated). The action
needs nothing beyond `permissions: contents: read` — no PR-comment scope, no
tokens, no telemetry, and no network beyond the npm install.

## How it works

A composite action: it installs Node 22 only when `node` is absent, then runs
`npx --yes @hedgeling/i18n@<pinned> check <working-directory> --github --json
--summary …` from the checkout root (so in a monorepo the annotation file
paths carry the subfolder prefix and attach to the PR diff). The pinned CLI
version bumps only via a release of the action; `hedgeling-version` overrides
it. The JSON report is written to
`$RUNNER_TEMP/hedgeling-check.json` (output `report-path`) and the markdown
impact summary is appended to the job summary.

Failure discipline follows the CLI's exit-code contract: **1** fails the step
as a localization regression (the findings are the annotations), **2** fails
with a distinguishable `configuration error` prefix so setup problems are
never misread as regressions.

With a base, the action fetches the base SHA when the shallow checkout lacks
it, retrying up to 3 attempts (backing off 2s, then 5s): GitHub's git
endpoint occasionally fails such fetches transiently (e.g. `RPC failed; HTTP
404 … expected 'packfile'`). If every attempt fails, the CLI cannot run
baseline mode and the step ends as a configuration error (exit 2) whose
message says so — that is almost always transient on GitHub's side; re-run
the job.

## Inputs

Inputs map 1:1 to `hedgeling check` flags — none invents semantics the CLI
lacks. All are optional.

| Input | CLI flag | Default | Notes |
|---|---|---|---|
| `working-directory` | positional `projectDir` | `.` | Directory holding `.hedgeling/extract.config.json`. |
| `base` | `--base <ref>` | PR base SHA on `pull_request`, else off | Branch-aware mode: gate/annotate introduced findings only. The action best-effort fetches the SHA when the checkout is shallow. |
| `max-untranslated` | `--max-untranslated <n\|p%>` | off | Absolute per-locale untranslated budget. |
| `max-new-untranslated` | `--max-new-untranslated <n>` | off | Delta budget vs. base (needs a base). |
| `fail-on` | `--fail-on <severity>` | off | `required` \| `definite` \| `potential` \| `safe`. |
| `require-verified` | `--require-verified <csv>` | off | Human-review gate for the named locales. |
| `allow-new` | `--allow-new` | `false` | Downgrade new drift strings to warnings. |
| `schema` | `--schema` | `false` | Verify committed artifacts are readable at this version. |
| `hedgeling-version` | — | pinned per release | Any npm package spec (version, dist-tag, or `file:` path). |
| `npm-token` | scoped `.npmrc` for the action's `npx` calls | — | Token for installing `@hedgeling/i18n`: `npm-token: ${{ secrets.HEDGELING_TOKEN }}`. A Hedgeling seat token (`hdg_...`, from the Seats panel in the console) installs from Hedgeling's registry; any other value is npm auth for the job's default registry. Env-indirected — the token value never lands on disk. Install failures are a configuration error (exit 2), never a check failure. |

## Outputs

| Output | Meaning |
|---|---|
| `passed` | `"true"` when every check passed. |
| `report-path` | Path to the full JSON report (the versioned `--json` contract). |
| `new-strings` | New strings missing from the committed catalog (the gated drift set). |
| `untranslated` | Total untranslated strings across checked locales (0 when no bundle is committed). |
| `introduced-findings` | Findings introduced by this change (with a base; without one, the total error count). |

Downstream use:

```yaml
      - id: i18n
        uses: Hedgeling/actions/check@v1
      - run: echo "passed=${{ steps.i18n.outputs.passed }} report=${{ steps.i18n.outputs.report-path }}"
        if: always()
```

## Versioning

`actions/check/v1` is a moving major alias over immutable `v1.x.y` tags
(standard Actions convention). Pin `@v1` for compatible updates or a full
`@v1.x.y` tag for byte-stable behavior.
