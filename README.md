# DebtDraft

A GitHub Action that scans a pull request for cyclomatic complexity, test coverage
gaps, code duplication, outdated dependencies, and known-vulnerable or malicious
dependencies, combines those five signals into one health score and letter grade, and
posts the result as a comment on the pull request (created on the first run, updated in
place on every later push to the same PR).

It also scans every push to your default branch. Those scans give your repository its
health score and trend on the DebtDraft dashboard, so the score follows the code you
merge, while each pull request shows what it would change.

See [`action.yml`](./action.yml) for the full list of inputs and outputs.

## Prerequisites

This Action requires a [DebtDraft](https://debtdraft.dev) account with this
repository connected. Before adding the workflow:

1. Sign in at the dashboard and connect this repository.
2. Create an API key under Settings > API keys.
3. Add it as a repository secret named `DEBTDRAFT_API_KEY`
   (Settings > Secrets and variables > Actions on this repository).

Every run checks the key and the repository's connection status before any analysis
starts, and fails immediately if either is missing or invalid (see "Authorization" below).

## Usage

```yaml
name: DebtDraft

on:
  pull_request:
  push:
    branches: [main] # your default branch

permissions:
  contents: read
  pull-requests: write

concurrency:
  group: debtdraft-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  health-check:
    name: DebtDraft
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: debtdraft/debtdraft-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          api-key: ${{ secrets.DEBTDRAFT_API_KEY }}
```

The `concurrency` block keeps one run per pull request (a new push cancels the older
run) and lets runs on your default branch finish one after another, so the dashboard
receives them in order.

## Requirements

The Action runs on Node 24. GitHub-hosted runners need nothing extra. A self-hosted
runner must be version 2.327.1 or newer, otherwise the run fails with
`'using: node24' is not supported`.

## Versioning

- `@v1` always points to the latest 1.x release. Non-breaking improvements reach you
  automatically.
- `@v1.0.0` (a full version) never changes.
- For the strictest pinning, use a full commit SHA. Every release lists its commit.

## Authorization

Before running any analysis, DebtDraft checks that `api-key` is valid and that this
repository is connected and active on your plan. The run fails immediately, with no
outputs, no PR comment, and no job summary, when:

- `api-key` is missing;
- the key is invalid or has been revoked;
- this repository is not connected to your DebtDraft team for this key;
- this repository is locked by your plan's repository limit (a Free-plan team keeps its
  oldest-connected repositories active up to the plan's limit; the rest need a Pro
  upgrade or a freed slot).

Each failure's log message says exactly which of these applies. A temporary problem on
DebtDraft's side does not block your CI: a warning is logged and the run continues.

## Permissions

Posting the PR comment needs `pull-requests: write`. Without it, the comment step fails
with an HTTP 403. This is logged as a warning, not a run failure, so the rest of the Action
(scores, outputs, job summary) still completes and succeeds.

A `pull_request` run triggered from a fork gets a **read-only** default `GITHUB_TOKEN`
regardless of the `permissions:` block above. This is a GitHub platform restriction on
fork-triggered workflows, not something this Action or its workflow config can change.
DebtDraft does not recommend switching to `pull_request_target` as a workaround: that
trigger runs with the base repository's elevated token permissions while still checking
out the fork's own code, which lets untrusted PR code run with write access. Every other
DebtDraft output (scores, job summary) works the same either way; only the PR comment
itself is affected.

## What is sent to the DebtDraft dashboard

The authorization check above sends only your repository's GitHub id, authenticated with
`api-key` (see Authorization).

The scan upload itself only happens on `pull_request` events and on pushes to the
repository's default branch. A push to any other branch uploads nothing. Each scan uploads:

- the repository's GitHub id and the commit SHA (the pull request's head commit, or the
  commit pushed to the default branch), plus the pull request number for a pull request;
- the six scores and the letter grade;
- a capped findings document that explains the scores. Every list holds at most 30 items
  plus the true total, and covers:
  - functions above the complexity threshold (file path, function name, line, complexity, length);
  - the files with the most uncovered lines, and the uncovered line ranges of this pull
    request's changed lines;
  - duplicated blocks (file paths and line numbers) and duplicated ranges in changed lines;
  - outdated dependencies, meaning a breaking release behind or deprecated (package name,
    installed and latest version);
  - known-vulnerable dependencies (package name, version, advisory id, severity, fixed
    version and the advisory's public one-line summary).

Never sent to DebtDraft: source code or code snippets, secrets or any secrets finding
(see Secrets detection below), environment variables, or the GitHub token.

File paths, function names and package names are sent, so they appear in your DebtDraft
dashboard, which only members of your team can see. The API key travels as a bearer
token, so `api-url` must use `https://` (plain `http://` is accepted only for localhost).

## Other network requests

To score your dependencies, the Action sends package names and versions, never your
code, to two public services operated by Google: [deps.dev](https://deps.dev) (latest
versions and publish dates) and [OSV.dev](https://osv.dev) (known vulnerabilities and
malicious-package records). A package from a scope mapped to a private registry in
`.npmrc` or `.yarnrc.yml` is never looked up.

The Action also calls the GitHub API with your `github-token`, only to post and update
the pull request comment.

## Secrets detection

DebtDraft scans every git-tracked text file for hardcoded credentials (cloud keys, API
tokens, private keys, database connection strings, and generic high-entropy values assigned
to names like `api_key` or `password`). It is reported **separately from the health score**
and never changes it: a leaked key is not something a good coverage number should offset.

When a pull request introduces a secret, DebtDraft tells you what and where in four places:

- an **annotation on the exact line** in the pull request's Files changed tab (an error for a
  high-confidence match, a warning for a heuristic one);
- a **Secrets section in the pull-request comment**, with the secret type, a link to the file
  and line, and remediation steps;
- a full table in the **job summary**, including secrets that were already in the repository;
- the Action log.

The secret value itself is never printed, logged, put in an output, or sent to the DebtDraft
dashboard.

By default (`fail-on-secrets: true`) the run **fails** when the pull request introduces a
high-confidence secret on one of its own changed lines. It does not fail for a secret already
in the repository, for a heuristic (medium confidence) match alone, or when the scan could not
complete; the comment and job summary are published first, so a failed run still shows what to
fix. Set `fail-on-secrets: false` to report without failing.

If a secret is real, **revoke or rotate it with the provider first**. Editing the line,
amending the commit, or force-pushing does not undo the exposure.

To mark a false positive, add a comment containing `secretlint-disable-next-line` on the line
above it (for example `// secretlint-disable-next-line`), or exclude the path with the
`exclude` input.

What it does not do:

- **It scans the checked-out files, not git history.** A secret committed and later deleted
  stays in history and stays compromised; use a dedicated history scanner for that.
- **It never checks whether a credential is still active.** That would mean sending it to a
  third party.
- Binary files, files over 1 MB, symlinks, and generated or vendored output (`node_modules`,
  `dist`, `build`, minified files, source maps, snapshots, SVGs) are skipped. Test files and
  fixtures are scanned, because real credentials leak into them.

Detection uses [secretlint](https://github.com/secretlint/secretlint) (MIT) for named-provider
patterns, plus a DebtDraft rule for generic high-entropy assignments.

## Support

Report a problem by opening an issue on this repository, or write to
[support@debtdraft.dev](mailto:support@debtdraft.dev). For a security problem, see
[SECURITY.md](./SECURITY.md). The service is covered by the
[Privacy Policy](https://debtdraft.dev/privacy) and the
[Terms and Conditions](https://debtdraft.dev/terms).

## License

All rights reserved. See [LICENSE](./LICENSE). Third-party components are listed in
[THIRD-PARTY-NOTICES.txt](./THIRD-PARTY-NOTICES.txt).
