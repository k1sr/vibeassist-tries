# VibeAssist generic tasks: handover

Compiled 18 Nov 2026 from the **VA Rebuild** Claude Code session (31 Oct – 2 Nov 2026), on the account with the test boards. This is the process we worked out and built for generic scheduled tasks. It comes from that session's transcript, not from reading the merged code. Check the code in `XIG-Australia/vibeassist-rebuild` against it before relying on details.

## What was built

| PR | What it does |
|---|---|
| #414 | Generic tasks 1: VibeAssist's instructions (generic prompts, all-boards hourly check, work in your own repo's folder) |
| #415 | Generic tasks 2: board page and setup wizard (finds generic tasks, "add this repository" path) |
| #416 | Generic tasks 3: setup in Claude Code, Claude makes the tasks |
| #418 | Re-lands #415 and #416. They were merged into the previous PR's branch instead of master by mistake, so #418 brings both into master unchanged. |
| #417 | Recommended models per task (Opus for build and hand-off, Haiku for the hourly check) |
| #419 | "Bring in the outside-board pull requests" (option A) |
| #420 | First install reloads onto the board when done |
| #421 | Project start: "Bring in my code" or "Start from an idea" (uses Develop with Claude) |
| #422 | New first-time setup and new-project wizard (stepper, phone layout). Not tried on a live board. |

Merge order used: #418, then #417, then #419.

## The process

### First-time setup (once per account)

1. Run setup in **Claude Code**, not a chat. You can make a Claude Code project for the board, but it must be in Code. The board is named after the Code project, or after the repository if there's no project. Updating a board still works from anywhere, including a chat.
2. Claude creates the three generic tasks (build, hand-off, hourly check), and only those that don't already exist.
   - Build and hand-off get **no schedule**. Only the hourly check runs on one.
   - Claude tries to attach the VibeAssist connector. On this account that was refused ("the connectors parameter is not available for this organization"), so it's added by hand.
   - Claude can't attach a repository when it makes a task.
   - Claude must make the tasks. Tasks made by hand on the routines page can't be started by Claude or VibeAssist ("Agents can only fire routines they created").
3. On the routines page (claude.ai/code/routines), in a **web browser, not the phone app**, open each task and add the **VibeAssist connector** and the **project's repository**.
   - If Save stays greyed out, change any other small field to wake it.
4. The first time the board finds its tasks, it gives any task with no model its recommended one: **Opus 5.5** for build and hand-off, **Haiku 5.5** for the hourly check. It does this once and never overrides a model you picked. Tasks made from Claude Code can't be given a model at creation, which is why the board does it.
5. The setup wizard tests each task.

### Each new project after that

- No new tasks. The wizard asks you to **add the new repository to all three tasks** on the routines page.
- Why: a task can only reach the repositories attached to it, so access is per task, not per board. Every project's repo must be on all three tasks, including the hourly check, which reads pull requests and deploys through GitHub.

### How the tasks run

- Build and hand-off take the board link from the start line, so one set of three tasks serves every board.
- Each run works only in the folder of its own repository and reads that repository's CLAUDE.md first. With two repositories attached, the session starts in the folder above both and neither CLAUDE.md loads by itself, so this instruction was added.
- The hourly check's scheduled run has no start line, so it goes through every board on the generic tasks (from `your_boards`).
- It **skips any board with nothing in flight** (nothing queued, building, waiting to merge, or merged but not yet live) and writes nothing on it. A board left alone for months costs a few reads an hour.
- Set start times on generic tasks begin within the hour after. The board saves your time zone so the hourly check knows when that is.

### Existing boards

- A board looks for its own tasks first, so boards with their own three tasks keep working unchanged after updating. Nothing forces a switch.
- Decision: move existing boards to the generic tasks **through chat** when wanted, not with a new feature.

## The test that settled it (1 Nov)

`vibeassist-app` (an older, larger VibeAssist) was added as a second repository on the quiet check task, and the model set to Haiku.

| | Baseline (1 repo) | Test (2 repos) |
|---|---|---|
| Model | Sonnet (no model set) | Haiku |
| Time | 47 s (had real work) | 33 s (idle) |
| Clone | 1.9 s | 1.9 s + 2.6 s |
| Starting tokens | 47,455 | 55,496 |
| Cost per run | $0.48 | $0.02 |

Findings:
- A second repository costs little: about 2.6 s of cloning and 8–12k starting tokens.
- Neither repository's CLAUDE.md loaded with two repos attached (see "How the tasks run").
- A model change on a task takes effect from the next run. The task doesn't need remaking.
- Tasks with no model set ran on Sonnet 5.5.

Measured hourly-check cost before generic tasks: Test board on Haiku about $0.014 per run (about $10/month); the VibeAssist board on default Sonnet about $0.21 per run (about $150/month). Almost all of it is fixed start-up cost, so an idle board cost as much as a busy one. That's why the idle-skip matters more than generic-vs-per-board.

Decision: use generic tasks. They're tidier, the extra repo is cheap, and one hourly run covers every board.

## Loose ends (as of 2 Nov)

1. **Boards on their own tasks don't skip idle hourly runs.** Only the generic hourly check skips. Moving those boards to the generic tasks fixes it.
2. **The hourly check and the board disagreed about "running".** The board trusts the status a part writes; the hourly check counted a part as running only if seen in the last 2 minutes, so it could start a second Builder. The fix was being written on 2 Nov together with: restore finding boards set up from Claude Code (it only worked in a Claude Project chat), and removing leftover "Claude Project" wording from board screens. The session hit the weekly limit before reporting it done. **Check whether it was committed or a PR opened.**
3. **#422 (new setup and project wizard)** hasn't been tried on a live board.
4. Still open from the earlier findings: the Builder waiting up to 30 minutes for build-notes approval (cost $8.22 at API rates in one test); `githubApp.repo` not refreshed after a typo fix; setup test times saved as dates only; inconsistent build branch names (`ask/…` vs `claude/…`); plan limits on Free and Pro untested.
5. **Housekeeping:** remove the 1am daily schedule from the Test board's build task. On the real board, once that account has credit, say "Bring in the outside-board pull requests", then make the two wording changes listed in the PR descriptions ("Work parts" and "Set up VibeAssist" asks).

## Related links

- Earlier handover (sharing, setup, builds; 30–31 Oct): https://claude.ai/artifact/Kypan6g963CZkYugvz5ZPv
- Setup and new-project mock-up (12 screens): https://claude.ai/artifact/YcSUvwrF8HoWEmTX3WmE1T
- Repository: `XIG-Australia/vibeassist-rebuild`. Its CLAUDE.md is the authority. A merge to master deploys to Cloudflare at once, for both accounts.
