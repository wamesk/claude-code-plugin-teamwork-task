# teamwork-task

A Claude Code plugin that fetches Teamwork.com tasks (single task or whole
tasklist) by URL, pulls every bit of context that already lives in Teamwork
(tasklist description, task & comment attachments, file comments), implements
each task in the current repository, moves the card across the Kanban board as
the work progresses, commits per task using the `TYPE(scope)[<task-id>]: Message`
convention, and logs spent time back to Teamwork as sequential, non-overlapping
entries. Push to remote is intentionally left to the user.

Part of the [`wame`](https://github.com/wamesk/claude-code) Claude Code plugin marketplace.

**Current version:** 1.8.0 — see [`CHANGELOG.md`](CHANGELOG.md) for the full release history.

---

## What's new in 1.8.0

- **Fast mode defers verification, not quality.** `ui_ux` (UI/UX and accessibility) stays on at build time in `fast` mode: when it is in `build_quality.dimensions`, Step 6.2 plans it and Step 6.3 follows its rules exactly as in `full`. `performance`, `security` and `reachability` get no plan line, but the code still covers them by taking the patterns the sibling code already uses (eager loading, the neighbours' policies, the menu entry and inbound links of a new screen). `framework` stays off — no version or docs lookups, sibling idioms only; `/work-mode full` checks frameworks, standards and best practices. The Step 6.5.5 self-check is still deferred to `/work-mode full`, and an explicit `--dimensions=` still wins. `full` is unchanged.

## What's new in 1.7.0

- **Work modes renamed: `fast` and `full`.** The companion plugin `wame-work-mode` 1.0.0 became [`work-mode`](https://github.com/wamesk/claude-code-plugin-work-mode) 2.0.0, and the names follow it: `--mode=fast|full` (`full` stays the default), `/work-mode fast|full|status` to switch (no argument opens a menu), and `/work-mode full` to run the deferred checks once and then switch to full (`/work-mode full --no-checks` only switches). The project files are `.claude/work-mode.local.md` (frontmatter `mode: fast|full`) and `.claude/work-mode-deferred.local.md` (same block format as before).
- **No `mode` key in the shared config any more.** The global default lives in Claude Code `/config` as the `work-mode` plugin option `default_mode`; that plugin's SessionStart hook writes it into the project's mode file, so this skill only reads the project file. The Step 2.6 migration deletes the `mode` key (idempotent). Resolution order: `--mode` > `.claude/work-mode.local.md` > legacy `.claude/wame-mode.local.md` > `full`.
- **The old names still work for one version.** `--mode=build` / `--mode=harden` map to `fast` / `full` and print one `⚠` line; a project that only has the legacy `.claude/wame-mode.local.md` is still read (`build` → `fast`, `harden` → `full`); a legacy `.claude/wame-deferred.local.md` is moved to the new name before the first append, so no deferred entry is lost.
- **The deferred list cannot ride along in a commit.** Step 6.7 stages explicit paths only and unstages both deferred-list names and both mode-file names before every commit, in every mode; the deferred block is appended only after the commit, and a list that is not git-ignored prints one `⚠` line.

The 1.7.0 migration only removes `mode` from your config. If you had set it to the fast value, set the `work-mode` plugin's `default_mode` to `fast` in `/config` instead.

## What's new in 1.6.0

- **Work modes: build fast, verify once.** Agents used to spend most of a run verifying instead of building — the quality dimensions, a proposed test per task, a test run and Pint per task, browser checks and the whole `/teamwork-task-test` pass on every task. `--mode=fast` (or `mode: fast` in the project's `.claude/work-mode.local.md` written by `/work-mode` from the [`work-mode`](https://github.com/wamesk/claude-code-plugin-work-mode) plugin) switches all of that off at once — the same as `--dimensions=none --test-after=false` plus `auto_propose_tests: false`, no per-task tests / Pint and no browser work. Each committed task is recorded in `.claude/work-mode-deferred.local.md`, and `/work-mode full` runs every skipped check once at the end. Fetching, board moves, commits and time logs run as usual. `full` (default) behaves exactly like 1.5.0; an explicit individual flag still wins over the mode. See [Work modes](#work-modes-v170). *(Shown with the 1.7.0 names — see [What's new in 1.7.0](#whats-new-in-170) for what was renamed; since 1.8.0 `fast` keeps `ui_ux` on at build time — see [What's new in 1.8.0](#whats-new-in-180).)*
- **No more browser tooling installed on the fly.** The skill never installs or uninstalls Playwright, Puppeteer or Dusk for a single run. It uses the chrome-devtools MCP or the runner the project already has; a missing runner is asked about **once**, and on yes it is installed permanently as a committed dev dependency.

The 1.6.0 migration only added a `mode` key to your config (removed again in 1.7.0) — nothing changed until you chose the fast mode.

## What's new in 1.5.0

- **Comments are actually read again.** Up to 1.4.2 the plugin could report *"no comments"* on a task that had eight — four independent defects each produced that same silent result (reported in a colleague's *"Štyri cesty k nule"* analysis): an invalid sort key that made the comments endpoint answer HTTP 400, a gate on a `commentsCount` field Teamwork v3 never returns, a wrong timestamp field name, and `echo "$JSON" | jq` pipes that break in zsh (the macOS default shell Claude Code runs every snippet in). All four are fixed; failures now print a `⚠` line naming the endpoint and the fallback instead of continuing with empty context.
- **The newest comment is always read.** `fetch_comments_mode: when_needed` now runs one cheap probe per task that returns the comment count *and* the newest comment. Whenever a task has comments, the newest one is read — it is the freshest truth and can overrule the description. The old heuristics (thin description, *"viď komentár"*) now only decide whether the **full thread** is fetched. `always` and `never` keep their meaning. A v1 fallback covers a failing v3 endpoint.
- **Board moves, the tasklist filter and subtasks work again.** v3 task objects carry no `projectId`; the plugin now reads `.tasklist.meta.projectId`, so workflow detection, board moves (also for subtasks) and the tasklist filter get a real project. The filter reads each task's column from `workflowStages` (the `?include=cards,stages` lookup always came back empty).
- **The new WAME board.** Work starts in **any of the start columns** — `tasklist_filter.todo_stages`, default *Ready for Development* and *To Do* — and a finished task moves to **Done - Local** (fallbacks *Internal testing* → *Testing*, so the older per-project boards keep landing where they did). *In progress* still resolves *In Progress* (case-insensitive). A task that is **not on the board** is now *analyse-only* ("not on the board") instead of being implemented — only the start columns are greenlit work; promote it from the plan when it is yours. See [Board workflow](#board-workflow) and [the migration rule](#migration-rule-for-the-board-columns).
- **A completed task behind a single-task URL is not reworked silently.** The skill asks first — *Skip* (recommended) or *Process anyway*, with the warning that processing moves the card out of its done column. `skip_completed_tasks` keeps skipping completed tasks silently in tasklists and subtasks.
- **Explicit `false` in the config is respected.** jq's `// true` treated `false` as missing, so `tasklist_filter.enabled`, `only_assigned_to_me`, `board_workflow.enabled`, `skip_completed_tasks`, `is_billable_by_default` and other switches could not be turned off (the migration even rewrote them to `true`).
- **Attachments come from the right place.** Task files are resolved via `GET /tasks/{id}.json?include=attachments`; the old fallback endpoint ignored its task filter and returned the whole workspace's files.
- **Build-time quality rules.** Every task is planned (Step 6.2), implemented (Step 6.3) and self-checked (new Step 6.5.5) against the same five dimensions `/teamwork-task-test` reviews at QA time — **UI/UX & accessibility, performance, security, page reachability** (a new screen gets its menu entry *and* inbound links from related screens in the same commit) **and framework best practices** (new or changed code uses the current idioms and built-in features of the framework / language versions the project actually has installed). What stays open is listed per task in the final summary and handed to `/teamwork-task-test`. Configure with `build_quality.dimensions` or `--dimensions=<csv>|none`.
- **Framework versions are detected, not remembered.** Once per run the plugin reads `composer.lock`, `composer.json` (`require.php` / `config.platform.php`), `package.json` + `node_modules` / lock, `browserslist` and `.nvmrc` into `/tmp/tw_job_<id>/framework_versions.tsv`, and looks APIs up in current docs (Laravel Boost `search-docs` → context7 → official docs). Guardrails: project conventions win over a newer idiom, no drive-by rewrites of untouched code (such opportunities are only listed as advisory tips), nothing deprecated in — or newer than — the installed versions and the browserslist target. `/teamwork-task-test` treats `framework` as advisory: it recommends, it never fails an acceptance criterion on it.
- **zsh-safe snippets.** Every snippet in the skill follows a documented *shell portability contract* (no echo pipes on API data, no swallowed parse errors, no unquoted `for` loops, no bash-only arrays, run state in files under `/tmp/tw_job_<id>/` instead of shell variables that die with each Bash call).

The 1.5.0 migration is automatic and idempotent — your config gains `build_quality.dimensions` (all five) on the next run. A list written by a pre-release 1.5.0 build that holds exactly the four original keys gains `framework` once; any list you customised stays as it is (see [the migration rule](#migration-rule-for-framework)). The board keys move to the new WAME board only where you kept the old defaults: `todo_stages` is added (`["Ready for Development", "To Do"]`, or `[<your column>]` when you had customised `todo_stage`), and `done_stage: "Internal testing"` becomes `"Done - Local"` with `["Internal testing", "Testing"]` as fallbacks; a customised `done_stage` is never touched (see [the board migration rule](#migration-rule-for-the-board-columns)).

---

## What's new in 1.4.0

- **Subtasks are first-class tasks.** When a parent task has subtasks (the common "container parent + N children" pattern — e.g. *Project bootstrap* with 8 subtasks for CI / auth / DB / …), the plugin now expands the parent in the working set and runs each subtask through the full pipeline: own description / acceptance criteria / comments / attachments fetch, own plan entry, own commit (`TYPE(scope)[<subtaskId>]: …`), own board move (*In progress → Internal testing* at the time; *Done - Local* with fallbacks since 1.5.0), own time log. The parent itself stays put on the board — it is just the container — and gets no commit and no time log.
- **Tasklist filter applies to subtasks.** The v1.3.0 *To Do + assigned to me* rules (since 1.5.0: *start columns + me*) run on the post-expansion set: a subtask in the wrong column or assigned to a teammate is dropped to *analyse-only* exactly like a top-level task. The single-task URL bypass still applies — paste the parent's URL, get all its subtasks processed regardless of column.
- **Parent context in the plan.** Each subtask's plan entry is prefixed with a one-line `Parent context:` showing the parent's name and a short description excerpt, so you can scan what the container is meant to deliver before reading the subtask. Disable with `"subtasks": {"include_parent_context": false}`.
- **Recursive expansion** up to `subtasks.max_depth` (default `2`) — covers parent → subtask → sub-subtask. Set to `1` to expand only direct children; raise for deeper hierarchies.
- **API shape tolerance.** Both v3 response shapes (`.tasks[]` and `.subtasks[]`) and both endpoints (`/tasks/{id}/subtasks.json` primary, `/tasks.json?parentTaskId={id}` fallback) are supported transparently.
- **Backward-compatible default-off path** — set `"subtasks": {"enabled": false}` to recover v1.3 behaviour entirely (parents stay in the working set, subtasks invisible). Existing single-task and tasklist runs that did not involve subtasks behave exactly the same as in v1.3.

The 1.4.0 migration is automatic and idempotent. Your existing config gains the `subtasks` block on the next run with safe defaults.

---

## What's new in 1.3.0

- **Tasklist filter (To Do + me)** — for **tasklist URLs**, only tasks in the board column `To Do` (exact, case-sensitive; since 1.5.0 any of the start columns, default *Ready for Development* + *To Do*) AND assigned to the current Teamwork user are actually implemented. The rest of the tasklist is still fetched and rendered in the plan with a 1–2 sentence *Quick read* opinion, but the worker loop skips every mutation (no commit, no time log, no board move). Solves the common multi-repo case where a single Teamwork project holds a Laravel backend + an Ionic frontend owned by different developers — without the filter, running `/teamwork-task` from the backend repo would implement the frontend developer's tasks in the wrong codebase.
- **Single-task URLs deliberately bypass the filter.** When the user opens a specific task by ID, we honour that intent regardless of which column the task sits in or who it is assigned to (since 1.5.0 a *completed* task asks first).
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
- **Conditional comments** — `fetch_comments_mode: when_needed` (default) only fetches comments when the description is thin or explicitly references them. *(Superseded in 1.5.0: the count now comes from a real probe — the `commentsCount` field this relied on is never returned by Teamwork v3 — and the newest comment is always read; see [Fetching comments](#fetching-comments).)*
- **Smart auto-commit gate** — UI/template files or diffs over 100 lines trigger a confirmation prompt; trivial textual changes commit and log automatically.
- **Auto-proposed tests** — when a task does not specify how to verify the change, the skill sniffs `composer.json` / `package.json` for installed test frameworks (Pest, PHPUnit, Dusk, Vitest, Jest, Playwright, Cypress, Selenium) and proposes the right kind of test alongside the implementation. Visual changes default to a browser test (Dusk for Laravel, Playwright/Cypress for JS); backend changes to feature / unit tests.
- **Smarter time cursor** — picks up from the end of today's most recent timelog (or skill start time for the day's first log). No overlap, no cross-midnight bleed, no future timestamps.
- **Board workflow** — task auto-moves to `In progress` on start and `Internal testing` on finish (with fallback to `Testing`). Missing workflow/columns degrade silently with a warning in the plan. *(1.5.0: the done target is `Done - Local`, fallbacks `Internal testing` → `Testing`; see [Board workflow](#board-workflow).)*
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
- `--tasklist-filter=true|false` — **v1.3.0**, tasklist URLs only: filter implementation set to the start columns + assigned to current user (default `true`). Single-task URLs ignore this flag.
- `--tasklist-todo-stage=<name>[,<name>…]` — override the start columns used by the tasklist filter for this run, comma-separated (default `"Ready for Development,To Do"`, each matched per `todo_stage_match_mode` — case-sensitive by default).
- `--tasklist-only-mine=true|false` — toggle the assignee check independently of the stage check (default `true`).
- `--worktree-handoff=ask|merge|push|leave` — **v1.3.0**, only when running inside a git worktree: what to do with the worktree's commits at end of run (default `ask`). `merge` = fast-forward / merge into a target branch; `push` = push the branch for a PR; `leave` = no-op.
- `--worktree-target=ask|parent|main|<branch>` — when `--worktree-handoff=merge`, decide the target branch (default `ask`).
- `--subtasks=true|false` — **v1.4.0**, expand parent tasks into their subtasks (default `true`).
- `--dimensions=<csv>|none` — **v1.5.0**, which build-time quality dimensions to plan and self-check: any subset of `ui_ux,performance,security,reachability,framework`, or `none` (default: `build_quality.dimensions`, all five). See [Build-time quality](#build-time-quality-v150).
- `--mode=fast|full` — **v1.6.0**, renamed in **v1.7.0**; work mode for this run (default: `.claude/work-mode.local.md`, then the legacy `.claude/wame-mode.local.md`, then `full`). `fast` = only `ui_ux` planned at build time (the other keys follow the sibling code, **v1.8.0**), no quality self-check, no test proposal, no per-task tests / Pint, no browser work and no `/teamwork-task-test` handoff; skipped checks go to `.claude/work-mode-deferred.local.md` for `/work-mode full`. The old values `build` / `harden` are deprecated aliases for one version (one `⚠` line). See [Work modes](#work-modes-v170).

## What the plugin does, step by step

1. **Parses** the Teamwork URL → workspace, kind (task/tasklist), entity ID.
2. **Loads or creates** the persistent config; prompts for credentials on first run; migrates v1.0.0 configs to the v1.1.0 schema.
3. **Fetches** the task(s) via Teamwork REST API v3, including:
   - the **tasklist description** (for tasklist runs) — surfaced as *"Tasklist context"* in the plan,
   - **comments** per task — the newest one whenever the task has any, the full thread when `fetch_comments_mode` says so,
   - **task attachments** into `./teamwork-task-<id>/`,
   - **comment attachments** into `./teamwork-task-<id>/comments/`,
   - **file comments** (when `fetch_file_comments=true`),
   - the description split into *acceptance criteria* and *final summary* (treated as the authoritative goal) — in the canonical WAME format the criteria are the block under `## Akceptačné kritériá` / `## Acceptance criteria` up to the next horizontal rule, `### Prierezové požiadavky` included; otherwise the description is **split on the first horizontal rule** (above / below),
   - for each unique project, the **workflow + stages** so the card can be moved to the right column,
   - for a single-task URL pointing at a **completed** task, a question whether to process it at all (default: skip — processing would move its card out of the done column).
4. **Renders a plan** (per `plan_mode`):
   - `overview` (default) — single tasklist-wide plan with goal / acceptance / approach / comments digest / attachments / board target per task → one approval.
   - `per_task` — same content but rendered and approved one task at a time inside the loop.
   - `none` — no plan approval, jump straight to work.
   You can **Approve / Skip tasks / Reorder / Add context / Cancel** before the work starts. Plan time is **not** logged to Teamwork.
5. For each task (in the approved order):
   - Starts a timer.
   - **Moves the card to "In progress"** on the project's board (if a workflow is configured).
   - Plans the implementation internally using the final summary + acceptance criteria + comments + attachments, including which of the five [build-time quality dimensions](#build-time-quality-v150) the change touches (and, once per run, the framework versions the project has installed). If a business decision or missing context is required, it **stops and asks** via `AskUserQuestion`.
   - Implements the change in the current repo, following the quality rules of every dimension that applies.
   - Runs relevant tests, if any.
   - Runs Pint formatting for PHP changes.
   - **Self-checks** its own diff against the applicable dimensions, fixes what is cheap, and records what stays open.
   - Stops the timer, rounds the duration **up** to the configured minute step (default 5).
   - **Safety gate** — if the diff touches UI/template/styling files or exceeds the line threshold, asks whether to commit, inspect first, or abort the task. Trivial edits skip this prompt.
   - Commits using the `TYPE(scope)[<task-id>]: Message` convention (no `Co-Authored-By` lines).
   - Logs time back to Teamwork via the API as a **sequential, non-overlapping** entry (see [Time logging behaviour](#time-logging-behaviour)) with a **business-oriented** Slovak description ending in `(commit: <short-hash>)`.
   - **Moves the card to "Done - Local"** (fallbacks **"Internal testing"** → **"Testing"**) once the time log succeeds.
   - Cleans up the attachment folder.
   - Optionally marks the task as completed in Teamwork.
6. **Prints a summary table** (task ID, title, minutes, commit hash, TW status, **board stage**, **build quality**) with the open quality items per task, hands them to `/teamwork-task-test` when it is installed, and reminds you to `git push` manually.

## Configuration reference

File: `~/.claude/plugins/data/teamwork-task-wamesk/config.json`

| Key                                       | Default                                                                  | Notes                                                                                                                                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `teamwork.base_url`                       | `https://<workspace>.teamwork.com`                                       | Your Teamwork workspace URL. Asked on first run.                                                                                                                                       |
| `teamwork.api_token`                      | (empty)                                                                  | Personal API key. Asked on first run. Stored at chmod 0600.                                                                                                                            |
| `plan_mode`                               | `overview`                                                               | `overview` (one approval for the whole tasklist), `per_task` (approval before each task), or `none`.                                                                                   |
| `fetch_comments_mode`                     | `when_needed`                                                            | `always`, `when_needed` (default), or `never`. `when_needed` always reads the newest comment and fetches the full thread only when the description is thin. See [Fetching comments](#fetching-comments). |
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
| `board_workflow.enabled`                  | `true`                                                                   | Master toggle for board moves (in progress / done column). |
| `board_workflow.in_progress_stage`        | `In progress`                                                            | Name of the column to move the card to when work starts. Match is case-insensitive, so it also resolves *In Progress* on the WAME board. |
| `board_workflow.done_stage`               | `Done - Local`                                                           | **1.5.0 default** (was `Internal testing`). Name of the column to move the card to when the time log succeeds — the WAME board's column for work finished locally. |
| `board_workflow.done_stage_fallbacks`     | `["Internal testing", "Testing"]`                                        | Tried in order if `done_stage` does not exist in the project's workflow (the older boards). First match wins. |
| `board_workflow.done_stage_schema`        | `2`                                                                      | **v1.5.0** — migration marker, do not edit: records that the done-target migration ran, so choosing `Internal testing` again later is never undone. See [the board migration rule](#migration-rule-for-the-board-columns). |
| `board_workflow.match_mode`               | `case_insensitive`                                                       | Reserved for future variants; today it is always case-insensitive.                                                                                                                     |
| `time_mode`                               | `real_rounded_5m`                                                        | `real_rounded_5m` measures actual time; `ask` prompts after every task.                                                                                                                |
| `time_rounding_minutes`                   | `5`                                                                      | Rounding step (minutes). Applies to both `real_rounded_5m` durations and the session cursor alignment.                                                                                |
| `round_threshold_minutes`                 | `4`                                                                      | **v1.3.0** — elapsed `< threshold` logs as raw minutes (1, 2, 3); elapsed `>= threshold` rounds to the **nearest** `time_rounding_minutes` step. Set equal to `time_rounding_minutes` to recover the pre-1.3.0 always-round-up behaviour. Old name `round_up_threshold_minutes` is auto-migrated. |
| `min_log_minutes`                         | `1`                                                                      | **v1.2.0** — smallest entry the skill will write. Lower than `round_threshold_minutes` enables the raw-minutes sub-round zone.                                                          |
| `branching_mode`                          | `current_branch`                                                         | `current_branch` or `new_feature_branch`.                                                                                                                                              |
| `default_language`                        | `sk`                                                                     | Language for the Teamwork time-log description (Slovak by default).                                                                                                                    |
| `auto_complete_finished_tasks`            | `false`                                                                  | If `true`, the plugin marks each task as completed in TW after the commit + time log. Independent from board moves.                                                                    |
| `skip_completed_tasks`                    | `true`                                                                   | If `true`, tasks already marked as completed in TW are skipped silently when iterating a tasklist or subtasks. A single-task URL pointing at a completed task always asks first (default: skip). |
| `is_billable_by_default`                  | `true`                                                                   | Sets `isbillable` on every logged time entry.                                                                                                                                          |
| `tasklist_filter.enabled`                 | `true`                                                                   | **v1.3.0** — for tasklist URLs only, filter to tasks in one of `tasklist_filter.todo_stages` AND assigned to the current user. Single-task URLs always bypass. |
| `tasklist_filter.todo_stages`             | `["Ready for Development", "To Do"]`                                     | **v1.5.0** — the start columns: a task is implementable when its column is **any** of them (order = display only). Each matched per `todo_stage_match_mode`. Tasks not on the board are never implementable. An empty list falls back to `todo_stage`, then the default. |
| `tasklist_filter.todo_stage`              | (legacy)                                                                 | Pre-1.5.0 single column name. Still read when `todo_stages` is absent or empty; the migration turns it into `todo_stages` once (`"To Do"` → the default pair, anything else → `[<name>]`) and leaves the key in place for older sibling plugins. |
| `tasklist_filter.todo_stage_match_mode`   | `case_sensitive`                                                         | `case_sensitive` (default, strict match per start column — `to do` does not match `To Do`) or `case_insensitive` (loose match). |
| `tasklist_filter.only_assigned_to_me`     | `true`                                                                   | Also require the current user to be in `task.assignees`. When `false`, only the stage check applies.                                                                                   |
| `tasklist_filter.analyze_all_tasks`       | `true`                                                                   | When `true`, tasks that fail the filter are rendered in the plan as analyse-only with a *Quick read* line. When `false`, they are dropped entirely.                                    |
| `tasklist_filter.skip_reason_render`      | `inline`                                                                 | `inline` (default) renders the skip reason next to each analyse-only task in the plan. Other modes reserved for future variants.                                                       |
| `subtasks.enabled`                        | `true`                                                                   | **v1.4.0** — expand every parent task with subtasks into the working set so each subtask is implemented, committed, board-moved and time-logged independently. `false` recovers v1.3 (parent stays, subtasks invisible). |
| `subtasks.max_depth`                      | `2`                                                                      | **v1.4.0** — recursion ceiling for subtask expansion. `1` = direct children only, `2` = also sub-subtasks, `3+` = deeper hierarchies. Safety cap against runaway recursion.            |
| `subtasks.include_parent_context`         | `true`                                                                   | **v1.4.0** — when expanding a subtask, render a `Parent context:` line (parent name + ≤ 300-char description excerpt) in the plan entry so the LLM understands the broader scope.       |
| `worktree_handoff.enabled`                | `true`                                                                   | **v1.3.0** — master toggle for the end-of-run worktree handoff. When `false`, the skill never auto-merges or pushes from a worktree.                                                  |
| `worktree_handoff.default_action`         | `ask`                                                                    | `ask` (default — render an AskUserQuestion), `merge`, `push`, or `leave`.                                                                                                              |
| `worktree_handoff.default_target`         | `ask`                                                                    | `ask` (default), `parent` (the branch the worktree was created from), `main` (origin/HEAD / main / master / trunk fallback), or a literal branch name.                                 |
| `worktree_handoff.merge_strategy`         | `ff_else_merge`                                                          | `ff_else_merge` (try fast-forward, fall back to merge commit), `ff_only`, `no_ff`, or `squash`.                                                                                        |
| `worktree_handoff.delete_branch_after_merge` | `true`                                                                | If `true`, after a successful merge the worktree's branch is deleted with `git branch -d` (refuses if branch has unmerged commits, as a sanity belt).                                  |
| `worktree_handoff.delete_worktree_after_merge` | `true`                                                              | If `true`, after a successful merge the worktree directory is removed with `git worktree remove`. The cwd may shift back to the main checkout.                                          |
| `worktree_handoff.push_remote`            | `origin`                                                                 | Remote name used by the "push branch for PR" path and the dirty-main fallback.                                                                                                         |
| `worktree_handoff.skip_if_no_commits`     | `true`                                                                   | If `true`, Step 9.5 is skipped silently when `HEAD` has not advanced this run (no new commits to hand off).                                                                            |
| `build_quality.dimensions`                | `["ui_ux", "performance", "security", "reachability", "framework"]`      | **v1.5.0** — build-time quality dimensions planned (Step 6.2), followed (Step 6.3) and self-checked (Step 6.5.5) per task. Same keys `/teamwork-task-test` reviews (`framework` there as advisory only). `[]` = none. Per run: `--dimensions=`. |
| `build_quality.dimensions_schema`         | `2`                                                                      | **v1.5.0** — migration marker, do not edit: records that the list was reconciled with the five-key set, so removing `framework` is never undone by a later run. See [the migration rule](#migration-rule-for-framework). |

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

## Tasklist filter (v1.3.0, start columns since v1.5.0)

For **tasklist** URLs only, the plugin filters which tasks get actually implemented:

- Only tasks **in one of the start columns** — `tasklist_filter.todo_stages`, default **`Ready for Development`** and **`To Do`** (the two start columns of the shared WAME board; an older board without *Ready for Development* simply uses *To Do*), each matched case-sensitively by default — `to do`, `TO DO`, `ToDo` do **not** match `To Do` — AND
- **assigned to the current Teamwork user**

are run through the worker loop. The rest of the tasklist's tasks — including every task that is **not on the board** — are still fetched and shown in the plan with a short *Quick read* sanity-check line, but the plugin **never** commits, time-logs, or moves the board card for them.

**Single-task URLs** (`/teamwork-task https://…/tasks/12345`) deliberately **bypass** the filter — when you point at a specific task by ID, the plugin honours that intent regardless of the task's column or assignee. The one exception is a **completed** task: the plugin asks first whether to process it (default: skip), because processing moves its card out of the done column. Completed tasks inside a tasklist or among subtasks are skipped silently (`skip_completed_tasks`).

The motivation is the standard multi-repo Kanban setup: a single Teamwork project commonly contains both a Laravel backend and an Ionic / iOS / Vue frontend, owned by different developers working in different repositories. Without the filter, running `/teamwork-task <tasklist-url>` from the Laravel repo would happily start implementing the frontend developer's tasks in the wrong codebase. The filter narrows the implementation set to what you actually own and is greenlit to work on, while still surfacing teammates' tasks in the same plan so you can sanity-check them in standup.

### How to override

- **Per run:** `--tasklist-filter=false` (process all fetched tasks), `--tasklist-only-mine=false` (skip the assignee check), or `--tasklist-todo-stage="Backlog"` / `--tasklist-todo-stage="Ready for Development,To Do,Backlog"` (other start columns, comma-separated).
- **From the plan-approval prompt:** *Promote an analyse-only task to implement* (flip specific task IDs back to the implement set — also the way to take a task that is not on the board) or *Disable the tasklist filter for this run*.
- **Persistently:** edit `~/.claude/plugins/data/teamwork-task-wamesk/config.json` and set `"tasklist_filter": {"enabled": false}`, or change `todo_stages` / `only_assigned_to_me` to whatever your team's convention is.

### Edge cases handled

- **Task has no card / is not on the board** → **v1.5.0** *analyse-only* with reason `no_card`, shown as *"not on the board"* — a backlog item nobody moved into a start column is not greenlit work. Promote it from the plan when it is yours. (1.3.0–1.4.2 documented the opposite, *process*.)
- **Project has no workflow at all** → every task is *not on the board*, so the whole tasklist is analyse-only and the plan asks how to proceed (disable the filter for this run, promote tasks, change the start columns, or cancel). A one-line note names the project.
- **Board has only some of the start columns** (the older per-project boards have *To Do* but no *Ready for Development*) → the missing name never matches; nothing else changes. A board with none of them is named once in the plan.
- **Task is on the board but its column cannot be read** (the workflow lookup failed, e.g. rate limit) → **v1.5.0** keeps it *analyse-only* with reason `stage_unresolved(<stageId>)` and a `⚠` line — an unreadable column is never treated as a start column.
- **`/me.json` is unreachable** (token without `users.read` scope, network blip) → assignee check is skipped so you are never silently locked out of your own work; the stage check still applies.

## Subtasks (v1.4.0)

Teamwork's data model allows a task to have **subtasks** — child tasks under a parent with their own description, acceptance criteria, comments, attachments, assignee, and stage on the board. A common pattern is a "container" parent task like *Project bootstrap* with 8 subtasks splitting the work by area (CI, auth, DB, …). Up to v1.3.0 the plugin ignored subtasks entirely: it planned, committed and time-logged against the empty parent, leaving the actual N units of work invisible. v1.4.0 closes that loop in **Step 3.42**.

### How expansion works

After the initial fetch and the tasklist-context lookup (Step 3.4), the plugin calls `GET /projects/api/v3/tasks/{id}/subtasks.json` (fallback when it fails: `GET /tasks.json?parentTaskId={id}` — singular, v3 ignores the plural `parentTaskIds` — filtered client-side) for every task in the working set. Each subtask's board column comes from its own `workflowStages`, its project from `.tasklist.meta.projectId`. When subtasks come back, the parent is **removed from the implementation working set** and each subtask takes its place. From that point on every subtask goes through the full pipeline as if it were a top-level task:

- Step 3.5 — own comments probe (newest comment always, full thread when needed).
- Step 3.6 — own description split (acceptance criteria above HR, final summary below).
- Step 3.7 — own attachment folder `./teamwork-task-<subtaskId>/`.
- Step 3.9 — own file-comments digest.
- Step 3.10 — local working-tree discovery.
- Step 3.45 — tasklist filter (when applicable) operates on the expanded set; a subtask outside the start columns (not on the board included) or assigned to a teammate is dropped to *analyse-only* by the same rules as a top-level task.
- Step 4 — own plan entry, prefixed with a `Parent context:` line (parent name + ≤ 300-char description excerpt) when `subtasks.include_parent_context = true`.
- Step 5 — own timer; the same sequential, non-overlapping 5-min cursor.
- Step 6 — own implementation, own `TYPE(scope)[<subtaskId>]: …` commit, own *In progress → Done - Local* board move (fallbacks *Internal testing* → *Testing*).
- Step 5/6 timelog — own `POST /projects/api/v3/tasks/{subtaskId}/time.json` entry.

### Behavioural decisions baked in

- **Parent stays put.** The parent is never moved across the workflow and gets no time log — it is a container, not a work item. When all its subtasks finish, the parent is **not** auto-clicked complete; close it manually if your team uses that convention.
- **Per-subtask commits and time logs.** No aggregation into one parent timelog. Every subtask gets its own commit with its own ID in square brackets and its own 5-min-aligned timelog.
- **Per-subtask board moves.** Each subtask has its own card and goes through *In progress → Done - Local* (or its fallback) independently.
- **Filter applies to subtasks.** A subtask outside the start columns or assigned to a teammate is dropped to `analyse_only` by Step 3.45 with the same *start columns + me* rules as a standalone task.

### Single-task URL on a parent with subtasks

`URL_KIND=task` and the URL points at a parent: Step 3.42 expands it into its 8 (or however many) subtasks. The tasklist filter is skipped (single-task URL bypass), so every open subtask is `process_mode=process` regardless of column or assignee (completed subtasks are skipped silently with `skip_completed_tasks`; a completed parent URL asks first) — you effectively get the same behaviour as feeding the plugin a tasklist URL with 8 children, minus the filter.

### Single-task URL on a subtask itself

Paste the subtask's URL directly: Step 3.42 finds no further subtasks (leaf) and the pipeline runs against that single subtask. Useful for hand-picking one specific child to work on.

### Configuration

Edit `~/.claude/plugins/data/teamwork-task-wamesk/config.json`:

- `subtasks.enabled = false` — recover v1.3 behaviour (parent stays in the working set, subtasks invisible). The whole rest of v1.4 is no-op.
- `subtasks.max_depth = 1` — expand only direct children; sub-subtasks stay attached to their parent subtask.
- `subtasks.max_depth = 3` (or higher) — go deeper for nested hierarchies.
- `subtasks.include_parent_context = false` — drop the `Parent context:` line from subtask plan entries (saves a few tokens per task; useful when the parent name is enough).

### Edge cases handled

- **Subtask in a different project than parent** → the per-project workflow cache (Step 3.3) picks up the extra project; the subtask's board move targets its own project's workflow.
- **API shape variance** → both `.tasks[]` and `.subtasks[]` response shapes are accepted; both `/tasks/{id}/subtasks.json` (primary) and `/tasks.json?parentTaskId={id}` (fallback) are tried.
- **`skip_completed_tasks = true`** → applies per subtask. A completed subtask is dropped from the expanded set the same way a completed top-level task is.
- **Subtask has no card / not on the board** → **v1.5.0** *analyse-only* (`no_card`) in a tasklist run, like a top-level task; promote it from the plan when it is yours. Under a single-task URL the filter is bypassed and it is processed.
- **`subtasks.enabled = false`** → step is a complete no-op; v1.3 behaviour.

## Plan modes

- **`overview` (default)** — after fetching tasks and comments, the plugin prints a single markdown plan covering every task in the run (goal, acceptance criteria, planned approach, comments digest, attachments, board target) and asks for one approval. You can **Approve / Skip tasks / Reorder / Add context / Cancel** before any code is touched.
- **`per_task`** — the same plan shape is rendered and approved **before each task**, right before its timer starts. Safest for high-stakes work; noisier for long tasklists.
- **`none`** — no plan approval gate; the plugin starts working as soon as the task list is fetched. Use only when you fully trust the task descriptions.

The plan generation itself is **not** counted in any task's time log — the timer for a task starts after planning, when implementation begins.

## Task description convention (acceptance criteria + final summary)

When the description is in the canonical WAME format written by `teamwork-task-analyze`, `-from-desk`, `-from-session` and `-from-dnr` (`[preamble] → --- → ## Akceptačné kritériá → --- → ## Cieľ → --- → ## Technický popis`), the acceptance criteria are the block from the `## Akceptačné kritériá` / `## Acceptance criteria` heading to the next horizontal rule — its `### Prierezové požiadavky` / `### Cross-cutting requirements` items included, which Step 6.2 treats as binding — and the preamble is context. Any other description is parsed by splitting on the **first horizontal rule** (`<hr>` in HTML or `---` / `***` / `___` on its own line in Markdown):

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
- **`when_needed`** (default) — one probe per task (`GET …/comments.json?pageSize=1&orderBy=date&orderMode=desc`) returns the comment **count** and the **newest comment** in a single call:
  - no comments → nothing to read;
  - one or more → the **newest comment is always read**, because the last comment is the freshest truth and can change the spec even on a well-described task;
  - more than one → the **full thread** is fetched as well when at least one of these holds:
    - The description has no horizontal rule (no *final summary*), or
    - The acceptance criteria above the HR is shorter than 100 characters, or
    - The description text explicitly references comments (e.g. *"viď komentár"*, *"see comments"*, *"viz nižšie"*), or
    - The newest comment refers back to earlier ones or contradicts the description.
- **`never`** — skip entirely (no probe either), even when the description is sparse. Useful when the tasks are known to be fully described.

Comments are sorted chronologically by `postedDateTime` (`orderBy=date`). If the v3 endpoint fails, the plugin prints a `⚠` line and falls back to the v1 endpoint (`/tasks/{id}/comments.json`, sorted by `datetime`); an unknown count means the full thread is read rather than assuming zero. The plan's *Comments context* line says exactly what was read (e.g. *"newest of 8 read — full thread not fetched"*). The **last comment is the freshest truth** when comments contradict each other or the description.

## Build-time quality (v1.5.0)

`/teamwork-task-test` reviews five cross-cutting dimensions at QA time. Since 1.5.0 this plugin applies the same five **while building**, so QA confirms instead of discovers:

| Key | Applies when the change … | Build rule (short form) |
| --- | --- | --- |
| `ui_ux` | touches templates / components / views (`test_visual_file_patterns`) or a class feeding them | copy the sibling screen's patterns; accessible names; disabled controls say why; labels and `alt`; loading / empty / error states; every string through the translation layer with a resolving English key |
| `performance` | adds queries, migrations, loops over records, batches, imports / exports | no query in a loop, eager loads, pagination, `chunkById`, indexes on new foreign keys and filtered columns, filter in SQL, slow work to a queue |
| `security` | adds routes, actions, buttons, endpoints, inputs, raw queries | same gate as the neighbours **plus** an object-scoped policy (no IDOR), tenant scope intact, FormRequest validation, explicit `$fillable`, handled errors instead of 500s, no internals or secrets in messages |
| `reachability` | adds, renames or removes a screen (page, view route, Nova resource / lens / dashboard / tool, SPA route) | the screen ships in the **same commit** with its **menu entry** and **inbound links** from related screens (relation tab on the parent, detail link, action, breadcrumb); a deliberately URL-only page is named as an `allow_orphans` candidate; renames leave no dead links; menu visibility matches route authorization |
| `framework` | writes or changes code in PHP, Laravel, Nova, Livewire, Inertia, Pest, JS / TS, Vue, React, CSS or Tailwind — almost every code task | new or changed code uses the current idiom / built-in feature of the **installed** version (looked up in current docs, not remembered) instead of a dated or hand-rolled pattern; project conventions win; no drive-by rewrites; nothing deprecated in, or newer than, the installed version / PHP floor / browserslist target; no new dependency for a built-in |

How it runs per task:

1. **Plan (Step 6.2)** — decide from the shape of the change which dimensions apply; a pure backend task records `ui_ux: not_applicable` instead of growing UI boilerplate.
2. **Implement (Step 6.3)** — follow the rules of every dimension that applies.
3. **Self-check (Step 6.5.5)** — walk the task's own diff, fix what is cheap now, and record one status per dimension: `checked`, `not_applicable`, `skipped(--dimensions)` or `open(<item> @ <file:line>)`. For `framework` it also records up to three advisory `suggest(<opportunity> @ <file:line>)` tips for code the task read but did not change. A dimension is never reported as checked when it was not read.
4. **Report & hand off** — the final summary shows a *Build quality* cell per task plus every open item, the detected framework versions and the advisory framework tips; open `security` / `reachability` items make the commit safety gate ask first (default `auto_commit_mode=when_safe`) — `framework` rows never do; the list is handed to `/teamwork-task-test` before it runs, with the `framework` lines marked advisory. Nothing is blocked silently.

Turn it off per run with `--dimensions=none`, or persistently with `"build_quality": {"dimensions": []}`.

### Framework best practices (`framework`)

The fifth dimension asks for the idioms and built-in features of the versions the project **actually has installed**, not the ones a model remembers from older releases:

1. **Detect the versions once per run** (Step 6.2) into `/tmp/tw_job_<id>/framework_versions.tsv` — `laravel/framework`, `laravel/nova`, `livewire/livewire`, `inertiajs/inertia-laravel`, `pestphp/pest`, `laravel/boost` from `composer.lock`; the PHP constraint from `composer.json` (`require.php`, `config.platform.php`); `vue`, `react`, `nuxt`, `vite`, `typescript`, `tailwindcss`, `@inertiajs/*`, `@ionic/*` from `node_modules` → `package-lock.json` → `yarn.lock` → the declared range; the `browserslist` target; the Node version (`.nvmrc` / `.node-version` / `engines.node`). The first task of the run detects, later tasks reuse (a task that changes a manifest or lock file drops the cache, so the next one detects again). PHP language features are bounded by the lowest PHP the project admits (`config.platform.php`, else the floor of `require.php`), not by your local `php -v`.
2. **Look the API up in current docs** — Laravel Boost `search-docs` when the project has `laravel/boost`, otherwise the context7 MCP, otherwise the official docs; per major version skim the upgrade guide for what is new and deprecated.
3. **Build within the guardrails** — the project's `CLAUDE.md` and sibling conventions win over a newer idiom; no second pattern next to an established one (unless the task is the refactor); no drive-by rewrites of untouched code (those become advisory tips in the summary); never a deprecated API; never a feature newer than the installed version, the PHP floor or the browserslist target (that one is a bug and is fixed before the commit); no new dependency for something the framework ships.

On the QA side `/teamwork-task-test` (1.2.0+) treats `framework` as **advisory**: recommendations only, no code edits, never a failed or downgraded acceptance criterion, never a blocker (only code that will not run on the installed versions / browserslist target is a real finding, filed under the other keys). Sibling plugins that write task descriptions (`teamwork-task-analyze`, `-from-desk`, `-from-session`, `-from-dnr`) put `framework` notes into the technical plan, never into the acceptance criteria.

### Migration rule for `framework`

`build_quality.dimensions_schema` records which key set your list was last reconciled with (absent = a four-key list from a pre-release 1.5.0 build; `2` = five keys). The Step 2.6 migration adds `framework` **once**, and only when the marker is absent and the list is exactly `ui_ux`, `performance`, `security`, `reachability` (each once, any order) — such a list cannot be told apart from the untouched old default. Every other list counts as customised and is left untouched: a subset, `[]`, a list that already names `framework`, or one with other keys. A missing list gets the five-key default. Afterwards the marker is `2`, so if you remove `framework` from the list it stays removed on every later run.


## Work modes (v1.7.0)

| | `full` (default) | `fast` |
|---|---|---|
| Quality dimensions planned (Step 6.2) | `build_quality.dimensions` | `ui_ux` only (the others follow the sibling code) |
| Quality self-check (Step 6.5.5) | run | skipped |
| Auto-proposed tests (Step 6.2.5) | `auto_propose_tests` | off |
| Per-task tests (6.4) and Pint (6.5) | run | skipped |
| Browser checks, docs / version lookups | as configured | skipped |
| `/teamwork-task-test` handoff (Step 8) | `auto_run_tests_after` | off (`--test-after=false`) |
| Fetching, board moves, commits, time logs | run | run |

Resolution: `--mode=` > the project's `.claude/work-mode.local.md` frontmatter `mode:` (written by `/work-mode`, plugin `work-mode`) > the legacy `.claude/wame-mode.local.md` (read for one more version; `build` → `fast`, `harden` → `full`) > `full`. There is no config key: the global default is the `work-mode` plugin option `default_mode` in Claude Code `/config`, and that plugin's SessionStart hook writes it into the project's mode file. An explicit `--dimensions=` or `--test-after=true` still wins over `fast`. In `fast` mode every committed task appends its files, screens and skipped checks to `.claude/work-mode-deferred.local.md` (a legacy `.claude/wame-deferred.local.md` is moved there first; the list is never staged or committed); `/work-mode full` runs those checks once — tests, Pint, the quality self-check, security and code review, one visual pass — clears the list and switches back to `full`.

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
| Laravel without Dusk     | Ask once to add Dusk permanently **or** write manual checklist       | Pest if installed, else PHPUnit         |
| PHP without Laravel      | Selenium standalone PHPUnit **or** manual checklist                  | Pest if installed, else PHPUnit         |
| JS with Playwright       | Playwright in `tests/e2e/`                                          | Vitest if installed, else Jest          |
| JS with Cypress          | Cypress in `cypress/e2e/`                                           | Vitest if installed, else Jest          |
| JS without any browser   | Ask once to add Playwright permanently **or** manual checklist       | Vitest if installed, else Jest          |
| Nothing detected         | Manual checklist                                                     | Manual checklist                        |

`test_frameworks.*_preference` lets you pin a specific choice (e.g. force `pest` even if PHPUnit is also present, or force `playwright` over a coexisting `cypress`). The default `"auto"` follows the table.

**Missing tool — ask once, install permanently, never on the fly**

The skill never installs or uninstalls Playwright, Puppeteer or Dusk for a single run; it prefers the chrome-devtools MCP or the runner the project already has. When the chosen framework is not installed (e.g. a visual change in a Laravel project without Dusk), it asks **once per run** via `AskUserQuestion`:

- **Add `<package>` to the project permanently** — installs it as a committed dev dependency and never removes it afterwards.
- **Skip browser tests, write a manual checklist** — proceeds without a test file; the checklist is emitted into the plan.
- **Abort the task** — leaves the working tree as-is.

**What gets committed**

When the skill auto-proposes tests, the implementation **and** the test file land in the same commit, so the time log line `(commit: <hash>)` covers both. The next board move to the done column (*Done - Local*, or *Internal testing* / *Testing* on older boards) signals that a human can pick up where automated verification stopped.

## Board workflow

When `board_workflow.enabled=true` (default), the plugin moves the Teamwork card across the project's Kanban board as work progresses:

- At the **start** of each task (right after the timer starts) → move to `board_workflow.in_progress_stage` (default `"In progress"`).
- After the **time log succeeds** → move to `board_workflow.done_stage` (default **`"Done - Local"`** since 1.5.0). If that column does not exist in the project, the plugin tries the names in `board_workflow.done_stage_fallbacks` (default `["Internal testing", "Testing"]`) in order; first match wins.

Stage matching is **case-insensitive** — `"DONE - LOCAL"`, `"Done - Local"`, and `"done - local"` all match the same column, and the default `"In progress"` matches *In Progress* on the WAME board.

Both board generations work with the defaults (verified with read-only requests on 2026-09-24):

| Board | Start columns (tasklist filter) | In progress | Done target |
| --- | --- | --- | --- |
| Shared **WAME workflow** (Planned tasks · … · Ready for Development · To Do · In Progress · On Hold · Done - Local · Deployed to DEV · …) | *Ready for Development*, *To Do* | *In Progress* | *Done - Local* |
| Older per-project boards (Next Sprint · Waiting for approval · To Do · In progress · On hold · Testing · Done - ready to deploy · …) | *To Do* | *In progress* | *Testing* (fallback) |

What happens if a stage is missing:

- **Both start and done stages found** — both moves happen.
- **Only `in_progress_stage` found** — start move happens; done move is skipped with a warning in the plan and the final summary. The task stays in *In progress* after completion.
- **Only the done stage found** — start move is skipped; done move still happens after the time log.
- **No workflow on the project, or neither stage found** — board moves are disabled for that project's tasks; a single warning is shown in the plan, and the *Board stage* column in the final summary shows `—`.

Failures of the move POST itself (HTTP non-2xx) are **non-fatal**: the plugin logs a warning and continues. Board moves are a nice-to-have, not a gating dependency. There are no retries — the most likely cause is a permissions or naming mismatch which retrying will not fix.

Interaction with `auto_complete_finished_tasks`: both run independently. If you enable task completion *and* board moves, the task ends up `completed=true` and in the done stage simultaneously, which Teamwork handles fine.

If the safety gate at commit time results in **Abort task**, the card stays in *In progress* (semantically correct — the task was not completed).

### Migration rule for the board columns

The shared WAME board starts work in *Ready for Development* or *To Do* and ends it in *Done - Local*; older boards end in *Internal testing* or *Testing*. The Step 2.6 migration moves your config to the new board **only where you kept the old defaults**, idempotently:

- **Start columns** — `tasklist_filter.todo_stages` is added once, only when it is missing (or `null`): `["Ready for Development", "To Do"]` when `todo_stage` was missing or the old default `"To Do"`, `[<your column>]` when you had customised it (`"Backlog"` → `["Backlog"]`). An existing `todo_stages` is never overwritten; the legacy `todo_stage` key stays in the file.
- **Done target** — `board_workflow.done_stage` moves from the old default `"Internal testing"` to `"Done - Local"`, and the fallbacks become `["Internal testing", …your other fallbacks except "Done - Local"…]` (deduplicated, `"Testing"` kept). A customised `done_stage` (anything but `"Internal testing"`) is never touched. `board_workflow.done_stage_schema: 2` records that the check ran, so if you set `"Internal testing"` again afterwards it stays.

Examples: the 1.4.2 default (`"Internal testing"` + `["Testing"]`) and a config that already listed `"Done - Local"` as a fallback (`"Internal testing"` + `["Done - Local", "Testing"]`) both end as `"Done - Local"` + `["Internal testing", "Testing"]`; `"QA review"` + `["Testing"]` stays as it is. `in_progress_stage` is not migrated — `"In progress"` already matches *In Progress* case-insensitively.

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

If a `POST` to Teamwork fails (network blip, 5xx), the cursor does **not** advance — the next successful log keeps the same start time so your timesheet stays contiguous, **and** the corresponding move to the done column (*Done - Local* or its fallback) is skipped (the task is not yet billed).

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
