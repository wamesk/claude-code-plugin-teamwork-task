# Changelog

All notable changes to the `teamwork-task` plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.7.0] - 2026-10-07

The companion plugin `wame-work-mode` 1.0.0 was renamed to
[`work-mode`](https://github.com/wamesk/claude-code-plugin-work-mode) 2.0.0
(decided 2026-10-07), and its modes and files were renamed with it. 1.7.0
follows the new names, drops the `mode` key from the shared config — the
global default moved to Claude Code `/config` — and keeps reading the old
names for one version so nothing breaks mid-feature.

### Changed

- **Modes renamed: `build` → `fast`, `harden` → `full`** (`full` stays the
  default). `--mode=fast|full` in the argument hint, Step 2.65 and every step
  that branches on the mode (6.2, 6.2.5, 6.4, 6.5, 6.7, 7, 8). The Step 8
  handoff now forwards `--mode=full` to `/teamwork-task-test`.
- **Commands renamed in the companion plugin.** `/wame-mode build|harden|status`
  is now `/work-mode fast|full|status` (no argument opens a menu);
  `/wame-harden` is now `/work-mode full`, which runs the pending deferred
  checks once and then switches to `full` (`/work-mode full --no-checks` only
  switches). All references point to the new commands.
- **Project files renamed.** The mode file is `.claude/work-mode.local.md`
  (frontmatter `mode: fast|full`), the deferred list
  `.claude/work-mode-deferred.local.md` (same block format as before).
- **Mode resolution (Step 2.65):** `--mode=` > `.claude/work-mode.local.md` >
  legacy `.claude/wame-mode.local.md` (`build` → `fast`, `harden` → `full`) >
  `full`. There is no config fallback any more: the global default is the
  `work-mode` plugin option `default_mode` in `/config`, which that plugin's
  SessionStart hook writes into the project's mode file, so this skill only
  reads the project file.
- **The deferred list is never staged.** Step 6.7 stages explicit paths only
  and runs `git reset -q --` on both deferred-list names and both mode-file
  names before every commit, in every mode; the block is appended after the
  commit; a list that is not git-ignored prints one `⚠` line
  (`git check-ignore`).

### Removed

- **The `mode` config key.** The Step 2.6 migration drops
  `(.mode //= "harden")` and runs an idempotent `del(.mode)` instead;
  `config.example.json` no longer carries the key. If you had set it to
  `build`, set the `work-mode` plugin's `default_mode` to `fast` in `/config`.

### Deprecated

- **The pre-rename names, read for one version only.** `--mode=build` /
  `--mode=harden` map to `fast` / `full` and print one `⚠` line; the legacy
  `.claude/wame-mode.local.md` is read when the new mode file is absent; when
  only the legacy `.claude/wame-deferred.local.md` exists, it is `mv`-ed to
  `.claude/work-mode-deferred.local.md` before the first append, so no
  deferred entry is lost.

---

## [1.6.0] - 2026-10-06

Most of a run went to verification instead of building: the five quality
dimensions, a proposed test per task, a filtered test run and Pint per task,
browser checks and the whole `/teamwork-task-test` pass — on every task. Models
also improvised installing and uninstalling Playwright / Puppeteer for a single
check. 1.6.0 adds a **build** work mode that defers all of it to one explicit
`/wame-harden` pass, and a fixed rule for browser tooling.

### Added

- **`--mode=build|harden` and the `mode` config key** (default `"harden"`,
  added by the Step 2.6 migration). New Step 2.65 resolves the mode — `--mode`
  > the project's `.claude/wame-mode.local.md` frontmatter `mode:` (written by
  `/wame-mode`, plugin `wame-work-mode`) > `config.mode` > `harden`. `build`
  maps onto the existing switches at once, in memory only:
  `build_quality.dimensions: []` (= `--dimensions=none`),
  `auto_run_tests_after: false` (= `--test-after=false`),
  `auto_propose_tests: false`, no per-task test run (6.4) or Pint (6.5), no
  browser work and no docs / version lookups. An explicit individual flag
  still wins over the mode. Fetching, the readiness gate, plan approval, board
  moves, commits and time logs are unchanged.
- **Deferred list.** In `build` mode each task appends its commit, files,
  screens and skipped checks to `.claude/wame-deferred.local.md` right after
  Step 6.7 (never staged); Step 7 prints a *Work mode* line pointing to
  `/wame-harden`.

### Changed

- **Step 8 checks the work mode first.** In `build` mode the
  `/teamwork-task-test` handoff is skipped unless `--test-after=true` is
  passed, in which case `--mode=harden` is forwarded so QA does not inherit
  the project's build mode.
- **Browser tooling is never installed on the fly.** Step 6.2.5's *"Install
  `<package>` now"* became *"Add `<package>` to the project permanently"*:
  asked once per run, installed as a committed dev dependency and never removed
  afterwards; the chrome-devtools MCP or the project's own runner is preferred.
  `npx playwright install` no longer passes `--with-deps`.

---

## [1.5.0] - 2026-09-24

The skill could report *"task has no comments"* on a task with eight — and
carry on with empty context, indistinguishable from a correct run. Four
independent defects each produced that silent result (reported in a
colleague's *"Štyri cesty k nule"* analysis). The same classes of bug also
emptied the subtask expansion, the board workflow, the tasklist filter and
the attachment list. All were confirmed against the live Teamwork API with
read-only requests and are fixed below; on top of that the skill now builds
every task against the five quality dimensions `teamwork-task-test` checks
at QA time — the fifth, `framework`, asks for the current idioms of the
framework versions the project actually has installed — and follows the new
WAME board: work starts in *Ready for Development* or *To Do* and finished
work lands in *Done - Local*.

### Fixed

- **Comments endpoint answered HTTP 400 and the error was read as "no
  comments".** Step 3.5 sorted with `orderBy=` a key v3 does not know; the
  endpoint answers `400 {"detail":"orderBy: unknown comment sort."}`, the
  status was never checked and the error body parsed as an empty list. Now
  `orderBy=date` (chronological) with an explicit HTTP check, pagination while
  `.meta.page.hasMore`, and a v1 fallback (`/tasks/{id}/comments.json`, sorted
  by `datetime`). Repro: `GET /projects/api/v3/tasks/45198800/comments.json`
  with the old sort key → 400; with `orderBy=date` → 8 comments.
- **The `when_needed` comment gate never opened.** It required
  `task.commentsCount > 0`, a field Teamwork v3 never returns, so with the
  default mode comments were never fetched. The count now comes from a one-item
  probe (`pageSize=1&orderBy=date&orderMode=desc` → `.meta.page.count`, v1
  fallback `."todo-item"."comments-count"`). Repro: `jq .task.commentsCount` on
  `GET /projects/api/v3/tasks/45198800.json` → `null`, while the task has 8
  comments.
- **Comment timestamps and bodies were read from fields that do not exist.**
  The skill documented the comment time under a field name v3 does not have,
  so chronological order and "the last comment wins" could not work. Now
  `postedDateTime`, `htmlBody` / `body`, `postedByUserId`, and `files[]`
  resolved through `?include=files`; comments are normalized (v1 mapped onto
  the v3 names) into one chronological file per task. Repro: the first
  comment of task 45198800 is `2026-08-14T09:32:28Z` in `postedDateTime`; the
  old field yields `null` for all 8.
- **`echo "$JSON" | jq` broke in zsh — the systemic one.** Claude Code runs
  every snippet in the user's login shell (zsh on macOS), whose builtin `echo`
  expands `\n`, `\\` inside the JSON; jq then rejects the response
  (`control characters from U+0000 through U+001F must be escaped`) and
  `2>/dev/null || echo 0` turned that into zero subtasks (Step 3.42), no
  workflow stages (Step 3.3), a broken tasklist filter (Step 3.45), no files
  (Steps 3.7 / 3.8) and no timelog cursor (Step 5.5). Every such pipe is now a
  here-string (`jq … <<<"$VAR"`) or `printf '%s\n'`, and every parse / HTTP
  failure prints a `⚠` line naming the endpoint and the fallback. Repro: the
  subtask list of task 45092249 contains `\\\n` in a description — the 1.4.2
  idiom yields `SUB_COUNT=0` in zsh, the here-string yields 5.
- **`projectId` was always null, so board moves never happened.** v3 task
  objects carry no `projectId` key; the project lives in
  `.tasklist.meta.projectId`. Step 3.3 therefore fetched workflows for an
  empty project, and the board moves (Steps 6.1.5 / 6.8.5) were always
  skipped. Now `.projectId // .tasklist.meta.projectId` (v1 fallback
  `."todo-item"."project-id"`) for tasks **and** subtasks, persisted to a
  `taskId → projectId` table. Repro: `jq .task.projectId` on task 45198800 →
  `null`; `.task.tasklist.meta.projectId` → `700336`.
- **Tasklist filter treated every task as having no column.** Step 3.45 read
  the column from `?include=cards,stages`, which returns an empty `.included`
  (and `cardId: null`) on the tasks endpoints, so every task looked as if it
  were off the board and the filter could not tell a *To Do* task from any
  other. The column now comes from each task's `workflowStages[0].stageId`
  resolved through the project's stage table, empty fields no longer shift
  TSV columns, and expanded parents leave the filter set while their subtasks
  join it. A task that *is* on a board but whose column cannot be named (the
  Step 3.3 workflows GET failed, e.g. on HTTP 429) fails closed as
  `analyse_only` with `stage_unresolved(<stageId>)` and a `⚠` line, instead
  of passing the stage check. (What happens to a task that is genuinely off
  the board is a deliberate change — see *Changed*.) Repro (tasklist 3361804,
  85 tasks): the 1.4.2 loader dies on jq in zsh and the filter sees 0 of 85
  tasks; in bash all 85 load with an empty column. Now each task carries its
  real column (`To Do`, `Next Sprint`, …).
- **An explicit `false` in the config was ignored — and rewritten.** Booleans
  were read as `.x // true`, and jq's `//` treats `false` like a missing
  value, so `"tasklist_filter": {"enabled": false}` (the documented way to
  switch the filter off), `only_assigned_to_me`, `analyze_all_tasks`,
  `board_workflow.enabled`, `subtasks.enabled`, `readiness_gate.enabled`,
  `worktree_cleanup.enabled` and the `worktree_handoff` switches all read back
  as `true`. Worse, the Step 2.6 migration's `(.x //= true)` rewrote
  `skip_completed_tasks`, `is_billable_by_default`, `fetch_attachments`,
  `fetch_file_comments`, `auto_propose_tests`, `auto_run_tests_after` and
  `include_commit_hash_in_log_description` from `false` back to `true` on
  every run — a user who switched billable time off got billable timelogs
  again. Boolean defaults now apply only to a missing / `null` value
  (`if . == null then true else . end`), and the rule is part of the
  portability contract. Repro: `jq -n '{"a": false} | .a // true'` → `true`;
  on tasklist 3367748 with `only_assigned_to_me: false` Step 3.45 still
  dropped every teammate's task as `wrong_assignee` before the fix and keeps
  the two *To Do* tasks after it.
- **Task attachments were fetched from a 404 endpoint and a fallback that
  returned the whole workspace.** `/projects/api/v3/tasks/{id}/files.json`
  answers 404; the `/projects/api/v3/files.json?taskIds=` fallback ignores the
  filter (100 of 19 042 workspace files per page) and would have downloaded
  unrelated files; `/files/{id}/download` is a 404 as well. Now
  `GET /tasks/{id}.json?include=attachments` → `.included.files` with
  `downloadURL` (or `/projects/api/v3/files/{id}.json`). Repro: task 45229457
  → 2 attachments, both downloaded.
- **The Step 3.42 subtask fallback asked a filter v3 ignores.** When
  `/tasks/{id}/subtasks.json` failed, the fallback called
  `tasks.json?parentTaskIds=<id>`; v3 silently ignores the plural parameter and
  returns the first 100 tasks of the whole site, which the client-side
  `parentTaskId` filter then reduced to "0 subtasks" — the parent stayed a leaf.
  The fallback now uses the singular `parentTaskId=<id>` (the same fix
  `teamwork-task-analyze` 1.3.0 ships); the client-side filter stays as a guard.
  Repro: `tasks.json?parentTaskIds=45379220` → 100 rows from 7 unrelated
  parents, `hasMore: true`; `tasks.json?parentTaskId=45379220` → its 8 subtasks.
- **Acceptance criteria in the canonical WAME format were read from the wrong
  block.** Step 3.6 split the description on the first HR, but the format the
  sibling plugins write (`[preamble] → HR → Akceptačné kritériá → HR → Cieľ → …`)
  puts the reporter's preamble — or nothing, when the description starts with
  the HR — above it, so the checklist Step 6 verified against was the preamble
  and the real criteria, including the new `### Prierezové požiadavky` block,
  were treated as summary. When an `## Akceptačné kritériá` /
  `## Acceptance criteria` heading exists, the criteria are now the block from
  that heading to the next HR (the same block `teamwork-task-test` ticks), and
  Step 6.2 treats a cross-cutting block there as binding; other descriptions
  keep the first-HR split.
- **zsh loops and arrays.** Unquoted `for E in $EXTS` / `for H in $HINTS`
  (Step 3.10) and `for SHA in $WT_PICKED_SHAS` (Step 9.5.5) iterate once in
  zsh, so local discovery built one `-iname` term that matched nothing and the
  cherry-pick got one argument made of every hash; they are `while read`
  loops now. The per-project `BOARD_MOVE_DISABLED[$PROJECT_ID]` /
  `TODO_STAGE_MISSING_FOR_PROJECT[...]` arrays (a 700 000-slot array in zsh,
  empty in every fresh shell) are replaced by per-project board files; a
  `local` repeated inside the subtask loop (prints `X=value` in zsh) is gone.
  Repro: `zsh -c 'E=$(printf "docx\npdf"); for x in $E; do echo $x; done'`
  prints one line.
- **Cross-step state lived in shell variables.** Every Bash tool call is a
  fresh shell, so `USER_ID`, `WORKFLOW_ID`, the comments, the tasklist JSON and
  the undefined `$CLAUDE_JOB_DIR` (Step 3.10 read `/tasks.json` from the
  filesystem root) were empty in later steps. State now lives in
  `/tmp/tw_job_<ENTITY_ID>/` (reset at Step 3, together with the per-entity
  Step 3.42 / 3.45 tables, so a stale `analyse_only` verdict cannot survive a
  `--tasklist-filter=false` re-run) and every snippet that reads Teamwork data
  or run state starts with a short run preamble; the Step 3 snippet now writes
  the Step 3.42 working-set file itself. The few worker-loop scalars (time
  cursor, timer start) are still carried by the executor between calls, as
  the portability contract states.
- **Step 8 never detected an installed `teamwork-task-test`.** The glob
  `~/.claude/plugins/*/teamwork-task-test/SKILL.md` does not match the cache
  layout (`cache/<marketplace>/teamwork-task-test/<version>/skills/…`) and is a
  hard `no matches found` error in zsh; it is a `find` now.
- The readiness gate (Step 6.0) formatted four values with three `%s` and
  scanned an empty description in a fresh shell; it now reads the
  description and the normalized comments from the run files.

### Added

- **Build-time quality rules** — the dimensions `teamwork-task-test`
  (Step 6.6) reviews at QA time, applied while building: `ui_ux`,
  `performance`, `security`, `reachability` and `framework` (next entry).
  Step 6.2 decides per task which dimensions the change shape touches (UI
  files → `ui_ux`; queries, migrations, loops → `performance`; new routes /
  actions / endpoints / inputs → `security`; any new screen →
  `reachability`; code in a versioned framework → `framework`) and plans the
  concrete steps — above all, a new screen gets its menu entry **and**
  inbound links from related screens in the same commit. Step 6.3 carries
  short imperative rules per dimension. The new **Step 6.5.5 Quality
  self-check** walks the task's diff before the timer stops, fixes what is
  cheap, and records one status per dimension (`checked` /
  `not_applicable` / `skipped(--dimensions)` / `open(<item> @ <file:line>)`)
  — never claiming a dimension it did not read. Open items appear per task in
  the Step 7 summary, make the Step 6.6.5 safety gate ask (in the default
  `when_safe` mode) when they are `security` / `reachability`, and are handed
  to `/teamwork-task-test` in the new Step 8.2.5. Why: QA used to be the first
  place anyone asked whether a new page was reachable from the menu or a new
  action was policy-checked; asking at build time is cheaper.
- **`framework` — the fifth dimension: best practices of the installed
  versions.** Model memory lags behind the frameworks, so new code tended to
  use dated patterns (or hand-roll what the framework ships) even in a
  project on Laravel 12 / Nova 5 / Tailwind 4. Step 6.2 now detects the real
  versions **once per run** from `composer.lock`, `composer.json`
  (`require.php`, `config.platform.php`), `package.json` + `node_modules` /
  `package-lock.json` / `yarn.lock`, `browserslist` and `.nvmrc` /
  `.node-version` / `engines.node`, caches them in
  `/tmp/tw_job_<id>/framework_versions.tsv` (dropped after a task that changes
  a manifest or lock file), and plans the idiom to use after
  looking the API up in current docs (Laravel Boost `search-docs` → context7
  → official docs). Step 6.3 applies it to **new or changed code only**, within
  guardrails: project `CLAUDE.md` and sibling conventions win over a newer
  idiom, no second pattern next to an established one, no drive-by rewrites,
  nothing deprecated in — or newer than — the installed version, the PHP floor
  or the browserslist target, no new dependency for a built-in. Step 6.5.5
  checks the diff for deprecated APIs, hand-rolled built-ins and too-new
  features (the last one is a bug and is fixed before the commit) and records
  up to three advisory `suggest` rows for opportunities in untouched code (a
  defect there — an N+1, a missing policy — stays an `open` row under its own
  key). `framework` rows never make the safety gate ask. Step 7 prints the detected
  versions and the advisory tips; Step 8.2.5 hands them to
  `/teamwork-task-test` marked advisory (its 1.2.0 only recommends on this
  key and never fails or downgrades an acceptance criterion on it).
- `build_quality.dimensions` config key (default all five) and the
  `--dimensions=<csv>|none` flag, mirroring `teamwork-task-test`. The Step 2.6
  migration merges the key idempotently (an explicit `[]` is preserved) and
  adds `framework` **once** to a list written before the key existed — only
  when that list is exactly the old four-key default (any order); a subset,
  `[]`, or a list with other keys is a user choice and stays untouched. A new
  `build_quality.dimensions_schema: 2` marker records that the check ran, so
  removing `framework` later is never undone. Verified on scratch configs in
  zsh and bash: a 1.4.2 config gets all five keys, a pre-release four-key
  list (in any order) gains `framework`, customised lists do not, and a second
  run changes nothing.
- **Shell portability contract** near the top of SKILL.md — the shell (and
  jq) rules behind the fixes above, in nine bullets, so future edits do not
  regress.
- A *Quality dimensions* line in the Step 4 plan entry and a *Build quality*
  column + *Comments read* / *Open build-quality items* / *Framework versions*
  / *Framework opportunities* lines in the Step 7 summary.

### Changed

- **`fetch_comments_mode: when_needed` always reads the newest comment.** The
  skill's own rule is that the last comment is the freshest truth; skipping it
  on well-described tasks implemented an outdated spec, because a comment can
  change the task after the description was written. The probe that counts
  comments returns the newest one in the same call, so this costs nothing
  extra. The existing heuristics (no final summary, short acceptance criteria,
  *"viď komentár"*) now only decide whether the **full thread** is fetched.
  `always` and `never` are unchanged (`never` skips the probe too). The plan's
  *Comments context* line states exactly what was read.
- Subtask expansion reuses the names / descriptions from the subtasks response
  instead of one extra `GET` per child, calls the `parentTaskId=` fallback only
  when the primary endpoint fails, and does not expand a parent whose subtasks
  are all completed (with `skip_completed_tasks=true`).
- Step 3.42's per-task subtask GETs retry HTTP 429 / 5xx
  (`curl --retry 3 --retry-max-time 120`, body written to a file so retried
  bodies are not concatenated). A big tasklist fires one request per task and
  can hit Teamwork's rate limit — observed while verifying this release — and
  a rate-limited parent must not quietly lose its subtasks; whatever still
  fails after the retries is named in a `⚠` line.
- **Start columns: *Ready for Development* and *To Do*.** The shared WAME
  board (Teamwork workflow *"WAME workflow"*, ~49 projects) greenlights
  work in two columns. The tasklist filter now implements a task whose column
  is **any** of the new `tasklist_filter.todo_stages` (default
  `["Ready for Development", "To Do"]`; the order is only the display order),
  each matched per `todo_stage_match_mode` (case-sensitive by default) — plus
  the assignee rule as before. The legacy string `todo_stage` is still read
  when the list is absent, and `--tasklist-todo-stage` accepts a
  comma-separated list. The plan banner, the empty-result message, the Step
  4.0a prompt (*Change the start columns*) and the final summary name the
  configured columns instead of a hard-coded *To Do*. Older boards without
  *Ready for Development* keep working with *To Do*.
- **A task that is not on the board is analyse-only.** Only the start
  columns are greenlit work, so a backlog item nobody moved onto the board —
  or any task of a project without a workflow — is no longer implemented from
  a tasklist run: the plan lists it as *"not on the board"* (`no_card`) and
  the user can promote it. This replaces the `no_card → process` rule 1.3.0
  documented (it never took effect, because the column lookup was broken —
  see *Fixed*), for tasks and subtasks alike. A column that cannot be named
  because a GET failed stays fail-closed (`stage_unresolved`).
- **Done target: *Done - Local*.** The WAME board ends local work in *Done -
  Local*; `board_workflow.done_stage` now defaults to it, with
  `done_stage_fallbacks` `["Internal testing", "Testing"]`, so the older
  boards keep landing where they did. `in_progress_stage` stays
  `"In progress"` — the case-insensitive match resolves *In Progress* on the
  new board. Plan (*Board target*), Step 6.8.5 and the final summary name the
  resolved column. Verified with GET only: the defaults resolve to *Ready for
  Development* 300980, *To Do* 300903, *In Progress* 300904 and *Done - Local*
  301170 on project 736882 (workflow 59165), and to *To Do* 295925,
  *In progress* 295926 and — via the fallbacks — *Testing* 295928 on project
  700336 (workflow 58171); the Step 3 → 3.3 → 3.45 snippets ran end-to-end on
  tasklists 3367748 and 3361804 in zsh and bash with identical results.
- **Board-column migration (Step 2.6), only from the old defaults.**
  `board_workflow.done_stage` moves to `"Done - Local"` only when it equals
  the old default `"Internal testing"`; the fallbacks then become
  `["Internal testing", …the existing fallbacks minus "Done - Local"…]`
  (deduplicated, `"Testing"` kept). A customised `done_stage` is never
  touched, and a new `board_workflow.done_stage_schema: 2` marker makes the
  check one-shot, so choosing *Internal testing* again later sticks.
  `tasklist_filter.todo_stages` is written once, only when missing: the
  default pair when the legacy `todo_stage` was missing or `"To Do"`,
  `[<custom>]` otherwise; an existing list is never overwritten and the
  legacy key stays for older sibling plugins. The config shape in use at WAME
  today (`done_stage: "Internal testing"`, fallbacks
  `["Done - Local", "Testing"]`, `todo_stage: "To Do"`) becomes
  `done_stage: "Done - Local"`, fallbacks `["Internal testing", "Testing"]`,
  `todo_stages: ["Ready for Development", "To Do"]`. Verified on scratch
  copies in zsh and bash — that shape, a fresh config, the 1.4.2 default,
  customised values, disabled and missing blocks — each unchanged by a second
  run.
- **A completed task behind a single-task URL asks first.**
  `skip_completed_tasks` keeps dropping completed tasks silently from
  tasklist and subtask iteration. A URL the user pasted keeps its task, and
  the new Step 3.35 asks whether to process it — *Skip* (recommended,
  default) or *Process anyway* — naming the task's current column and warning
  that processing moves the card out of the done column (to *In progress*,
  then to the done target) and logs time on a completed task. Why: pasting a
  finished task is as often a mistake as a rework request, and doing either
  silently is wrong half the time.
- Config migration banner reads "1.5.0 schema".

---

## [1.4.2] - 2026-06-11

### Fixed

- **Subtask expansion (Step 3.42) no longer silently no-ops.** The 1.4.0 / 1.4.1
  builds shipped the expansion driver loop fed by an empty process substitution
  (`done < <( true )`), with the working-set enumeration left to the executor as
  a "semantic, executed by the skill" step. In practice the loop never ran:
  `expand_subtasks` was never invoked, so a parent task's subtasks were never
  detected and the **parent itself** received the commit, board move and time
  log instead of its subtasks. The loop now reads a concrete `WORKING_SET_FILE`
  that the skill must materialize first (one `taskId<TAB>name<TAB>description`
  row per Step 3 task), guarded by an empty-file warning, and a new **MANDATORY**
  callout makes both the working-set write and the "`GET /tasks/{id}/subtasks.json`,
  never trust `subTasksCount`" detection rule explicit and non-optional.
- **`parentTaskIds` fallback no longer absorbs the whole project.** Some Teamwork
  instances ignore the `parentTaskIds` query filter and return *every* task in
  the project; the fallback now filters client-side to children whose
  `parentTaskId` actually equals the parent task, so a genuine leaf task can no
  longer pick up unrelated project tasks as bogus subtasks.
- **Step 1 URL parser no longer fails on macOS/BSD `sed`.** The `ENTITY_ID` and
  `URL_KIND` substitutions used `|` as both the `s|||` delimiter and the regex
  alternation operator inside `(tasks|tasklists)`; BSD `sed` (the typical macOS
  developer platform) reads the first inner `|` as the closing delimiter and
  aborts with `RE error: parentheses not balanced`, returning an **empty** id
  and kind for every URL — Step 1 then cannot identify the task at all. The
  delimiter is now `#`, which does not clash with the alternation.

### Changed

- **Removed the `version` field from the SKILL.md frontmatter** — `plugin.json`
  is now the single source of truth for the version. The frontmatter value had
  silently gone stale at `1.3.0` (two minor versions behind), the same class of
  bug recorded back at 1.1.3; dropping the field removes the footgun entirely.
  The runtime migration banner string was also updated from `1.3.0 schema` to
  `1.4.2 schema` (Step 2.6).
- Added `--subtasks=true|false` to the SKILL.md `argument-hint` — the per-run
  toggle was documented in the CHANGELOG / README but missing from the hint.

---

## [1.4.1] - 2026-06-10

### Fixed

- **Time logs no longer land hours early / overlap (timezone bug).** `parse_iso`
  parsed the UTC `timeLogged` returned by Teamwork with macOS BSD `date -j -f
  "…Z"` **without `-u`**, so the trailing `Z` was treated as a literal and the
  wall-clock was read as **local** time — shifting the session cursor by the
  local UTC offset and producing timelogs that started hours before the real
  work and overlapped existing entries. The BSD `Z` branch now uses `date -ju`.
- Documented the **TIMEZONE CONTRACT** on `parse_iso` (Step 5.5) and at the POST
  `time` formatting (Step 6.8): Teamwork **returns** `timeLogged` in UTC but
  **interprets** the POST/PATCH `time` field in the user's local/profile
  timezone — so the cursor is parsed as UTC (`date -ju`) and the POST `time` is
  formatted as local (`date -r`, no `-u`). The two must never be mixed.

---

## [1.4.0] - 2026-06-03

**Subtasks are first-class tasks.** A common Teamwork pattern is a "container"
parent task with N subtasks underneath — each subtask has its own
description, acceptance criteria, comments, attachments, assignee, and board
stage. Up to v1.3.0 the skill ignored subtasks entirely: it planned, committed
and time-logged against the empty parent, leaving the actual N units of work
invisible. v1.4.0 closes that loop. After the initial fetch, every parent
task with subtasks is **expanded** in the working set — the parent steps
aside (no board move, no commit, no time log), and each subtask runs through
the full pipeline as if it were a standalone task.

### Added

- **Step 3.42 — Expand subtasks.** Runs between Step 3.4 (tasklist context)
  and Step 3.45 (tasklist filter). For each fetched task it calls
  `GET /projects/api/v3/tasks/{id}/subtasks.json?include=cards,stages`
  (fallback: `GET /tasks.json?parentTaskIds={id}`) and, when subtasks are
  returned, replaces the parent in the working set with its subtasks. Each
  subtask carries forward through Steps 3.5 (comments), 3.6 (description
  split), 3.7 (attachments), 3.9 (file comments), 3.10 (local discovery),
  3.45 (tasklist filter), 4 (plan), 5 (timer), 6 (worker loop), 7 (final
  summary) exactly like a top-level task — per-subtask attachment folder
  (`./teamwork-task-{subtaskId}/`), per-subtask commit
  (`TYPE(scope)[<subtaskId>]: …`), per-subtask board move
  (*In progress → Internal testing*), per-subtask timelog
  (`POST /tasks/{subtaskId}/time.json`) with the same sequential
  non-overlapping 5-min cursor as today.
- **Recursive expansion** up to `subtasks.max_depth` (default `2`) — covers
  parent → subtask → sub-subtask. Increase for deeply nested projects;
  decrease to `1` to expand only direct children.
- **Parent context in the plan.** When `subtasks.include_parent_context = true`
  (default), each subtask's plan entry gets a `Parent context:` line with
  the parent's name plus a ≤ 300-char description excerpt so the LLM
  understands the broader containing scope before implementing.
- **Tasklist filter parity for subtasks.** Step 3.45 (the `To Do + me` filter)
  now operates on the post-expansion working set. A subtask in the wrong
  stage or assigned to a teammate is dropped to `analyse_only` with the
  same rules as a top-level task. Step 3.42 captures per-subtask
  `stageName` + `assignees` from the `?include=cards,stages` shape and
  feeds them into `TASK_STAGE_FILE` when the tasklist endpoint did not
  return the subtask on its own (the common case — subtasks are not on the
  parent tasklist's board view).
- **Config block `subtasks`** with three tunables:
  - `enabled` (default `true`) — master switch. `false` recovers v1.3
    behaviour entirely (parent stays in the working set, subtasks invisible).
  - `max_depth` (default `2`) — recursion ceiling. `1` = direct children
    only.
  - `include_parent_context` (default `true`) — emit the `Parent context:`
    line in the plan for subtasks.
- **API shape tolerance.** Both `.tasks[]` and `.subtasks[]` response shapes
  are accepted (Teamwork v3 has shipped both at different times). Endpoint
  404 on `/tasks/{id}/subtasks.json` triggers fallback to
  `/tasks.json?parentTaskIds={id}`.

### Behavioural decisions baked in

- **Parent stays on the board.** The parent task is not moved across the
  workflow and gets no time log. It is a container; the work lives in the
  subtasks.
- **Each subtask moves itself.** Per-subtask board move
  *In progress → Internal testing*, per-subtask `mark complete` (when
  `auto_complete_finished_tasks = true`).
- **Per-subtask time logs.** No aggregation into a single parent timelog —
  every subtask gets its own entry with sequential 5-min cursoring.
- **No auto-complete on the parent.** When all subtasks finish, the parent
  is **not** auto-clicked complete. The user closes the container manually.

### Edge cases handled

- **Single-task URL on a parent with subtasks** → working set becomes the N
  subtasks; tasklist filter is bypassed (single-task URL rule); every
  subtask is `process`.
- **Single-task URL on a subtask itself** → no further expansion; runs as a
  single standalone task.
- **Subtask in a different project than parent** → Step 3.3's per-project
  workflow cache picks up the extra project; the subtask's board move
  targets its own project's workflow.
- **`skip_completed_tasks = true`** → applies per subtask, exactly like
  top-level tasks today.
- **Subtask has no card / not on the board** → falls back to `process` (same
  rule as a top-level task with no card; backlog subtasks never get silently
  stranded).
- **`subtasks.enabled = false`** → step is a no-op; parents stay in the
  working set; subtasks invisible (v1.3 behaviour).

### Migration

The 1.4.0 migration is automatic and idempotent. Your existing config gains
the `subtasks` block on the next run with safe defaults (`enabled = true`,
`max_depth = 2`, `include_parent_context = true`). If you do not want the
subtask expansion in a specific run, pass `--subtasks=false` (per-run
override) or set `"subtasks": {"enabled": false}` in your config to disable
permanently. Existing single-task and tasklist runs that did not involve
subtasks behave exactly the same.

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
