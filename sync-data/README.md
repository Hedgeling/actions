# Hedgeling sync data — GitHub Action

The **closed git loop for dynamic content** (dynamic-data spec 07). The
UI-string loop is commit-triggered (code merges → `actions/translate` →
localization PR → merge → sync push). Dynamic content changes the trigger —
**API churn, not commits** — so this loop is **cron-driven**. One tick runs,
in this fixed order:

```
cron ─▶ sync pull ─▶ sync-data ─▶ commit artifacts ─▶ sync push ─▶ publish
        (human truth   (fetch Δ,     (state, data-      (console     (bundle →
         in first)      translate,    cursors, dynamic   rows)        storage URL)
                        recompile)    bundle)
                                          ▲
                console review (async) ───┘  (next cycle's pull)
```

- **Pull first** so human verifications land before recompile — a console
  approval reaches the served translation in ONE cycle. Pull conflicts
  (exit 1) warn and continue (conflicts wait for humans, data keeps
  flowing); transport failures (exit 2) abort before any commit.
- **Direct commit, no localization PR.** Dynamic batches are high-frequency,
  validator-gated, additive, and reviewed asynchronously in the console — a
  PR per cron tick is bookkeeping noise. Safety is preserved by
  `localize-plan`: the commit refuses when ANY dirty path is not a
  committable Hedgeling artifact, so unrelated changes are never swept in.
- **Idempotence is the no-op path.** Nothing new from the API ⇒ no commit
  (a cursor-only delta is run metadata), no push HTTP (`stateHash`
  short-circuit), no publish (bundle bytes unchanged). A tick whose only
  change is a pulled console verdict also skips the push: the pull records
  the post-pull hash (`lastPush.pulledStateHash`), so the console's own
  decisions are never re-uploaded or metered as a state import. A quiet night costs
  one fetch and zero writes anywhere — aggressive schedules are cheap.
- **Budget-stopped runs (exit 2) still commit and push** what completed:
  cataloged-untranslated entries are valid state and the data cursor was
  deliberately held, so the next tick resumes. The run stays green with a
  warning annotation carrying the report counts — repeated budget stops are
  the operator's signal to raise the budget or tighten fields.
- **Publish decouples serving from deploying.** `publish-command` runs LAST,
  only after commit + sync push succeed and only when the bundle bytes
  changed. Git remains the source of truth; the published bundle is a
  derived, reproducible artifact. The spec-04 runtime reloads the published
  URL every `reloadIntervalMs`, so verified translations reach production
  with **no deploy**.

## Quickstart: hourly loop with a Supabase Storage publish

```yaml
# .github/workflows/hedgeling-sync-data.yml
name: Hedgeling sync data
on:
  schedule:
    - cron: '17 * * * *'   # hourly is a sane default for listing data —
                           # the no-op path makes aggressive schedules cheap
  workflow_dispatch:

permissions:
  contents: write           # the direct artifact commit

# Serialize THIS workflow's ticks (its own group — never one shared with the
# translate/sync workflows; see "Concurrency" below).
concurrency:
  group: hedgeling-sync-data
  cancel-in-progress: false

jobs:
  sync-data:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: Hedgeling/actions/sync-data@v1
        env:
          MLS_TOKEN: ${{ secrets.MLS_TOKEN }}   # spec-01 `env:` indirection for adapter auth
        with:
          console-token: ${{ secrets.HEDGELING_CONSOLE_TOKEN }}
          console-url: https://console.example.com
          publish-command: >-
            npx supabase storage cp "$HEDGELING_DYNAMIC_BUNDLE"
            ss:///hedgeling/hedgeling-dynamic-bundle.json --experimental
```

The server then loads the published URL:

```ts
const dyn = await loadDynamicBundle({
  fetch: "https://<project>.supabase.co/storage/v1/object/public/hedgeling/hedgeling-dynamic-bundle.json",
  reloadIntervalMs: 5 * 60_000,
});
```

No `publish-command` ⇒ the publish step is skipped and the bundle serves
from the deployed repo copy — translations then ride deploys (degraded but
correct). **Exception:** `external`-retention sources REQUIRE a publish
command — their bundles are never committed, so publish is the only
delivery path (the CLI validates this up front).

## Concurrency: one group per workflow

Give this workflow its **own** `concurrency:` group, as `actions/translate`
(`hedgeling-localize`) and `actions/sync` (`hedgeling-sync`) document for
theirs:

```yaml
concurrency:
  group: hedgeling-sync-data
  cancel-in-progress: false
```

**Do not put all three workflows in one shared group.** GitHub keeps one
running and only ONE pending run per group: when a third run queues, the
pending one is cancelled without ever starting — and it shows as
"cancelled", not as a failure. In a shared group that dropped run can be a
*different* writer (a merge's localization run, a console push). In a
per-workflow group the dropped run is always superseded by a newer run of
the SAME writer, so nothing is lost: the action's first step fast-forwards
to the branch's current head, so a run that waited in the queue (or a cron
tick that fired while the previous tick was still running) works from the
newest commit rather than the one that triggered it.

Runs of different writers may then overlap; each is built for that:
artifact commits rebase-and-retry (never force), `actions/translate` writes
its own branch, and the sync push's `staleBase` lineage check is the
console-side backstop — the tick surfaces a warning, skips publish, and the
next cron tick self-heals from the fresh checkout.

## Failure modes per step

| Step | Exit | Behavior |
|---|---|---|
| `sync pull` | 1 (conflicts) | Warning, continue — conflicts wait for humans, data keeps flowing. |
| `sync pull` | 2 (transport) | Retry with backoff, then **abort before any commit**. |
| `sync-data` | 1 (failure) | **Abort before any commit.** |
| `sync-data` | 2 (budget) | Warning annotation with report counts; **commit + push anyway** (held cursor resumes next tick). |
| `sync-data` | 2 without a report | Configuration/transport failure (e.g. missing console credentials for `external` retention, or the data source feed unreachable or timed out) — abort before any commit, even when another `--source` budget-stopped. |
| commit | plan `error` | Fail — an unrelated dirty path is never swept into the commit. |
| commit | push rejected | Rebase-and-retry (3 attempts, never force), then fail loudly. |
| `sync push` | 2 (transport) | Retry with backoff, then fail — the artifact commit already landed; the console push retries next tick. |
| cursor bookkeeping | push rejected | Warning only — losing the cursor commit costs efficiency (the next tick does a full, still-correct push), never correctness. |
| publish | non-zero | Fail — git state is already correct; re-run or let the next byte change republish. |

Any other exit code anywhere (an `npx` install/network meltdown, a signal)
fails the step — nothing falls through to a green tick.

**Cursor bookkeeping**: a `sync push` that moved something rewrites
`.hedgeling/sync.json` *after* the artifact commit, and every cron tick
starts from a fresh checkout — so the push step commits the updated cursor
back to the branch (`[skip ci]`, rebase-and-retry, same bot-token override).
Without it the `stateHash` short-circuit would never engage and every quiet
tick would re-push the full state.

## Secrets

The console token and the data-source API credentials arrive as workflow
secrets → env (`console-token` input / job-level `env:` for the spec-01
adapter `env:` indirection). The action never writes them to any artifact —
same law as the sync cursor ("no tokens EVER") — and never puts them on
argv. The publish step receives only `PUBLISH_COMMAND` and the bundle path;
wire storage credentials into it via job-level env.

A `rest` source never forwards credentials off its endpoint's origin: a
next-page link or an HTTP redirect to another origin fails the source
(exit 1) instead of sending `auth.headers` there.

## Timeouts

Every feed request (each page, redirects and body included) has a timeout:
the source's `options.timeoutMs`, else job-level env
`HEDGELING_FEED_TIMEOUT_MS` (milliseconds), else 30 s. A feed that accepts
the connection and never answers fails the tick as a transport error
(exit 2 without a report) instead of holding the job until GitHub kills it.
Console calls use `HEDGELING_CONSOLE_TIMEOUT_MS` the same way (default 30 s;
`sync push` 10 min).

## Inputs

| Input | CLI flag | Default | Notes |
|---|---|---|---|
| `working-directory` | positional `projectDir` | `.` | Directory holding `.hedgeling/extract.config.json`. |
| `sources` | `--source <name>` per name | all configured | Comma-separated subset. |
| `budget-strings` | `--budget-strings` | config `dynamicBudgetStrings` | Per-source translation budget. |
| `publish-command` | — | — | Runs last; bundle path in `$HEDGELING_DYNAMIC_BUNDLE`. |
| `console-token` | env `HEDGELING_CONSOLE_TOKEN` | — | Seat token or service-account credential. Wire from secrets. Never echoed. |
| `console-url` | `--origin` | cursor artifact, then `HEDGELING_CONSOLE_URL` | Console origin. |
| `project` | `--project` | config `projectSlug` | Console project slug. |
| `token` | — | checkout credentials | Bot token for the direct commit push (HTTP(S) origins). |
| `max-attempts` | — | `3` | Retry budget for exit 2 on pull/push. |
| `retry-delay` | — | `10` | Base seconds between retries (linear backoff). |
| `hedgeling-version` | — | pinned per release | Any npm package spec. |
| `npm-token` | scoped `.npmrc` for the action's `npx` calls | — | Token for installing `@hedgeling/i18n`: `npm-token: ${{ secrets.HEDGELING_TOKEN }}`. A Hedgeling seat token (`hdg_...`, from the Seats panel in the console) installs from Hedgeling's registry; any other value is npm auth for the job's default registry. Env-indirected — the token value never lands on disk. Install failures are a configuration error (exit 2) that aborts the tick before any write. |

## Outputs

| Output | Meaning |
|---|---|
| `budget-stopped` | `"true"` when any source stopped at the translation budget. |
| `bundle-changed` | `"true"` when the dynamic bundle bytes changed (the publish gate). |
| `json-path` | Path to the `sync-data --json` report (array for multi-source runs) — set only when the report is non-empty; an empty string means no report was produced. |
| `pull-conflicts` | Conflicts the opening pull classified (never fail the tick). |
| `committed` | `"true"` when the direct artifact commit was pushed. |
| `stale-base` | `"true"` when an out-of-band writer was detected (publish skipped). |
| `published` | `"true"` when the publish command ran; empty when skipped. |

## Versioning

`actions/sync-data/v1` is a moving major alias over immutable `v1.x.y` tags
(standard Actions convention). Pin `@v1` for compatible updates or a full
`@v1.x.y` tag for byte-stable behavior.
