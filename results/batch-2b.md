# Batch 2b results — hosting services, second pass

- **Date/time (start):** 2026-11-15, see each step for times (UTC)
- **Model:** `claude-opus-5-5`
- **How:** `gh api` in the shell for every GitHub REST call. The raw JSON of each call's final read is saved under `results/batch-2b-raw/`. Polls ran every 30 s for up to 10 min, reading `commits/SHA/statuses`, `commits/SHA/check-runs` and `deployments?sha=SHA`, and stopped early once Netlify, Vercel, Cloudflare Pages and Cloudflare Workers Builds all had a final (non-pending / completed) entry.

---

## STEP 1 — A commit to main

This file's first version is the commit pushed to main for this step.

Pushed with `git push origin main` → `5a6e8ce..81e8a52  main -> main` at 20:58:15. Commit: **`81e8a529ded6d6f50c4bdb427d70295547246933`**.

Commands, each run every 30 s:
- `gh api repos/k1sr/vibeassist-tries/commits/81e8a529ded6d6f50c4bdb427d70295547246933/statuses`
- `gh api repos/k1sr/vibeassist-tries/commits/81e8a529ded6d6f50c4bdb427d70295547246933/check-runs`
- `gh api "repos/k1sr/vibeassist-tries/deployments?sha=81e8a529ded6d6f50c4bdb427d70295547246933"`

Poll log (`results/batch-2b-raw/step1-poll-log.txt`). Polls 2–19 are identical to poll 1:
```
poll 0 at 20:58:18 | statuses: Vercel=pending | checks: cloudflare-workers-and-pages/Cloudflare Pages=in_progress/null | deployments: 
poll 1 at 20:58:49 | statuses: Vercel=success, Vercel=pending | checks: cloudflare-workers-and-pages/Cloudflare Pages=completed/success | deployments: 
...
poll 20 at 21:08:43 | statuses: Vercel=success, Vercel=pending | checks: cloudflare-workers-and-pages/Cloudflare Pages=completed/success | deployments: 
time up at 21:08:44 (2 of 4 had reported)
```
The full 10 minutes ran out with only 2 of the 4 hosts reported.

**Every status** (final read):
```
{"creator":"vercel[bot]","context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/Ac6LYVgMvCNaPqt16k2KY5ZWTZ9Y","created_at":"2026-11-15T20:58:24Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/Ac6LYVgMvCNaPqt16k2KY5ZWTZ9Y","created_at":"2026-11-15T20:58:17Z"}
```
**Every check run** (final read):
```
{"app":"cloudflare-workers-and-pages","name":"Cloudflare Pages","status":"completed","conclusion":"success","details_url":"https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/79669eab-6102-4272-a87f-d77a6b4f9e16","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132955638036","external_id":"79669eab-6102-4272-a87f-d77a6b4f9e16","started_at":"2026-11-15T20:58:28Z","completed_at":"2026-11-15T20:58:28Z","output":{"title":"Deployed successfully","summary":"<table><tr><td><strong>Latest commit:</strong> </td><td>\n<code>81e8a52</code>\n</td></tr>\n<tr><td><strong>Status:</strong></td><td>&nbsp;✅&nbsp; Deploy successful!</td></tr>\n<tr><td><strong>Preview URL:</strong></td><td>\n<a href='https://79669eab.vibeassist-tries.pages.dev'>https://79669eab.vibeassist-tries.pages.dev</a>\n</td></tr>\n</table>\n\n[View logs](https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/79669eab-6102-4272-a87f-d77a6b4f9e16)\n","text":null,"annotations_count":0}}
```
**Deployments for this commit:** `[]`. A final unfiltered `gh api repos/k1sr/vibeassist-tries/deployments` also returned `[]`.

**Did each host write the main line's deploy result?**
- **Netlify:** **no.** Nothing in statuses, check runs or deployments in 10 min 29 s.
- **Vercel:** **yes, in a commit status.** Context `Vercel`: `pending` "Vercel is deploying your app" at 20:58:17, then `success` "Deployment has completed" at 20:58:24. target_url (both) `https://vercel.com/xig2/vibeassist-tries/Ac6LYVgMvCNaPqt16k2KY5ZWTZ9Y` (dashboard).
- **Cloudflare Pages:** **yes, in a check run.** "Cloudflare Pages" (app `cloudflare-workers-and-pages`): `completed` / `success`, output.title "Deployed successfully". output.summary gives Latest commit `81e8a52`, Status "✅ Deploy successful!", Preview URL `https://79669eab.vibeassist-tries.pages.dev`, and a "View logs" link. details_url `https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/79669eab-6102-4272-a87f-d77a6b4f9e16`.
- **Cloudflare Workers Builds:** **no.** No check run (or anything else) for this commit.
- No host created a GitHub **deployment**.

