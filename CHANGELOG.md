# Changelog

All notable changes to the `teamwork-task` plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.1.1] - 2026-05-27

Implement → verify in a single command. When the companion `teamwork-task-test`
plugin is installed, this skill now hands off to it at the very end of the
worker loop so each task's acceptance criteria get individually verified before
the user pushes.

### Added

- **`auto_run_tests_after`** config key (default `true`) — at the end of Step 7
  (after the implementation summary), the skill detects whether the
  `teamwork-task-test` skill is available and, if so, invokes it via the `Skill`
  tool with the original Teamwork URL. The verification skill's per-task and
  tasklist reports are rendered inline before the final push reminder, so the
  user sees the QA verdict before deciding to push.
- **`--test-after=true|false`** CLI flag to override the config for the current
  run only.
- **Test skill detection** is best-effort and silent — the available-skills list
  is consulted first, with a filesystem check under `~/.claude/plugins/**/
  teamwork-task-test/SKILL.md` as a fallback. If neither signal confirms the
  skill is installed, a single-line tip is printed pointing at
  `/plugin install teamwork-task-test@wame` and the run ends cleanly.
- **`Skill` added to `allowed-tools`** so the test skill can be invoked from
  within this skill's execution context.

### Changed

- The "push manually when ready" reminder moved from the end of Step 7 to a
  dedicated Step 9, so the verification output (Step 8) lands *before* the push
  prompt — the user sees test results, then decides about pushing.
- Config migration banner now says *"config migrated to 1.1.1 schema"* when any
  key was added by the idempotent `jq` merge.

### Compatibility

- Fully backward compatible. Users on 1.1.0 who do not install
  `teamwork-task-test` see exactly the same behaviour as before, plus the one
  tip line at the end. The new config key is added silently on first run by the
  existing migration step.

---

## [1.1.0] - 2026-05-27

Context-rich runs, smart gating, and board-aware automation. The skill now pulls
more of the surrounding signal that already lives in Teamwork (tasklist
description, task & comment attachments, file comments), asks for review only
when the change is visibly risky, picks up the time cursor where your last
timelog left off, and moves the card across the Kanban board as the work
progresses.

### Added

- **Tasklist context** — when the URL points at a tasklist, its description is
  extracted and rendered on top of the plan as *"Tasklist context"*, so the
  whole batch of tasks gets framed before per-task planning.
- **Attachments handling** — task files (and comment files) are downloaded into
  `./teamwork-task-<TASK_ID>/` and `./teamwork-task-<TASK_ID>/comments/`.
  Text-based files are read inline by Claude for context; binaries are listed
  in the plan. The folder is auto-cleaned after the time log succeeds.
- **File comments fetch** — comments attached to file objects (not the task)
  are fetched via `GET /projects/api/v3/files/{fileId}/comments.json` and
  surfaced in the per-task plan section.
- **`.gitignore` auto-update** — first run in a repo appends
  `/teamwork-task-*/` to the project's `.gitignore` so downloaded attachments
  never reach a commit. The skill asks whether to commit that `.gitignore`
  change before starting work.
- **Smart auto-commit gate (`auto_commit_mode`)** — before committing, the
  skill inspects the diff. UI/template/styling files (`.vue`, `.tsx`,
  `.blade.php`, `.css`, …) or diffs larger than `auto_commit_max_diff_lines`
  (default 100) trigger an `AskUserQuestion` prompt to *Approve / Inspect /
  Abort*. Trivial textual changes commit and log straight through.
- **Time cursor from Teamwork's last timelog** — at the start of the worker
  loop the skill calls `GET /projects/api/v3/time.json` filtered by the
  current user and `startDate=<today>` (and looks up the user via the V1
  `/me.json` endpoint). Today's most recent timelog's *end* becomes the
  starting cursor (rounded up to the 5-minute grid). If there is no log yet
  today, the cursor starts at the skill's launch time (`floor(now)`). Future
  timestamps and cross-midnight cases fall back to `floor(now)`.
- **Board workflow integration** — when the project has a workflow, the task
  is moved to **"In progress"** at the start of implementation and to
  **"Internal testing"** after the time log succeeds. If "Internal testing"
  doesn't exist, the skill falls back to **"Testing"**. Stage matching is
  case-insensitive. Missing workflows or stages degrade silently with a
  warning in the plan; the task still runs end-to-end.
- **Commit hash in time-log description** — every Teamwork time-log
  description now ends with ` (commit: <short-hash>)` so a stakeholder
  reading the timesheet can jump straight to the change.
- **Board stage column in the final summary** — the run summary table gains
  a *"Board stage"* column showing where each task landed (e.g. *Internal
  testing*, *In progress* for aborted tasks, *—* when board moves are
  disabled).
- **Auto-proposed tests** — when a task description does not specify how the
  work should be verified, the skill sniffs the project (`composer.json` and
  `package.json`) for installed test frameworks (Pest, PHPUnit, Laravel
  Dusk, Vitest, Jest, Playwright, Cypress, Selenium) and proposes concrete
  test files alongside the implementation. Visual changes default to a
  browser test (Dusk on Laravel, Playwright/Cypress on JS), backend changes
  to feature / unit tests (Pest on Laravel, Vitest on JS). The user can opt
  out per task via the description (*"no tests"*, *"bez testov"*), or
  globally via `auto_propose_tests=false`. Missing tools (e.g. visual
  change in a Laravel project without Dusk) trigger an `AskUserQuestion`
  with three options: *Install*, *Skip and write a manual checklist*, or
  *Abort task* — never an unsolicited install.
- **`CHANGELOG.md`** — this file.

### Changed

- **`fetch_comments` → `fetch_comments_mode`** — the boolean is replaced by an
  enum: `always`, `when_needed` (default), `never`. The `when_needed`
  heuristic skips fetching comments when the task already has a clear
  description (HR-split final summary present and acceptance criteria
  ≥ 100 chars), and fetches them when the description is thin or explicitly
  references comments (e.g. *"viď komentár"*, *"see comments"*). The check is
  short-circuited when the task's `commentsCount == 0`.
- **Attachments cleanup default is `after_timelog`** (not `after_commit`) so
  files stay available for debugging if a time log POST fails. Override via
  `attachments_cleanup: "after_commit" | "after_timelog" | "never"`.
- **`plugin.json`** gains an explicit `"version"` field, and the plugin
  description is updated to reflect the broader 1.1.0 capabilities.

### Migration

On the first run after upgrading, the skill rewrites
`~/.claude/plugins/data/teamwork-task-wamesk/config.json` to introduce the new
keys:

- Existing `fetch_comments: true` → `fetch_comments_mode: "always"` (key
  removed).
- Existing `fetch_comments: false` → `fetch_comments_mode: "never"` (key
  removed).
- Missing `auto_commit_mode` → defaults to `"when_safe"`.
- Missing `time_cursor_strategy` → defaults to `"last_teamwork_timelog"`.
- Missing `board_workflow` → defaults to enabled with `"In progress"` /
  `"Internal testing"` (and `"Testing"` as fallback).
- Other new keys (`fetch_attachments`, `fetch_file_comments`,
  `max_attachment_size_mb`, `attachments_cleanup`, `auto_commit_risky_patterns`,
  `auto_commit_max_diff_lines`, `include_commit_hash_in_log_description`,
  `auto_propose_tests`, `test_frameworks`, `test_visual_file_patterns`,
  `test_opt_out_keywords`) receive their defaults if absent.

The migration is idempotent — subsequent runs are no-ops. No manual action is
required; the API token and base URL are preserved unchanged.

---

## [1.0.0] - 2026-05-25

Initial release.

### Added

- Fetch Teamwork tasks via REST API v3 from a single-task or tasklist URL.
- First-run setup flow that prompts for the workspace base URL and an API
  token, storing them at chmod 0600 outside the plugin cache
  (`~/.claude/plugins/data/teamwork-task-wamesk/config.json`).
- Plan modes (`overview` / `per_task` / `none`) with approval gates before
  any code is touched.
- Per-task implementation loop with optional Pint formatting for PHP repos.
- Per-task git commit using the `TYPE(scope)[<task-id>]: Message` convention,
  with `Co-Authored-By` lines deliberately suppressed.
- Sequential, non-overlapping time logs aligned to a configurable rounding
  step (default 5 minutes). The session cursor starts only after the plan is
  approved, so planning time is never billed.
- Task description parsing that splits on the first horizontal rule into
  *acceptance criteria* (above) and *final summary* (below — treated as the
  authoritative goal).
- Comment fetching per task to surface decision history and clarifications.
- Configurable branching strategy (`current_branch` or
  `new_feature_branch`) and CLI overrides (`--time-mode`, `--branching`,
  `--plan-mode`).
- Final summary table with task ID, title, minutes, commit hash, and
  Teamwork completion status.

[1.1.0]: https://github.com/wamesk/claude-code/releases/tag/teamwork-task-1.1.0
[1.0.0]: https://github.com/wamesk/claude-code/releases/tag/teamwork-task-1.0.0
