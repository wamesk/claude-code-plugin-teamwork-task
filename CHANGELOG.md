# Changelog

All notable changes to the `teamwork-task` plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.2.0] - 2026-05-29

**End-of-run worktree housekeeping + sub-rounding timelog fallback.**
Background sessions, the `EnterWorktree` tool, and the `/teamwork-task`
skill itself all create git worktrees inside `.claude/worktrees/<name>` —
but nothing in the existing toolchain ever deletes them automatically
(the only auto-cleanup is the `EnterWorktree` no-change case). Without an
explicit cleanup step, every successful background run leaves another
worktree behind. Step 10 closes that loop. The release also fixes a
fast-run footgun where the future-timestamp guard skipped *every* timelog
of a fast model run because the headroom never reached the 5-minute
rounding step — now a `min_log_minutes`-aware sub-rounding fallback lets
the skill write smaller entries instead of silently losing the record.

### Added

- **Step 10 — Worktree cleanup (end-of-run housekeeping).** After Step 9's
  push reminder, the skill enumerates all worktrees of the current
  repository (`git worktree list --porcelain`), classifies each one
  (`main`, `current`, `merged-clean`, `unmerged-clean`, `dirty`, `ghost`),
  and offers to remove the safe ones via **AskUserQuestion**. Removal is
  two-step: `git worktree remove <path>` then `git branch -d <branch>` (or
  `-D` after explicit confirmation when the branch has unmerged commits).
  The current session's own worktree is always excluded. Failures are
  non-blocking and reported in the Step 7 final summary.
- **Config block `worktree_cleanup`** with five tunables: `enabled`
  (default `true`), `auto_remove_merged_clean` (default `false` — always
  ask), `stale_age_days` (default `14`, used to highlight old worktrees in
  the question), `ignore_paths` (default `[]`), and `report_when_empty`
  (default `false`).
- **CLI flag `--worktree-cleanup=true|false|ask`** for per-run override.
- **Sub-rounding timelog fallback (Step 6.6.1).** When the real-time
  headroom between the session cursor and `now()` is smaller than
  `time_rounding_minutes` (default 5) but at least `min_log_minutes`
  (default 1), the skill writes a smaller-than-round entry instead of
  skipping the task entirely. The cursor still advances by the exact
  logged amount, so the next log resumes seamlessly from where this one
  ended. Pre-1.2.0 behaviour is recovered by setting `min_log_minutes`
  equal to `time_rounding_minutes`.
- **Config key `min_log_minutes`** (default `1`).
- **Step 7 status block** now surfaces `SUB_ROUND_TIMELOGS` alongside
  `SKIPPED_TIMELOGS` and `CLAMPED_TIMELOGS` so reviewers can spot the
  entries that broke the 5-minute cosmetic alignment.

### Why

Two pain points from real production runs:

1. **Worktree accumulation.** `git worktree list` on a long-lived repo
   after a few weeks of background jobs typically shows 5–10 worktree
   entries. None are cleaned by git or by Claude Code; users must remember
   `git worktree remove`. The skill now does it at the same point it
   already handles per-run housekeeping (board moves, time logs,
   attachment cleanup) — explicitly, transparently, only with user
   confirmation for anything that could lose commits.
2. **Lost timelogs on fast runs.** A run where the model produces work in
   seconds (not minutes) had every `TIMELOG_SKIPPED` because headroom
   never reached 5 min from `floor(now)`. The user then either had to
   manually log each task or re-run the skill later just to catch the
   cursor up. Sub-rounding fallback writes the few-minute entry so the
   record survives even on fast runs.

### Notes

- The classification uses the **default branch** of the repo (resolved
  from `origin/HEAD`, falling back to `main` / `master` / `trunk` /
  current HEAD). Merged-status is determined by `git merge-base
  --is-ancestor`, which correctly handles the `+` prefix that appears on
  branches checked out in other worktrees.
- Dirty worktrees are listed with a warning but never auto-removed — the
  user has to handle them manually (`git stash` or commit-and-push first).
- Sub-rounding logs deliberately break the 5-min start-time alignment for
  that one entry. The sequence guarantee (cursor advances by exactly the
  logged minutes) is preserved, so the next log starts at, e.g., 10:37
  instead of 10:35. Reviewers reading the timesheet see a 2-min entry
  followed by a 10-min entry — clear breadcrumb that something fast
  happened.

---

## [1.1.3] - 2026-05-28

**Ask before assuming.** Two new gates in front of the worker loop close the
"the skill produced code against a synthetic placeholder while the real input
was sitting in the project folder" failure mode that surfaced on a real run
(tasklist [#3335436](https://wame.teamwork.com/app/tasklists/3335436/list) —
Tatra banka bmail import). The fix is two-fold: scan the cwd for unattached
context, and react to the task body's own warnings.

### Added

- **Step 3.10 — Local working-tree discovery.** After the Teamwork attachments
  are fetched (Steps 3.7–3.9), the skill now scans the project root
  (configurable depth + dirs + extensions) for files whose names overlap with
  the tasklist/task keywords (`vzor`, `dnr`, `sample`, `specifikác`, project
  name like `Strečnianska/`, …). Matches are offered to the user via
  **AskUserQuestion** with multi-select; picked files are read inline
  (`pandoc`/`textutil` for `.docx`, `pdftotext` for `.pdf`, `xlsx2csv`/`in2csv`
  for `.xlsx`) and become part of the per-task `context_files` array consumed
  by Step 6.2. Fall-back: even when disabled, runs if any task description
  mentions a filename pattern but no attachment was downloaded.
- **Step 6.0 — Readiness gate (per task).** Before the timer starts for each
  task, the description + comments are scanned against a configurable list of
  "blocked-by-external-input" phrases (`pred začatím vyžiadať`, `bez vzorky
  nemá zmysel`, `prisľúbené`, `⏳`, `waiting for client`, …). On a hit the
  skill stops and asks the user, offering: (1) point me at the input now —
  reuses Step 3.10 results plus free-text; (2) proceed with a synthetic
  placeholder, but prefix the time-log description with `⚠️` and append a
  `Synthetic-Input:` trailer to the commit body for later grep; (3) skip the
  task — no commit, no log, no board move; (4) cancel the run.
- **Plan template now renders two new rows per task** — `Context files
  (local)` (output of Step 3.10) and `Missing inputs` (gating phrases or
  filename hints without a matching file). If any task has a non-empty
  `Missing inputs` row, the plan-approval question grows an *"I have the
  input — let me paste a path"* option.
- **Two new config keys, both default-on:** `local_context_discovery`
  (object with `enabled/max_depth/scan_dirs/extensions/filename_hints/ignore_globs/min_keyword_score/max_files_to_offer`)
  and `readiness_gate` (object with `enabled/patterns/filename_hint_pattern/on_block_default`).
  Defaults match the patterns that produced the original failure so existing
  users get the new behaviour without touching their config.
- **Two new CLI flags:** `--local-discovery=true|false` and
  `--readiness-gate=true|false`.
- **Filename-hint fall-through in Step 3.7** — if a task description mentions
  a filename (regex `[A-Za-z0-9_-]+\.(docx|pdf|xlsx|eml|msg|csv|sql|md|json|txt)`)
  but the task ended with zero downloaded attachments, set `FILENAME_HINT_PRESENT=1`
  for the session so Step 3.10 runs **even if local discovery is disabled in
  config**. Somebody clearly referenced a file; assume it just lives outside
  Teamwork rather than ignoring it.

### Why

A real run on tasklist #3335436 went like this: Task 44740924
("Rozpoznanie obsahu notifikácie z Tatra banky") explicitly said *"Pred
začatím vyžiadať reálny vzor notifikácie od klienta. Bez vzorky nemá zmysel
písať regex."* The skill blew past that sentence and shipped a parser against
a fixture invented from the DNR description. Meanwhile,
`Strečnianska/Bmail o pohybe na ucte - vzor.docx` was sitting one directory
above the cwd — never attached to the task, never noticed by the skill. The
v1.1.2 architecture had no mechanism to either (a) read the gating sentence
or (b) discover the local file. v1.1.3 adds both.

### Side effects on other steps

- **Step 6.2.5** auto-proposed test fixtures must use a `*_synthetic.*`
  suffix when the readiness gate's "proceed anyway" path was taken, so a
  reviewer can grep the fixtures and see immediately which ones are
  filler.
- **Step 6.8** time-log description gets the `⚠️ Implementované so
  syntetickou náhradou` prefix on the same path. The log still
  contributes to the running cursor so subsequent logs stay sequential.
- **SKILL.md frontmatter version was 1.1.1 even after 1.1.2 shipped** —
  this release brings it into sync with `plugin.json` at `1.1.3`.

### Compatibility

- Existing config files automatically gain the new keys via the idempotent
  Step 2.6 migration; nothing the user previously set is touched.
- Users who explicitly want the v1.1.2 silent-barrel-through behaviour can
  set `"local_context_discovery": {"enabled": false}` and
  `"readiness_gate": {"enabled": false}`, or pass both `--local-discovery=false
  --readiness-gate=false` on a single run.

---

## [1.1.2] - 2026-05-27

Future-timestamp guard for the session time cursor. On a fast run the model
produces work much faster than wall-clock time, so the cursor (which only
advances by *logged* minutes) drifts ahead of `now()` and the next POST writes
a timelog with a start/end in the future. Teamwork accepts those entries
silently, but the resulting timesheet is useless for billing.

### Added

- **Step 6.6.1 "Future-timestamp guard"** in `SKILL.md` — before each timelog
  POST, the skill now caps `DURATION_MIN` to the remaining `HEADROOM_MIN`
  (= floor distance from the cursor to `now()` in `ROUND` increments). If the
  cursor has already caught up with `now()`, the POST is skipped, the cursor is
  not advanced, and the board move to *Internal testing* is also skipped.
- **`SKIPPED_TIMELOGS` and `CLAMPED_TIMELOGS` accounting arrays** rendered in
  the Step 7 final summary, with a short paragraph telling the user why those
  entries did not land (so they can re-run later or fill the minutes in
  manually).

### Why

The Step 5.5 *no-future-timestamps* safeguard only ran at cursor
initialization — it did nothing about subsequent drift. Documented user-visible
bug: an 8-task run starting at 12:00 logged its last entry ending at 16:00
while the wall clock was at 13:33. The guard closes that gap at the source.

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
