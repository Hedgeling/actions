# Hedgeling sync — GitHub Action

`hedgeling sync push|pull` is the **one sanctioned transport** between repo
state and the Hedgeling console (ad-hoc import scripts are the documented
anti-pattern). This composite action runs that same CLI command with the CI
ergonomics around it:

- **push after merge to main** — repo-owned truth (sources, contexts, machine
  translations) upserted to the console; console rows for keys absent from
  state are retired. Unchanged state short-circuits to a no-op (no HTTP).
- **retry/backoff on transport failures** — a flaky 502 never fails a
  merge-to-main workflow. Conflicts are *never* retried (the sync completed;
  a retry cannot change the answer).
- **cursor bookkeeping** — after a push that moved something, the updated
  `.hedgeling/sync.json` is committed back to the branch (`[skip ci]`), so
  the next run's idempotency short-circuit and incremental pulls work.

For the **pull** direction you usually don't use this action directly: set
`sync: 'true'` on [`actions/translate`](../translate/README.md) instead, so
console review verdicts and human edits land inside the reviewable
localization PR rather than a direct commit.

## Quickstart: push repo state after merge to main

```yaml
# .github/workflows/hedgeling-sync.yml
name: Hedgeling sync
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: write          # the cursor-artifact bookkeeping commit

concurrency: hedgeling-sync   # serialize pushes — see "Concurrency" below

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Hedgeling/actions/sync@v1
        with:
          console-token: ${{ secrets.HEDGELING_CONSOLE_TOKEN }}
          console-url: https://console.example.com
```

The project slug comes from `projectSlug` in
`.hedgeling/extract.config.json` (recommended) or the `project` input. On a
run where nothing changed since the last push, the CLI short-circuits on the
recorded `stateHash` — no HTTP call, no cursor churn, exit 0. The same holds
when the only change since the last push was pulled from the console (review
verdicts, human edits): the pull records the resulting hash as
`lastPush.pulledStateHash` and the push reports `reason: "state-from-pull"`.

**Keep `console-url` identical across workflows and local runs.** The cursor
in `.hedgeling/sync.json` is scoped to one console origin, and `console-url`
is passed to the CLI as `--origin`. A run with a different origin string is
treated as a different console: the cursor's `lastPull` and `lastPush` are
dropped, the pull is a full pull, and the push uploads the whole state (the
`stateHash` short-circuit above no longer applies) and rewrites the cursor
for the new origin. The comparison is exact, so `http://localhost:8080` and
`http://127.0.0.1:8080` are two consoles — a setup that alternates them
churns the cursor (and a bookkeeping commit) on every switch. When
`console-url` is empty, the origin recorded in the cursor wins over
`HEDGELING_CONSOLE_URL`.

## First-time setup

1. **Create the console project** (or note its slug). Put the slug in
   `.hedgeling/extract.config.json` as `projectSlug` so every CLI command and
   workflow agrees on the target.
2. **Choose the credential.** Your seat token (`hdg_…`, the one that installs
   the package) also signs in to the console, so the simplest setup passes
   the same secret as `console-token`. For a credential that reaches less or
   expires, mint a service account in the console instead (see
   [Token scoping](#token-scoping--rotation)).
3. **Add the CI secret**: repository → Settings → Secrets →
   `HEDGELING_CONSOLE_TOKEN` (the seat token or the service-account
   credential).
4. **Preflight locally**: `HEDGELING_CONSOLE_TOKEN=… hedgeling sync doctor
   --origin https://console.example.com`. Doctor is read-only and names the
   broken link (slug resolution, credentials, origin, cursor consistency,
   one live console probe) — the same failures CI flattens into "exit 2".
5. **First push** — run the workflow above (or locally:
   `HEDGELING_CONSOLE_TOKEN=… hedgeling sync push --origin https://console.example.com`).
   This seeds the console with repo state and writes
   `.hedgeling/sync.json`.
6. **Commit `.hedgeling/sync.json`.** The cursor artifact is committed,
   canonical JSON and never carries tokens. It records the console origin,
   the pushed `stateHash`, and the pull watermark — losing it costs you the
   idempotency short-circuit and incremental pulls (the next sync is a full
   one), but never correctness.
7. **Wire the pull side**: add `sync: 'true'` + `console-token` to your
   [`actions/translate`](../translate/README.md) workflow so review verdicts
   flow back inside the localization PR.

## Token scoping & rotation

| Direction | Permission tier | Entitlement |
|---|---|---|
| `sync pull` | `strings.view` (read-only) | — |
| `sync push` | `imports.run` (the manage tier) | `state_imports` |

A seat token covers both rows in every project of its workspace; it cannot
edit the glossary or manage credentials. Revoking the seat (or issuing it a
new token) in the console cuts it off at once.

With service accounts, use **two** when your workflows split directions: the
localization-PR workflow only ever pulls, so its token should not carry the
manage tier. Push metering mirrors `/api/import-state` — sync is never a
billing bypass.

Rotation: mint the replacement account, update the
`HEDGELING_CONSOLE_TOKEN` secret, then revoke the old credential. Nothing in
the repo references the token (the cursor artifact stores origin + hashes
only), so rotation is a secrets-manager-only operation. A revoked/expired
token surfaces as exit 2 (HTTP 401/403) — the workflow fails loudly rather
than drifting.

## Conflict runbook (exit 1)

`sync pull` exits **1** when the merge law found true conflicts: both sides
changed the same entry since the last watermark, and both changes are human.
This is "sync completed, attention required" — *not* a failure of transport:

- the **repo text is kept** (console human truth never silently overwrites
  repo human truth),
- the entry is marked **needs-review** in state, so `check
  --require-verified` holds the gate until a human decides,
- the console text is recorded in **`.hedgeling/sync-revisions.jsonl`**
  (append-only, `revisionSource: "sync"`) — nothing is lost; the log is the
  audit trail of what the console said and when,
- each conflict is listed on stderr with key, locale, both texts, and the
  review note.

To resolve: open the console's review queue (the needs-review entries carry
the conflict), pick or edit the winning text, verify — the next `sync pull`
brings the verdict back and clears the flag. Failed pull-side validation
(placeholder/ICU mismatches, terminology violations) lands in the same
conflict listing: the console edit is rejected, not written.

In workflows where conflicts should not fail the job (e.g. they ride a PR
for review anyway), set `fail-on-conflicts: 'false'` — the count still lands
in the `conflicts` output and the job summary.

Exit **2** is different: configuration or transport (bad origin, bad token,
missing slug, 5xx after retries). Nothing was synced; no partial state was
written. When CI fails with exit 2, run **`hedgeling sync doctor`** locally
(with the same token) — it names which link is broken instead of making you
bisect four fallback chains by hand.

## Transient failures & retries

The action retries **only exit 2** — up to `max-attempts` (default 3), with
linear backoff (`retry-delay` × attempt number, default 10 s). Conflicts
(exit 1) and success (exit 0) never retry. If you call the CLI directly in a
custom workflow, prescribe the same discipline: retry 2, never 1.

## Concurrency

Serialize sync pushes with a `concurrency:` group (as in the quickstart) so
merge-queue bursts and re-runs don't race each other. Ordering convention
for the two directions:

- **push** runs on merge to main — it carries repo truth and updates the
  cursor on main;
- **pull** runs inside the localization-PR workflow — its
  `.hedgeling/sync.json` update rides the PR. If a sync push lands on main
  while that PR is open, the PR's cursor file conflicts like any other file:
  re-run the localization workflow (the branch is regenerated from the new
  head) rather than resolving by hand.

A lost race on the cursor bookkeeping commit is harmless by design: the
winning run's cursor carries a newer `stateHash`, and the losing run's
rebase retries, then gives up loudly.

## Branch protection

The cursor bookkeeping commit pushes directly to the branch the workflow ran
on (`git push origin HEAD:main`). On a **protected** main — required PRs,
required status checks — that push is rejected and the action fails loudly
after the sync itself succeeded. Pick one of these, in order of preference:

1. **`commit-cursor: 'false'`** (simplest, always works): the push to the
   console still happens; the updated `.hedgeling/sync.json` is left in the
   workspace for you to deliver through your normal PR flow (e.g. a
   `peter-evans/create-pull-request` step, or fold it into the next
   localization PR). Until the cursor lands, re-runs skip the CLI's
   `stateHash` short-circuit and contact the console. If the state is
   unchanged, the console recognizes the exact state it last accepted and
   answers `alreadyAccepted` — nothing is re-imported, no usage is recorded,
   and the run reports `outcome: unchanged`. If the state did change, the
   push applies normally but reports a **stale base** (below), because the
   console last accepted a push the committed cursor never recorded.
2. **Bot token with bypass**: pass a PAT or GitHub App token that is on the
   branch-protection bypass list to `actions/checkout` (`token:`), so the
   persisted credential the cursor push rides can push to main. Note a PAT
   push does trigger workflows — the `[skip ci]` in the commit message is
   what prevents a sync→sync loop.
3. **Repository rule exception**: with repository rulesets you can exempt
   the `github-actions[bot]` app for pushes that touch only
   `.hedgeling/sync.json` paths, keeping the default token workable.

The action cannot detect protection up front (the failure mode is the
rejected push), so if your org protects main by default, start at option 1.

## Monitoring the loop

The console records every sync on the project (`project_sync_status`):

- **`GET /api/sync/status?project=<slug>`** (`strings.view`) returns
  `lastPushAt` / `lastPushStateHash` / `lastPushActor` / `lastPullAt` /
  `lastPullActor` — nulls for a direction that never ran. A repo whose
  workflow died (revoked token, disabled workflow, deleted secret) shows up
  here as staleness instead of being discovered via translation drift.
- **Last-writer detection**: the CLI declares the `stateHash` its committed
  cursor says it builds on (`baseStateHash`). When that is not what the
  console last accepted — two repos or a misconfigured fork pushing into one
  project — the push response carries `staleBase: true` and the CLI prints a
  warning (the push still applies, under the never-overwrite-human law).
  Two causes look identical to the console:
  - **another writer**: two repos or workflows push into one project —
    consolidate on one pushing repo/workflow;
  - **a lost cursor**: this repo's previous push landed but its cursor
    commit did not (`commit-cursor: 'false'` before the cursor was
    delivered, or a rejected cursor push) and the state has changed since.
    Harmless — once a cursor from this run lands, the warning stops.
  An *identical* re-push after a lost cursor never warns (see
  `alreadyAccepted` under Branch protection).

## Inputs

| Input | CLI flag | Default | Notes |
|---|---|---|---|
| `command` | `sync push` / `sync pull` | `push` | Direction. |
| `working-directory` | positional `projectDir` | `.` | Directory holding `.hedgeling/extract.config.json`. |
| `console-token` | env `HEDGELING_CONSOLE_TOKEN` | — | Seat token or service-account credential. Wire from secrets. Never echoed. |
| `console-url` | `--origin` | cursor artifact, then `HEDGELING_CONSOLE_URL` | Console origin. |
| `project` | `--project` | config `projectSlug` | Console project slug. |
| `locales` | `--locales <csv>` | all | Pull only; a filtered pull never advances the shared watermark. |
| `glossary` | `--glossary <path>` | `.hedgeling/glossary.json` | Pull-side validation glossary. |
| `dry-run` | `--dry-run` | `false` | Print the plan; write and push nothing. |
| `fail-on-conflicts` | — | `true` | Pull: fail the step on exit 1, or warn and continue. |
| `commit-cursor` | — | `true` | Push: commit `.hedgeling/sync.json` back to the branch. |
| `max-attempts` | — | `3` | Retry budget for exit 2. |
| `retry-delay` | — | `10` | Base seconds between retries (linear backoff). |
| `hedgeling-version` | — | pinned per release | Any npm package spec. |
| `npm-token` | scoped `.npmrc` for the action's `npx` calls | — | Token for installing `@hedgeling/i18n`: `npm-token: ${{ secrets.HEDGELING_TOKEN }}`. A Hedgeling seat token (`hdg_...`, from the Seats panel in the console) installs from Hedgeling's registry; any other value is npm auth for the job's default registry. Env-indirected — the token value never lands on disk. Install failures are a configuration error (exit 2), never a sync failure. |

## Outputs

| Output | Meaning |
|---|---|
| `outcome` | `synced`, `unchanged` (idempotent no-op), `conflicts`, or `dry-run`. |
| `changed` | `"true"` when the sync moved something. |
| `conflicts` | Conflict count (0 for push). |
| `json-path` | Path to the CLI's versioned `--json` document — set only when that document is non-empty. An empty string means no report was produced (e.g. an exit-2 configuration/transport failure); never read a path you were not given. |
| `cursor-committed` | Always `"true"` or `"false"`. `"true"` only when the bookkeeping commit was pushed; `"false"` on an unchanged run, when the commit step is skipped (`pull`, `dry-run`, `commit-cursor: 'false'`, or an earlier step failed), and when the commit/push failed. |

## Version compatibility

The CLI and console speak a versioned sync contract (`{ version: 1, … }`).
A mismatch hard-fails with exit 2 ("update one side") rather than guessing.
Pin `hedgeling-version` deliberately: when the hosted console upgrades its
contract version, bump the pinned CLI in the same change.

This is a stated commitment, not just today's behavior — the policy
(additive fields never bump the version; a contract bump ships in a CLI
minor; the console accepts both versions for **at least 90 days** of
overlap) lives in `docs/VERSIONING.md` of the `@hedgeling/i18n` package, "Sync contract". You
can build a release process on it. `hedgeling sync doctor` probes the live
contract version, so checking your pin against the hosted console is one
local command.

## Versioning

`actions/sync/v1` is a moving major alias over immutable `v1.x.y` tags
(standard Actions convention). Pin `@v1` for compatible updates or a full
`@v1.x.y` tag for byte-stable behavior.
