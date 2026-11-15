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

---

## STEP 2 — A pull request

**What I did:**
- `git checkout -b tries/preview-1` from main (`ec9f5d5`), changed `<h1>vibeassist tries</h1>` to `<h1>vibeassist tries — preview 1</h1>` in index.html, committed, and ran `git push -u origin tries/preview-1` → pushed. Head commit: **`2534760d0d5f5b9c95fcef836eb883028edee8d6`**.
- `gh api repos/k1sr/vibeassist-tries/pulls -f title="Batch 2: preview test (do not merge)" -f head=tries/preview-1 -f base=main -f body=...` →
  `{"created_at":"2026-11-15T20:36:14Z","head":"2534760d0d5f5b9c95fcef836eb883028edee8d6","html_url":"https://github.com/k1sr/vibeassist-tries/pull/4","number":4}`
- A polling script ran `gh api repos/k1sr/vibeassist-tries/commits/2534760d…/status`, `…/statuses` and `…/check-runs` every 30 s, stopping once every host had a non-pending status or a completed check run. Poll log (`results/batch-2-raw/step2-poll-log.txt`):
```
poll 0 at 20:36:26 | statuses: Vercel=pending, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
poll 1 at 20:36:57 | statuses: Vercel=pending, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: cloudflare-workers-and-pages/Workers Builds: vibeassist-tries=in_progress/null, vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
poll 2 at 20:37:29 | statuses: Vercel=pending, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: cloudflare-workers-and-pages/Workers Builds: vibeassist-tries=completed/failure, vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
all three reported at 20:37:29
```
- All three had reported by **20:37:29**, 75 s after the pull request was opened. The final reads are saved raw as `results/batch-2-raw/step2-status.json`, `step2-statuses.json` and `step2-check-runs.json`. At 20:37:42 the check-runs count was still 5.

**Combined status (`/status`):** `{"state":"success","total_count":2}`

**Every status entry (`/statuses`), newest first:**
```
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"success","description":"Deploy Preview ready!","target_url":"https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app","created_at":"2026-11-15T20:36:23Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/6ABXP5FiS5ybaNDamiWPEXm6rz5Y","created_at":"2026-11-15T20:36:21Z"}
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"pending","description":"Deploy Preview processing.","target_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa17c0621639d3f0f442b7","created_at":"2026-11-15T20:36:18Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/6ABXP5FiS5ybaNDamiWPEXm6rz5Y","created_at":"2026-11-15T20:36:15Z"}
```

**Every check run (`/check-runs`):**
```
{"app":"cloudflare-workers-and-pages","name":"Workers Builds: vibeassist-tries","status":"completed","conclusion":"failure","details_url":"https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/e2687254-6d3b-44b2-951d-47e698e4d83e","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132949565412","external_id":"e2687254-6d3b-44b2-951d-47e698e4d83e","started_at":"2026-11-15T20:37:00Z","completed_at":"2026-11-15T20:37:00Z","output":{"title":"Workers Builds: vibeassist-tries","summary":"\nBuild ID: [e2687254-6d3b-44b2-951d-47e698e4d83e](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/e2687254-6d3b-44b2-951d-47e698e4d83e)\nScript: [vibeassist-tries](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production)\n","text":null}}
{"app":"vercel","name":"Vercel Preview Comments","status":"completed","conclusion":"success","details_url":"https://vercel.com/github","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132949384133","external_id":"","started_at":"2026-11-15T20:36:22Z","completed_at":"2026-11-15T20:36:22Z","output":{"title":"✅ No unresolved feedback","summary":"💬 0 unresolved, 0 resolved. [Go to feedback](https://vercel.live/open-feedback/vibeassist-tries-git-tries-preview-1-xig2.vercel.app?via=pr-comment-feedback-link)","text":null}}
{"app":"netlify","name":"Redirect rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa17c0621639d3f0f442b7","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132949371400","external_id":"6afa17c0621639d3f0f442b7","started_at":"2026-11-15T20:36:17Z","completed_at":"2026-11-15T20:36:21Z","output":{"title":"No redirect rules processed","summary":"This deploy did not include any redirect rules.","text":null}}
{"app":"netlify","name":"Header rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa17c0621639d3f0f442b7","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132949369205","external_id":"6afa17c0621639d3f0f442b7","started_at":"2026-11-15T20:36:17Z","completed_at":"2026-11-15T20:36:21Z","output":{"title":"No header rules processed","summary":"This deploy did not include any header rules.","text":null}}
{"app":"netlify","name":"Pages changed - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa17c0621639d3f0f442b7","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132949367156","external_id":"6afa17c0621639d3f0f442b7","started_at":"2026-11-15T20:36:17Z","completed_at":"2026-11-15T20:36:21Z","output":{"title":"1 new file uploaded","summary":"1 generated page and no assets changed.","text":"Changed pages include:\n- index.html - [view](https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app/)\n"}}
```

**By host:**
- **Netlify** (`netlify[bot]`, site `zesty-bienenstitch-c3a6cb`):
  - Status `netlify/zesty-bienenstitch-c3a6cb/deploy-preview`: `pending` "Deploy Preview processing." (target_url `https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa17c0621639d3f0f442b7`), then `success` "Deploy Preview ready!" (target_url **`https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`**).
  - 3 check runs, all `neutral`, details_url = the deploy page above:
    - "Redirect rules" ("No redirect rules processed")
    - "Header rules" ("No header rules processed")
    - "Pages changed" ("1 new file uploaded"). Its output.text carries `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app/`.
- **Vercel** (`vercel[bot]`):
  - Status `Vercel`: `pending` "Vercel is deploying your app", then `success` "Deployment has completed". Both have target_url `https://vercel.com/xig2/vibeassist-tries/6ABXP5FiS5ybaNDamiWPEXm6rz5Y` (dashboard, not the site).
  - Check run "Vercel Preview Comments": `success`, details_url `https://vercel.com/github`. Its output.summary links `https://vercel.live/open-feedback/vibeassist-tries-git-tries-preview-1-xig2.vercel.app?via=pr-comment-feedback-link`.
- **Cloudflare** (`cloudflare-workers-and-pages` app):
  - **No status.** One check run, "Workers Builds: vibeassist-tries": `completed` / **`failure`**.
  - details_url `https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/e2687254-6d3b-44b2-951d-47e698e4d83e`. Its output.summary links the same build page and the script page `…/workers/services/view/vibeassist-tries/production`; output.text null.
  - Note: it is connected as a **Workers** project ("Workers Builds"), not Cloudflare Pages, and the build of this unchanged-config commit **failed**.

**`gh api repos/k1sr/vibeassist-tries/issues/4/comments`** → exit 0, 3 comments (full bodies in `results/batch-2-raw/step2-comments.json`):
```
{"id":6900975077,"user":"vercel[bot]","created_at":"2026-11-15T20:36:16Z","updated_at":"2026-11-15T20:36:21Z","html_url":"https://github.com/k1sr/vibeassist-tries/pull/4#issuecomment-6900975077"}
{"id":6900975686,"user":"netlify[bot]","created_at":"2026-11-15T20:36:18Z","updated_at":"2026-11-15T20:36:24Z","html_url":"https://github.com/k1sr/vibeassist-tries/pull/4#issuecomment-6900975686"}
{"id":6900990768,"user":"cloudflare-workers-and-pages[bot]","created_at":"2026-11-15T20:37:29Z","updated_at":"2026-11-15T20:37:29Z","html_url":"https://github.com/k1sr/vibeassist-tries/pull/4#issuecomment-6900990768"}
```
- **vercel[bot]:** a table with project / deployment "Ready" (`https://vercel.com/xig2/vibeassist-tries/6ABXP5FiS5ybaNDamiWPEXm6rz5Y`) / **Preview `https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app`**, plus a `[vc]: #…` base64 header whose JSON includes `"previewUrl":"vibeassist-tries-git-tries-preview-1-xig2.vercel.app"`.
- **netlify[bot]:** "Deploy Preview for *zesty-bienenstitch-c3a6cb* ready!", with the latest commit `2534760…`, the deploy log link, and Deploy Preview `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`.
- **cloudflare-workers-and-pages[bot]:** "Deploying Preview to Cloudflare". Its table shows **Build: Failed ❌** for commit 2534760 with a "View logs" link to `https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/e2687254-…`. **The Deployment URL cell is empty.**

---

## STEP 3 — The preview link, per host

From the reads recorded in step 2 (no new calls):

- **Netlify:** preview link **`https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`**.
  - Field: `target_url` of the **status** with context `netlify/zesty-bienenstitch-c3a6cb/deploy-preview`, state `success`, from `GET /commits/2534760…/statuses` (also in `/status`, `.statuses[]`). **It is on a status.**
  - The same link (with a trailing `/`) is also in the **check run** "Pages changed - zesty-bienenstitch-c3a6cb", in `output.text`, and in netlify[bot]'s PR comment.
  - The *pending* status's target_url is the Netlify dashboard deploy page, not the preview.
- **Vercel:** preview link **`https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app`**.
  - The field it comes from is the **comment** by vercel[bot] (the "Preview" link in its table, and `previewUrl` in the comment's base64 `[vc]` header).
  - It is **not** on the status: the `Vercel` status's `target_url` is the dashboard page `https://vercel.com/xig2/vibeassist-tries/6ABXP5FiS5ybaNDamiWPEXm6rz5Y`.
  - It is **not** in any check run's details_url (`https://vercel.com/github`). The only place the hostname shows up in a check run is embedded inside a different link in the "Vercel Preview Comments" check's `output.summary`: `https://vercel.live/open-feedback/vibeassist-tries-git-tries-preview-1-xig2.vercel.app?via=pr-comment-feedback-link`. That is a vercel.live feedback link, not the preview link itself.
  - **Vercel gives no direct preview link in any status or check run; the direct link is only in its comment.**
- **Cloudflare:** **no preview link at all.** Its only check run ("Workers Builds: vibeassist-tries") ended in `failure`, and its details_url and output.summary link only to the dash.cloudflare.com build and script pages. It wrote no status. Its comment's "Deployment URL" cell is empty because the build failed.
  - **Cloudflare gives no preview link in any status, check run or comment for this pull request.**

---

## STEP 4 — A failed deploy

### 4a. Breaking the build

On `tries/preview-1` I changed package.json's `"build"` to `"echo intentional build failure for batch 2 && exit 1"` (locally `npm run build` → exit 1), committed and pushed: `2534760..62cdf18  tries/preview-1 -> tries/preview-1` at 20:38:28. Head: **`62cdf1804babce89e5d11b588ab106ce3783e6a8`**.

Poll log (`results/batch-2-raw/step4-fail-poll-log.txt`):
```
poll 0 at 20:38:31 | statuses: Vercel=pending | checks: vercel/Vercel Preview Comments=completed/success
poll 1 at 20:39:02 | statuses: Vercel=failure, Vercel=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
poll 2 at 20:39:34 | statuses: Vercel=failure, Vercel=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: cloudflare-workers-and-pages/Workers Builds: vibeassist-tries=completed/failure, vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
all three reported at 20:39:34
```
`/status`: `{"state":"failure","total_count":2}`

`/statuses`:
```
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"success","description":"Deploy Preview ready!","target_url":"https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app","created_at":"2026-11-15T20:38:38Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"failure","description":"Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_ApCCLJgyx39eh9GKrYfcaToh9SSf --logs","target_url":"https://vercel.com/xig2/vibeassist-tries/ApCCLJgyx39eh9GKrYfcaToh9SSf","created_at":"2026-11-15T20:38:36Z"}
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"pending","description":"Deploy Preview processing.","target_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1847576519d1f025d91d","created_at":"2026-11-15T20:38:32Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/ApCCLJgyx39eh9GKrYfcaToh9SSf","created_at":"2026-11-15T20:38:30Z"}
```
`/check-runs`:
```
{"app":"cloudflare-workers-and-pages","name":"Workers Builds: vibeassist-tries","status":"completed","conclusion":"failure","details_url":"https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/5cf30b6b-4d8f-438f-b1c6-44127515d10b","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950233324","output":{"title":"Workers Builds: vibeassist-tries","summary":"\nBuild ID: [5cf30b6b-4d8f-438f-b1c6-44127515d10b](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/5cf30b6b-4d8f-438f-b1c6-44127515d10b)\nScript: [vibeassist-tries](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production)\n","text":null}}
{"app":"vercel","name":"Vercel Preview Comments","status":"completed","conclusion":"success","details_url":"https://vercel.com/github","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950027333","output":{"title":"✅ No unresolved feedback","summary":"💬 0 unresolved, 0 resolved.","text":null}}
{"app":"netlify","name":"Redirect rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1847576519d1f025d91d","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950013720","output":{"title":"No redirect rules processed","summary":"This deploy did not include any redirect rules.","text":null}}
{"app":"netlify","name":"Header rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1847576519d1f025d91d","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950011755","output":{"title":"No header rules processed","summary":"This deploy did not include any header rules.","text":null}}
{"app":"netlify","name":"Pages changed - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1847576519d1f025d91d","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950010070","output":{"title":"1 new file uploaded","summary":"No generated pages and 1 asset changed.","text":null}}
```

**How each host reported the failure:**
- **Vercel:** status `Vercel` → **`failure`**, description "Deployment has failed — run this Vercel CLI command: npx vercel inspect dpl_ApCCLJgyx39eh9GKrYfcaToh9SSf --logs", target_url `https://vercel.com/xig2/vibeassist-tries/ApCCLJgyx39eh9GKrYfcaToh9SSf`. Its check run "Vercel Preview Comments" stayed `success` (it only tracks feedback). Its PR comment was updated at 20:38:36 to show **Error** (link `https://vercel.com/xig2/vibeassist-tries/ApCCLJgyx39eh9GKrYfcaToh9SSf`) with an empty Preview cell.
- **Netlify: did NOT report a failure.**
  - Status → **`success`** "Deploy Preview ready!", target_url `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`. Its 3 check runs were `neutral`; "Pages changed" said "No generated pages and **1 asset changed**".
  - Its comment was updated to "Deploy Preview … ready!" for commit 62cdf18.
  - The combined `/status` state is still `failure` because of Vercel.
  - So Netlify is not actually running `npm run build` with a failing exit code, or ignores it. It looks as if it publishes repository files directly: the changed package.json counted as "1 asset changed". I could not open the preview to confirm (step 5).
- **Cloudflare:** check run "Workers Builds: vibeassist-tries" → **`failure`**. details_url `https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/5cf30b6b-4d8f-438f-b1c6-44127515d10b`; output.summary has the build ID link and script link; output.text null. No status.
  - Its PR comment was **not updated**: it still shows only commit 2534760, last updated 20:37:29.
  - Cloudflare's builds had already failed before I broke anything (step 2), so this failure is not distinguishable from that one.

### 4b. Restoring the build

Restored `"build": "mkdir -p dist && cp index.html dist/"`, committed and pushed: `62cdf18..e587e1b  tries/preview-1 -> tries/preview-1` at 20:39:51. Head: **`e587e1bdfeca9db5e8cb27157a31c7cd3f088ddb`**.

Poll log (`results/batch-2-raw/step4-fix-poll-log.txt`):
```
poll 0 at 20:39:56 | statuses: Vercel=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending | checks: netlify/Header rules - zesty-bienenstitch-c3a6cb=in_progress/null, netlify/Pages changed - zesty-bienenstitch-c3a6cb=in_progress/null, cloudflare-workers-and-pages/Workers Builds: vibeassist-tries=in_progress/null, vercel/Vercel Preview Comments=completed/success
poll 1 at 20:40:27 | statuses: Vercel=pending, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success | checks: cloudflare-workers-and-pages/Workers Builds: vibeassist-tries=completed/failure, vercel/Vercel Preview Comments=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral
all three reported at 20:40:27
```
`/status`: `{"state":"success","total_count":2}`

`/statuses`:
```
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"success","description":"Deploy Preview ready!","target_url":"https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app","created_at":"2026-11-15T20:40:01Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/6kXwTpvewzh977VqQGBsBMkEArKp","created_at":"2026-11-15T20:39:59Z"}
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"pending","description":"Deploy Preview processing.","target_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa189a0d5bcdd4105a2864","created_at":"2026-11-15T20:39:55Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/6kXwTpvewzh977VqQGBsBMkEArKp","created_at":"2026-11-15T20:39:54Z"}
```
`/check-runs`:
```
{"app":"cloudflare-workers-and-pages","name":"Workers Builds: vibeassist-tries","status":"completed","conclusion":"failure","details_url":"https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/db2ec5aa-b728-4342-b740-9a62fa5ba9b4","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950476989","output":{"title":"Workers Builds: vibeassist-tries","summary":"\nBuild ID: [db2ec5aa-b728-4342-b740-9a62fa5ba9b4](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production/builds/db2ec5aa-b728-4342-b740-9a62fa5ba9b4)\nScript: [vibeassist-tries](https://dash.cloudflare.com/5ab782b7007da791a8e2bd06113aeb72/workers/services/view/vibeassist-tries/production)\n","text":null}}
{"app":"vercel","name":"Vercel Preview Comments","status":"completed","conclusion":"success","details_url":"https://vercel.com/github","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950421542","output":{"title":"✅ No unresolved feedback","summary":"💬 0 unresolved, 0 resolved. [Go to feedback](https://vercel.live/open-feedback/vibeassist-tries-git-tries-preview-1-xig2.vercel.app?via=pr-comment-feedback-link)","text":null}}
{"app":"netlify","name":"Redirect rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa189a0d5bcdd4105a2864","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950406909","output":{"title":"No redirect rules processed","summary":"This deploy did not include any redirect rules.","text":null}}
{"app":"netlify","name":"Header rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa189a0d5bcdd4105a2864","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950405017","output":{"title":"No header rules processed","summary":"This deploy did not include any header rules.","text":null}}
{"app":"netlify","name":"Pages changed - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa189a0d5bcdd4105a2864","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132950403132","output":{"title":"All files already uploaded","summary":"All files already uploaded by a previous deploy with the same commits.","text":null}}
```

**Back to success?**
- **Vercel:** yes. `success` "Deployment has completed", target_url `https://vercel.com/xig2/vibeassist-tries/6kXwTpvewzh977VqQGBsBMkEArKp`.
- **Netlify:** `success` "Deploy Preview ready!" (it never left success). "Pages changed" said "All files already uploaded by a previous deploy with the same commits."
- **Cloudflare:** **no.** "Workers Builds: vibeassist-tries" is `failure` again (build `db2ec5aa-b728-4342-b740-9a62fa5ba9b4`). Its comment still shows only 2534760 / Failed. Cloudflare has failed on every commit of this pull request, including the two with a working build script.

---

## STEP 5 — Opening each preview link from this session

Commands and output (the TLS/CA lines curl prints are trimmed; the rest is exact):
```
$ curl -sS -v -m 20 https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app
(at 20:41:05 UTC)
* Uses proxy env variable no_proxy == 'localhost,127.0.0.1,::1,127.0.0.0/8,0.0.0.0/8,::,169.254.0.0/16,api.anthropic.com,api-staging.anthropic.com,api-pr-preview.anthropic.com,mcp-proxy.anthropic.com,mcp-proxy-staging.anthropic.com,registry.npmjs.org,jsr.io,npm.jsr.io,pypi.org,files.pythonhosted.org,index.crates.io,proxy.golang.org,host.docker.internal,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,100.64.0.0/10,.svc.cluster.local,*.svc.cluster.local'
* Uses proxy env variable https_proxy == 'http://127.0.0.1:41683'
* Connected to 127.0.0.1 (127.0.0.1) port 41683
* CONNECT tunnel: HTTP/1.1 negotiated
* allocate connect buffer
* Establish HTTP proxy tunnel to deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app:443
> CONNECT deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app:443 HTTP/1.1
> Host: deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app:443
> User-Agent: curl/8.5.0
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 403 Forbidden
< Content-Type: text/plain; charset=utf-8
< X-Content-Type-Options: nosniff
< Content-Length: 113
< Connection: close
< 
* CONNECT tunnel failed, response 403
* Closing connection
curl: (56) CONNECT tunnel failed, response 403
[curl exit 56]

$ curl -sS -v -m 20 https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app
(at 20:41:05 UTC)
* Uses proxy env variable no_proxy == 'localhost,127.0.0.1,::1,127.0.0.0/8,0.0.0.0/8,::,169.254.0.0/16,api.anthropic.com,api-staging.anthropic.com,api-pr-preview.anthropic.com,mcp-proxy.anthropic.com,mcp-proxy-staging.anthropic.com,registry.npmjs.org,jsr.io,npm.jsr.io,pypi.org,files.pythonhosted.org,index.crates.io,proxy.golang.org,host.docker.internal,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,100.64.0.0/10,.svc.cluster.local,*.svc.cluster.local'
* Uses proxy env variable https_proxy == 'http://127.0.0.1:41683'
* Connected to 127.0.0.1 (127.0.0.1) port 41683
* CONNECT tunnel: HTTP/1.1 negotiated
* allocate connect buffer
* Establish HTTP proxy tunnel to vibeassist-tries-git-tries-preview-1-xig2.vercel.app:443
> CONNECT vibeassist-tries-git-tries-preview-1-xig2.vercel.app:443 HTTP/1.1
> Host: vibeassist-tries-git-tries-preview-1-xig2.vercel.app:443
> User-Agent: curl/8.5.0
> Proxy-Connection: Keep-Alive
> 
< HTTP/1.1 403 Forbidden
< Content-Type: text/plain; charset=utf-8
< X-Content-Type-Options: nosniff
< Content-Length: 110
< Connection: close
< 
* CONNECT tunnel failed, response 403
* Closing connection
curl: (56) CONNECT tunnel failed, response 403
[curl exit 56]
```
(Earlier, at 20:40:02, the same two URLs plus `…netlify.app/package.json` without `-v` gave the same `curl: (56) CONNECT tunnel failed, response 403` and `http_code=000`.)

- **Netlify** `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`: **refused**. This session's outbound proxy answered the CONNECT with `HTTP/1.1 403 Forbidden`; curl exit 56. The request never reached Netlify.
- **Vercel** `https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app`: **refused** the same way (`403 Forbidden` on CONNECT, curl exit 56).
- **Cloudflare:** **not tried.** There is no preview link to open, because every Cloudflare build on this pull request failed and it never published a URL (step 3).

The refusal comes from this cloud environment's network policy (the preview hosts are not on its allowed list), not from the hosts. So I could not check what the previews actually serve, including whether Netlify's "successful" preview of the broken commit is just the raw repository files.

---

## Result lines

**TRY 9 RESULT: passes** — a session can read Netlify's deploy result for a commit. It is the commit status with context `netlify/zesty-bienenstitch-c3a6cb/deploy-preview` (`pending` "Deploy Preview processing." → `success` "Deploy Preview ready!"), readable with `gh api …/commits/SHA/statuses` or `/status`, plus 3 `neutral` check runs linking the deploy page. **Caveat:** Netlify reported `success` on commit 62cdf18, whose build script exits 1. So the readable result does not reflect `npm run build` failing. Netlify apparently is not running that build. Also, on main (step 1) Netlify wrote nothing at all.

**TRY 10 RESULT, Netlify: passes** — preview link `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`, from the `target_url` of the `success` **status** `netlify/zesty-bienenstitch-c3a6cb/deploy-preview`. It is also in the "Pages changed" check run's `output.text`.

**TRY 10 RESULT, Vercel: fails** — the preview link `https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app` is only in vercel[bot]'s PR **comment**. The `Vercel` status's `target_url` is the vercel.com dashboard page. The only check run, "Vercel Preview Comments", has details_url `https://vercel.com/github`, and the preview hostname appears only inside a vercel.live feedback URL in its `output.summary`, not as a preview link.

**TRY 10 RESULT, Cloudflare: fails** — no preview link anywhere. It writes only a check run ("Workers Builds: vibeassist-tries", app `cloudflare-workers-and-pages`), whose links go to dash.cloudflare.com build pages. All its builds on this pull request failed, including those with a working build script, and its comment's Deployment URL cell is empty. It is also set up as a **Workers** project, not Cloudflare Pages.

---

## Left in place

- **Open pull request:** https://github.com/k1sr/vibeassist-tries/pull/4 "Batch 2: preview test (do not merge)" — not merged. Head `e587e1b` (build script restored).
- **Branches:** `tries/preview-1` (this batch). From batch 1: `tries/pr-a`, `tries/pr-b`, `tries/workflows`.
- **Files on main from this batch:** `results/batch-2.md`, `results/batch-2-raw/*` (raw API output; no secrets).
- **Finished:** 2026-11-15 ~20:42 UTC.
