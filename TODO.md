# TODO — vetlock-web-scans

Repo-specific follow-ups. See `hosted-scan.yml`'s README section for the
current end-to-end status of the hosted-scan API path.

- [ ] Wait for `vetlock@0.7.0` npm publish (tracked as the P0 item in
      `vetlock/TODO.md`). Until it lands, `hosted-scan.yml`'s
      `npx vetlock@latest` step 404s and every hosted scan reports back
      verdict `"failed"` — the workflow itself is correct, it just has
      nothing to `npx`.
- [ ] Add a schedule trigger to `cleanup.yml` for the `results/` dir — old
      scan output files grow unbounded otherwise. `results/` currently has
      16 files. `cleanup.yml` already has `on.schedule` wired
      (`"17 4 * * *"`, daily) and is running successfully, so this item is
      about confirming the purge threshold (currently `-mtime +0`, i.e.
      >24h) keeps pace as `scan.yml` volume grows — revisit if `results/`
      starts trending upward between runs.
- [ ] Consider a self-hosted runner if we hit the 20-concurrent-job GitHub
      Actions cap (public-repo free tier). Currently at low volume — not
      needed yet.
- [ ] Add a smoke-test workflow that fires `hosted-scan.yml` once a day
      with a known-good lockfile pair and asserts the Worker gets a proper
      `/result-scan` callback. Catches Worker↔runner contract drift (e.g.
      the `scan_id` shape mismatch fixed in 505d711) before it silently
      breaks the hosted-scan path.
