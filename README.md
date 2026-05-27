# teamwork-task

A Claude Code plugin that fetches Teamwork.com tasks (single task or whole
tasklist) by URL, pulls every bit of context that already lives in Teamwork
(tasklist description, task & comment attachments, file comments), implements
each task in the current repository, moves the card across the Kanban board as
the work progresses, commits per task using the `TYPE(scope)[<task-id>]: Message`
convention, and logs spent time back to Teamwork as sequential, non-overlapping
entries. Push to remote is intentionally left to the user.

Part of the [`wame`](https://github.com/wamesk/claude-code) Claude Code plugin marketplace.

**Current version:** 1.1.0 — see [`CHANGELOG.md`](CHANGELOG.md) for the full release history.

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
| `branching_mode`                          | `current_branch`                                                         | `current_branch` or `new_feature_branch`.                                                                                                                                              |
| `default_language`                        | `sk`                                                                     | Language for the Teamwork time-log description (Slovak by default).                                                                                                                    |
| `auto_complete_finished_tasks`            | `false`                                                                  | If `true`, the plugin marks each task as completed in TW after the commit + time log. Independent from board moves.                                                                    |
| `skip_completed_tasks`                    | `true`                                                                   | If `true`, tasks already marked as completed in TW are skipped when iterating a tasklist.                                                                                              |
| `is_billable_by_default`                  | `true`                                                                   | Sets `isbillable` on every logged time entry.                                                                                                                                          |

To change a setting, edit the file directly and rerun the command.

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
2. **Everything aligned to the rounding step (default 5 min).** Both the **duration** and the **start time** are multiples of `time_rounding_minutes`. You will never see a log starting at `:17` or lasting `13 min`; the plugin rounds the duration **up** and aligns the cursor to the nearest 5-minute boundary.
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
