# Hedgeling GitHub Actions

Run [Hedgeling](https://thehedgeling.com) in your GitHub workflows: check
localization on pull requests, receive translations as a pull request, and
keep the Hedgeling console in step with your repository.

| Action | What it does |
|---|---|
| [`check`](./check/README.md) | The localization quality gate for pull requests. |
| [`translate`](./translate/README.md) | Extracts and translates after a merge, and delivers the result as one standing pull request. |
| [`sync`](./sync/README.md) | Pushes repository state to the Hedgeling console and pulls review verdicts back. |
| [`sync-data`](./sync-data/README.md) | Translates content your app loads at runtime, on a schedule. |

## Use

You need a Hedgeling workspace and a seat token (console: Admin, Billing &
usage, Package seats). Store it as the repository secret `HEDGELING_TOKEN`.

```yaml
- uses: actions/checkout@v7
- uses: Hedgeling/actions/check@v1
  with:
    npm-token: ${{ secrets.HEDGELING_TOKEN }}
```

Each action's README lists its inputs and outputs. Complete workflow files to
copy are in [`examples/`](./examples).

## Versions

`@v1` always points at the latest release of version 1 and receives fixes.
To stay on one exact release, pin a full tag such as `@v1.7.0`. Each release
runs the matching version of the `@hedgeling/i18n` package by default.

## About this repository

These actions install and run the `@hedgeling/i18n` package, which is
licensed separately and installs with your seat token. This repository is
published from Hedgeling's own source on each release; changes made here
directly are overwritten. Questions and problems:
support@thehedgeling.com.
