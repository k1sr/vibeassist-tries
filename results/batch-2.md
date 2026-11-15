# Batch 2 results — what hosting services write on GitHub

- **Date/time (start):** 2026-11-15 20:35 UTC
- **Model:** `claude-opus-5-5` (as reported by `get_session` earlier in this session)
- **How I called GitHub:** `gh api` in the shell, as asked. Every call's raw JSON output is saved, byte for byte, under `results/batch-2-raw/` (named in each step). Below I show the fields that matter. Where a response is long (the combined-status response repeats the whole repository object, and every status repeats its creator's user object), the full text is in the raw file.

---

## STEP 1 — The main line

Latest commit on main: `a6d685b7e523328f3013f5b140701bddd837bcdd` ("Results batch 1a: TRY 1 follow-up", committed 11:20:08Z). Calls made at 20:35:47 UTC.

**1. `gh api repos/k1sr/vibeassist-tries/commits/a6d685b7e523328f3013f5b140701bddd837bcdd/status`** → exit 0, 4974 bytes (raw: `results/batch-2-raw/step1-status.json`):
```
{"state":"success","sha":"a6d685b7e523328f3013f5b140701bddd837bcdd","total_count":1,"statuses":[{"context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/DDpdjwZUTGuqLhhtUAYLW3fAqXKt","created_at":"2026-11-15T20:34:47Z","avatar_url":"https://avatars.githubusercontent.com/in/8329?v=4"}]}
```
(The remaining key is `repository`: the full repository object.)

**2. `gh api repos/k1sr/vibeassist-tries/commits/a6d685b7e523328f3013f5b140701bddd837bcdd/statuses`** → exit 0, 12140 bytes, 8 entries (raw: `results/batch-2-raw/step1-statuses.json`). Each entry, newest first; every `creator` is `vercel[bot]`:
```
{"context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/DDpdjwZUTGuqLhhtUAYLW3fAqXKt","created_at":"2026-11-15T20:34:47Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/DDpdjwZUTGuqLhhtUAYLW3fAqXKt","created_at":"2026-11-15T20:34:41Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"failure","description":"Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_DsFwtSasNusNg9ZsH5D76NqJ6c9a --logs","target_url":"https://vercel.com/xig2/vibeassist-tries/DsFwtSasNusNg9ZsH5D76NqJ6c9a","created_at":"2026-11-15T20:34:21Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/DsFwtSasNusNg9ZsH5D76NqJ6c9a","created_at":"2026-11-15T20:34:15Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"failure","description":"Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_2GaQBWuqpncznfyGcBHmx4xjNruM --logs","target_url":"https://vercel.com/xig2/vibeassist-tries/2GaQBWuqpncznfyGcBHmx4xjNruM","created_at":"2026-11-15T20:32:57Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/2GaQBWuqpncznfyGcBHmx4xjNruM","created_at":"2026-11-15T20:32:51Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"failure","description":"Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_8k1boGev7nU2Dba3qD2g9bErUFNY --logs","target_url":"https://vercel.com/xig2/vibeassist-tries/8k1boGev7nU2Dba3qD2g9bErUFNY","created_at":"2026-11-15T20:28:32Z","creator":"vercel[bot]"}
{"context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/8k1boGev7nU2Dba3qD2g9bErUFNY","created_at":"2026-11-15T20:28:26Z","creator":"vercel[bot]"}
```

**3. `gh api repos/k1sr/vibeassist-tries/commits/a6d685b7e523328f3013f5b140701bddd837bcdd/check-runs`** → exit 0:
```
{"total_count":0,"check_runs":[]}
```

**4. `gh api repos/k1sr/vibeassist-tries/deployments`** → exit 0 (not refused):
```
[]
```

**Per host, for commit a6d685b:**
- **Vercel:** wrote **commit statuses** only (context `Vercel`, creator `vercel[bot]`). It deployed this commit four times between 20:28 and 20:34 UTC. The first three ended in `failure` ("Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_… --logs"); the fourth ended in `success` ("Deployment has completed"). The combined state is `success`. Every `target_url` is a vercel.com dashboard link (`https://vercel.com/xig2/vibeassist-tries/<id>`), not a site URL. No check runs, no deployments.
- **Netlify:** **nothing** in statuses, check runs or deployments.
- **Cloudflare Pages:** **nothing** in statuses, check runs or deployments.

