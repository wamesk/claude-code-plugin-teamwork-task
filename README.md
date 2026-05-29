# teamwork-task

A Claude Code plugin that fetches Teamwork.com tasks (single task or whole
tasklist) by URL, pulls every bit of context that already lives in Teamwork
(tasklist description, task & comment attachments, file comments), implements
each task in the current repository, moves the card across the Kanban board as
the work progresses, commits per task using the `TYPE(scope)[<task-id>]: Message`
convention, and logs spent time back to Teamwork as sequential, non-overlapping
entries. Push to remote is intentionally left to the user.

Part of the [`wame`](https://github.com/wamesk/claude-code) Claude Code plugin marketplace.

**Current version:** 1.3.0 — see [`CHANGELOG.md`](CHANGELOG.md) for the full release history.

---

## What's new in 1.3.0

- **Tasklist filter (To Do + me)** — for **tasklist URLs**, only tasks in the board column `To Do` (exact, case-sensitive) AND assigned to the current Teamwork user are actually implemented. The rest of the tasklist is still fetched and rendered in the plan with a 1–2 sentence *Quick read* opinion, but the worker loop skips every mutation (no commit, no time log, no board move). Solves the common multi-repo case where a single Teamwork project holds a Laravel backend + an Ionic frontend owned by different developers — without the filter, running `/teamwork-task` from the backend repo would implement the frontend developer's tasks in the wrong codebase.
- **Single-task URLs deliberately bypass the filter.** When the user opens a specific task by ID, we honour that intent regardless of which column the task sits in or who it is assigned to.
- **Three new CLI flags** for per-run overrides: `--tasklist-filter=true|false`, `--tasklist-todo-stage=<name>`, `--tasklist-only-mine=true|false`.
- **Plan-approval gate gains two new options:** *Promote an analyse-only task to implement* (flip one or more skipped tasks back to the implement set) and *Disable the tasklist filter for this run* (process everything regardless of stage/assignee).
- Step 7 final summary now lists the filter activity and the analyse-only task IDs so the run report is honest about what was and was not implemented.
- **Honest time logs in both directions (`round_threshold_minutes`, default `4`).** Up to v1.2.0 every entry rounded **up** to ≥ 5 min — a 30-second README typo fix got billed as 5, an 11-min hotfix as 15 (a 36 % over-bill). v1.3.0 splits the decision into two zones: elapsed below the threshold logs as raw minutes (1, 2, 3); elapsed at or above the threshold rounds to the **nearest** 5-min step. Concretely: 4 → 5, 5 → 5, 6 → 5, 7 → 5, 8 → 10, 9 → 10, 11 → 10, 12 → 10, 13 → 15, 14 → 15. Set the threshold equal to `time_rounding_minutes` (e.g. both `5`) to recover the pre-1.3.0 always-round-up behaviour.
- **Empty-result message for the tasklist filter.** If the v1.3.0 "To Do" + me filter ends up with 0 tasks to implement, the skill explains *which rules* produced the empty set, *per-task reasons*, and *most common causes*, then asks a focused prompt: disable the filter, pick analyse-only tasks to promote, change the stage name, toggle the assignee check off, or cancel. No more silent "skill did nothing" runs.
- **Worktree handoff at end of run (Step 9.5).** When the skill executes inside a git worktree, every commit lands on the worktree's branch and never reaches `main` on its own. v1.3.0 asks at the end of the run what to do with those commits: *Merge into a target branch* (default — FF if possible, fall back to merge commit; target asked separately as parent / main / custom), *Push the branch for a PR*, *Leave as-is*, or *Cherry-pick specific commits*. On a successful merge the worktree branch + directory are removed by default. Disable with `--worktree-handoff=leave` or `"worktree_handoff": {"enabled": false}` for runs where you want the v1.2.x "leave it all in the worktree" behaviour back.

The 1.3.0 migration is automatic and idempotent — your existing config gains the `tasklist_filter` block on the next run, with safe defaults.

---

## What's new in 1.1.0

- **Tasklist context** — tasklist description is shown on top of the plan so the whole batch is framed before per-task planning.
- **Attachments** — task files, comment files, and file comments are pulled into `./teamwork-task-<id>/` and used as inline context. `.gitignore` is auto-updated; the folder is cleaned up after the time log succeeds.
- **Conditional comments** — `fetch_comments_mode: when_needed` (default) only fetches comments when the description is thin or explicitly references them, short-circuited when `commentsCount=0`.
- **Smart auto-commit gate** — UI/template files or diffs over 100 lines trigger a confirmation prompt; trivial textual changes commit and log automatically.
- **Auto-proposed tests** — when a task does not specify how to verify the change, the skill sniffs `composer.json` / `package.json` for installed test frameworks (Pest, PHPUnit, Dusk, Vitest, Jest, Playwright, Cypress, Selenium) and proposes the right kind of test alongside the implementation. Visual changes default to a browser test (Dusk for Laravel, Playwright/Cypress for JS); backend changes to feature / unit tests.
- **Smarter time cursor** — picks up from the end of today's most recent timelog (or skill start time for the day's first log). No overlap, no cross-midnight bleed, no future timestamps.
- **Board workflow** — task auto-moves to `In progress` on start and `Internal testing` on finish (with fallback to `Testing`). Missing workflow/columns degrade silently with a warning in the plan.
- **Commit hash in time-log description** — every Teamwork timelog now ends with `(commit: <short-hash>)` for direct traceability.
- **CHANGELOG.md** — proper Keep-a-Changelog file for the project.

Full migration notes are in [`CHANGELOG.md`](CHANGELOG.md). The 1.1.0 migration is automatic and idempotent.

---

## Installation

```text
/plugin marketplace add wamesk/claude-code
/plugin install teamwork-task@wame
```

## Prerequisites

- A Teamwork.com account with API access enabled.
- A personal Teamwork **API Key**. Create one here:
  `https://<workspace>.teamwork.com/launchpad/apikey/manage` → *Create API Key*.
  See the [Teamwork API authentication docs](https://apidocs.teamwork.com/docs/teamwork/v3/getting-started/authentication) for details.
- `curl` and `jq` available in your shell (preinstalled on macOS; on Linux: `apt install jq` / `brew install jq`).
- `git` installed and configured.

## First run

On the first invocation the plugin will:

1. Create `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (copied from the bundled `config.example.json`) with permissions `0600`.
2. Prompt you (via `AskUserQuestion`) for your Teamwork **base URL** (prefilled from the URL you passed in) and **API token**.
3. Save the values back into that config file.
4. Append `/teamwork-task-*/` to your project's `.gitignore` (if not already covered) so downloaded attachments cannot accidentally land in a commit. You will be asked whether to commit that change first.

The config lives **outside** the plugin cache, so reinstalling or updating the plugin will **not** wipe your API token. Existing v1.0.0 configs are migrated to the v1.1.0 schema automatically on the next run — no manual action required.

## Usage

```text
/teamwork-task <teamwork-url>
```

Supported URL shapes:

- Single task — `https://<workspace>.teamwork.com/app/tasks/<taskId>`
- Tasklist — `https://<workspace>.teamwork.com/app/tasklists/<tasklistId>`

Examples:

```text
/teamwork-task https://acme.teamwork.com/app/tasks/12345
/teamwork-task https://acme.teamwork.com/app/tasklists/678 --time-mode=ask
/teamwork-task https://acme.teamwork.com/app/tasklists/678 --branching=new_feature_branch
/teamwork-task https://acme.teamwork.com/app/tasks/12345 --auto-commit=never
```

Optional flags (override the saved config **for this run only**, not persisted):

- `--plan-mode=overview` — generate a single tasklist-wide plan and ask for approval before any work (default).
- `--plan-mode=per_task` — render and approve a plan **before each individual task**.
- `--plan-mode=none` — skip planning approval entirely.
- `--time-mode=real_rounded_5m` — measure actual elapsed time and round up to the nearest 5 minutes (default).
- `--time-mode=ask` — ask after each task how many minutes to log; the measured value is suggested as the default.
- `--branching=current_branch` — commit on the current branch (default).
- `--branching=new_feature_branch` — create `feature/teamwork-tasklist-<id>` (or `feature/teamwork-task-<id>`) and commit there.
- `--auto-commit=always` — commit and log without prompting, no matter how risky the diff looks.
- `--auto-commit=when_safe` — ask only for UI/template files or large diffs (default).
- `--auto-commit=never` — always prompt before committing.
- `--tasklist-filter=true|false` — **v1.3.0**, tasklist URLs only: filter implementation set to "To Do" + assigned to current user (default `true`). Single-task URLs ignore this flag.
- `--tasklist-todo-stage=<name>` — override the column name used by the tasklist filter (default `"To Do"`, case-sensitive).
- `--tasklist-only-mine=true|false` — toggle the assignee check independently of the stage check (default `true`).
- `--worktree-handoff=ask|merge|push|leave` — **v1.3.0**, only when running inside a git worktree: what to do with the worktree's commits at end of run (default `ask`). `merge` = fast-forward / merge into a target branch; `push` = push the branch for a PR; `leave` = no-op.
- `--worktree-target=ask|parent|main|<branch>` — when `--worktree-handoff=merge`, decide the target branch (default `ask`).

## What the plugin does, step by step

1. **Parses** the Teamwork URL → workspace, kind (task/tasklist), entity ID.
2. **Loads or creates** the persistent config; prompts for credentials on first run; migrates v1.0.0 configs to the v1.1.0 schema.
3. **Fetches** the task(s) via Teamwork REST API v3, including:
   - the **tasklist description** (for tasklist runs) — surfaced as *"Tasklist context"* in the plan,
   - **comments** per task (when `fetch_comments_mode` says so),
   - **task attachments** into `./teamwork-task-<id>/`,
   - **comment attachments** into `./teamwork-task-<id>/comments/`,
   - **file comments** (when `fetch_file_comments=true`),
   - the description **split on the first horizontal rule** into *acceptance criteria* (above HR) and *final summary* (below HR — treated as the authoritative goal),
   - for each unique project, the **workflow + stages** so the card can be moved to the right column.
4. **Renders a plan** (per `plan_mode`):
   - `overview` (default) — single tasklist-wide plan with goal / acceptance / approach / comments digest / attachments / board target per task → one approval.
   - `per_task` — same content but rendered and approved one task at a time inside the loop.
   - `none` — no plan approval, jump straight to work.
   You can **Approve / Skip tasks / Reorder / Add context / Cancel** before the work starts. Plan time is **not** logged to Teamwork.
5. For each task (in the approved order):
   - Starts a timer.
   - **Moves the card to "In progress"** on the project's board (if a workflow is configured).
   - Plans the implementation internally using the final summary + acceptance criteria + comments + attachments. If a business decision or missing context is required, it **stops and asks** via `AskUserQuestion`.
   - Implements the change in the current repo.
   - Runs relevant tests, if any.
   - Runs Pint formatting for PHP changes.
   - Stops the timer, rounds the duration **up** to the configured minute step (default 5).
   - **Safety gate** — if the diff touches UI/template/styling files or exceeds the line threshold, asks whether to commit, inspect first, or abort the task. Trivial edits skip this prompt.
   - Commits using the `TYPE(scope)[<task-id>]: Message` convention (no `Co-Authored-By` lines).
   - Logs time back to Teamwork via the API as a **sequential, non-overlapping** entry (see [Time logging behaviour](#time-logging-behaviour)) with a **business-oriented** Slovak description ending in `(commit: <short-hash>)`.
   - **Moves the card to "Internal testing"** (or fallback **"Testing"**) once the time log succeeds.
   - Cleans up the attachment folder.
   - Optionally marks the task as completed in Teamwork.
6. **Prints a summary table** (task ID, title, minutes, commit hash, TW status, **board stage**) and reminds you to `git push` manually.

## Configuration reference

File: `~/.claude/plugins/data/teamwork-task-wamesk/config.json`

| Key                                       | Default                                                                  | Notes                                                                                                                                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `teamwork.base_url`                       | `https://<workspace>.teamwork.com`                                       | Your Teamwork workspace URL. Asked on first run.                                                                                                                                       |
| `teamwork.api_token`                      | (empty)                                                                  | Personal API key. Asked on first run. Stored at chmod 0600.                                                                                                                            |
| `plan_mode`                               | `overview`                                                               | `overview` (one approval for the whole tasklist), `per_task` (approval before each task), or `none`.                                                                                   |
| `fetch_comments_mode`                     | `when_needed`                                                            | `always`, `when_needed` (default), or `never`. See [Fetching comments](#fetching-comments).                                                                                            |
| `fetch_attachments`                       | `true`                                                                   | If `true`, task and comment file attachments are downloaded into `./teamwork-task-<id>/`.                                                                                              |
| `fetch_file_comments`                     | `true`                                                                   | If `true`, comments attached to file objects are fetched and surfaced in the plan.                                                                                                     |
| `max_attachment_size_mb`                  | `25`                                                                     | Files larger than this are skipped (listed in the plan + final summary, not downloaded).                                                                                               |
| `attachments_cleanup`                     | `after_timelog`                                                          | `after_commit`, `after_timelog` (default), or `never`. Recommended `after_timelog` keeps files available for debugging if the time log POST fails.                                     |
| `auto_commit_mode`                        | `when_safe`                                                              | `always` (auto-commit, never ask), `when_safe` (ask only if the diff looks risky — see [Smart auto-commit](#smart-auto-commit)), or `never` (always ask).                              |
| `auto_commit_risky_patterns`              | UI/template/styling regex                                                | Regex (one per array entry) applied to changed file paths. Match ⇒ the safety gate prompts. Default covers `.vue`, `.jsx`, `.tsx`, `.svelte`, `.blade.php`, `.css/.scss/.sass/.less`, `.html`. |
| `auto_commit_max_diff_lines`              | `100`                                                                    | Total insertions + deletions over this threshold triggers the safety gate.                                                                                                             |
| `auto_propose_tests`                      | `true`                                                                   | If `true`, when the task description doesn't mention testing, propose tests appropriate for the detected stack. See [Auto-proposed tests](#auto-proposed-tests).                       |
| `test_frameworks.php_unit_preference`     | `auto`                                                                   | `auto`, `pest`, or `phpunit`. `auto` picks Pest if installed, otherwise PHPUnit.                                                                                                       |
| `test_frameworks.php_browser_preference`  | `auto`                                                                   | `auto`, `dusk`, or `selenium`. `auto` picks Dusk in Laravel projects, otherwise Selenium standalone.                                                                                  |
| `test_frameworks.js_unit_preference`      | `auto`                                                                   | `auto`, `vitest`, or `jest`. `auto` follows what `package.json` already declares.                                                                                                      |
| `test_frameworks.js_browser_preference`   | `auto`                                                                   | `auto`, `playwright`, or `cypress`. `auto` follows what `package.json` already declares.                                                                                              |
| `test_visual_file_patterns`               | UI / template regex                                                      | Regex list applied to changed paths. A match classifies the task as a *visual change* (browser test preferred); no match means *backend change* (unit/feature test preferred).         |
| `test_opt_out_keywords`                   | `["no tests", "skip tests", "without tests", "bez testov", "netreba testy"]` | Phrases in the task description that disable auto-propose for that task.                                                                                                          |
| `time_cursor_strategy`                    | `last_teamwork_timelog`                                                  | `last_teamwork_timelog` (default — pick up from today's last timelog) or `floor_now` (always start at the rounding boundary of the skill's launch time).                              |
| `include_commit_hash_in_log_description`  | `true`                                                                   | Append ` (commit: <short-hash>)` to every Teamwork time-log description.                                                                                                               |
| `board_workflow.enabled`                  | `true`                                                                   | Master toggle for board moves (in progress / internal testing).                                                                                                                        |
| `board_workflow.in_progress_stage`        | `In progress`                                                            | Name of the column to move the card to when work starts. Match is case-insensitive.                                                                                                    |
| `board_workflow.done_stage`               | `Internal testing`                                                       | Name of the column to move the card to when the time log succeeds.                                                                                                                     |
| `board_workflow.done_stage_fallbacks`     | `["Testing"]`                                                            | Tried in order if `done_stage` does not exist in the project's workflow. First match wins.                                                                                             |
| `board_workflow.match_mode`               | `case_insensitive`                                                       | Reserved for future variants; today it is always case-insensitive.                                                                                                                     |
| `time_mode`                               | `real_rounded_5m`                                                        | `real_rounded_5m` measures actual time; `ask` prompts after every task.                                                                                                                |
| `time_rounding_minutes`                   | `5`                                                                      | Rounding step (minutes). Applies to both `real_rounded_5m` durations and the session cursor alignment.                                                                                |
| `round_threshold_minutes`                 | `4`                                                                      | **v1.3.0** — elapsed `< threshold` logs as raw minutes (1, 2, 3); elapsed `>= threshold` rounds to the **nearest** `time_rounding_minutes` step. Set equal to `time_rounding_minutes` to recover the pre-1.3.0 always-round-up behaviour. Old name `round_up_threshold_minutes` is auto-migrated. |
| `min_log_minutes`                         | `1`                                                                      | **v1.2.0** — smallest entry the skill will write. Lower than `round_threshold_minutes` enables the raw-minutes sub-round zone.                                                          |
| `branching_mode`                          | `current_branch`                                                         | `current_branch` or `new_feature_branch`.                                                                                                                                              |
| `default_language`                        | `sk`                                                                     | Language for the Teamwork time-log description (Slovak by default).                                                                                                                    |
| `auto_complete_finished_tasks`            | `false`                                                                  | If `true`, the plugin marks each task as completed in TW after the commit + time log. Independent from board moves.                                                                    |
| `skip_completed_tasks`                    | `true`                                                                   | If `true`, tasks already marked as completed in TW are skipped when iterating a tasklist.                                                                                              |
| `is_billable_by_default`                  | `true`                                                                   | Sets `isbillable` on every logged time entry.                                                                                                                                          |
| `tasklist_filter.enabled`                 | `true`                                                                   | **v1.3.0** — for tasklist URLs only, filter to tasks in `tasklist_filter.todo_stage` AND assigned to the current user. Single-task URLs always bypass.                                  |
| `tasklist_filter.todo_stage`              | `"To Do"`                                                                | Name of the board column that gates implementation. Matched per `todo_stage_match_mode`.                                                                                               |
| `tasklist_filter.todo_stage_match_mode`   | `case_sensitive`                                                         | `case_sensitive` (default, strict match — `to do` does not match `To Do`) or `case_insensitive` (loose match).                                                                         |
| `tasklist_filter.only_assigned_to_me`     | `true`                                                                   | Also require the current user to be in `task.assignees`. When `false`, only the stage check applies.                                                                                   |
| `tasklist_filter.analyze_all_tasks`       | `true`                                                                   | When `true`, tasks that fail the filter are rendered in the plan as analyse-only with a *Quick read* line. When `false`, they are dropped entirely.                                    |
| `tasklist_filter.skip_reason_render`      | `inline`                                                                 | `inline` (default) renders the skip reason next to each analyse-only task in the plan. Other modes reserved for future variants.                                                       |
| `worktree_handoff.enabled`                | `true`                                                                   | **v1.3.0** — master toggle for the end-of-run worktree handoff. When `false`, the skill never auto-merges or pushes from a worktree.                                                  |
| `worktree_handoff.default_action`         | `ask`                                                                    | `ask` (default — render an AskUserQuestion), `merge`, `push`, or `leave`.                                                                                                              |
| `worktree_handoff.default_target`         | `ask`                                                                    | `ask` (default), `parent` (the branch the worktree was created from), `main` (origin/HEAD / main / master / trunk fallback), or a literal branch name.                                 |
| `worktree_handoff.merge_strategy`         | `ff_else_merge`                                                          | `ff_else_merge` (try fast-forward, fall back to merge commit), `ff_only`, `no_ff`, or `squash`.                                                                                        |
| `worktree_handoff.delete_branch_after_merge` | `true`                                                                | If `true`, after a successful merge the worktree's branch is deleted with `git branch -d` (refuses if branch has unmerged commits, as a sanity belt).                                  |
| `worktree_handoff.delete_worktree_after_merge` | `true`                                                              | If `true`, after a successful merge the worktree directory is removed with `git worktree remove`. The cwd may shift back to the main checkout.                                          |
| `worktree_handoff.push_remote`            | `origin`                                                                 | Remote name used by the "push branch for PR" path and the dirty-main fallback.                                                                                                         |
| `worktree_handoff.skip_if_no_commits`     | `true`                                                                   | If `true`, Step 9.5 is skipped silently when `HEAD` has not advanced this run (no new commits to hand off).                                                                            |

To change a setting, edit the file directly and rerun the command.

## Worktree handoff (v1.3.0)

When you run `/teamwork-task` inside a **git worktree** — typically because Claude Code launched a background job in `.claude/worktrees/<name>` or because you started the session from a worktree manually — every commit produced by the worker loop lands on the worktree's branch (e.g. `claude/teamwork-task-12345`). Up to v1.2.0 those commits stayed there forever; you had to remember to merge them by hand. v1.3.0 closes that loop in **Step 9.5**, right after the final summary and the optional verification handoff.

### What you are asked

At the end of the run, the skill renders an `AskUserQuestion`:

> Run finished in worktree `.claude/worktrees/foo` on branch `claude/foo` with **N** new commits this session: ⟨short list⟩. What do you want to do with them?

The default options are:

- **Merge into `<parent>`** — fast-forward into the parent branch (the one the worktree was created from) when possible, fall back to a merge commit otherwise. After a successful merge, the worktree's branch and directory are removed (configurable). Recommended for most flows.
- **Merge into a different branch…** — opens a follow-up question with `main`, the parent branch, and *Other (free text)* so you can point at any local branch.
- **Push the branch to `origin` for a PR** — `git push -u origin <branch>`; the branch and worktree stay, so you can open a PR manually.
- **Leave as-is — I will handle the handoff manually** — no-op.
- **Cherry-pick specific commits into a target branch** — power option; multi-select the commits, pick the target, the skill does the cherry-picks.

### Defaults you can preset for unattended runs

Edit `~/.claude/plugins/data/teamwork-task-wamesk/config.json`:

- `worktree_handoff.default_action = "merge"` — skip the first question.
- `worktree_handoff.default_target = "parent"` — skip the target question.
- `worktree_handoff.merge_strategy = "ff_else_merge"` — default.
- `worktree_handoff.delete_branch_after_merge = true` — default.
- `worktree_handoff.delete_worktree_after_merge = true` — default.

Combined, this gives you a fully hands-off pipeline: the skill implements, commits, time-logs, moves the board, runs verification, and merges everything back into the parent branch, all in one command.

### Safety rules

- **The main checkout must be clean.** If `git status` on the main repo shows uncommitted changes, Step 9.5 refuses to switch branches there (could lose your work) and falls back to *push-only* — your commits go to `origin/<branch>` and you finish the merge manually.
- **Merge conflicts pause the run.** The skill never auto-resolves merge conflicts. On failure the worktree is left intact, the error is surfaced, and you can `cd` in and finish manually.
- **Cherry-pick conflicts pause the run.** Same rule.
- **`skip_if_no_commits = true`** (default) — if the worker loop did not commit anything (every task aborted / skipped / timelog failed), Step 9.5 is silently skipped. Nothing to merge.

### Bypass

- **Per run:** `--worktree-handoff=leave` (skip entirely) or `--worktree-handoff=push` (skip the merge question).
- **Persistently:** `"worktree_handoff": {"enabled": false}` in your config.
- **Single-task runs from the main checkout** never trigger this step — the detection in Step 5.1 marks `WT_RUN_IN_WORKTREE=0` and Step 9.5 is a no-op.

## Tasklist filter (v1.3.0)

For **tasklist** URLs only, the plugin filters which tasks get actually implemented:

- Only tasks **in the board column `To Do`** (matched case-sensitively by default — `to do`, `TO DO`, `ToDo` do **not** match) AND
- **assigned to the current Teamwork user**

are run through the worker loop. The rest of the tasklist's tasks are still fetched and shown in the plan with a short *Quick read* sanity-check line, but the plugin **never** commits, time-logs, or moves the board card for them.

**Single-task URLs** (`/teamwork-task https://…/tasks/12345`) deliberately **bypass** the filter — when you point at a specific task by ID, the plugin honours that intent regardless of the task's column or assignee.

The motivation is the standard multi-repo Kanban setup: a single Teamwork project commonly contains both a Laravel backend and an Ionic / iOS / Vue frontend, owned by different developers working in different repositories. Without the filter, running `/teamwork-task <tasklist-url>` from the Laravel repo would happily start implementing the frontend developer's tasks in the wrong codebase. The filter narrows the implementation set to what you actually own and is greenlit to work on, while still surfacing teammates' tasks in the same plan so you can sanity-check them in standup.

### How to override

- **Per run:** `--tasklist-filter=false` (process all fetched tasks), `--tasklist-only-mine=false` (skip the assignee check), or `--tasklist-todo-stage="Ready"` (use a different column name).
- **From the plan-approval prompt:** *Promote an analyse-only task to implement* (flip specific task IDs back to the implement set) or *Disable the tasklist filter for this run*.
- **Persistently:** edit `~/.claude/plugins/data/teamwork-task-wamesk/config.json` and set `"tasklist_filter": {"enabled": false}`, or change `todo_stage` / `only_assigned_to_me` to whatever your team's convention is.

### Edge cases handled

- **Project has no workflow at all** → filter degrades to "process everything" (no `To Do` to filter by). A one-line note is shown in the plan.
- **Task has no card / is not on the board** → falls back to `process` so backlog items never get silently stranded.
- **`/me.json` is unreachable** (token without `users.read` scope, network blip) → assignee check is skipped so you are never silently locked out of your own work; the stage check still applies.

## Plan modes

- **`overview` (default)** — after fetching tasks and comments, the plugin prints a single markdown plan covering every task in the run (goal, acceptance criteria, planned approach, comments digest, attachments, board target) and asks for one approval. You can **Approve / Skip tasks / Reorder / Add context / Cancel** before any code is touched.
- **`per_task`** — the same plan shape is rendered and approved **before each task**, right before its timer starts. Safest for high-stakes work; noisier for long tasklists.
- **`none`** — no plan approval gate; the plugin starts working as soon as the task list is fetched. Use only when you fully trust the task descriptions.

The plan generation itself is **not** counted in any task's time log — the timer for a task starts after planning, when implementation begins.

## Task description convention (acceptance criteria + final summary)

The plugin parses each task's description by splitting on the **first horizontal rule** (`<hr>` in HTML or `---` / `***` / `___` on its own line in Markdown):

```
Acceptance criteria:
- Must allow PDF export
- Must respect tenant theme
- Must work offline

---

Final summary: implement an `InvoicePdfExporter` action that uses the Dompdf engine,
respects the active tenant theme via the existing `ThemeResolver` service, and queues
the job for offline support. Reuse the existing `Invoice::toArray()` shape.
```

- Content **above** the HR is treated as the **acceptance criteria** (the checklist).
- Content **below** the HR is the **final summary** — the authoritative goal — and is used as the basis for the commit message body and the Teamwork time-log description.
- If no HR is present, the whole description is treated as acceptance criteria and the plan will warn that no final summary was provided.

When `URL_KIND=tasklist`, the **tasklist's own description** is rendered on top of the plan as *"Tasklist context"* to frame the whole batch before per-task planning.

## Fetching comments

`fetch_comments_mode` controls when the plugin pulls comments:

- **`always`** — fetch comments for every task (legacy v1.0 behavior, more API calls).
- **`when_needed`** (default) — fetch only when the task description is unlikely to be self-contained. Specifically: fetch only if `commentsCount > 0` *and* at least one of these holds:
  - The description has no horizontal rule (no *final summary*), or
  - The acceptance criteria above the HR is shorter than 100 characters, or
  - The description text explicitly references comments (e.g. *"viď komentár"*, *"see comments"*, *"viz nižšie"*).
- **`never`** — skip entirely, even when the description is sparse. Useful when the tasks are known to be fully described.

Comments are kept in chronological order. The **last comment is the freshest truth** when comments contradict each other.

## Attachments

When `fetch_attachments=true` (default), the plugin downloads:

- **Task files** → `./teamwork-task-<task-id>/`
- **Comment files** → `./teamwork-task-<task-id>/comments/`
- (Optionally) **File comments** → surfaced as a one-line digest per file in the plan, not downloaded as separate files.

Each downloaded file is prefixed with its Teamwork file ID (e.g. `987_mockup.png`) so name collisions never overwrite each other.

A few hard rules:

- The first run in a repo appends `/teamwork-task-*/` to your project's `.gitignore` and asks whether to commit that `.gitignore` change before starting work. Subsequent runs only touch `.gitignore` if the pattern is not already there.
- Files larger than `max_attachment_size_mb` (default 25 MB) are **skipped** — listed in the plan and the final summary as *"skipped: file (size)"*, never downloaded.
- After the time log succeeds, the attachment folder is removed per `attachments_cleanup` (default `after_timelog`). If the time log POST fails, the folder is intentionally **kept** so you can inspect what was being worked on.
- Text-based files (`.md`, `.txt`, `.json`, `.yaml`, `.csv`, `.log`, source files, etc.) are read inline by Claude for additional context. Binary files (images, PDFs, archives) are listed in the plan but not opened.

## Smart auto-commit

The `auto_commit_mode` setting decides whether a commit is auto-approved or gated on a confirmation prompt:

- **`always`** — every commit is auto-approved.
- **`when_safe`** (default) — auto-commit unless the diff trips a heuristic that suggests human review:
  - A changed file matches one of `auto_commit_risky_patterns` (UI / template / styling by default), **or**
  - The diff size is greater than `auto_commit_max_diff_lines` (default 100 lines of insertions + deletions).
  If any of those fires, the plugin asks via `AskUserQuestion`:
  - **Approve commit** → commit and continue.
  - **Inspect first** → pause; you `git diff` / open the editor; the plugin re-asks afterwards.
  - **Abort task** → leave the working tree as-is, do not commit, do not log time, do not move the board card. The next task (if any) still runs.
- **`never`** — always ask before committing, regardless of diff content.

`auto_commit_risky_patterns` is a JSON array of regex strings applied to changed file paths via `grep -iE`. Adjust it to match your codebase: e.g. add `\\.tpl$` for Smarty templates, or `^database/migrations/` if you want database changes gated even when small.

## Auto-proposed tests

If a Teamwork task forgets to describe how the change should be verified, the plugin proposes tests for you instead of skipping the topic. The behaviour is controlled by `auto_propose_tests` (default `true`).

**How the skill decides what to do**

For each task, the plugin classifies the description as one of:

- **`described`** — text mentions any of *test*, *unit test*, *feature test*, *browser test*, *Pest*, *PHPUnit*, *Dusk*, *Selenium*, *Playwright*, *Cypress*, *e2e*, *cover with tests*, *napíš testy*, etc. The skill uses your description as-is.
- **`opted_out`** — text matches any phrase in `test_opt_out_keywords` (defaults include *"no tests"*, *"skip tests"*, *"bez testov"*, *"netreba testy"*). The skill writes a note in the plan and writes no tests.
- **`missing`** — neither of the above. Auto-propose mode kicks in.

**How the framework is chosen**

For `missing`, the skill sniffs `composer.json` and `package.json` for installed frameworks:

| Project signal           | Visual change (`.vue` / `.tsx` / `.blade.php` / `resources/views/…`) | Backend change                         |
| ------------------------ | -------------------------------------------------------------------- | -------------------------------------- |
| Laravel + Dusk installed | Laravel Dusk in `tests/Browser/`                                    | Pest if installed, else PHPUnit         |
| Laravel without Dusk     | Ask to install Dusk **or** write manual checklist                    | Pest if installed, else PHPUnit         |
| PHP without Laravel      | Selenium standalone PHPUnit **or** manual checklist                  | Pest if installed, else PHPUnit         |
| JS with Playwright       | Playwright in `tests/e2e/`                                          | Vitest if installed, else Jest          |
| JS with Cypress          | Cypress in `cypress/e2e/`                                           | Vitest if installed, else Jest          |
| JS without any browser   | Ask to install Playwright **or** manual checklist                    | Vitest if installed, else Jest          |
| Nothing detected         | Manual checklist                                                     | Manual checklist                        |

`test_frameworks.*_preference` lets you pin a specific choice (e.g. force `pest` even if PHPUnit is also present, or force `playwright` over a coexisting `cypress`). The default `"auto"` follows the table.

**Missing tool — ask, never auto-install**

When the chosen framework is not installed (e.g. a visual change in a Laravel project without Dusk), the skill **never** runs `composer require` or `npm install` silently. It asks via `AskUserQuestion`:

- **Install `<package>` now** — runs the install command and continues.
- **Skip browser tests, write a manual checklist** — proceeds without a test file; the checklist is emitted into the plan.
- **Abort the task** — leaves the working tree as-is.

**What gets committed**

When the skill auto-proposes tests, the implementation **and** the test file land in the same commit, so the time log line `(commit: <hash>)` covers both. The next *Internal testing* board move signals that a human can pick up where automated verification stopped.

## Board workflow

When `board_workflow.enabled=true` (default), the plugin moves the Teamwork card across the project's Kanban board as work progresses:

- At the **start** of each task (right after the timer starts) → move to `board_workflow.in_progress_stage` (default `"In progress"`).
- After the **time log succeeds** → move to `board_workflow.done_stage` (default `"Internal testing"`). If that column does not exist in the project, the plugin tries the names in `board_workflow.done_stage_fallbacks` (default `["Testing"]`) in order; first match wins.

Stage matching is **case-insensitive** — `"INTERNAL TESTING"`, `"Internal testing"`, and `"internal testing"` all match the same column.

What happens if a stage is missing:

- **Both start and done stages found** — both moves happen.
- **Only `in_progress_stage` found** — start move happens; done move is skipped with a warning in the plan and the final summary. The task stays in *In progress* after completion.
- **Only the done stage found** — start move is skipped; done move still happens after the time log.
- **No workflow on the project, or neither stage found** — board moves are disabled for that project's tasks; a single warning is shown in the plan, and the *Board stage* column in the final summary shows `—`.

Failures of the move POST itself (HTTP non-2xx) are **non-fatal**: the plugin logs a warning and continues. Board moves are a nice-to-have, not a gating dependency. There are no retries — the most likely cause is a permissions or naming mismatch which retrying will not fix.

Interaction with `auto_complete_finished_tasks`: both run independently. If you enable task completion *and* board moves, the task ends up `completed=true` and in the done stage simultaneously, which Teamwork handles fine.

If the safety gate at commit time results in **Abort task**, the card stays in *In progress* (semantically correct — the task was not completed).

## Time logging behaviour

Hard rules every entry the plugin writes to Teamwork obeys:

1. **Sequential, non-overlapping entries.** Even though implementation work can overlap in real time (parallel tool calls, interleaved investigations), the time **logged to Teamwork** is laid out strictly back-to-back. The plugin maintains a single *session cursor* that advances by exactly the logged duration after each entry. Result: if task A is logged as `start 10:15, 20 min`, task B will be logged as `start 10:35, …` — never `10:32`, never overlapping `10:25–10:45`.
2. **Round-to-nearest beyond the threshold, raw below it (default 5 min step, 4 min threshold — v1.3.0).** The duration decision is split into two zones controlled by `round_threshold_minutes` (default `4`):
   - **Elapsed at or above the threshold** rounds to the **nearest** `time_rounding_minutes` step using half-up integer rounding. With defaults (`threshold=4`, `ROUND=5`): 4 → 5, 5 → 5, 6 → 5, 7 → 5, 8 → 10, 9 → 10, 10 → 10, 11 → 10, 12 → 10, 13 → 15, 14 → 15, 15 → 15. This is honest in both directions — 7-min work bills as 5, 8-min work bills as 10. Over many tasks the bias averages out.
   - **Elapsed strictly below the threshold but at least `min_log_minutes` (default 1)** logs as **raw minutes**. With defaults: 1 → 1, 2 → 2, 3 → 3. The cursor advances by the exact raw amount, so the next log starts off the 5-min grid (e.g. `:17`). This keeps a trivial-edit log honest at 1 min instead of padding it to 5.
   - Set `round_threshold_minutes` equal to `time_rounding_minutes` (e.g. both `5`) to recover the pre-1.3.0 "always round up to ROUND" behaviour where every entry is ≥ 5 min and the cursor never falls off the 5-min grid.
3. **Plan time is never logged.** The session cursor starts only after the plan is approved. Time spent on planning, reading the user's clarifications, or waiting for an `AskUserQuestion` response is **not** billed.

### Time cursor strategy

The session cursor's starting position is controlled by `time_cursor_strategy`:

- **`last_teamwork_timelog`** (default) — pick up where you left off. The plugin asks the Teamwork API for the most recent timelog **from today** (via `GET /projects/api/v3/time.json?assignedToUserIds=<you>&startDate=<today>&orderBy=date&orderMode=desc&pageSize=1`) and uses its *end* timestamp (rounded up to the 5-minute boundary). If there is no log today, the cursor starts at `floor(skill-launch-time, 5min)`. Future timestamps (clock skew) and cross-midnight cases fall back to `floor(now)`.
- **`floor_now`** — always start at `floor(skill-launch-time, 5min)`. Classic v1.0 behavior, useful if you keep parallel timesheets manually and don't want the plugin to discover them.

The final summary prints which source was used (`last_timelog @ 10:50` or `skill start (first log of day)`), so you can always verify which strategy actually fired.

If a `POST` to Teamwork fails (network blip, 5xx), the cursor does **not** advance — the next successful log keeps the same start time so your timesheet stays contiguous, **and** the corresponding *Move to "Internal testing"* board step is skipped (the task is not yet billed).

## Time-log description tone

The text that lands in Teamwork's time-log description is written from the perspective of someone reading the timesheet for billing or status (PM, client, accountant) — **not** a developer reading a code review. State **what was delivered for the user or the business**. Include a technical detail only when it materially helps identify the work (a specific module name, a feature flag, a migration ID). Avoid variable names, line counts, library versions, and diff stats. Keep it to 1–2 sentences in `default_language` (Slovak by default), followed by ` (commit: <short-hash>)` if `include_commit_hash_in_log_description=true` (default).

**Good (business):**
- *„Pridaná možnosť exportu faktúr do PDF s podporou témy nájomcu. (commit: a1b2c3d)"*
- *„Opravený výpadok prihlasovania pri súbežnom obnovení relácie. (commit: 9f4e2b1)"*

**Avoid (technical):**
- *„Refactored InvoiceController::export() to use new DompdfRenderer, added 3 tests."*
- *„Updated 8 files, +142 −37 lines, bumped filament/filament to 5.2."*

## Time tracking modes

- **`real_rounded_5m` (default)** — the plugin measures wall-clock time between starting the task and the commit, then rounds **up** to the nearest `time_rounding_minutes`. Best for billable work where you trust the measurement.
- **`ask`** — after each commit, the plugin asks you how many minutes to log, with the measured value prefilled. Use this when you frequently take long breaks, multitask, or want explicit control.

Either mode still feeds into the sequential, 5-minute-aligned session cursor above — `ask` only lets you override the measured duration, not the start time.

## Branching modes

- **`current_branch` (default)** — every commit lands on whatever branch is currently checked out. The plugin verifies the working tree is clean before starting (after the `.gitignore` auto-update step).
- **`new_feature_branch`** — the plugin creates (or switches to) `feature/teamwork-tasklist-<tasklistId>` for tasklist runs, or `feature/teamwork-task-<taskId>` for single-task runs, and commits there.

## Commit format

The plugin extends the conventions in your `~/.claude/CLAUDE.md` with the **Teamwork task ID in square brackets** right after the scope, so every commit is traceable back to its source task:

```
TYPE(scope)[<task-id>]: Short summary

Optional detailed description on next lines (based on the task's final summary).
```

- `TYPE` ∈ `CREATE`, `UPDATE`, `EDIT`, `FIX`, `REMOVE`, `MOVE`, `UPGRADE`, `DELETE`, …
- `scope` is the module / model / area touched (lowercase, e.g. `user`, `auth`, `invoice`).
- `<task-id>` is the numeric Teamwork task ID, no `#` prefix — so `git log --grep '\[123456\]'` finds every commit related to that task.
- The plugin **never** adds `Co-Authored-By` lines.

Examples:

```
CREATE(user)[123456]: Create user module
UPDATE(invoice)[123789]: Add PDF export action
FIX(auth)[124001]: Resolve session timeout race condition
```

## Blockers

When something genuinely cannot be decided (ambiguous spec, missing context, business call), the plugin **stops and asks** via `AskUserQuestion`. It will not invent decisions for product/business-shaped questions, and it will not skip the task silently.

Non-essential side-effects are **never blockers** — board move failures, attachment download failures, file comments fetch failures, and `auto_complete_finished_tasks` POST failures all degrade silently with a warning. The core task → commit → time log pipeline still runs end-to-end.

## Security

- The Teamwork API token is stored in plaintext at `~/.claude/plugins/data/teamwork-task-wamesk/config.json` with file permissions `0600`.
- The token is never echoed to stdout, never written to commit messages, and never passed to `curl` via the URL — only via `-u "$TOKEN:xxx"`.
- The plugin's own `.gitignore` excludes `config.json` from the repo itself as a safety net.
- Downloaded attachments may contain confidential customer material. The auto-added `/teamwork-task-*/` entry in your project's `.gitignore` prevents accidental commits, and the post-timelog cleanup removes the folder once the work is logged.

If you suspect your token has leaked, revoke it in Teamwork (`/launchpad/apikey/manage`) and rerun the plugin — it will re-prompt.

## Roadmap

- Branch per task (currently: tasklist-level branch or current branch).
- Automatic `git push` after the run.
- Open a Pull Request per task or per tasklist.
- Multi-repo orchestration (when a tasklist spans multiple repositories).
- Resume after interruption — recover from the last successful timelog / commit when a run was killed mid-way (today only the time cursor handles part of this).

## License

MIT — see [`LICENSE`](LICENSE).
