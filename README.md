# vetlock-web-scans

Public backend for the lockfile-scan demo on [oj-uday.github.io](https://oj-uday.github.io).

**Don't dispatch this workflow by hand.** Use the site.

## What this is

When someone drops a `package-lock.json` pair into the scanner on
[oj-uday.github.io](https://oj-uday.github.io), a Cloudflare Worker
(source: [`vetlock-scan.oj-uday.workers.dev`](https://vetlock-scan.oj-uday.workers.dev))
dispatches `.github/workflows/scan.yml` on this repo with the two lockfiles
base64+gzip-encoded as workflow inputs. The workflow:

1. Checks out [`vetlock`](https://github.com/OJ-Uday/vetlock) at tag `v0.3.0`.
2. Installs + builds it with pnpm.
3. Runs `node dist/cli.js diff before.json after.json --json`.
4. Commits the JSON result to `results/<scan_id>.json` in this repo.

The Worker polls the raw file URL every 2s. When it appears, the site renders
the vetlock findings in-page. Results are ephemeral — a cleanup workflow purges
files older than 24 h.

## Why a whole separate repo

The scanner needs a place to write scan results that's:
- **public** — no auth wall for the Worker to read
- **cheap** — GitHub Actions is free for public repos
- **auditable** — anyone can see the exact code the workflow runs

Splitting it out of `oj-uday.github.io` keeps the portfolio repo itself
never-mutated-by-a-bot, which is nice for `git log` hygiene.

## Cost

$0/month. GitHub Actions has no minute cap on public repos.

## Abuse protections (at the Worker)

- Rate limit: 3 scans / 60s per IP.
- Payload cap: 500 KB per lockfile, 1 MB total.
- Content-type must be `application/json`.
- Both lockfiles must parse as npm/yarn/pnpm lockfiles at the Worker before we spend a workflow run.

## Local repro of what the workflow does

```bash
# Same command the workflow runs.
git clone https://github.com/OJ-Uday/vetlock.git && cd vetlock
git checkout v0.3.0
pnpm install --frozen-lockfile
pnpm -r build
node packages/cli/dist/cli.js diff /path/to/before.json /path/to/after.json --json
```

Or, faster:

```bash
npx @oj-uday/vetlock@0.8.0 diff before.json after.json --json
```

## Setup (repo owner only)

The workflow just needs `contents: write` on this repo — no PATs required. The
Cloudflare Worker needs a fine-grained PAT with `actions:write` on this repo.
See [SETUP.md](https://github.com/OJ-Uday/OJ-Uday.github.io/blob/main/SETUP.md)
in the portfolio repo.

## License

MIT.

## Hosted scan API workflow

Workflow: [`.github/workflows/hosted-scan.yml`](.github/workflows/hosted-scan.yml).

`hosted-scan.yml` runs vetlock against a lockfile pair delivered via
`repository_dispatch` and reports the verdict straight back over HTTPS —
no commit to this repo, no polling.

It's triggered by a `repository_dispatch` event with `event_type: hosted-scan`,
sent by the Cloudflare Worker's `POST /scan` endpoint at
[`vetlock-app.oj-uday.workers.dev/scan`](https://vetlock-app.oj-uday.workers.dev/scan) —
not manually. The Worker POSTs the two lockfiles base64+gzip-encoded as
`client_payload`, and the workflow POSTs the result to the Worker's
`/result-scan` endpoint when it's done (success, WARN/BLOCK, or crash — it
always reports back).

Repo owner setup: add `VETLOCK_HOSTED_INGEST_SECRET` to this repo's Actions
secrets (must match the Worker's `HOSTED_SCAN_INGEST_SECRET` — note this is a
*different* secret from `RESULT_INGEST_SECRET`, which only guards the
GitHub App webhook path) so the workflow can authenticate its callback to
`/result-scan`.

Cost: $0/month, same as everything else here — GitHub Actions has no minute
cap on public repos.

**Verified working 2026-07-15:** end-to-end lifecycle (Worker `POST /scan` →
`repository_dispatch` → this workflow → `POST /result-scan` callback)
succeeds. It currently returns a `"failed"` verdict because `@oj-uday/vetlock@latest`
isn't published to npm yet (`npx @oj-uday/vetlock@latest` has nothing to install) —
this workflow will start returning real verdicts once P3 npm-publish
completes on the `vetlock` repo.

### Runbook: manually redeliver a failed dispatch

`hosted-scan.yml`'s only trigger is `repository_dispatch` — it has no
`workflow_dispatch` block, so plain `gh workflow run hosted-scan.yml` will
be rejected with "workflow does not have a workflow_dispatch trigger". To
redeliver a dispatch by hand for debugging, hit the same `dispatches` API
endpoint the Worker uses, via `gh api`:

```bash
gh api repos/OJ-Uday/vetlock-web-scans/dispatches \
  -f event_type=hosted-scan \
  -f 'client_payload[scan_id]=debug-manual-test' \
  -f 'client_payload[lockfile_before_b64]=<base64+gzip of before lockfile>' \
  -f 'client_payload[lockfile_after_b64]=<base64+gzip of after lockfile>'
```

Then watch the run and check the callback landed:

```bash
gh run list --repo OJ-Uday/vetlock-web-scans --workflow=hosted-scan.yml --limit 5
gh run watch --repo OJ-Uday/vetlock-web-scans <run-id>
```

The result never gets committed to this repo (that's the point of this
workflow) — confirm success by checking the Worker's `/scan/:id` endpoint
for the `scan_id` you used, or by tailing the Worker with `wrangler tail`.
