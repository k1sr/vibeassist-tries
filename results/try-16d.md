# TRY 16d results: what builds on merge into try16/jobs

Run date: 2026-11-18 (UTC). Raw API responses are in `results/try-16d-raw/`.

Step 0: `origin/main` (2a5c6f7) merged into try16/jobs with a normal merge commit `fe05ecc` ("try 16d: bring try16/jobs up to main [skip ci]"), pushed directly. `.github/workflows/try-16c-ci.yml` is on try16/jobs afterwards.

## Requests

| Req | Branch | PR | Title | Head commit | Merge commit | Merged (UTC) | Evidence read (UTC) |
|---|---|---|---|---|---|---|---|
| A | try16/d-a-074316 | #37 | TRY 16d request A | 3e16d51 | df84af1 | 07:43:28 | ~07:48:40 (5 min after merge) |
| B | try16/d-b-074850 | #38 | [skip ci] TRY 16d request B | b841ea4 | 3ec053e | 07:48:52 | 07:53:56 (5 min after merge) |

Each squash merge was a `PUT pulls/<n>/merge` with `merge_method=squash`, and the reply is in `results/try-16d-raw/{A,B}-merge-reply.json`. Commit statuses, check runs, deployments, deployment statuses, Actions runs and PR comments were read for both the head and merge commits.

## Result table

Cell values are "built", "skipped", "ignored" or "nothing seen". Times are UTC.

| Host | A | B |
|---|---|---|
| Cloudflare Pages | **built**. Check run "Cloudflare Pages" = success on head (07:43:33) and merge (07:43:46). | **nothing seen**. No Pages check run or status on head or merge. |
| Cloudflare Workers | **built**. Check run "Workers Builds: vibeassist-tries" = success on head (07:43:55) and merge (07:44:32). | **skipped**. Same check run = skipped on head (07:48:56) and merge (07:48:59). |
| Vercel | **ignored**. Status "Vercel" = success, description "Canceled by Ignored Build Step", on head (07:43:23) and merge (07:43:32). | **ignored**. Status "Vercel" = success, description "Canceled by Ignored Build Step", on head (07:48:56) and merge (07:49:00). |
| Netlify | **nothing seen**. No Netlify status or check run on head or merge. | **nothing seen**. Same. |
| Railway | **nothing seen**. No Railway status or check run on head or merge. | **nothing seen**. Same. |
| Render | **nothing seen**. No Render status or check run on head or merge. | **nothing seen**. Same. |
| GitHub Actions (try-16c-ci) | **built**. `actions/runs?head_sha=3e16d51` returned run 44261085946 (push, 07:43:20) and run 44261103466 (pull_request, 07:43:27), both success. `head_sha=df84af1` returned run 44261107329 (push, 07:43:29), success. Check runs "try-16c-ci" = success on head and merge. | **nothing seen**. 0 runs for head or merge. |
| TRY 16 target started | **yes, 07:43:49**. "TARGET RAN 2026-11-18T07:43:49Z" line on main, from commit 9273272. | **yes, 07:49:13**. "TARGET RAN 2026-11-18T07:49:13Z" line on main, from commit 8698441. |

Deployments API (`deployments?sha=`) returned an empty list for every head and merge commit in both runs. The Vercel and Cloudflare results above come from commit statuses and check runs, not from deployments.

## Branches pushed after request A's merge (07:43:28)

- `origin/try16/d-b-074850` at b841ea4 (07:48:50)
- `origin/try16/jobs` at 3ec053e (07:48:53, the B squash merge)
- `origin/main` at 8698441 (07:49:13). Its commits 9273272 (07:43:49) and 8698441 (07:49:13) each add one "TARGET RAN" line to `results/try-16-log.md`.

## Notes

- Vercel reports "success" with the description "Canceled by Ignored Build Step" on every head and merge commit. The table calls this "ignored". Vercel still sets a "pending / Vercel is deploying your app" status first, then the cancel status replaces it on both commits. Vercel on B was not built.
- Vercel's "Vercel Preview Comments" check ran on both A and B, and both PRs have one issue comment each (see raw files). It is not a deploy.
- B produced no Actions runs on head or merge, and Workers was skipped. The `[skip ci]` title is the only difference from A that I can point to; I did not test whether it caused this.
- Both "TARGET RAN" lines are on main in commits authored "Claude" (9273272 and 8698441). I did not write them and cannot confirm from here which session did. They are recorded as found in the log.
- The step 0 merge commit uses the exact message given, so it has no Co-Authored-By or Claude-Session trailer. The two request commits do have the trailers.
- Compared with 16c, Vercel on this run did not build on either commit: the Ignored Build Step ran and cancelled on both A and B. In 16c, Vercel built on all three requests.
