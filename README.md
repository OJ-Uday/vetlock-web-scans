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
npx vetlock@0.3.0 diff before.json after.json --json
```

## Setup (repo owner only)

The workflow just needs `contents: write` on this repo — no PATs required. The
Cloudflare Worker needs a fine-grained PAT with `actions:write` on this repo.
See [SETUP.md](https://github.com/OJ-Uday/OJ-Uday.github.io/blob/main/SETUP.md)
in the portfolio repo.

## License

MIT.

## Hosted scan API workflow

`hosted-scan.yml` runs vetlock against a lockfile pair delivered via
`repository_dispatch` and reports the verdict straight back over HTTPS —
no commit to this repo, no polling.

It's triggered by the Cloudflare Worker at
[`vetlock-app.oj-uday.workers.dev`](https://vetlock-app.oj-uday.workers.dev),
not manually. The Worker POSTs a `repository_dispatch` event of type
`hosted-scan` with the two lockfiles base64+gzip-encoded, and the workflow
POSTs the result to the Worker's `/result-scan` endpoint when it's done
(success, WARN/BLOCK, or crash — it always reports back).

Repo owner setup: add `VETLOCK_HOSTED_INGEST_SECRET` to this repo's Actions
secrets (must match the Worker's `RESULT_INGEST_SECRET`) so the workflow can
authenticate its callback to `/result-scan`.

Cost: $0/month, same as everything else here — GitHub Actions has no minute
cap on public repos.
