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

---

## STEP 2 — A visible change on pull request #4

On `tries/preview-1` I changed the paragraph to `<p>A tiny static site used for tests. Preview pass 2b.</p>`, then committed and pushed: `e587e1b..ca969ad  tries/preview-1 -> tries/preview-1` at 21:09:13. Head: **`ca969ad50f1fb3f715ede4f2b713d13af692e5e2`**.

Poll log (`results/batch-2b-raw/step2-poll-log.txt`). Polls 2–19 are identical to poll 1:
```
poll 0 at 21:09:17 | statuses: Vercel=pending | checks: vercel/Vercel Preview Comments=completed/success, cloudflare-workers-and-pages/Cloudflare Pages=in_progress/null | deployments: 
poll 1 at 21:09:48 | statuses: netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, Vercel=pending | checks: cloudflare-workers-and-pages/Cloudflare Pages=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, vercel/Vercel Preview Comments=completed/success, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral | deployments: 
...
poll 20 at 21:19:42 | statuses: netlify/zesty-bienenstitch-c3a6cb/deploy-preview=success, Vercel=success, netlify/zesty-bienenstitch-c3a6cb/deploy-preview=pending, Vercel=pending | checks: cloudflare-workers-and-pages/Cloudflare Pages=completed/success, netlify/Redirect rules - zesty-bienenstitch-c3a6cb=completed/neutral, vercel/Vercel Preview Comments=completed/success, netlify/Header rules - zesty-bienenstitch-c3a6cb=completed/neutral, netlify/Pages changed - zesty-bienenstitch-c3a6cb=completed/neutral | deployments: 
time up at 21:19:44 (3 of 4 had reported)
```
3 of 4 hosts reported by 21:09:48. **Cloudflare Workers Builds wrote nothing for this commit in 10 min 31 s.**

**Every status:**
```
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"success","description":"Deploy Preview ready!","target_url":"https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app","created_at":"2026-11-15T21:09:29Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"success","description":"Deployment has completed","target_url":"https://vercel.com/xig2/vibeassist-tries/4ugpEZjiQfcESw3SxxpXxAtZZb8S","created_at":"2026-11-15T21:09:20Z"}
{"creator":"netlify[bot]","context":"netlify/zesty-bienenstitch-c3a6cb/deploy-preview","state":"pending","description":"Deploy Preview processing.","target_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1f7c6fb055bedcfb4721","created_at":"2026-11-15T21:09:20Z"}
{"creator":"vercel[bot]","context":"Vercel","state":"pending","description":"Vercel is deploying your app","target_url":"https://vercel.com/xig2/vibeassist-tries/4ugpEZjiQfcESw3SxxpXxAtZZb8S","created_at":"2026-11-15T21:09:15Z"}
```
**Every check run:**
```
{"app":"cloudflare-workers-and-pages","name":"Cloudflare Pages","status":"completed","conclusion":"success","details_url":"https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132958969074","external_id":"9c07c340-a13b-4e1f-98b9-f00d5b178d13","started_at":"2026-11-15T21:09:34Z","completed_at":"2026-11-15T21:09:34Z","output":{"title":"Deployed successfully","summary":"<table><tr><td><strong>Latest commit:</strong> </td><td>\n<code>ca969ad</code>\n</td></tr>\n<tr><td><strong>Status:</strong></td><td>&nbsp;✅&nbsp; Deploy successful!</td></tr>\n<tr><td><strong>Preview URL:</strong></td><td>\n<a href='https://9c07c340.vibeassist-tries.pages.dev'>https://9c07c340.vibeassist-tries.pages.dev</a>\n</td></tr>\n<tr><td><strong>Branch Preview URL:</strong></td><td>\n<a href='https://tries-preview-1.vibeassist-tries.pages.dev'>https://tries-preview-1.vibeassist-tries.pages.dev</a>\n</td></tr>\n</table>\n\n[View logs](https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13)\n","text":null}}
{"app":"netlify","name":"Redirect rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1f7c6fb055bedcfb4721","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132958906070","external_id":"6afa1f7c6fb055bedcfb4721","started_at":"2026-11-15T21:09:19Z","completed_at":"2026-11-15T21:09:27Z","output":{"title":"No redirect rules processed","summary":"This deploy did not include any redirect rules.","text":null}}
{"app":"vercel","name":"Vercel Preview Comments","status":"completed","conclusion":"success","details_url":"https://vercel.com/github","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132958904386","external_id":"","started_at":"2026-11-15T21:09:21Z","completed_at":"2026-11-15T21:09:21Z","output":{"title":"✅ No unresolved feedback","summary":"💬 0 unresolved, 0 resolved. [Go to feedback](https://vercel.live/open-feedback/vibeassist-tries-git-tries-preview-1-xig2.vercel.app?via=pr-comment-feedback-link)","text":null}}
{"app":"netlify","name":"Header rules - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1f7c6fb055bedcfb4721","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132958903990","external_id":"6afa1f7c6fb055bedcfb4721","started_at":"2026-11-15T21:09:19Z","completed_at":"2026-11-15T21:09:27Z","output":{"title":"No header rules processed","summary":"This deploy did not include any header rules.","text":null}}
{"app":"netlify","name":"Pages changed - zesty-bienenstitch-c3a6cb","status":"completed","conclusion":"neutral","details_url":"https://app.netlify.com/projects/zesty-bienenstitch-c3a6cb/deploys/6afa1f7c6fb055bedcfb4721","html_url":"https://github.com/k1sr/vibeassist-tries/runs/132958901673","external_id":"6afa1f7c6fb055bedcfb4721","started_at":"2026-11-15T21:09:19Z","completed_at":"2026-11-15T21:09:27Z","output":{"title":"1 new file uploaded","summary":"1 generated page and no assets changed.","text":"Changed pages include:\n- index.html - [view](https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app/)\n"}}
```
**Deployments for this commit:** `[]`

**Cloudflare Pages check run, every field** (`jq '.check_runs[] | select(.name=="Cloudflare Pages")'` on `results/batch-2b-raw/step2-check-runs.json`):
```json
{
  "id": 132958969074,
  "name": "Cloudflare Pages",
  "node_id": "CR_kwDOVvDoas8AAAAe9Pjc8g",
  "head_sha": "ca969ad50f1fb3f715ede4f2b713d13af692e5e2",
  "external_id": "9c07c340-a13b-4e1f-98b9-f00d5b178d13",
  "url": "https://api.github.com/repos/k1sr/vibeassist-tries/check-runs/132958969074",
  "html_url": "https://github.com/k1sr/vibeassist-tries/runs/132958969074",
  "details_url": "https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13",
  "status": "completed",
  "conclusion": "success",
  "started_at": "2026-11-15T21:09:34Z",
  "completed_at": "2026-11-15T21:09:34Z",
  "output": {
    "title": "Deployed successfully",
    "summary": "<table><tr><td><strong>Latest commit:</strong> </td><td>\n<code>ca969ad</code>\n</td></tr>\n<tr><td><strong>Status:</strong></td><td>&nbsp;✅&nbsp; Deploy successful!</td></tr>\n<tr><td><strong>Preview URL:</strong></td><td>\n<a href='https://9c07c340.vibeassist-tries.pages.dev'>https://9c07c340.vibeassist-tries.pages.dev</a>\n</td></tr>\n<tr><td><strong>Branch Preview URL:</strong></td><td>\n<a href='https://tries-preview-1.vibeassist-tries.pages.dev'>https://tries-preview-1.vibeassist-tries.pages.dev</a>\n</td></tr>\n</table>\n\n[View logs](https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13)\n",
    "text": null,
    "annotations_count": 0,
    "annotations_url": "https://api.github.com/repos/k1sr/vibeassist-tries/check-runs/132958969074/annotations"
  },
  "check_suite": {
    "id": 118477280293
  },
  "app": {
    "id": 85455,
    "client_id": "Iv1.0879b6ff6c8f7a1a",
    "slug": "cloudflare-workers-and-pages",
    "node_id": "MDM6QXBwODU0NTU=",
    "owner": {
      "login": "cloudflare",
      "id": 314135,
      "node_id": "MDEyOk9yZ2FuaXphdGlvbjMxNDEzNQ==",
      "avatar_url": "https://avatars.githubusercontent.com/u/314135?v=4",
      "gravatar_id": "",
      "url": "https://api.github.com/users/cloudflare",
      "html_url": "https://github.com/cloudflare",
      "followers_url": "https://api.github.com/users/cloudflare/followers",
      "following_url": "https://api.github.com/users/cloudflare/following{/other_user}",
      "gists_url": "https://api.github.com/users/cloudflare/gists{/gist_id}",
      "starred_url": "https://api.github.com/users/cloudflare/starred{/owner}{/repo}",
      "subscriptions_url": "https://api.github.com/users/cloudflare/subscriptions",
      "organizations_url": "https://api.github.com/users/cloudflare/orgs",
      "repos_url": "https://api.github.com/users/cloudflare/repos",
      "events_url": "https://api.github.com/users/cloudflare/events{/privacy}",
      "received_events_url": "https://api.github.com/users/cloudflare/received_events",
      "type": "Organization",
      "user_view_type": "public",
      "site_admin": false
    },
    "name": "Cloudflare Workers and Pages",
    "description": "The **Cloudflare GitHub App** automatically deploys your code to Cloudflare when you merge a pull request to your GitHub repository. It can also create a new repository on your GitHub account when you get started with a Cloudflare template. \r\n\r\n\r\nThis integration also:\r\n- Displays the status of your deployments as check runs\r\n- Posts links to preview URLs as comments on each pull request (Cloudflare Pages only)\r\n\r\n\r\n<img src=\"https://imagedelivery.net/wSMYJvS3Xw-n339CbDyDIA/b3c1fc84-7d73-4fc4-8835-69e85980df00/public\" width=67% >\r\n\r\n\r\n<img src=\"https://imagedelivery.net/wSMYJvS3Xw-n339CbDyDIA/e9a5ffb3-4ec3-4dd6-8bc8-ed0eec5ddb00/public\" width=67% >\r\n\r\n\r\nFor more information, refer to the [Cloudflare Workers docs](https://developers.cloudflare.com/workers/ci-cd/builds/) and [Cloudflare Pages docs](https://developers.cloudflare.com/pages/configuration/git-integration/).",
    "external_url": "https://developers.cloudflare.com/workers/ci-cd/builds/",
    "html_url": "https://github.com/apps/cloudflare-workers-and-pages",
    "created_at": "2020-10-19T20:23:00Z",
    "updated_at": "2026-10-30T00:49:56Z",
    "permissions": {
      "administration": "write",
      "checks": "write",
      "contents": "write",
      "deployments": "write",
      "metadata": "read",
      "pull_requests": "write"
    },
    "events": [
      "pull_request",
      "push"
    ]
  },
  "pull_requests": [
    {
      "url": "https://api.github.com/repos/k1sr/vibeassist-tries/pulls/4",
      "id": 5326934989,
      "number": 4,
      "head": {
        "ref": "tries/preview-1",
        "sha": "ca969ad50f1fb3f715ede4f2b713d13af692e5e2",
        "repo": {
          "id": 1458628714,
          "url": "https://api.github.com/repos/k1sr/vibeassist-tries",
          "name": "vibeassist-tries"
        }
      },
      "base": {
        "ref": "main",
        "sha": "ec9f5d50aedb85fb9159d096480edc7c587d6479",
        "repo": {
          "id": 1458628714,
          "url": "https://api.github.com/repos/k1sr/vibeassist-tries",
          "name": "vibeassist-tries"
        }
      }
    }
  ]
}
```
So **output.summary** is:
```
<table><tr><td><strong>Latest commit:</strong> </td><td>
<code>ca969ad</code>
</td></tr>
<tr><td><strong>Status:</strong></td><td>&nbsp;✅&nbsp; Deploy successful!</td></tr>
<tr><td><strong>Preview URL:</strong></td><td>
<a href='https://9c07c340.vibeassist-tries.pages.dev'>https://9c07c340.vibeassist-tries.pages.dev</a>
</td></tr>
<tr><td><strong>Branch Preview URL:</strong></td><td>
<a href='https://tries-preview-1.vibeassist-tries.pages.dev'>https://tries-preview-1.vibeassist-tries.pages.dev</a>
</td></tr>
</table>

[View logs](https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13)
```
and **output.text** is `null`.

**Comments** (`gh api repos/k1sr/vibeassist-tries/issues/4/comments`, read at about 21:19:45; full bodies in `results/batch-2b-raw/step2-comments.json`):
```
{"id":6900975077,"user":"vercel[bot]","created_at":"2026-11-15T20:36:16Z","updated_at":"2026-11-15T21:09:20Z"}
{"id":6900975686,"user":"netlify[bot]","created_at":"2026-11-15T20:36:18Z","updated_at":"2026-11-15T21:09:30Z"}
{"id":6900990768,"user":"cloudflare-workers-and-pages[bot]","created_at":"2026-11-15T20:37:29Z","updated_at":"2026-11-15T20:43:05Z"}
{"id":6901398325,"user":"cloudflare-workers-and-pages[bot]","created_at":"2026-11-15T21:09:15Z","updated_at":"2026-11-15T21:09:35Z"}
```
- vercel[bot] (6900975077): updated at 21:09:20. Full text in step 4.
- netlify[bot] (6900975686): updated at 21:09:30. "Deploy Preview for *zesty-bienenstitch-c3a6cb* ready!", Latest commit `ca969ad…`, Deploy Preview `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`.
- cloudflare-workers-and-pages[bot] (6900990768, the **Workers Builds** comment): last updated 20:43:05. It now lists e587e1b and 2534760, both "Build: Failed ❌" with empty Deployment URL cells. **Not updated for ca969ad.**
- cloudflare-workers-and-pages[bot] (6901398325, a **new Cloudflare Pages** comment): created 21:09:15, updated 21:09:35. Full text:
```
## Deploying vibeassist-tries with &nbsp;<a href="https://pages.dev"><img alt="Cloudflare Pages" src="https://user-images.githubusercontent.com/23264/106598434-9e719e00-654f-11eb-9e59-6167043cfa01.png" width="16"></a> &nbsp;Cloudflare Pages

<table><tr><td><strong>Latest commit:</strong> </td><td>
<code>ca969ad</code>
</td></tr>
<tr><td><strong>Status:</strong></td><td>&nbsp;✅&nbsp; Deploy successful!</td></tr>
<tr><td><strong>Preview URL:</strong></td><td>
<a href='https://9c07c340.vibeassist-tries.pages.dev'>https://9c07c340.vibeassist-tries.pages.dev</a>
</td></tr>
<tr><td><strong>Branch Preview URL:</strong></td><td>
<a href='https://tries-preview-1.vibeassist-tries.pages.dev'>https://tries-preview-1.vibeassist-tries.pages.dev</a>
</td></tr>
</table>

[View logs](https://dash.cloudflare.com/?to=/5ab782b7007da791a8e2bd06113aeb72/pages/view/vibeassist-tries/9c07c340-a13b-4e1f-98b9-f00d5b178d13)
```

**Preview link per host, and the exact field:**
- **Netlify:** `https://deploy-preview-4--zesty-bienenstitch-c3a6cb.netlify.app`, from the `target_url` of the **status** `netlify/zesty-bienenstitch-c3a6cb/deploy-preview` (state `success`). The same link with a trailing `/` is in the "Pages changed" **check run**'s `output.text`, and it is also in the comment.
- **Vercel:** `https://vibeassist-tries-git-tries-preview-1-xig2.vercel.app`, **only in the comment** (see step 4). The status target_url is the dashboard page `https://vercel.com/xig2/vibeassist-tries/4ugpEZjiQfcESw3SxxpXxAtZZb8S`. The check run's details_url is `https://vercel.com/github`, and its `output.summary` has the hostname only inside the vercel.live feedback link.
- **Cloudflare Pages:** per-deploy `https://9c07c340.vibeassist-tries.pages.dev` and branch `https://tries-preview-1.vibeassist-tries.pages.dev`. Both come from the "Cloudflare Pages" **check run**'s `output.summary` (an HTML table: rows "Preview URL:" and "Branch Preview URL:", in `<a href='…'>`). They also appear in comment 6901398325. Its `details_url` is the dashboard, and there is no status.
- **Cloudflare Workers Builds:** **none.** It did not run for this commit, and its earlier runs all failed with empty Deployment URL cells.

