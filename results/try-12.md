# Try 12: GitHub merge trigger, 20 merges in 20 minutes

The question: when a scheduled task has a GitHub trigger for merged pull requests, how many of 20 merges in 20 minutes start a session?

## How the task was made

- `create_trigger` with name "TRY 12 event counter", no cron and no run-once time (so it has no schedule), `create_new_session_on_fire: true`, and the prompt from the brief, word for word. Made at 2026-11-15T21:56:00Z. ID: trig_01PL4wtxtDBsCUucEodvyT4t.
- `create_trigger` has no model field, so the model was set right after with `update_trigger` (model `claude-sonnet-5-5`) at 2026-11-15T21:56:03Z.
- The create call warned: "this trigger stores no MCP connectors, so the sessions it fires will run without connector (mcp__<server>__*) tools."
- The task has no repository on it (`sources: []`). The person adds the repository and the GitHub trigger by hand on the web page.

## PART 1: the task as made

Everything `list_triggers` (limit 100) returned, word for word:

```json
{"data":[{"id":"trig_01PL4wtxtDBsCUucEodvyT4t","name":"TRY 12 event counter","cron_expression":"","enabled":true,"created_at":"2026-11-15T21:56:00.376787Z","updated_at":"2026-11-15T21:56:03.397951Z","persist_session":false,"mcp_connections":[],"creator":{"account_uuid":"fe780672-62e7-4163-b531-308ddf470285"},"api_token_hint":"","enabled_plugins":[],"extra_marketplaces":[],"ended_reason":"","created_via":"meta_mcp","suspension_reason":"","derived_state":{"folders_state":"FOLDERS_STATE_NONE","folders":[],"prompt":"You were started by a GitHub event for try 12. Do only this, then end. Do not use any VibeAssist tool or memory.\n1. Note the time now in UTC, to the second.\n2. Write down, word for word, everything your first message and your starting context say about the event that started you: the pull request number, the event name, the branch and commit you started on.\n3. In k1sr/vibeassist-tries, make one new file results/try-12-runs/RUN-<UTC time as YYYYMMDD-HHMMSS>-<6 random letters>.md holding the lines from steps 1 and 2 and the output of `git rev-parse HEAD` and `git branch --show-current`.\n4. Commit it and push it to main. If the push is refused because main moved, run `git pull --rebase origin main` and push again, up to 5 times. If pushing to main is refused for any other reason, push the commit to a new branch named try12-run-<the same 6 letters> instead, and say why in the file.","model":"claude-sonnet-5-5","files":[]},"created_surface":"ROUTINE_CREATED_SURFACE_UNSPECIFIED","created_kind":"ROUTINE_CREATED_KIND_ROUTINE"}],"has_more":false}
```

Notes on what the list shows: `cron_expression` is empty (no schedule), the model is `claude-sonnet-5-5`, there is no field for repositories or GitHub triggers in this listing, and `mcp_connections` is empty.

## PART 2: the task after the hand edit

Read back with `get_trigger` at about 22:12 UTC (the list tool returns the same fields). Everything it returned, word for word:

```json
{"trigger":{"id":"trig_01PL4wtxtDBsCUucEodvyT4t","name":"TRY 12 event counter","cron_expression":"","enabled":true,"created_at":"2026-11-15T21:56:00.376787Z","updated_at":"2026-11-15T22:06:57.941516Z","persist_session":false,"mcp_connections":[],"creator":{"account_uuid":"fe780672-62e7-4163-b531-308ddf470285","display_name":"Simon"},"api_token_hint":"","enabled_plugins":[],"extra_marketplaces":[],"ended_reason":"","created_via":"meta_mcp","suspension_reason":"","derived_state":{"folders_state":"FOLDERS_STATE_NONE","folders":[],"prompt":"You were started by a GitHub event for try 12. Do only this, then end. Do not use any VibeAssist tool or memory.\n1. Note the time now in UTC, to the second.\n2. Write down, word for word, everything your first message and your starting context say about the event that started you: the pull request number, the event name, the branch and commit you started on.\n3. In k1sr/vibeassist-tries, make one new file results/try-12-runs/RUN-<UTC time as YYYYMMDD-HHMMSS>-<6 random letters>.md holding the lines from steps 1 and 2 and the output of `git rev-parse HEAD` and `git branch --show-current`.\n4. Commit it and push it to main. If the push is refused because main moved, run `git pull --rebase origin main` and push again, up to 5 times. If pushing to main is refused for any other reason, push the commit to a new branch named try12-run-<the same 6 letters> instead, and say why in the file.","model":"claude-sonnet-5-5","files":[]},"created_surface":"ROUTINE_CREATED_SURFACE_UNSPECIFIED","created_kind":"ROUTINE_CREATED_KIND_ROUTINE"}}
```

What changed compared with Part 1: only `updated_at` (21:56:03 to 22:06:57), and `display_name` is now shown in `creator`. The read-back shows no repository and no GitHub trigger at all, so the tool can't see the hand edit. Only the timestamp shows that something was saved.

What the person said and showed:
- "The repo was already attached, and there are countless github events to add." The page already had k1sr/vibeassist-tries, even though the task was made with an empty repository list. It probably came from the environment or a default.
- A screenshot of the "Edit scheduled task" dialog showed:
  - Name "TRY 12 event counter", the prompt, "Permissions set by environment", model "Sonnet 5.5".
  - A notice: "Claude created this scheduled task, so it runs in Auto mode — connector calls are checked by a classifier".
  - Trigger: "GitHub event", repository "k1sr/vibeassist-tries".
  - Event: "Pull request: Merged". The page has a merged action of its own, so "closed" wasn't needed.
  - Filter: "Is merged" "equals" "true". There's a + to add more filters and a bin icon to remove one.
  - Advanced settings tabs: General, Connectors, Behavior, Notifications. On General, Repository is set to k1sr/vibeassist-tries.
  - "Save changes" was greyed out, so nothing was left unsaved.
- The person did not list the full set of events or filter fields. They said only that there are "countless" events.

## Pull requests opened

20 branches try12/pr-01 to try12/pr-20 were made from main at e97bf40. Each adds one file, try12/NN.txt, and nothing else. A pull request into main was opened for each with `gh api`, titled "TRY 12 merge NN". All 20 were opened between 22:13:54 and 22:14:20 UTC.

| PR | branch | opened (UTC) |
|---|---|---|
| #5 | try12/pr-01 | 2026-11-15T22:13:54Z |
| #6 | try12/pr-02 | 2026-11-15T22:13:55Z |
| #7 | try12/pr-03 | 2026-11-15T22:13:57Z |
| #8 | try12/pr-04 | 2026-11-15T22:13:58Z |
| #9 | try12/pr-05 | 2026-11-15T22:14:00Z |
| #10 | try12/pr-06 | 2026-11-15T22:14:01Z |
| #11 | try12/pr-07 | 2026-11-15T22:14:02Z |
| #12 | try12/pr-08 | 2026-11-15T22:14:03Z |
| #13 | try12/pr-09 | 2026-11-15T22:14:05Z |
| #14 | try12/pr-10 | 2026-11-15T22:14:07Z |
| #15 | try12/pr-11 | 2026-11-15T22:14:08Z |
| #16 | try12/pr-12 | 2026-11-15T22:14:09Z |
| #17 | try12/pr-13 | 2026-11-15T22:14:11Z |
| #18 | try12/pr-14 | 2026-11-15T22:14:12Z |
| #19 | try12/pr-15 | 2026-11-15T22:14:14Z |
| #20 | try12/pr-16 | 2026-11-15T22:14:15Z |
| #21 | try12/pr-17 | 2026-11-15T22:14:16Z |
| #22 | try12/pr-18 | 2026-11-15T22:14:17Z |
| #23 | try12/pr-19 | 2026-11-15T22:14:19Z |
| #24 | try12/pr-20 | 2026-11-15T22:14:20Z |

### Sessions started by opening

The check ran at 2026-11-15T22:19:34Z, 5 min 14 s after the last pull request was opened and before the first merge. It found none:
- `results/try-12-runs/` did not exist on main (GitHub returned 404 Not Found).
- No branch named try12-run-* existed.
- None of the 20 RUN files found later says it came from an opening.

So opening the pull requests started no sessions.

## Merges

Each pull request was merged with `gh api -X PUT repos/k1sr/vibeassist-tries/pulls/N/merge -f merge_method=merge`, one every 60 seconds, in order. Every call returned `merged: true`. The time below is when the merge call returned.

| PR | merge returned (UTC) | merge commit |
|---|---|---|
| #5 | 2026-11-15T22:19:37Z | 9b0f85a491505a7904e6478225e423c351713ca4 |
| #6 | 2026-11-15T22:20:38Z | da3bebf46eff2526032af7676f8a4c7febdeb050 |
| #7 | 2026-11-15T22:21:37Z | 82c993d93a47e3b38b3e0476ff95b6babfc3ee6e |
| #8 | 2026-11-15T22:22:38Z | 9e0ae92d118f1e9d9e77689cd0dbd694c5492873 |
| #9 | 2026-11-15T22:23:38Z | 9434e1d404dece756a4a6c2cfe14fb15d0c6b01e |
| #10 | 2026-11-15T22:24:38Z | 160a0ea0fc00d71b6f770c7083761dad00bc77b4 |
| #11 | 2026-11-15T22:25:37Z | a9639db68f19debf1a1b2b8c41cf1ac465595715 |
| #12 | 2026-11-15T22:26:38Z | ee4a4a63477e6b6a6a8da9fc86bc2a0d475f172f |
| #13 | 2026-11-15T22:27:38Z | cb92f7b158429a132755710ec3a3cba304bc9166 |
| #14 | 2026-11-15T22:28:38Z | 69dd9b610d9a40516f321a9da87963a654869178 |
| #15 | 2026-11-15T22:29:37Z | b6a0b7fad35d0244ef28d1192be611f39dfd3805 |
| #16 | 2026-11-15T22:30:38Z | bc224f01cd2cf5e8a2adadf257c0350bcf499158 |
| #17 | 2026-11-15T22:31:38Z | d426add4d1736bc81964f8470b361fecfc86e29e |
| #18 | 2026-11-15T22:32:37Z | fadcd2df71117d6d1e32fa890c573eab3493a901 |
| #19 | 2026-11-15T22:33:38Z | 4d261a220b268f9de5bb5f712e09db80c0f81e79 |
| #20 | 2026-11-15T22:34:37Z | 8bd9a8d05c94cc58ff5833d2b36078129368a5f6 |
| #21 | 2026-11-15T22:35:38Z | 07a1cd8eff47a8c9f570ee20238154b693d148e7 |
| #22 | 2026-11-15T22:36:38Z | 7e2a85ad30b59d0ab0e1854ef1ef8705532c5253 |
| #23 | 2026-11-15T22:37:38Z | 2389117a8f91c6dfe0873370b1be0c9b6ec0dfc6 |
| #24 | 2026-11-15T22:38:38Z | 1c6cb0b10be6b5cdb8f7134770d8578416f2fdb1 |

## After the last merge

The last merge returned at 22:38:38Z. After 15 minutes (22:53:38Z), a check with `gh api` and then `git fetch --all` at 22:53:43Z found:

- **results/try-12-runs/ on main: 20 files**
  - results/try-12-runs/RUN-20261115-221945-wyrtbj.md
  - results/try-12-runs/RUN-20261115-222045-qzvkmt.md
  - results/try-12-runs/RUN-20261115-222144-yklmcx.md
  - results/try-12-runs/RUN-20261115-222245-qkzvwm.md
  - results/try-12-runs/RUN-20261115-222345-qlmyky.md
  - results/try-12-runs/RUN-20261115-222445-qzmtvk.md
  - results/try-12-runs/RUN-20261115-222545-pqovts.md
  - results/try-12-runs/RUN-20261115-222645-vjxmcx.md
  - results/try-12-runs/RUN-20261115-222745-yjljya.md
  - results/try-12-runs/RUN-20261115-222845-qwkzpm.md
  - results/try-12-runs/RUN-20261115-222944-qzkvmw.md
  - results/try-12-runs/RUN-20261115-223046-qkzvhm.md
  - results/try-12-runs/RUN-20261115-223145-eypzqx.md
  - results/try-12-runs/RUN-20261115-223244-slahcl.md
  - results/try-12-runs/RUN-20261115-223345-zpqbtq.md
  - results/try-12-runs/RUN-20261115-223445-qkvzmt.md
  - results/try-12-runs/RUN-20261115-223546-gsmwqc.md
  - results/try-12-runs/RUN-20261115-223645-fkkfap.md
  - results/try-12-runs/RUN-20261115-223745-pshjxj.md
  - results/try-12-runs/RUN-20261115-223845-xflcrn.md
- **Branches named try12-run-*: none.** Every run pushed to main without trouble.
- **Other branches the runs pushed:** 17 runs also pushed their own session branch, `claude/brave-euler-<6 chars>`. The other three runs (wo0iw6, uvbyvl, ohxppe) left none on the remote:
  - claude/brave-euler-4odxr9
  - claude/brave-euler-5157su
  - claude/brave-euler-5pm8ff
  - claude/brave-euler-5sfqoi
  - claude/brave-euler-73dg87
  - claude/brave-euler-azcx22
  - claude/brave-euler-dcshiy
  - claude/brave-euler-dzovlf
  - claude/brave-euler-g3d3t3
  - claude/brave-euler-mc9ogg
  - claude/brave-euler-qdyju2
  - claude/brave-euler-qioox2
  - claude/brave-euler-spbmko
  - claude/brave-euler-w9a3ds
  - claude/brave-euler-whb6bh
  - claude/brave-euler-ws55ux
  - claude/brave-euler-yc4d7l

## RESULT

Each RUN file names the pull request that started it, its head branch and its head SHA, so every run matches exactly one merge. "Session start" below is the time the run wrote down in its step 1, which is also in its file name. The real start of the session was a little earlier. "Pushed" is the commit time of the RUN file on main.

| PR | merge returned (UTC) | session started? | session start (UTC, from the run) | delay (s) | RUN file pushed (UTC) | merge to push (s) | RUN file |
|---|---|---|---|---|---|---|---|
| #5 | 2026-11-15T22:19:37Z | yes | 2026-11-15T22:19:45Z | 8 | 2026-11-15T22:19:52Z | 15 | RUN-20261115-221945-wyrtbj.md |
| #6 | 2026-11-15T22:20:38Z | yes | 2026-11-15T22:20:45Z | 7 | 2026-11-15T22:20:56Z | 18 | RUN-20261115-222045-qzvkmt.md |
| #7 | 2026-11-15T22:21:37Z | yes | 2026-11-15T22:21:44Z | 7 | 2026-11-15T22:21:55Z | 18 | RUN-20261115-222144-yklmcx.md |
| #8 | 2026-11-15T22:22:38Z | yes | 2026-11-15T22:22:45Z | 7 | 2026-11-15T22:22:56Z | 18 | RUN-20261115-222245-qkzvwm.md |
| #9 | 2026-11-15T22:23:38Z | yes | 2026-11-15T22:23:45Z | 7 | 2026-11-15T22:23:51Z | 13 | RUN-20261115-222345-qlmyky.md |
| #10 | 2026-11-15T22:24:38Z | yes | 2026-11-15T22:24:45Z | 7 | 2026-11-15T22:24:52Z | 14 | RUN-20261115-222445-qzmtvk.md |
| #11 | 2026-11-15T22:25:37Z | yes | 2026-11-15T22:25:45Z | 8 | 2026-11-15T22:25:52Z | 15 | RUN-20261115-222545-pqovts.md |
| #12 | 2026-11-15T22:26:38Z | yes | 2026-11-15T22:26:45Z | 7 | 2026-11-15T22:26:51Z | 13 | RUN-20261115-222645-vjxmcx.md |
| #13 | 2026-11-15T22:27:38Z | yes | 2026-11-15T22:27:45Z | 7 | 2026-11-15T22:27:51Z | 13 | RUN-20261115-222745-yjljya.md |
| #14 | 2026-11-15T22:28:38Z | yes | 2026-11-15T22:28:45Z | 7 | 2026-11-15T22:28:50Z | 12 | RUN-20261115-222845-qwkzpm.md |
| #15 | 2026-11-15T22:29:37Z | yes | 2026-11-15T22:29:44Z | 7 | 2026-11-15T22:29:51Z | 14 | RUN-20261115-222944-qzkvmw.md |
| #16 | 2026-11-15T22:30:38Z | yes | 2026-11-15T22:30:46Z | 8 | 2026-11-15T22:30:55Z | 17 | RUN-20261115-223046-qkzvhm.md |
| #17 | 2026-11-15T22:31:38Z | yes | 2026-11-15T22:31:45Z | 7 | 2026-11-15T22:31:54Z | 16 | RUN-20261115-223145-eypzqx.md |
| #18 | 2026-11-15T22:32:37Z | yes | 2026-11-15T22:32:44Z | 7 | 2026-11-15T22:33:00Z | 23 | RUN-20261115-223244-slahcl.md |
| #19 | 2026-11-15T22:33:38Z | yes | 2026-11-15T22:33:45Z | 7 | 2026-11-15T22:33:51Z | 13 | RUN-20261115-223345-zpqbtq.md |
| #20 | 2026-11-15T22:34:37Z | yes | 2026-11-15T22:34:45Z | 8 | 2026-11-15T22:34:53Z | 16 | RUN-20261115-223445-qkvzmt.md |
| #21 | 2026-11-15T22:35:38Z | yes | 2026-11-15T22:35:46Z | 8 | 2026-11-15T22:35:52Z | 14 | RUN-20261115-223546-gsmwqc.md |
| #22 | 2026-11-15T22:36:38Z | yes | 2026-11-15T22:36:45Z | 7 | 2026-11-15T22:36:56Z | 18 | RUN-20261115-223645-fkkfap.md |
| #23 | 2026-11-15T22:37:38Z | yes | 2026-11-15T22:37:45Z | 7 | 2026-11-15T22:37:52Z | 14 | RUN-20261115-223745-pshjxj.md |
| #24 | 2026-11-15T22:38:38Z | yes | 2026-11-15T22:38:45Z | 7 | 2026-11-15T22:38:56Z | 18 | RUN-20261115-223845-xflcrn.md |

- **How many started a session:** 20 of 20. Every merge started exactly one session. There were no duplicates and no runs left over.
- **Merges with no session:** none.
- **Cap:** none seen. The 20th merge (#24) started a session just like the 1st did. Through 20 merges in 19 minutes (22:19:37 to 22:38:38), there was no throttling, no dropped events and no growing delay.
- **Delay from merge to the time the run noted:** shortest 7 s and longest 8 s (14 runs took 7 s, 6 took 8 s). From merge to the RUN file landing on main took 12 s to 23 s.
- **What the runs said about how they were started.** These quotes come from the RUN files, which repeat what each session was given at the start.
  - The first message was headed "[SCHEDULED TASK - AUTOMATED FIRING OF A CONFIGURED PROMPT]" and then the prompt.
  - It came with a `<github-trigger-context>` block: "This routine was triggered by a GitHub webhook. Event: pull_request.closed / Repository: k1sr/vibeassist-tries / PR: #N — https://github.com/k1sr/vibeassist-tries/pull/N / Branch: try12/pr-NN → main / Head SHA: <sha>", followed by "If your workspace was freshly provisioned by this event and its source is this repository, the PR branch is already checked out. Otherwise fetch it ... `git fetch https://github.com/k1sr/vibeassist-tries refs/pull/N/head`."
  - **The event name the session saw is `pull_request.closed`, not "merged".** The page's "Pull request: Merged" choice is shown to the session as the closed action. The merged part comes from the "Is merged equals true" filter.
  - **What each session started on:** `git rev-parse HEAD` was the head commit of the pull request branch (for example #5 → 7577581e…, which is try12/pr-01's commit), not the merge commit on main. `git branch --show-current` was a fresh session branch named `claude/brave-euler-<6 chars>`, which the system prompt told the session to develop on.
- **Model:** the RUN files don't say which model ran. The task was set to `claude-sonnet-5-5`.

TRY 12: 20 of 20 merges started a session.
Cap seen: no cap seen up to 20 in 20 minutes.
Delay: shortest 7 s, longest 8 s.
Left in place:
- the task "TRY 12 event counter" (trig_01PL4wtxtDBsCUucEodvyT4t), still on, with its GitHub trigger
- the branches try12/pr-01 to try12/pr-20, and the 17 claude/brave-euler-* branches the runs pushed
- the files try12/01.txt to try12/20.txt and the 20 RUN files in results/try-12-runs/, all on main
- the 20 merged pull requests #5 to #24
