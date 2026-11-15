# Hedgeling translate — GitHub Action

Merged source changes come back as a reviewable localization PR. The action
runs the **same CLI pipeline a developer runs** — `hedgeling translate`
(extract → translate, all writes through `writeState`/compile) — then
delivers the catalog/state/bundle diff as ONE standing pull request on the
fixed branch `hedgeling/localize`. A re-run regenerates the branch from the
current default-branch head, force-pushes it, and edits the existing PR:
serial, idempotent, no PR pile-up.

## Quickstart

```yaml
name: Hedgeling localization
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write

concurrency: hedgeling-localize   # serialize runs — the branch is singleton

jobs:
  localize:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Hedgeling/actions/translate@v1
        with:
          api-key: ${{ secrets.HEDGELING_TRANSLATE_API_KEY }}
```

When a run produces no catalog/state/bundle diff there is no branch and no
PR — the step exits 0 with "nothing to localize" in the job summary, so
re-running on an unchanged tree is a no-op.

## How it works

A composite action: it installs Node 22 only when `node` is absent, runs
`npx --yes @hedgeling/i18n@<pinned> translate <working-directory> --report
… --report-markdown …` (writing both reports to `$RUNNER_TEMP`, outside the
checkout), then asks the CLI's `localize-plan` for the decision:

- **unrelated dirty path** (anything outside the committable Hedgeling
  artifact set — catalog, state, compiled bundles, key maps, `.hedgeling/`)
  → error: the action never sweeps unrelated changes into a localization PR,
- **no diff** → no-op, "nothing to localize",
- **diff** → create (first run) or update (a PR is already open on
  `hedgeling/localize`).

The commit touches only those artifact paths (`git add
--pathspec-from-file`), the branch is force-pushed with `--force-with-lease`,
and the PR body is the `--report-markdown` twin: a per-locale table of
translated / carried-by-rung / TM hits / failed / needs-review, the
provider+model, and a file:line list of needs-review items capped at 50.
The resulting PR triggers `hedgeling check` like any other — `--require-verified`
locales keep a human in the loop.

If every item the run attempted failed (none translated, glossary-filled, or
TM-hit), `translate` exits 1 and the step fails before any PR is touched — a
broken provider or prompt is a red run, not an empty PR. Partial failure still
opens/updates the PR with the failures listed in the report.

## Permissions

The workflow needs `contents: write` (push the regenerated branch) and
`pull-requests: write` (create/edit the PR). The check action stays
`contents: read` — this one is honest about writing. `GITHUB_TOKEN` is used
for `gh` only; nothing is sent anywhere else.

**Checks on the localization PR** — GitHub does not trigger workflow runs
for events caused by the default `GITHUB_TOKEN` (anti-recursion), so a PR
the action creates will not run `hedgeling check` or any other
PR-triggered gate on its own. To make the localization PR behave like a
human-authored PR — checks fire, reviews gate — wire a PAT or GitHub App
token through the `token` input. The token then authenticates both the `gh`
calls **and** the branch push (`actions/checkout` persists `GITHUB_TOKEN`
for git; pushing the branch with it would keep suppressing the PR's checks
on every update, so the action overrides the push credentials for that one
command — nothing is persisted or rewritten):

```yaml
      - uses: Hedgeling/actions/translate@v1
        with:
          api-key: ${{ secrets.HEDGELING_TRANSLATE_API_KEY }}
          token: ${{ secrets.HEDGELING_BOT_TOKEN }}
```

**A "failed" check with zero jobs on a closed localization PR** — when the
PR is opened with the default `GITHUB_TOKEN`, GitHub may still create a
check suite/run for it, parked in the *approval required* state (workflow
runs triggered by `GITHUB_TOKEN` activity wait for approval rather than
run). If the PR is then closed without approving, GitHub concludes that run
as `failure` with **zero jobs** — it never executed anything, so it is not a
real localization or check failure. This is GitHub behavior, not the
action's. The recommended fix is the same as above: open the PR with a
GitHub App installation token (preferred — short-lived, scoped per repo) or
a fine-grained PAT via the `token` input, so the PR's checks run normally
and conclude with real results.

## Console sync (pull review verdicts into the PR)

With `sync: 'true'` the action runs `hedgeling sync pull` **before**
translating: console review verdicts (verified / rejected / needs-review) and
human edits land in repo state through the normal compile path and ride this
localization PR — the sanctioned way console truth returns to the repo
(spec 10). The pull is incremental from the committed
`.hedgeling/sync.json` watermark.

```yaml
      - uses: Hedgeling/actions/translate@v1
        with:
          api-key: ${{ secrets.HEDGELING_TRANSLATE_API_KEY }}
          sync: 'true'
          console-token: ${{ secrets.HEDGELING_CONSOLE_TOKEN }}
          console-url: https://console.example.com
```

The token can be your seat token, or a service-account credential that only
needs the `strings.view` permission (pull is read-only on the console side). Transport failures
retry 3× with backoff before failing the run; conflicts never fail it (the
entries are marked needs-review, the repo text kept, and they arrive in this
PR for a human — the count rides the `sync-conflicts` output). The push
direction (repo state → console, after merge to main) is the
[`actions/sync`](../sync/README.md) action, which also carries the setup
guide, token-scoping table, and conflict runbook.

## Secrets

The provider API key is an action input wired from repo secrets and reaches
the CLI exclusively through `HEDGELING_TRANSLATE_API_KEY` (the generic env
every provider honors). It is never echoed and never written to the run
report — the report records provider and model *names* only. Provider-specific
env vars (`OPENAI_API_KEY`, `DEEPL_API_KEY`, …) set at job level also work.

## Inputs

| Input | CLI flag | Default | Notes |
|---|---|---|---|
| `working-directory` | positional `projectDir` | `.` | Directory holding `.hedgeling/extract.config.json`. |
| `api-key` | env `HEDGELING_TRANSLATE_API_KEY` | — | Wire from secrets: `api-key: ${{ secrets.HEDGELING_TRANSLATE_API_KEY }}`. |
| `locales` | `--locales <csv>` | config locales | Restrict the run to these locales. |
| `force-locales` | `--force-locales <csv>` | off | Re-translate even if already translated. |
| `glossary` | `--glossary <path>` | `.hedgeling/glossary.json` | Glossary override. |
| `base-branch` | PR `--base` | branch that triggered the run | PR base. |
| `sync` | `sync pull` before translate | `false` | Pull console review verdicts/edits into the PR (spec 10). |
| `console-token` | env `HEDGELING_CONSOLE_TOKEN` | — | Seat token, or a service-account credential (`strings.view` suffices), for the pull. |
| `console-url` | `--origin` | cursor artifact, then `HEDGELING_CONSOLE_URL` | Console origin for the pull. |
| `project` | `--project` | config `projectSlug` | Console project slug for the pull. |
| `token` | `GH_TOKEN` for `gh` calls + branch-push auth | `GITHUB_TOKEN` | PAT/App token so the bot PR's checks fire — on creation AND on every branch update (GITHUB_TOKEN events don't trigger workflows). |
| `hedgeling-version` | — | pinned per release | Any npm package spec (version, dist-tag, or `file:` path). |
| `npm-token` | scoped `.npmrc` for the action's `npx` calls | — | Token for installing `@hedgeling/i18n`: `npm-token: ${{ secrets.HEDGELING_TOKEN }}`. A Hedgeling seat token (`hdg_...`, from the Seats panel in the console) installs from Hedgeling's registry; any other value is npm auth for the job's default registry. Env-indirected — the token value never lands on disk. Install failures are a configuration error (exit 2), never a translation failure. |

## Outputs

| Output | Meaning |
|---|---|
| `changed` | `"true"` when a branch was pushed (a diff existed). |
| `pr-url` | URL of the created/updated localization PR. |
| `report-path` | Path to the versioned JSON run report. |
| `translated` / `failed` / `needs-review` | Outcome counts from the run report. |
| `sync-conflicts` | Conflicts the pre-translate `sync pull` classified (empty unless `sync: 'true'`). |

## Versioning

`actions/translate/v1` is a moving major alias over immutable `v1.x.y` tags
(standard Actions convention). Pin `@v1` for compatible updates or a full
`@v1.x.y` tag for byte-stable behavior.
