# Try 11: a pull request open for 24 hours while main changes twice

## Set up

- **23:00:09Z:** on main, added try11/a.txt (lines "1. line 1" to "10. line 10") and results/try-11-log.md (one line: "# Try 11 log"). This is commit 0e2a889, pushed straight to main.
- **Branch try11/long:** made from main. Line 2 of try11/a.txt became "2. line 2 (changed on try11/long)" in commit d8ccfb4, which was pushed.
- **Pull request #25:** https://github.com/k1sr/vibeassist-tries/pull/25, "TRY 11 long pull request (do not merge)", opened 23:00:12Z.
- **The two tasks:** both made with `create_trigger` at 23:00:22Z, with `create_new_session_on_fire: true` and the prompts from the brief, word for word. `create_trigger` has no model field, so both were set to `claude-sonnet-5-5` with `update_trigger` at 23:00:27Z.
  - "TRY 11 bring up to date" (trig_019yTs2kSBTpZvuwWqMnz1t1) has no schedule.
  - "TRY 11 hourly check" (trig_013g6bhmG2c3rKX6Z8tVjogF) runs on `0 * * * *`. The server tied the hour to the minute it was made, so the first run is due at 2026-11-16T00:06:04Z.
- **Warning on both create calls:** "this trigger stores no MCP connectors, so the sessions it fires will run without connector (mcp__<server>__*) tools."
- **Repository on the tasks, as the tool sees it:** the create and update replies include `session_request.config`, and for both tasks it holds `"sources":[], "outcomes":[]`. So no repository was set when the tool made them. The read-back tools (`get_trigger`, `list_triggers`) don't show `session_request` at all.
- **For comparison:** when TRY 12 was switched off, the `update_trigger` reply showed `"sources":[{"git_repository":{"url":"https://github.com/k1sr/vibeassist-tries", ...}}]` and `"outcomes":[{"git_repository":{"git_info":{"repo":"k1sr/vibeassist-tries","branches":["claude/brave-euler"]}}}]`. TRY 12 was also made with empty sources, so that repository came from the hand edit on the web page. The update reply shows it; the read-back does not.

## The tasks as read back

`get_trigger` for "TRY 11 bring up to date", word for word:

```json
{"trigger":{"id":"trig_019yTs2kSBTpZvuwWqMnz1t1", "name":"TRY 11 bring up to date", "cron_expression":"", "enabled":true, "created_at":"2026-11-15T23:00:22.572234Z", "updated_at":"2026-11-15T23:00:27.709473Z", "persist_session":false, "mcp_connections":[], "creator":{"account_uuid":"fe780672-62e7-4163-b531-308ddf470285", "display_name":"Simon"}, "api_token_hint":"", "enabled_plugins":[], "extra_marketplaces":[], "ended_reason":"", "created_via":"meta_mcp", "suspension_reason":"", "derived_state":{"folders_state":"FOLDERS_STATE_NONE", "folders":[], "prompt":"Try 11. Do only this, then end. Use no VibeAssist tool or memory. In k1sr/vibeassist-tries: fetch everything, check out try11/long, merge origin/main into it, push it. Then on main append one line to results/try-11-log.md: \"UPDATE | <UTC time now> | <what your first message says about how you were started> | merged main into try11/long: <ok or the error>\". Push to main; if refused because main moved, pull --rebase and push again, up to 5 times.", "model":"claude-sonnet-5-5", "files":[]}, "created_surface":"ROUTINE_CREATED_SURFACE_UNSPECIFIED", "created_kind":"ROUTINE_CREATED_KIND_ROUTINE"}}
```

`get_trigger` for "TRY 11 hourly check", word for word:

```json
{"trigger":{"id":"trig_013g6bhmG2c3rKX6Z8tVjogF", "name":"TRY 11 hourly check", "cron_expression":"0 * * * *", "enabled":true, "next_run_at":"2026-11-16T00:06:04.625203Z", "created_at":"2026-11-15T23:00:22.835722Z", "updated_at":"2026-11-15T23:00:27.995873Z", "persist_session":false, "mcp_connections":[], "creator":{"account_uuid":"fe780672-62e7-4163-b531-308ddf470285", "display_name":"Simon"}, "api_token_hint":"", "enabled_plugins":[], "extra_marketplaces":[], "ended_reason":"", "created_via":"meta_mcp", "suspension_reason":"", "derived_state":{"folders_state":"FOLDERS_STATE_NONE", "folders":[], "prompt":"Try 11 hourly check. Do only this, then end. Use no VibeAssist tool or memory. In k1sr/vibeassist-tries:\n1. Note the UTC time now, and copy word for word what your first message says about when you fired.\n2. Fetch everything. Your run number N is 1 plus the number of lines starting \"HOURLY\" in results/try-11-log.md on origin/main. If N is more than 25, append \"HOURLY | N | <time> | past the end, did nothing\", push, and end.\n3. Let B be `git merge-base origin/main origin/try11/long`. List the files changed from B to origin/main, and from B to origin/try11/long. OVERLAP is yes if any file is in both lists.\n4. If OVERLAP is yes: list every tool you have that could start another scheduled task now, by name. If you have one, use it to start the task \"TRY 11 bring up to date\" and record what it returned. If you have none, or it fails, do that task's merge yourself (merge origin/main into try11/long and push) and record that.\n5. If N is 3: on main, add a new file try11/other.txt. If N is 8: on main, change line 9 of try11/a.txt. Make this change after step 4, in its own commit.\n6. Append one line to results/try-11-log.md on main: \"HOURLY | N | <UTC time now> | fired: <step 1 text> | overlap: yes/no | action: none / started the task (<result>) / merged it myself (<why>) | main change made: none/other.txt/a.txt line 9\". Push; if refused because main moved, pull --rebase and push again, up to 5 times.", "model":"claude-sonnet-5-5", "files":[]}, "created_surface":"ROUTINE_CREATED_SURFACE_UNSPECIFIED", "created_kind":"ROUTINE_CREATED_KIND_ROUTINE"}}
```

`list_triggers` (limit 100) gave the same fields for both tasks. It also listed TRY 12 with `"enabled":false` and `"last_run":{"status":"ROUTINE_RUN_STATUS_SUCCEEDED","fired_at":"2026-11-15T22:38:39.635388Z","finished_at":"2026-11-15T22:39:12.538267Z","session_id":"cse_01YRXdXPnBitEaDBCyfEkrRj"}`.

## Repository already on the tasks? (the person's answer)

The person's answer: "Neither task had the repo; I've added it to both. However, the bring up to date had saved greyed out, so I added the github trigger, which is now on there, but might need to remove it".

- **TRY 11 bring up to date:** the repository was NOT already shown. The person added it. The Save button stayed greyed out after adding just the repository, so they also added a GitHub trigger, which made Save work. That GitHub trigger is now on the task. Which event it is set to hasn't been recorded yet.
- **TRY 11 hourly check:** the repository was NOT already shown. The person added it.

I checked again at 23:04:41Z with a harmless `update_trigger` (`enabled: true`) on each task, because its reply shows the hidden `session_request`:
- **Bring up to date:** `sources` = https://github.com/k1sr/vibeassist-tries, `outcomes` branch "claude/kind-cori". The reply has no field for the GitHub trigger, so the tool can't see it.
- **Hourly check:** `sources` = https://github.com/k1sr/vibeassist-tries, `outcomes` branch "claude/vigilant-gauss".
- **The hourly schedule changed:** the page's save rewrote it from `0 * * * *` to `2 * * * *`, so the first run moved from 00:06:04Z to **2026-11-16T00:02:00Z**.
- **Both tasks** now have `allowed_tools` "preset:default", Task, Bash, Glob, Grep, Read, Edit, MultiEdit, Write, NotebookEdit, WebFetch, TodoWrite, WebSearch, BashOutput, KillBash, Skill, Tmux, Monitor, SendUserFile and REPL. When the tool made them, this list was empty.

### The GitHub trigger was removed from "TRY 11 bring up to date"

The person: "Removed it, repo still there, please check again. Added API with no details and it went back to manual trigger".

- **What they did:** removed the GitHub trigger, added an "API" trigger with no details, and the page then showed the task as a manual trigger again. The GitHub trigger's event was never recorded.
- **Checked at 23:06:12Z** with a harmless `update_trigger` (`enabled: true`): `cron_expression` is "" and `sources` = https://github.com/k1sr/vibeassist-tries. The repository is still there. `outcomes` branch is now "claude/exciting-noether", and it changes with each save.
- **What the tool can't show:** any field for GitHub, API or manual triggers. So the tool can't confirm the GitHub trigger is gone; that rests on what the person saw.
- **Ready to start:** "TRY 11 bring up to date" can now be started only by the hourly check, by hand, or through the API.
