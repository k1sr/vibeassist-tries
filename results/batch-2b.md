# Batch 2b results — hosting services, second pass

- **Date/time (start):** 2026-11-15, see each step for times (UTC)
- **Model:** `claude-opus-5-5`
- **How:** `gh api` in the shell for every GitHub REST call. The raw JSON of each call's final read is saved under `results/batch-2b-raw/`. Polls ran every 30 s for up to 10 min, reading `commits/SHA/statuses`, `commits/SHA/check-runs` and `deployments?sha=SHA`, and stopped early once Netlify, Vercel, Cloudflare Pages and Cloudflare Workers Builds all had a final (non-pending / completed) entry.

---

## STEP 1 — A commit to main

This file's first version is the commit pushed to main for this step.
