# Changelog

All notable changes to the `teamwork-task` plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.3.0] - 2026-05-29

**Tasklist filter: only "To Do" + me.** A single Teamwork project commonly
holds tasks for two (or more) repositories and two (or more) people — a
typical setup is a Laravel backend in one repo plus an Ionic / iOS frontend
in another, each owned by a different developer. Up to v1.2.0, running
`/teamwork-task <tasklist-url>` from the backend repo happily started
implementing the frontend developer's tasks in the wrong codebase, with no
warning. v1.3.0 closes that loop with an explicit board-column + assignee
filter that runs automatically for tasklist URLs and gets out of the way for
single-task URLs.

### Added

- **Step 3.45 — Tasklist filter (To Do + me).** For tasklist URLs only, the
  skill fetches each task's current board stage (via the
  `?include=cards,stages` shape on the tasklists endpoint) and current
  assignees, then marks every task with one of:
  - `process` — stage matches `tasklist_filter.todo_stage` (default `To Do`,
    **case-sensitive** by default) AND assignees contain the current user.
    The worker loop runs as in v1.2.x.
  - `analyse_only` — fetched and rendered in the plan with a one-line
    "Quick read" opinion, but the worker loop's Step 6.-1 gate skips
    every mutation (no commit, no time log, no board move).
  - `drop` — only when `analyze_all_tasks=false`; the task is removed from
    the plan entirely.
- **Step 6.-1 — Process-mode gate.** Runs before the readiness gate.
  Analyse-only tasks `continue` past the worker loop without starting a
  timer, writing a commit, posting a time log, or moving the board card.
- **Step 2.7 — Resolve current user (cached for the whole run).** The
  `/me.json` lookup that Step 5.5 used to do lazily for the time cursor is
  now executed up front and cached so Step 3.45 can use it too. Empty
  result is non-fatal — assignee check falls back to "skip" so a permission
  glitch never silently locks the user out of their own tasklist.
- **Config block `tasklist_filter`** with six tunables:
  - `enabled` (default `true`).
  - `todo_stage` (default `"To Do"`).
  - `todo_stage_match_mode` (default `"case_sensitive"`; supports
    `"case_insensitive"` for teams that mix casings).
  - `only_assigned_to_me` (default `true`).
  - `analyze_all_tasks` (default `true` — show teammates' tasks in the plan
    as analyse-only; set `false` for strict mode where they are dropped).
  - `skip_reason_render` (default `"inline"` — render the skip reason next
    to the task in the plan).
- **Three CLI flags** for per-run overrides: `--tasklist-filter=true|false`,
  `--tasklist-todo-stage=<name>`, `--tasklist-only-mine=true|false`.
- **Step 4 plan template — two-section layout.** Tasklist plans now split
  into *"To implement"* and *"Analyse only — not implemented in this run"*.
  Each analyse-only entry shows the skip reason (e.g.
  `wrong_stage(In progress) + wrong_assignee`) and a 1–2 sentence "Quick
  read" sanity-check.
- **Two new plan-approval options:** *Promote an analyse-only task to
  implement* and *Disable the tasklist filter for this run* — both
  re-render the plan in place without forcing a re-fetch.
- **Step 7 final summary** now lists `Tasklist filter` activity and
  `Analyse-only tasks (not touched)` so the run report is honest about
  what was and was not implemented.
- **Step 3.45 empty-result message + Step 4.0a short-circuit prompt** —
  when the tasklist filter ends up with `TF_COUNT_PROCESS == 0` (no
  tasks survived the "To Do" + me check), the skill no longer renders
  the normal six-option plan-approval question. Instead it prints a
  detailed explanation of *which filter rules ran*, *per-task reasons*
  for the top 10 skipped tasks, and *most common causes* (wrong stage
  name, casing mismatch, all tasks assigned to teammates), then asks a
  focused prompt: *Disable the filter for this run* / *Pick tasks from
  the analyse-only list to implement* / *Change the required stage
  name* / *Toggle the assignee check off* / *Cancel*. Solves the
  "skill silently did nothing" confusion when the user assumes the
  defaults match their team's column naming.
- **`round_threshold_minutes` config key** (default `4`) — splits the
  Step 6.6 rounding decision into two zones so short runs are billed
  honestly in *both directions* (no padding short tasks up to 5, no
  systematic over-billing of every long task to the next step). Elapsed
  below the threshold logs as raw minutes (1, 2, 3); elapsed at or above
  the threshold rounds to the **nearest** `time_rounding_minutes` step
  using half-up integer rounding:
  ```
  4 → 5    7 → 5     11 → 10    13 → 15
  5 → 5    8 → 10    12 → 10    14 → 15
  6 → 5    9 → 10
  ```
  Set the threshold equal to `time_rounding_minutes` to recover the
  pre-1.3.0 "always round up to ROUND" behaviour.
- **Step 6.6 rewritten** to honour the new threshold — `DURATION_SOURCE`
  is now exposed (`rounded_nearest`, `sub_round_elapsed`,
  `floored_min_log`) and the sub-round path registers
  `SUB_ROUND_TIMELOGS` with a reason ("elapsed below ${THRESHOLD}m
  threshold") so Step 7 can explain why an entry is below 5 min.
- **Step 6.6.1 simplified** — the clamp no longer re-rounds to ROUND. It
  just clamps `DURATION_MIN` down to raw headroom when an overshoot is
  about to happen, then flags the entry as sub-round with reason
  "clamped by headroom guard". A round-up entry whose headroom is 7 min
  used to be clamped to 5 (a 2-min loss); v1.3.0 logs the full 7 min and
  the cursor stays accurate.
- **Step 9.5 — Worktree handoff (merge / push / leave).** When the skill
  runs inside a git worktree (typical for background jobs launched via
  `EnterWorktree` or sessions started from `.claude/worktrees/<name>`),
  every commit lands on the worktree's branch and never reaches `main`
  on its own. Up to v1.2.0 the user had to remember to merge them by
  hand; Step 10 cleanup would happily remove the worktree's parent
  directory but never touched the merge. v1.3.0 closes that loop by
  asking, at the end of the run, what to do with the new commits:
  - *Merge into a target branch* (default — fast-forward if possible,
    fall back to merge commit; target asked separately with parent /
    main / custom options),
  - *Push the worktree branch to `origin` for a PR*,
  - *Leave as-is*,
  - *Cherry-pick specific commits* (power option).
  On a successful merge, both the worktree branch and the worktree
  directory are removed by default, which chains into Step 10 cleanup
  without re-asking.
- **Step 5.1 — Detect worktree mode + cache parent branch.** New early
  detection step that resolves `WT_RUN_IN_WORKTREE`, `WT_HEAD_BEFORE`
  (HEAD before the worker loop, so Step 9.5 can compute "commits made
  this run" only), and `WT_PARENT_BRANCH` (origin/HEAD with main /
  master / trunk fallback). All cheap, no API calls.
- **Config block `worktree_handoff`** with eight tunables: `enabled`
  (default `true`), `default_action` (default `ask`; supports `merge`,
  `push`, `leave`), `default_target` (default `ask`; supports `parent`,
  `main`, `<branch>`), `merge_strategy` (default `ff_else_merge`;
  supports `ff_only`, `no_ff`, `squash`), `delete_branch_after_merge`
  (default `true`), `delete_worktree_after_merge` (default `true`),
  `push_remote` (default `"origin"`), `skip_if_no_commits` (default
  `true` — do not even ask when nothing was committed this run).
- **Two new CLI flags:** `--worktree-handoff=ask|merge|push|leave` and
  `--worktree-target=ask|parent|main|<branch>`.
- **Step 10.2 deduplication** — if Step 9.5 already merged and removed
  the current worktree, Step 10 filters it out of its discovery so the
  user is not asked about the same worktree twice.

### Why

A real run on a multi-repo Teamwork project surfaced the failure: a
developer running the skill from the Laravel repo expected only "their"
backend tasks to be implemented. The skill instead grabbed every task in
the tasklist, including the frontend ones marked for the iOS engineer,
moved them to *In progress* on the board (visible to the whole team), and
started writing PHP code for an iOS task description. The implementation
phase eventually failed at the safety gate (the diff did not match the
description), but by that point the board state had been wrongly mutated
and the time logs were wasted. v1.3.0 narrows the implementation set to
what the developer actually owns and is greenlit to work on, while still
showing teammates' tasks for context so the developer can comment on them
in standup.

The `round_threshold_minutes` addition came from the opposite end of the
honest-billing problem: v1.2.0 always rounded **up** to ≥ 5 min, which
was wrong on both ends — a 30-second typo fix got logged as 5 minutes
(embarrassing for a trivial commit), and an 11-minute hotfix got logged
as 15 (a 36 % over-bill). The split-zone logic with round-**to-nearest**
makes both ends honest: trivial work logs as 1–3 min raw; substantial
work rounds in either direction (7 → 5, 8 → 10, 12 → 10, 13 → 15) so the
billing grid averages out over many tasks instead of biasing upward
every time.

The Step 9.5 worktree handoff comes from a recurring pain point: a
background run completes, commits land in `.claude/worktrees/<name>` on a
branch like `claude/<task>` — and **none of it reaches `main`**. The user
sees a clean Step 7 summary, walks away thinking the work is done, and
discovers a week later (when reviewing `git log` on `main`) that the
changes never made it home. The fix is structural: the skill that produced
the commits is the right place to also place them in their final home, or
to explicitly hand the decision back to the user before walking away. The
end-of-run timing was chosen deliberately — by Step 9.5 the user has
already seen the implementation summary and the test verification result
in Step 7 + Step 8, so they have full context to decide whether the
commits are ready for `main`, ready for a PR, or need further work before
either.

### Why case-sensitive "To Do"

The existing board_workflow stage matcher is case-insensitive on purpose —
"In progress" and "INTERNAL TESTING" are typical real-world variants and
matching loosely keeps the configuration low-friction. The tasklist
*filter* has the opposite stakes: a loose match would also accept
"In progress" tasks (because both start with "I" / share two letters under
some matchers) or, worse, accidentally accept a column literally named
"Todo" that has a different meaning on a freeform Kanban board. Strict
match means the user explicitly approves the column name they expect, and
the default `"To Do"` matches the literal Teamwork column the user pointed
at in the brief that drove this release. Users who want loose matching can
flip `todo_stage_match_mode` to `case_insensitive`.

### Why single-task URLs bypass

When the user passes `/teamwork-task https://…/tasks/12345`, they have
deliberately pointed at one task — typically because they want to debug it,
re-run it after a fix, or run a task that lives outside the normal "To Do"
ramp. Forcing the same filter there would mean refusing to run on a task
the user is staring at on the board. Single-task URLs are an explicit user
intent; honour it.

### Notes

- The filter assumes the project has a workflow attached. If the project
  has none (Step 3.3 marks it `BOARD_MOVE_DISABLED`), the filter falls
  back to "process everything" rather than dropping the whole tasklist,
  with a one-line note in the plan.
- Tasks that have no card (added before a workflow was attached) fall back
  to `process` as well, so the filter never silently strands a backlog
  task that was never put on the board.
- `analyse_only` tasks still go through Step 3.5/3.7/3.10 fetches because
  the analysis output should be informed by comments + attachments. Only
  the mutation steps (6.1.5 in-progress move, 6.2 implementation, 6.7
  commit, 6.8 timelog, 6.8.5 done move, 6.9 cleanup, 6.10 task complete)
  are gated. This is the right trade-off — the extra fetches are cheap and
  the user gets a better "Quick read" line for analyse-only tasks.

### Compatibility

- Fully backward compatible for **single-task URLs** — they always bypass
  the filter, so `/teamwork-task https://…/tasks/12345` behaves exactly as
  in v1.2.x.
- For **tasklist URLs** the default behaviour changes: tasks that are not
  in `To Do` + assigned to the current user end up `analyse_only`. Users
  who want the v1.2.x "implement everything I'm given" behaviour can:
  - Pass `--tasklist-filter=false` for a single run, or
  - Set `"tasklist_filter": {"enabled": false}` in
    `~/.claude/plugins/data/teamwork-task-wamesk/config.json` to disable it
    persistently.
- Existing config files automatically gain the `tasklist_filter` block via
  the idempotent Step 2.6 migration; nothing the user previously set is
  touched.

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
