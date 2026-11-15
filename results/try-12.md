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
