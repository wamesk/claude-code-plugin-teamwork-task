---
name: teamwork-task
description: "Use when the user provides a Teamwork.com URL (tasklist or task) and asks to 'work on these tasks', 'urob tasky z teamworku', 'spracuj tasky z teamwork', 'vypracuj tasky z teamworku', or invokes '/teamwork-task'. Fetches tasks via the Teamwork REST API (v3), pulls task description, attachments, comments (always the newest one, the full thread when needed), and file comments for context, **scans the local working tree for unattached specs / samples / DNR docs that match the task keywords and asks the user whether to use them**, **detects gating phrases in the task body (e.g. 'Bez vzorky nemá zmysel písať regex') and pauses with a question before implementing instead of barreling through with synthetic data**, implements tasks one by one in the current repository (planning and self-checking each one against five build-time quality dimensions — UI/UX & accessibility, performance, security, page reachability: every new screen gets its menu entry and inbound links in the same commit — and framework best practices: new or changed code uses the current idioms and built-in features of the framework / language versions the project actually has installed, detected from its lock files and looked up in current docs, never newer than installed and never as a drive-by rewrite), moves the task on the board (In progress → Done - Local, with fallbacks Internal testing → Testing), commits per task using the TYPE(scope)[<task-id>]: Message convention, and logs time back to Teamwork as sequential, non-overlapping 5-min-aligned entries that pick up from your last timelog of the day. **For tasklist URLs the skill applies a board-column + assignee filter — only tasks in one of the start columns (`Ready for Development` or `To Do` by default, exact case-sensitive match) AND assigned to the current user are actually implemented; tasks not on the board and every other task in the tasklist are still fetched, analysed, and briefly commented on so the developer can sanity-check teammates' work without touching it. Single-task URLs deliberately bypass the filter (a completed task is only processed after the user confirms).** Configurable safety gate asks for review when the diff touches UI/template files or grows beyond 100 lines. When the companion `teamwork-task-test` skill is installed, hands off to it at the very end so each task's acceptance criteria get individually verified before the user pushes. Pauses and asks the user via AskUserQuestion on blockers."
argument-hint: "<teamwork-url> [--time-mode=real_rounded_5m|ask] [--branching=current_branch|new_feature_branch] [--plan-mode=overview|per_task|none] [--auto-commit=always|when_safe|never] [--local-discovery=true|false] [--readiness-gate=true|false] [--test-after=true|false] [--worktree-cleanup=true|false|ask] [--worktree-handoff=ask|merge|push|leave] [--worktree-target=ask|parent|main|<branch>] [--tasklist-filter=true|false] [--tasklist-todo-stage=<name>[,<name>…]] [--tasklist-only-mine=true|false] [--subtasks=true|false] [--dimensions=ui_ux,performance,security,reachability,framework|none] [--mode=fast|full]"
allowed-tools: [Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion, Skill]
---

# Teamwork Task Worker

Fetch tasks from Teamwork.com (single task or whole tasklist), pull every bit
of context that is already attached to them in Teamwork (description, comments,
file attachments, file comments, tasklist description), implement them one by
one in the current repository, move the card across the board as work
progresses, commit per task, and log spent time back to Teamwork as sequential
non-overlapping entries. The user pushes to remote manually.

The user invoked this skill with: `$ARGUMENTS`

---

## Shell portability contract (v1.5.0)

Every bash block below runs in the user's login shell through Claude Code's
Bash tool — **zsh 5.9 on macOS**, bash elsewhere — and **every Bash tool call
is a fresh shell**: variables and functions do not survive between calls.
Each rule below has shipped a *silent* failure (empty comments, no subtasks,
no board moves) before; keep them when editing this file:

- **Never `echo "$VAR" | jq …`** (or `| grep` / `| sed` / `| tr`) on API JSON,
  descriptions, comments or file lists. zsh's builtin `echo` expands `\n`,
  `\t`, `\\` inside the payload and jq rejects it. Use `jq … <<<"$VAR"` or
  `printf '%s\n' "$VAR" | …`.
- **Never swallow a parse or HTTP failure** (`2>/dev/null || echo 0`). Capture
  the status (`curl -w '\n%{http_code}'`, then `HTTP=${RESP##*$'\n'};
  BODY=${RESP%$'\n'*}`), print a `⚠` line naming the endpoint, and state the
  fallback the run continues with.
- **`for X in $VAR` iterates once in zsh** (no word splitting). Use
  `while IFS= read -r X; do …; done <<<"$VAR"`.
- **Empty TSV fields collapse** under `IFS=$'\t' read` (tab is IFS whitespace
  in both shells) — write `-` for an empty field and map it back, or parse the
  file with `awk -F '\t'`.
- **No bash-only syntax:** `[ a == b ]` (use `=`), `${!arr[@]}` / `${!name}`,
  numeric array subscripts (zsh arrays are 1-based; `ARR[$PROJECT_ID]=1`
  allocates a 700 000-slot array) — use slices `${ARR[@]:i:1}` or a file
  keyed by id.
- **Loops that GET once per task** (Step 3.42) use `curl --retry 3 -o <file>`:
  curl then retries HTTP 429 / 5xx with backoff and honours `Retry-After`;
  the file matters because on stdout a retried request concatenates every
  failed body in front of the good one. Never `--retry` a POST / PUT.
- **zsh traps:** an unmatched glob is fatal (`no matches found`) — use
  `find … -name '…'` and quote every URL (they contain `?` and `&`); a bare
  `local X` repeated inside a loop prints `X=<value>` — declare every `local`
  once, at the top of the function.
- **jq's `//` treats `false` like a missing value.** `.x // true` turns an
  explicit `"x": false` into `true`, and `(.x //= true)` in the migration
  rewrites it on every run. Boolean defaults use
  `.x | if . == null then true else . end` (reads) and
  `(.x |= if . == null then true else . end)` (migration); `// "text"`,
  `// []` and `//= {…}` are fine.
- **Cross-step state lives in files, never in shell variables.** Snippets
  that read Teamwork data or run state start with the *run preamble* (Step 3)
  that re-derives `CONFIG_FILE`, `AUTH`, `BASE`, `ENTITY_ID` and
  `TW_JOB_DIR=/tmp/tw_job_<ENTITY_ID>`; tables shared between steps live in
  `TW_JOB_DIR` or `/tmp/tw_*_<ENTITY_ID>.tsv`. The few scalars the worker
  loop still carries between calls (`USER_ID`, `START_TS`,
  `SESSION_CURSOR_TS`, `DURATION_MIN`, the Step 7 accounting lists) are
  remembered by you, the executor, and written literally into the next
  snippet — never assumed to survive on their own.

---

## Arguments

Expected first positional argument: a Teamwork.com URL pointing to either a tasklist or a single task.

Recognized URL shapes:
- Single task: `https://<workspace>.teamwork.com/app/tasks/<taskId>` (also `/#/tasks/<id>`, `/tasks/<id>` etc.)
- Tasklist: `https://<workspace>.teamwork.com/app/tasklists/<tasklistId>` (also legacy `/tasklists/<id>`)

Optional flags (override config for this run only — not persisted):
- `--time-mode=real_rounded_5m` | `--time-mode=ask`
- `--branching=current_branch` | `--branching=new_feature_branch`
- `--plan-mode=overview` | `--plan-mode=per_task` | `--plan-mode=none`
- `--auto-commit=always` | `--auto-commit=when_safe` | `--auto-commit=never`
- `--local-discovery=true|false` — scan the working tree for unattached spec/sample files matching task keywords (default `true`, see Step 3.10)
- `--readiness-gate=true|false` — pause with a question when a task body says it needs an external input that may not yet be available (default `true`, see Step 6.0)
- `--test-after=true|false` — hand off to `/teamwork-task-test` after a clean finish (default `true`; `false` in `fast` mode)
- `--worktree-cleanup=true|false|ask` — at end of run, scan the repo for other worktrees and offer to remove ones that are merged & clean (default `true`, see Step 10)
- `--worktree-handoff=ask|merge|push|leave` — when running inside a worktree, decide at the end of the run what to do with the worktree's commits (default `ask`, see Step 9.5). `merge` = fast-forward into parent, fall back to merge commit if FF impossible; `push` = push current branch to remote and leave for a PR; `leave` = no-op.
- `--worktree-target=ask|parent|main|<branch>` — when `--worktree-handoff=merge`, decide where to merge into (default `ask`).
- `--tasklist-filter=true|false` — applies only when `URL_KIND=tasklist`: filter tasks down to the ones in one of the configured start columns AND assigned to the current user (default `true`, see Step 3.45). Has no effect on single-task URLs.
- `--tasklist-todo-stage=<name>[,<name>…]` — override the start columns `tasklist_filter.todo_stages` for this run; comma-separated for several (default `Ready for Development,To Do`, each matched per `tasklist_filter.todo_stage_match_mode` — **case-sensitively** by default). The Step 3.3 / 3.45 snippets take the value through their `TODO_STAGES_CLI` line.
- `--tasklist-only-mine=true|false` — override `tasklist_filter.only_assigned_to_me` for this run (default `true`).
- `--subtasks=true|false` — override `subtasks.enabled` for this run (default `true`, see Step 3.42).
- `--dimensions=<csv>|none` — which build-time quality dimensions to plan (Step 6.2), follow (Step 6.3) and self-check (Step 6.5.5). Any subset of `ui_ux,performance,security,reachability,framework` (the same keys `/teamwork-task-test` reviews in its Step 6.6 — `framework` there as advisory recommendations only), or `none` to skip them (default: `config.build_quality.dimensions`, all five). Unknown keys are dropped with a `⚠` line.
- `--mode=fast|full` — work mode for this run (default: resolved in Step 2.65 — the project's `.claude/work-mode.local.md`, then the legacy `.claude/wame-mode.local.md`, then `full`). `fast` builds fast: the quality dimensions, the test proposal, the per-task test run and Pint, browser checks and the `/teamwork-task-test` handoff are all off at once, and every skipped check is recorded for `/work-mode full`. `full` keeps every check exactly as before. An explicit individual flag (`--dimensions=…`, `--test-after=…`) still wins over the mode. The pre-1.7.0 values `--mode=build` / `--mode=harden` are deprecated aliases of `fast` / `full` (one `⚠` line).

If `$ARGUMENTS` is empty or does not contain a URL, ask the user via **AskUserQuestion** for the Teamwork URL before doing anything else.

---

## Step 1 — Parse the URL

Extract from the URL:
- `WORKSPACE` — subdomain (e.g. `acme` from `https://acme.teamwork.com/...`). Derive `BASE_URL = https://<WORKSPACE>.teamwork.com`.
- `URL_KIND` — `task` or `tasklist`.
- `ENTITY_ID` — numeric task or tasklist ID.

Use a simple shell regex via `Bash`:
```bash
URL="<the url>"
WORKSPACE=$(echo "$URL" | sed -nE 's|https?://([^.]+)\.teamwork\.com/.*|\1|p')
# NOTE: use '#' as the s-command delimiter, NOT '|' — the regex alternation
# (tasks|tasklists) contains a literal '|', and BSD/macOS sed treats the first
# inner '|' as the closing delimiter, aborting with "RE error: parentheses not
# balanced" and returning an EMPTY id/kind for every URL. '#' avoids the clash.
ENTITY_ID=$(echo "$URL" | sed -nE 's#.*/(tasks|tasklists)/([0-9]+).*#\2#p')
URL_KIND=$(echo "$URL" | sed -nE 's#.*/(tasks|tasklists)/[0-9]+.*#\1#p' | sed 's/s$//')
BASE_URL="https://${WORKSPACE}.teamwork.com"
```

If parsing fails (any of the three is empty), ask the user via **AskUserQuestion** to confirm the URL or paste a corrected one.

---

## Step 2 — Load or create config (first-run flow)

Runtime config lives **outside** the plugin cache, so it survives plugin updates:

```
~/.claude/plugins/data/teamwork-task-wamesk/config.json
```

Algorithm:
1. `CONFIG_DIR="$HOME/.claude/plugins/data/teamwork-task-wamesk"` — `mkdir -p "$CONFIG_DIR"`.
2. `CONFIG_FILE="$CONFIG_DIR/config.json"`.
3. If `CONFIG_FILE` does not exist: copy the bundled template from `${CLAUDE_PLUGIN_ROOT}/config.example.json` into `CONFIG_FILE`, then `chmod 600 "$CONFIG_FILE"`.
4. Read the config with `jq` (always validate first: `jq . "$CONFIG_FILE" >/dev/null` — if invalid, report to the user and stop).
5. **First-run check** — if any of these is missing/empty/`<workspace>` placeholder:
   - `.teamwork.base_url`
   - `.teamwork.api_token`

   Then prompt the user via **AskUserQuestion**:
   - **Base URL**: prefill with the `BASE_URL` derived from the input URL (just confirm or override).
   - **API token**: ask the user to paste it. Instruct: *"Get it from `${BASE_URL}/launchpad/apikey/manage` → Create API Key. Token is stored in `~/.claude/plugins/data/teamwork-task-wamesk/config.json` with chmod 600."*

   After collecting values, write them back to the config:
   ```bash
   jq --arg url "$BASE_URL" --arg tok "$API_TOKEN" \
      '.teamwork.base_url=$url | .teamwork.api_token=$tok' \
      "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
   chmod 600 "$CONFIG_FILE"
   ```

6. **Apply CLI flag overrides** to in-memory config (`--time-mode`, `--branching`, `--plan-mode`, `--auto-commit`, `--dimensions`, `--mode`, …) — do not persist them. Resolve the work mode (Step 2.65) first and apply the individual flags after it, so an explicit flag wins over what `fast` implies. `--dimensions=none` means an empty active set; `--dimensions=ui_ux,security` keeps only the listed known keys.

7. **Never echo the API token** in shell output. When invoking `curl`, pass auth via `-u` to keep it out of `ps`.

### Step 2.6 — Config migration (silent, idempotent)

Before reading any new key, normalize the config to the 1.1.0 shape:

```bash
# Map legacy `fetch_comments` (bool) → `fetch_comments_mode` (enum)
LEGACY=$(jq -r '.fetch_comments // empty' "$CONFIG_FILE")
if [ "$LEGACY" = "true" ]; then
  jq 'del(.fetch_comments) | .fetch_comments_mode = "always"' "$CONFIG_FILE" \
    > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
elif [ "$LEGACY" = "false" ]; then
  jq 'del(.fetch_comments) | .fetch_comments_mode = "never"' "$CONFIG_FILE" \
    > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
fi

# Fill in any new key that the user has not set explicitly
jq '
  (.plan_mode //= "overview") |
  # v1.7.0: the work mode left this shared file — Step 2.65 reads it from the
  # project file `.claude/work-mode.local.md` only. Deleting an absent key is
  # a no-op, so this stays idempotent.
  del(.mode) |
  (.fetch_comments_mode //= "when_needed") |
  (.fetch_attachments |= if . == null then true else . end) |
  (.fetch_file_comments |= if . == null then true else . end) |
  (.max_attachment_size_mb //= 25) |
  (.attachments_cleanup //= "after_timelog") |
  (.auto_commit_mode //= "when_safe") |
  (.auto_commit_risky_patterns //= ["\\.(vue|jsx|tsx|svelte|blade\\.php|css|scss|sass|less|html)$"]) |
  (.auto_commit_max_diff_lines //= 100) |
  (.auto_propose_tests |= if . == null then true else . end) |
  (.test_frameworks //= {
    "php_unit_preference": "auto",
    "php_browser_preference": "auto",
    "js_unit_preference": "auto",
    "js_browser_preference": "auto"
  }) |
  (.test_visual_file_patterns //= [
    "\\.(vue|jsx|tsx|svelte|blade\\.php|html)$",
    "^resources/views/"
  ]) |
  (.test_opt_out_keywords //= [
    "no tests", "skip tests", "without tests", "bez testov", "netreba testy"
  ]) |
  (.time_cursor_strategy //= "last_teamwork_timelog") |
  (.include_commit_hash_in_log_description |= if . == null then true else . end) |
  (.time_mode //= "real_rounded_5m") |
  (.time_rounding_minutes //= 5) |
  (.min_log_minutes //= 1) |
  # One-shot rename for users who tested an early 1.3.0 build with the
  # old `round_up_threshold_minutes` name. The key was never on a released
  # tag, so this rename is best-effort cleanup — no fallback if both keys
  # somehow co-exist (the new name wins).
  (if has("round_up_threshold_minutes") and (has("round_threshold_minutes") | not)
   then .round_threshold_minutes = .round_up_threshold_minutes
        | del(.round_up_threshold_minutes)
   else . end) |
  (.round_threshold_minutes //= 4) |
  (.branching_mode //= "current_branch") |
  (.default_language //= "sk") |
  (.auto_complete_finished_tasks //= false) |
  (.skip_completed_tasks |= if . == null then true else . end) |
  (.is_billable_by_default |= if . == null then true else . end) |
  (.board_workflow //= {
    "enabled": true,
    "in_progress_stage": "In progress",
    "done_stage": "Done - Local",
    "done_stage_fallbacks": ["Internal testing", "Testing"],
    "match_mode": "case_insensitive",
    "done_stage_schema": 2
  }) |
  # v1.5.0: the new WAME board ends in "Done - Local". The done target moves
  # ONLY when it is still the old default "Internal testing" and the marker
  # says this check never ran; "Internal testing" then becomes the first
  # fallback (existing fallbacks follow, minus "Done - Local", deduplicated
  # case-insensitively), so old boards keep landing where they did. A
  # customised done_stage is never touched. `done_stage_schema: 2` records
  # that the check ran, so choosing "Internal testing" again later sticks.
  (if (.board_workflow | type) == "object"
      and (.board_workflow.done_stage_schema // 1) < 2
      and .board_workflow.done_stage == "Internal testing"
   then .board_workflow.done_stage = "Done - Local"
        | .board_workflow.done_stage_fallbacks =
            (["Internal testing"]
             + [ (.board_workflow.done_stage_fallbacks
                   | if type == "array" then .[] elif type == "string" then . else empty end)
                 | select(type == "string" and length > 0)
                 | select(ascii_downcase as $n | $n != "done - local" and $n != "internal testing") ]
             | reduce .[] as $s ([];
                 if any(.[]; ascii_downcase == ($s | ascii_downcase)) then . else . + [$s] end))
   else . end) |
  (if (.board_workflow | type) == "object" and (.board_workflow.done_stage_schema // 1) < 2
   then .board_workflow.done_stage_schema = 2
   else . end) |
  (.auto_run_tests_after |= if . == null then true else . end) |
  (.local_context_discovery //= {
    "enabled": true,
    "max_depth": 3,
    "scan_dirs": [".", "docs", "specs", "samples", "Strečnianska"],
    "extensions": ["docx","pdf","md","txt","eml","msg","csv","xlsx","sql","json","html"],
    "filename_hints": ["vzor","sample","dnr","specifikác","otazky","q&a","podklad","priloh","attachment"],
    "ignore_globs": [".git/**","vendor/**","node_modules/**","storage/**",".claude/**","public/**","bootstrap/cache/**",".idea/**",".vscode/**","teamwork-task-*/**"],
    "min_keyword_score": 1,
    "max_files_to_offer": 12,
    "doc_converters": {
      "docx": "auto",
      "pdf": "auto",
      "xlsx": "auto"
    }
  }) |
  (.readiness_gate //= {
    "enabled": true,
    "patterns": [
      "pred[[:space:]]+začatím[[:space:]]+vyžiadať",
      "bez[[:space:]]+(vzor|sample|fixture|údajov|dát|prílohy|prikladu)[[:space:]]+(nemá[[:space:]]+zmysel|nie[[:space:]]+je[[:space:]]+možné)",
      "vyžiadať[[:space:]]+(reálny[[:space:]]+)?(vzor|sample|prílohu)",
      "waiting[[:space:]]+for[[:space:]]+(client|customer|stakeholder|sample)",
      "requires[[:space:]]+(real|production)[[:space:]]+(sample|data|fixture)",
      "prisľúbené",
      "pending[[:space:]]+from",
      "blocked[[:space:]]+by",
      "⏳"
    ],
    "filename_hint_pattern": "[A-Za-z0-9_-]+\\.(docx|pdf|xlsx|eml|msg|csv|sql|md|json|txt)\\b",
    "on_block_default": "ask"
  }) |
  (.worktree_cleanup //= {
    "enabled": true,
    "auto_remove_merged_clean": false,
    "stale_age_days": 14,
    "ignore_paths": [],
    "report_when_empty": false
  }) |
  (.tasklist_filter //= {
    "enabled": true,
    "todo_stages": ["Ready for Development", "To Do"],
    "todo_stage_match_mode": "case_sensitive",
    "only_assigned_to_me": true,
    "analyze_all_tasks": true,
    "skip_reason_render": "inline"
  }) |
  # v1.5.0: start columns. `todo_stages` (array) supersedes the single legacy
  # `todo_stage` string. Written once, only when absent (or null) — an
  # existing `todo_stages` is never overwritten. A legacy value that is
  # missing or the old default "To Do" gets the new default pair; a
  # customised one is kept as a one-item list. The legacy key itself stays
  # (older builds of the sibling plugins that share this file still read it).
  (if (.tasklist_filter | type) == "object" and .tasklist_filter.todo_stages == null
   then .tasklist_filter.todo_stages =
          (.tasklist_filter.todo_stage as $legacy
           | if ($legacy | type) == "string" and $legacy != "" and $legacy != "To Do"
             then [$legacy]
             else ["Ready for Development", "To Do"] end)
   else . end) |
  (.worktree_handoff //= {
    "enabled": true,
    "default_action": "ask",
    "default_target": "ask",
    "merge_strategy": "ff_else_merge",
    "delete_branch_after_merge": true,
    "delete_worktree_after_merge": true,
    "push_remote": "origin",
    "skip_if_no_commits": true
  }) |
  # v1.5.0: build-time quality dimensions (Steps 6.2 / 6.3 / 6.5.5). Merged key
  # by key so an existing block keeps its values — including an explicit `[]`
  # ("none"), which `//=` preserves because only null/false are replaced.
  (.build_quality //= {}) |
  # The fifth key `framework` joins a list written before it existed ONLY when
  # that list is exactly the old four-key default (any order). Any other list
  # — a subset, `[]`, a list that already names `framework`, unknown keys —
  # is a user choice and stays as it is. `dimensions_schema: 2` records that
  # this check ran, so a user who later removes `framework` keeps that choice.
  (if (.build_quality.dimensions_schema // 1) < 2
      and (.build_quality.dimensions | type) == "array"
      and (.build_quality.dimensions | sort) == ["performance", "reachability", "security", "ui_ux"]
   then .build_quality.dimensions += ["framework"]
   else . end) |
  (.build_quality.dimensions //= ["ui_ux", "performance", "security", "reachability", "framework"]) |
  (if (.build_quality.dimensions_schema // 1) < 2
   then .build_quality.dimensions_schema = 2
   else . end)
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

The migration is idempotent — running it twice produces the same file. Do not print the config to stdout; only mention "config migrated to 1.5.0 schema" once if any change was made.

The `build_quality` key (added in 1.5.0) controls the build-time quality rules —
Step 6.2 decides per task which of the five dimensions (`ui_ux`, `performance`,
`security`, `reachability`, `framework`) the change shape touches, Step 6.3
follows the matching rules while implementing, and Step 6.5.5 self-checks the
task's diff against them before the timer stops. The keys are exactly the ones
`/teamwork-task-test` reviews at QA time (its Step 6.6), so whatever is left
open here is handed to it in Step 8. `framework` is active here (it shapes
the new or changed code) but **advisory** on the QA side: the tester only
recommends, it never fails or downgrades an acceptance criterion on it.
Default is all five; set `"build_quality": {"dimensions": []}` or pass
`--dimensions=none` to skip them.

**Migration rule for the fifth key.** `build_quality.dimensions_schema` marks
which key set the list was last reconciled with (absent = the four-key list
of a pre-release 1.5.0 build; `2` = five keys). The Step 2.6 migration adds
`framework` **once**, and only when all of these hold: the marker is absent
(or `< 2`), the list is an array, and it contains exactly `ui_ux`,
`performance`, `security`, `reachability` — each once, in any order. Every
other list is treated as customised and left untouched: a subset
(`["security", "reachability"]`), an explicit `[]`, a list that already names
`framework`, or one with unknown keys. A missing / `null` list gets the
five-key default. Either way the marker is then set to `2`, so a user who
removes `framework` afterwards is never overridden by a later run. A list of
exactly the four keys is indistinguishable from the untouched old default,
so it gains `framework` — remove it once to opt out; it stays out.

**Migration rule for the board columns (1.5.0).** The shared WAME board
(Teamwork workflow *"WAME workflow"*) starts work in *Ready for Development*
or *To Do* and ends it in *Done - Local*; older per-project boards still end
in *Internal testing* or *Testing*. Both keep working:
- **Start columns** — `tasklist_filter.todo_stages` is written **once, only
  when absent or `null`** (an existing list is never overwritten): the new
  default `["Ready for Development", "To Do"]` when the legacy
  `tasklist_filter.todo_stage` is missing, empty or the old default
  `"To Do"`; `[<legacy value>]` when it was customised (`"Backlog"` →
  `["Backlog"]`). The legacy string stays in the file and is still read when
  `todo_stages` is absent or empty (see Step 3.3).
- **Done target** — `board_workflow.done_stage` moves to `"Done - Local"`
  **only** when it equals the old default `"Internal testing"` and
  `board_workflow.done_stage_schema` is absent; the fallbacks become
  `["Internal testing", …the existing fallbacks minus "Done - Local"…]`
  (deduplicated case-insensitively, `"Testing"` kept). A customised
  `done_stage` is never touched. Either way the marker is then set to `2`, so
  a user who deliberately picks `"Internal testing"` again later keeps it.
  Examples: the old default (`"Internal testing"` + `["Testing"]`) and a
  config already carrying `["Done - Local", "Testing"]` as fallbacks both end
  as `"Done - Local"` + `["Internal testing", "Testing"]`; `"QA"` stays
  `"QA"`.
- `board_workflow.in_progress_stage` stays `"In progress"` — the match is
  case-insensitive, so it resolves *In Progress* on the WAME board as well.

The sibling plugins that read these keys from the same shared file
(`teamwork-task-analyze`, `teamwork-tasks-from-session`) must apply exactly
these rules — add a missing key; move the done target only away from the old
default and only while `done_stage_schema` is absent, then set it to `2` —
so the file ends in the same state whichever plugin runs first, and none of
them rewrites a value the user chose.

The top-level `mode` key that 1.6.0 added is **gone since 1.7.0**: the migration deletes it (`del(.mode)`) and nothing reads it any more — `teamwork-task-test`, which shares this file, neither reads nor writes it either. The work mode is a per-project setting in `.claude/work-mode.local.md` (plugin `work-mode`); its global default is the `work-mode` plugin option `default_mode` in Claude Code `/config`, which that plugin's SessionStart hook writes into the project file. See Step 2.65.

The `auto_run_tests_after` key (added in 1.1.1) controls whether this skill, on a clean finish, hands off to `/teamwork-task-test` to verify the acceptance criteria of every implemented task. Default is `true`. Disable per run with `--test-after=false`.

The `local_context_discovery` key (added in 1.1.3) controls Step 3.10 — scanning the working tree for unattached specs / samples that the user dropped into the project folder but did not attach to the Teamwork task. Default is `true`. Disable per run with `--local-discovery=false`.

The `readiness_gate` key (added in 1.1.3) controls Step 6.0 — detecting phrases in the task description that mark an external blocker (e.g. *"Bez vzorky nemá zmysel písať regex"*) and pausing with **AskUserQuestion** before implementing, so the skill does not silently produce code against synthetic placeholders. Default is `true`. Disable per run with `--readiness-gate=false`.

The `worktree_cleanup` key (added in 1.2.0) controls Step 10 — at the end of a successful run, scan the repo for other git worktrees (typically `.claude/worktrees/<name>` left behind by prior background jobs) and offer to delete the ones that are clean and merged into the default branch. Default is `true`. Disable per run with `--worktree-cleanup=false`.

The `min_log_minutes` key (added in 1.2.0) controls Step 6.6.1 — when the real-time headroom between the session cursor and `now()` is smaller than `time_rounding_minutes` (default 5) but at least `min_log_minutes` (default 1), the skill writes a smaller-than-round timelog entry instead of skipping the task entirely. The cursor still advances by the exact logged amount, so the next log resumes seamlessly. Set `min_log_minutes` to the same value as `time_rounding_minutes` to keep the legacy v1.1.x "5-min minimum or skip" behaviour.

The `round_threshold_minutes` key (added in 1.3.0) decouples the rounding
decision from how the skill treats short runs in Step 6.6. **Elapsed time
strictly below this threshold is logged as raw minutes** (1, 2, or 3 min for
the default `4`), while elapsed time *at or above* the threshold is rounded
to the **nearest** `time_rounding_minutes` step using half-up integer
rounding. With the defaults (`threshold=4`, `ROUND=5`):

| elapsed | logged | reason                       |
| ------- | ------ | ---------------------------- |
| 1–3 min | 1–3    | below threshold — raw        |
| 4 min   | 5      | nearest multiple of 5        |
| 5 min   | 5      | exact                        |
| 6 min   | 5      | 6 is closer to 5 than to 10  |
| 7 min   | 5      | 7 is closer to 5 than to 10  |
| 8 min   | 10     | 8 is closer to 10            |
| 11 min  | 10     | 11 is closer to 10 than to 15 |
| 12 min  | 10     | 12 is closer to 10           |
| 13 min  | 15     | 13 is closer to 15           |

The motivation is honest billing in both directions: a 30-second
documentation tweak should not be billed as 5 minutes (handled by the
sub-round zone), and an 11-minute hotfix should not be padded to 15 just
because the previous policy was "always round up" (handled by the
round-to-nearest zone). Set `round_threshold_minutes` equal to
`time_rounding_minutes` to recover the pre-1.3.0 "everything rounds up to
ROUND" behaviour (every entry will be ≥ 5 min and bias upward). Set it to
`1` to apply nearest-rounding to every entry and disable the sub-round
raw-minutes zone entirely. Note: the **old key name**
`round_up_threshold_minutes` from early 1.3.0 builds is silently renamed
to `round_threshold_minutes` by the Step 2.6 migration; the value is
preserved.

The `tasklist_filter` key (added in 1.3.0) controls Step 3.45 — when the user
hands a **tasklist** URL, the skill only implements tasks that are (a) currently
in **one of the start columns** listed in `todo_stages` (default
`["Ready for Development", "To Do"]` since 1.5.0; the list order is only the
display order; each name matched **case-sensitively** by default — i.e.
`to do`, `TO DO`, `ToDo` do **not** match `To Do`) and (b) assigned to the
current authenticated user. A task that is **not on the board** (or whose
project has no workflow) has no start column and is analyse-only as well
(1.5.0). The remaining tasks in the tasklist are still fetched and briefly
analysed in the plan (Step 4) but the worker loop (Step 6) skips their
implementation, commit, time log, and board moves — so a Laravel-backend
developer running the skill against a shared "Backend + Ionic frontend"
tasklist sees teammates' tasks for context and can comment on them, but does
not start coding them. Single-task URLs (`URL_KIND=task`) bypass the filter,
no matter the config — when a user opens a specific task by ID they
typically want it processed regardless of where it sits on the board; only a
**completed** task asks first (Step 3.35). Disable entirely with
`"tasklist_filter": {"enabled": false}` or per run with
`--tasklist-filter=false`.

The `worktree_handoff` key (added in 1.3.0) controls Step 9.5 — when the
skill is executing **inside a git worktree** (typical for background runs
launched via `EnterWorktree`, or whenever the user starts the session from
`.claude/worktrees/<name>`), commits land on the worktree's branch and never
reach `main` on their own. v1.3.0 closes that loop: after the final summary
(Step 7) and the optional verification handoff (Step 8) but **before** the
push reminder (Step 9), the skill detects that it is running in a worktree
with at least one new commit and offers, via **AskUserQuestion**, one of:
- *Merge into a target branch* (default — fast-forward if possible, fall
  back to merge commit). The target is asked separately (parent / main /
  custom), so the user can confirm where the commits go.
- *Push the worktree branch and let me open a PR.*
- *Leave as-is — I will handle the handoff manually.*
On a successful merge, both the branch and the worktree are removed by
default (`delete_branch_after_merge` + `delete_worktree_after_merge`), which
chains into Step 10 cleanup without re-asking. Disable per run with
`--worktree-handoff=leave` or persistently with
`"worktree_handoff": {"enabled": false}`. Power users can preset
`default_action=merge` (skip the first question) and/or
`default_target=parent` (skip the target question) for unattended runs.

---

## Step 2.65 — Work mode: fast vs. full (v1.6.0, renamed in v1.7.0)

Most of a run's wall-clock time used to go to verification, not to building:
the five-dimension plan and self-check, a proposed test file per task, a
filtered test run and Pint per task, browser checks and finally the whole
`/teamwork-task-test` pass — on every task. The work mode lets the user defer
all of that to **one** explicit full pass at the end of a feature.

**Resolution order** (first hit wins):
1. `--mode=fast|full` on the command line;
2. the project's `.claude/work-mode.local.md` YAML frontmatter `mode:` —
   written by `/work-mode` from the `work-mode` plugin, so one switch drives
   every WAME plugin in that project. A global default lives in Claude Code
   `/config` as the `work-mode` plugin option `default_mode`; that plugin's
   SessionStart hook writes it into this file, so this skill only ever reads
   the project file — never `/config` and never the shared config;
3. the legacy `.claude/wame-mode.local.md` (plugin `wame-work-mode` 1.0.0),
   read only when the new file is absent — `build` maps to `fast`, `harden`
   to `full`;
4. `full`.

`--mode=build` and `--mode=harden` are deprecated aliases of `fast` and
`full`, kept for one version; each prints one `⚠` line.

```bash
WORK_MODE="<value of --mode, or empty>"
case "$WORK_MODE" in
  build)  WORK_MODE=fast; echo "⚠ --mode=build is deprecated — use --mode=fast (renamed in 1.7.0)." ;;
  harden) WORK_MODE=full; echo "⚠ --mode=harden is deprecated — use --mode=full (renamed in 1.7.0)." ;;
esac
MODE_FILE=""
if [ -z "$WORK_MODE" ]; then
  if [ -f .claude/work-mode.local.md ]; then
    MODE_FILE=.claude/work-mode.local.md
  elif [ -f .claude/wame-mode.local.md ]; then
    MODE_FILE=.claude/wame-mode.local.md
  fi
fi
if [ -n "$MODE_FILE" ]; then
  WORK_MODE=$(sed -n '/^---$/,/^---$/{s/^mode:[[:space:]]*//p;}' "$MODE_FILE" \
    | head -n 1 | tr -d "\"' \r")
  case "$WORK_MODE" in build) WORK_MODE=fast ;; harden) WORK_MODE=full ;; esac
fi
case "$WORK_MODE" in fast|full) ;; *) WORK_MODE=full ;; esac
echo "WORK_MODE=$WORK_MODE (${MODE_FILE:-flag or default})"
```

Remember `WORK_MODE` like `USER_ID` and write it literally into later snippets.

**What `fast` switches off at once** (in memory only — no file is ever
rewritten by this step):

| Switch | `full` (default) | `fast` |
|---|---|---|
| Quality dimensions (Steps 6.2 / 6.5.5) | `config.build_quality.dimensions` | `[]` — same as `--dimensions=none` |
| Test proposal (Step 6.2.5) | `config.auto_propose_tests` | `false` |
| Per-task test run (Step 6.4) and Pint (Step 6.5) | run | skipped |
| `/teamwork-task-test` handoff (Step 8) | `config.auto_run_tests_after` | `false` — same as `--test-after=false` |
| Browser / click-through checks, docs and version lookups | as the steps say | skipped |

An explicit individual flag still wins: `--mode=fast --dimensions=security`
keeps the security self-check, `--mode=fast --test-after=true` still hands
off to QA. Fast mode skips **verification, not bookkeeping** — fetching,
the readiness gate, the plan approval, board moves, the safety gate, one
commit per task and the time logs all run as usual.

**Deferred list.** In `fast` mode, right after each task's commit (Step 6.7)
append one block to the project's `.claude/work-mode-deferred.local.md`. When
only the legacy `.claude/wame-deferred.local.md` exists, `mv` it to the new
name first so earlier entries are not lost, then append (create the file when
neither exists):

```bash
DEFERRED=.claude/work-mode-deferred.local.md
mkdir -p .claude
if [ ! -f "$DEFERRED" ] && [ -f .claude/wame-deferred.local.md ]; then
  mv .claude/wame-deferred.local.md "$DEFERRED"
  echo "ℹ Renamed .claude/wame-deferred.local.md → $DEFERRED (work-mode 2.0.0 name)."
fi
git check-ignore -q "$DEFERRED" \
  || echo "⚠ $DEFERRED is not git-ignored — add .claude/*.local.md to .gitignore; it must never be committed."
```

Then append the block (same format as before 1.7.0):

```markdown
## [<task-id>] <task title> — <YYYY-MM-DD HH:MM> — <commit hash>
- Files: <paths in the commit>
- Screens: <URLs / Nova resources the change shows up on, or none>
- Skipped: dimensions, test proposal, tests, Pint, QA handoff
```

`/work-mode full` (plugin `work-mode`) reads this file, runs the skipped
checks once, fixes what they find, clears it and switches the project to
`full`. **Never stage or commit** the deferred list or the mode file — under
either name. Step 6.7 stages explicit paths only and unstages these four
files before every commit, so a stray `git add .claude` cannot carry them in.

**Browser tooling rule (every mode).** Never install or uninstall Playwright,
Puppeteer or Laravel Dusk for a single run. Use the chrome-devtools MCP or the
runner the project already has. When a check needs a runner the project
lacks, ask the user **once**; on yes, install it permanently as a committed
dev dependency and never remove it afterwards; on no, write a manual
checklist instead.

---

## Step 2.7 — Resolve current user (cached for the whole run)

Step 3.45 (tasklist filter) and Step 5.5 (time cursor) both need the
authenticated user's numeric ID. Teamwork's v3 API does not expose a `/me`
endpoint, so fetch it once from the legacy v1 `/me.json`:

```bash
CONFIG_FILE="$HOME/.claude/plugins/data/teamwork-task-wamesk/config.json"
AUTH="$(jq -r '.teamwork.api_token' "$CONFIG_FILE"):xxx"
BASE=$(jq -r '.teamwork.base_url' "$CONFIG_FILE")

USER_ID=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/me.json" | jq -r '.person.id // empty')

if [ -z "$USER_ID" ]; then
  echo "  ⚠ could not resolve the current Teamwork user — assignee filter (Step 3.45) and last-timelog cursor (Step 5.5) will fall back to safe defaults." >&2
else
  echo "  ℹ current Teamwork user id: ${USER_ID}" >&2
fi
```

"Cached" means *remembered by you, the executor*: a shell variable does not
survive into the next Bash call (see the shell portability contract), so carry
the printed id literally into the `USER_ID="…"` line of the Step 3.45 and
Step 5.5 snippets.

Treat the empty case as **non-fatal**:
- Step 3.45 falls back to "stage filter only" (skip the assignee check) so the
  user is never silently locked out of their own tasklist by a missing
  permission.
- Step 5.5 falls back to `floor(now)` as it already does today.

Do not echo the token in the error message even when the call fails — leave
the token entirely out of stdout/stderr.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Authentication: HTTP Basic, username = API token, password = any string (Teamwork convention: use `xxx`).

**Run preamble.** Every Bash call is a fresh shell, so every snippet from here
on starts with these lines (substitute the Step 1 values literally):

```bash
CONFIG_FILE="$HOME/.claude/plugins/data/teamwork-task-wamesk/config.json"
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq -r '.teamwork.base_url' "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
ENTITY_ID="<ENTITY_ID from Step 1>"
URL_KIND="<URL_KIND from Step 1>"
TW_JOB_DIR="/tmp/tw_job_${ENTITY_ID}"   # per-run state shared between steps
```

Endpoints (Teamwork API v3 — see `https://apidocs.teamwork.com/docs/teamwork/v3/`):
- **Single task** (`URL_KIND=task`) — `GET /projects/api/v3/tasks/{id}.json` → `.task`.
- **Tasklist** (`URL_KIND=tasklist`) — `GET /projects/api/v3/tasklists/{id}.json`
  → `.tasklist` (name, description; a *completed* tasklist answers 404) and
  `GET /projects/api/v3/tasklists/{id}/tasks.json?pageSize=100&page=N` →
  `.tasks[]`, paginated while `.meta.page.hasMore == true`.

The fetch resets `TW_JOB_DIR` and the per-entity tables of Steps 3.42 / 3.45
that live next to it in `/tmp` (so a previous run of the same URL cannot leak
stale state into this one — e.g. an old `analyse_only` verdict surviving a
`--tasklist-filter=false` re-run, or old subtasks surviving `--subtasks=false`)
and writes three files the later steps read:

| File | Shape | Read by |
| --- | --- | --- |
| `$TW_JOB_DIR/tasks.json` | `{tasklist: {…} or null, tasks: [v3 task objects]}` | 3.4, 3.45, 3.10 |
| `/tmp/tw_working_set_${ENTITY_ID}.tsv` | `taskId<TAB>name<TAB>description` (plain text, ≤ 300 chars, `-` when empty) | 3.42 |
| `$TW_JOB_DIR/task_projects.tsv` | `taskId<TAB>projectId` (Step 3.42 appends subtasks) | 3.3, 6.1.5, 6.8.5 |

```bash
# (run preamble)
SKIP_DONE=$(jq -r '.skip_completed_tasks | if . == null then true else . end' "$CONFIG_FILE")
rm -rf "$TW_JOB_DIR" && mkdir -p "$TW_JOB_DIR"
# Per-entity tables outside TW_JOB_DIR (Steps 3.42 / 3.45 / 6.0a read them).
# Explicit names, no glob — an unmatched glob is fatal in zsh.
rm -f "/tmp/tw_expanded_${ENTITY_ID}.tsv" "/tmp/tw_parent_context_${ENTITY_ID}.tsv" \
      "/tmp/tw_task_stages_${ENTITY_ID}.tsv" "/tmp/tw_tasks_process_${ENTITY_ID}.tsv"
TASKS_FILE="$TW_JOB_DIR/tasks.json"
WORKING_SET_FILE="/tmp/tw_working_set_${ENTITY_ID}.tsv"
TASK_PROJECT_FILE="$TW_JOB_DIR/task_projects.tsv"
: > "$WORKING_SET_FILE"; : > "$TASK_PROJECT_FILE"

if [ "$URL_KIND" = "task" ]; then
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
    "${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" = "200" ]; then
    # A single-task URL keeps a completed task here — skip_completed_tasks
    # only filters tasklist / subtask iteration. Step 3.35 asks the user
    # whether to process it (default: skip).
    jq '{tasklist: null, tasks: [.task]}' <<<"$BODY" > "$TASKS_FILE" \
      || { echo "  ⚠ GET /projects/api/v3/tasks/${ENTITY_ID}.json returned unparsable JSON — nothing to work on" >&2; rm -f "$TASKS_FILE"; }
  else
    echo "  ⚠ GET /projects/api/v3/tasks/${ENTITY_ID}.json → HTTP ${HTTP}: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null)" >&2
  fi
else
  TL_JSON=null
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
    "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ] || ! TL_JSON=$(jq -c '.tasklist // null' <<<"$BODY"); then
    TL_JSON=null
    echo "  ⚠ GET /projects/api/v3/tasklists/${ENTITY_ID}.json → HTTP ${HTTP} — continuing without the tasklist name/description (a completed tasklist answers 404)" >&2
  fi

  PAGES_FILE="$TW_JOB_DIR/tasks.pages.jsonl"; : > "$PAGES_FILE"
  PAGE=1
  while :; do
    RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
      "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=${PAGE}")
    HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
    if [ "$HTTP" != "200" ]; then
      echo "  ⚠ GET /projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?page=${PAGE} → HTTP ${HTTP}: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — the working set stops at the tasks fetched so far" >&2
      break
    fi
    if ! jq -c '.tasks[]?' <<<"$BODY" >> "$PAGES_FILE"; then
      echo "  ⚠ tasklist page ${PAGE} is not valid JSON — the working set stops at the tasks fetched so far" >&2
      break
    fi
    [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
    PAGE=$((PAGE + 1))
  done

  jq -s --argjson tl "$TL_JSON" --arg skip "$SKIP_DONE" \
    '{tasklist: $tl, tasks: [.[] | select($skip != "true" or .status != "completed")]}' \
    "$PAGES_FILE" > "$TASKS_FILE"
fi

if [ -s "$TASKS_FILE" ]; then
  # Working set for Step 3.42 — plain-text description, empty fields as "-"
  # (an empty TSV field would collapse under `IFS=$'\t' read`).
  jq -r '.tasks[] | [
      (.id | tostring),
      ((.name // "") | if . == "" then "-" else . end),
      ((.description // "") | gsub("<[^>]+>"; "") | gsub("\\s+"; " ") | .[0:300]
        | if . == "" then "-" else . end)
    ] | @tsv' "$TASKS_FILE" > "$WORKING_SET_FILE"

  # v3 task objects carry NO `projectId` — it lives in `.tasklist.meta.projectId`.
  jq -r '.tasks[] | [(.id | tostring), ((.projectId // .tasklist.meta.projectId // "") | tostring)] | @tsv' \
    "$TASKS_FILE" > "$TASK_PROJECT_FILE"

  # v1 fallback for any task whose project id is still unknown.
  MISSING=$(awk -F '\t' '$2 == "" { print $1 }' "$TASK_PROJECT_FILE")
  if [ -n "$MISSING" ]; then
    while IFS= read -r TID; do
      PID=$(curl -sS -u "$AUTH" -H "Accept: application/json" "${BASE}/tasks/${TID}.json" \
        | jq -r '."todo-item"."project-id" // empty')
      if [ -n "$PID" ]; then
        awk -F '\t' -v OFS='\t' -v id="$TID" -v pid="$PID" '$1 == id { $2 = pid } { print }' \
          "$TASK_PROJECT_FILE" > "$TASK_PROJECT_FILE.tmp" && mv "$TASK_PROJECT_FILE.tmp" "$TASK_PROJECT_FILE"
      else
        echo "  ⚠ task #${TID}: project id unknown in v3 and v1 — board moves for this task will be skipped" >&2
      fi
    done <<<"$MISSING"
  fi
  echo "  ℹ Step 3: $(grep -c . "$WORKING_SET_FILE") task(s) in the working set" >&2
else
  echo "  ⚠ Step 3: no task data fetched — report the HTTP line above to the user and stop" >&2
fi
```

Fields used later (v3 names, verified against the live API):
- `id`, `name`, `description` (HTML), `status`, `priority`, `estimateMinutes`,
  `parentTaskId`, `tasklistId`.
- **Project id = `.projectId // .tasklist.meta.projectId`.** v3 task objects —
  the single fetch *and* the list items — carry **no `projectId` key**; the id
  sits in `.tasklist = {id, type, meta: {name, projectId}}`. Reading
  `.projectId` alone yields `null`, which silently disabled workflow detection
  (Step 3.3), the start-column lookup (Step 3.45) and every board move
  (Steps 6.1.5 / 6.8.5). v1 fallback: `GET /tasks/{id}.json` →
  `."todo-item"."project-id"`.
- **There is no `commentsCount` either** — v3 never returns it, so a gate
  on it never opens. Step 3.5 counts comments with a one-item probe instead.
- `workflowStages[] = {workflowId, stageId, stageTaskDisplayOrder}` — the
  task's board column (`stageId: 0` = not on the board). Step 3.45 reads the
  column from here; `?include=cards,stages` returns an empty `.included` on
  the task endpoints.
- `assigneeUserIds` (may be `null`) / `assignees[] = {id, type}` — Step 3.45.
- `attachments[] = {id, type: "files"}` — references only; Step 3.7 resolves them.

Filtering:
- If `config.skip_completed_tasks == true` → completed tasks are dropped
  silently from **tasklist** and **subtask** (Step 3.42) iteration. A
  single-task URL pointing at a completed task is kept in the working set and
  Step 3.35 **asks** whether to process it (default: skip).
- Sort tasks by priority then id ascending (stable processing order).

**v1.4.0 — subtasks expansion runs right after Step 3.4** (the tasklist
context fetch). See Step 3.42 — every parent task with subtasks is replaced in
the working set by its subtasks, so the rest of this section (comments,
description split, attachments, …) operates on the expanded set. To recover
v1.3 behaviour set `subtasks.enabled = false`.

If the API returns HTTP 401 → token is invalid. Re-prompt the user for a new token (re-run the first-run flow in Step 2 / Step 2.6 to overwrite `.teamwork.api_token`), save, retry. If 403/404 → report to the user and stop (cannot recover automatically).

### Step 3.3 — Resolve board workflow stages

Fetch the workflow + stages once per unique project id in
`$TW_JOB_DIR/task_projects.tsv` (Step 3). The stages are needed twice: for the
board moves (only when `config.board_workflow.enabled == true`) and for the
Step 3.45 start-column filter (always), so the fetch itself is not gated.

Stage name → id lookup goes through a `name<TAB>id` text table (associative
arrays behave differently in bash 3.2, bash 4+ and zsh). The per-project
result is written to `$TW_JOB_DIR/board_<projectId>.tsv` (`key<TAB>value`
rows), because Steps 3.45, 6.1.5 and 6.8.5 run in later Bash calls where no
variable survives. Every stage of every project also lands in
`$TW_JOB_DIR/stage_names.tsv` (`stageId<TAB>name`) for Steps 3.42 / 3.45.

```bash
# (run preamble — see Step 3)
TASK_PROJECT_FILE="$TW_JOB_DIR/task_projects.tsv"
STAGE_NAMES_FILE="$TW_JOB_DIR/stage_names.tsv"
touch "$STAGE_NAMES_FILE"

BW_ENABLED=$(jq -r '.board_workflow.enabled | if . == null then true else . end' "$CONFIG_FILE")
IN_PROGRESS_NAME=$(jq -r '.board_workflow.in_progress_stage // "In progress"' "$CONFIG_FILE")
DONE_NAME=$(jq -r       '.board_workflow.done_stage // "Done - Local"'        "$CONFIG_FILE")
TODO_MATCH_MODE=$(jq -r '.tasklist_filter.todo_stage_match_mode // "case_sensitive"' "$CONFIG_FILE")

# v1.5.0 start columns, one name per line: `todo_stages` (array) → legacy
# `todo_stage` (string) → default pair. An empty list counts as absent.
# Keep this jq identical in Step 3.45.
TODO_STAGES_CLI=""   # --tasklist-todo-stage for this run (comma-separated); empty = config
TODO_NAMES=$(jq -r '
  (.tasklist_filter.todo_stages
     | if type == "array" then map(select(type == "string" and length > 0))
       elif type == "string" and length > 0 then [.] else [] end
     | if length > 0 then . else null end)
  // (.tasklist_filter.todo_stage | if type == "string" and length > 0 then [.] else null end)
  // ["Ready for Development", "To Do"]
  | .[]' "$CONFIG_FILE")
if [ -n "$TODO_STAGES_CLI" ]; then
  TODO_NAMES=$(printf '%s\n' "$TODO_STAGES_CLI" | tr ',' '\n' \
    | sed -E 's/^[[:space:]]+//; s/[[:space:]]+$//' | grep -v '^$')
fi

# Build FALLBACKS array (bash 3.2-safe: no `mapfile`). Default since 1.5.0:
# "Internal testing", then "Testing" — the done columns of the older boards.
FALLBACKS=()
while IFS= read -r FB; do
  [ -n "$FB" ] && FALLBACKS+=("$FB")
done < <(jq -r '(.board_workflow.done_stage_fallbacks // ["Internal testing", "Testing"])
                | if type == "array" then .[] else . end' "$CONFIG_FILE")

# Lookup helpers — defined and used inside this one snippet. The
# case-insensitive one lowers ASCII only (LC_ALL=C), exactly like jq's
# ascii_downcase that built the table, so a name with an uppercase non-ASCII
# letter ("Čaká …") still finds itself.
lookup_stage_id() {                 # case-insensitive
  local needle=""
  needle=$(printf '%s' "$1" | LC_ALL=C tr '[:upper:]' '[:lower:]')
  awk -F '\t' -v n="$needle" '$1 == n { print $2; exit }' "$STAGE_TABLE_FILE"
}
lookup_stage_id_cs() {              # case-sensitive (Step 3.45 start columns)
  awk -F '\t' -v n="$1" '$1 == n { print $2; exit }' "$STAGE_TABLE_CS_FILE"
}

while IFS= read -r PROJECT_ID; do
  [ -z "$PROJECT_ID" ] && continue
  BOARD_FILE="$TW_JOB_DIR/board_${PROJECT_ID}.tsv"
  # Already resolved in this run — a re-run after Step 3.42 only fetches
  # projects that subtasks added.
  [ -s "$BOARD_FILE" ] && continue

  WORKFLOW_ID=""; IN_PROGRESS_STAGE_ID=""; DONE_STAGE_ID=""; DONE_RESOLVED_NAME=""
  TODO_STAGE_IDS=""; TODO_SUMMARY=""
  STAGE_TABLE_FILE="/tmp/tw_stages_${PROJECT_ID}.tsv"        # lowercase_name<TAB>id
  STAGE_TABLE_CS_FILE="/tmp/tw_stages_cs_${PROJECT_ID}.tsv"  # raw_name<TAB>id

  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
    "${BASE}/projects/api/v3/workflows.json?projectIds=${PROJECT_ID}&include=stages")
  HTTP=${RESP##*$'\n'}; WF_RESP=${RESP%$'\n'*}
  if [ "$HTTP" != "200" ]; then
    echo "  ⚠ GET /projects/api/v3/workflows.json?projectIds=${PROJECT_ID} → HTTP ${HTTP}: $(jq -r '.errors[0].detail // .message // empty' <<<"$WF_RESP" 2>/dev/null) — board moves and the start-column lookup are disabled for project #${PROJECT_ID}" >&2
    WF_RESP='{}'
  fi

  if ! WORKFLOW_ID=$(jq -r '.workflows[0].id // empty' <<<"$WF_RESP"); then
    echo "  ⚠ workflows.json for project #${PROJECT_ID} is not valid JSON — board moves disabled for it" >&2
    WF_RESP='{}'; WORKFLOW_ID=""
  fi

  # Teamwork v3 returns `.included.stages` as an OBJECT keyed by string ID; the
  # stage's own `.value.id` may or may not duplicate the key, so we prefer
  # `.key` as the canonical ID. An array shape is tolerated just in case.
  STAGES_JQ='
    if (.included.stages | type) == "object" then
      .included.stages | to_entries[] | [(.value.name // ""), (.key // (.value.id|tostring))]
    elif (.included.stages | type) == "array" then
      .included.stages[]                  | [(.name // ""), (.id|tostring)]
    else empty end'
  jq -r "$STAGES_JQ"' | "\(.[0] | ascii_downcase)\t\(.[1])"' <<<"$WF_RESP" > "$STAGE_TABLE_FILE"
  jq -r "$STAGES_JQ"' | "\(.[0])\t\(.[1])"'                  <<<"$WF_RESP" > "$STAGE_TABLE_CS_FILE"
  jq -r "$STAGES_JQ"' | "\(.[1])\t\(.[0])"'                  <<<"$WF_RESP" >> "$STAGE_NAMES_FILE"

  # Resolve target stages (case-insensitive — "In progress" also resolves the
  # WAME board's "In Progress"; "Done - Local" falls back to "Internal
  # testing" / "Testing" on the older boards).
  IN_PROGRESS_STAGE_ID=$(lookup_stage_id "$IN_PROGRESS_NAME")
  DONE_STAGE_ID=$(lookup_stage_id "$DONE_NAME")
  if [ -n "$DONE_STAGE_ID" ]; then
    DONE_RESOLVED_NAME="$DONE_NAME"
  else
    for FB in "${FALLBACKS[@]}"; do
      CANDIDATE=$(lookup_stage_id "$FB")
      if [ -n "$CANDIDATE" ]; then
        DONE_STAGE_ID="$CANDIDATE"
        DONE_RESOLVED_NAME="$FB"
        break
      fi
    done
  fi

  # --- v1.5.0: tasklist filter start columns ---------------------------------
  # Every configured start column is looked up per board. `case_sensitive`
  # (default) uses the strict lookup — `to do`, `TO DO`, `ToDo` must NOT match
  # an explicit "To Do"; `case_insensitive` delegates to lookup_stage_id. A
  # board that lacks some start columns is normal (the older per-project
  # boards have no "Ready for Development") — the ids are informational, Step
  # 3.45 matches by name and reports a board with none of them in the plan.
  while IFS= read -r TS; do
    [ -z "$TS" ] && continue
    if [ "$TODO_MATCH_MODE" = "case_insensitive" ]; then
      SID=$(lookup_stage_id "$TS")
    else
      SID=$(lookup_stage_id_cs "$TS")
    fi
    [ -n "$SID" ] && TODO_STAGE_IDS="${TODO_STAGE_IDS:+${TODO_STAGE_IDS},}${SID}"
    TODO_SUMMARY="${TODO_SUMMARY:+${TODO_SUMMARY}, }${TS}=${SID:-—}"
  done <<<"$TODO_NAMES"

  DISABLED=0
  if [ "$BW_ENABLED" != "true" ] || [ -z "$WORKFLOW_ID" ] \
     || { [ -z "$IN_PROGRESS_STAGE_ID" ] && [ -z "$DONE_STAGE_ID" ]; }; then
    DISABLED=1
  fi

  {
    printf 'workflow_id\t%s\n'          "$WORKFLOW_ID"
    printf 'in_progress_stage_id\t%s\n' "$IN_PROGRESS_STAGE_ID"
    printf 'done_stage_id\t%s\n'        "$DONE_STAGE_ID"
    printf 'done_resolved_name\t%s\n'   "$DONE_RESOLVED_NAME"
    printf 'todo_stage_ids\t%s\n'       "$TODO_STAGE_IDS"
    printf 'disabled\t%s\n'             "$DISABLED"
  } > "$BOARD_FILE"

  echo "  ℹ project #${PROJECT_ID}: workflow ${WORKFLOW_ID:-none}, in progress=${IN_PROGRESS_STAGE_ID:-—}, done=${DONE_STAGE_ID:-—} (${DONE_RESOLVED_NAME:-no match}), start columns: ${TODO_SUMMARY:-—}, board moves $( [ "$DISABLED" = "1" ] && echo disabled || echo enabled )" >&2
done < <(cut -f2 "$TASK_PROJECT_FILE" | grep -v '^$' | sort -u)

if [ -z "$(cut -f2 "$TASK_PROJECT_FILE" 2>/dev/null | grep -v '^$')" ]; then
  echo "  ⚠ Step 3.3: no project id in ${TASK_PROJECT_FILE} — board moves and the start-column filter cannot be resolved (did Step 3 run?)" >&2
fi
```

Outcomes (per project, read back from `board_<projectId>.tsv`):
- **Both stages found** → board moves enabled (`workflow_id`, `in_progress_stage_id`, `done_stage_id`).
- **Only `in_progress_stage_id` found** → start move enabled, done move skipped with a warning in the plan ("Project #X has none of the done columns 'Done - Local' / 'Internal testing' / 'Testing' — task will stay in 'In progress' after completion." — name the configured `done_stage` + fallbacks).
- **Only `done_stage_id` found** → start move skipped, done move enabled.
- **Neither / no workflow / `board_workflow.enabled=false`** → `disabled 1`, plan warning *"Project #X has no workflow — board moves disabled."*
- **`todo_stage_ids` empty** (independent of the moves) → the board has none of the start columns; every task of that project ends up `wrong_stage` or `no_card` in Step 3.45, and the plan says so once for the project.

Live check (GET only, 2026-09-24): on the shared *WAME workflow* (59165, e.g.
project 736882) the defaults resolve to *Ready for Development* 300980 and
*To Do* 300903 (start), *In Progress* 300904, *Done - Local* 301170; on the
older board of project 700336 (workflow 58171) to *To Do* 295925 (no *Ready
for Development*), *In progress* 295926 and — via the fallbacks — *Testing*
295928.

After Step 3.42 appended subtasks, run this snippet once more if
`task_projects.tsv` gained a project that has no `board_<id>.tsv` yet
(subtask in a different project than its parent) — resolved projects are
skipped.

Silent degradation: never block the run, never `AskUserQuestion` here. Report state in the plan and the final summary.

### Step 3.35 — Completed single-task URL: ask before processing (v1.5.0)

Runs only when `URL_KIND=task`. `skip_completed_tasks` drops completed tasks
**silently** from tasklist and subtask iteration, but a URL the user pasted
is an explicit choice — and a completed task usually sits in a done column
(*Done - Local*, *Testing*, …) where processing it would pull the card back
out. So the skill asks instead of guessing either way. It runs after
Step 3.3 so the question can name the task's current column.

```bash
# (run preamble — see Step 3)
if [ "$URL_KIND" = "task" ] && [ -s "$TW_JOB_DIR/tasks.json" ]; then
  T_STATUS=$(jq -r '.tasks[0].status // ""' "$TW_JOB_DIR/tasks.json")
  if [ "$T_STATUS" = "completed" ]; then
    T_NAME=$(jq -r '.tasks[0].name // ""' "$TW_JOB_DIR/tasks.json")
    T_SID=$(jq -r '((.tasks[0].workflowStages // [])[0].stageId // 0) | tostring' "$TW_JOB_DIR/tasks.json")
    T_COL="not on the board"
    if [ "$T_SID" != "0" ]; then
      T_COL=$(awk -F '\t' -v id="$T_SID" '$1 == id { print $2; exit }' "$TW_JOB_DIR/stage_names.tsv")
      [ -n "$T_COL" ] || T_COL="column #${T_SID} (name unresolved — Step 3.3)"
    fi
    # printf, not echo — zsh's echo would expand backslashes in the task name.
    printf '  ⚠ [#%s] %s is COMPLETED in Teamwork (board: %s) — ask the user before processing it\n' \
      "$ENTITY_ID" "$T_NAME" "$T_COL" >&2
  else
    echo "  ℹ [#${ENTITY_ID}] status: ${T_STATUS:-unknown} — no completed-task question" >&2
  fi
fi
```

When the snippet printed the `COMPLETED` line, ask via **AskUserQuestion**
before anything else (plan, timer, board move):

> *Task #<id> "<name>" is already completed in Teamwork (board: <column>).
> Processing it reworks a finished task: its card moves out of <column> to
> In progress when work starts and on to the done target (Done - Local,
> fallbacks Internal testing → Testing) after the time log, and time is
> logged on the completed task. The skill does not reopen the task.*

- **Skip it — end the run** *(recommended, default)* — print one line
  (*"[#<id>] completed — skipped at the user's request; nothing changed."*)
  and end the run here: no plan, no timer, no commit, no board move, no
  time log — none of the later steps run.
- **Process it anyway** — continue with Step 3.4. The plan entry's stage
  reads *"<column> — completed task, processing confirmed"*, so the board
  move out of the done column is visible before approval.

Subtasks of a processed parent still follow `skip_completed_tasks` silently
(Step 3.42) — the question is asked once, for the URL the user pasted.

### Step 3.4 — Tasklist description (when `URL_KIND=tasklist`)

Pull the tasklist's own description (fetched in Step 3; `null` when the
tasklist answered 404) and surface it at the top of the plan:

```bash
# (run preamble — see Step 3)
TL_DESC_RAW=$(jq -r '.tasklist.description // empty' "$TW_JOB_DIR/tasks.json")

# Strip HTML tags down to plain text (best-effort; sufficient for plan
# rendering). printf, not echo — zsh's echo expands backslashes in the text.
TASKLIST_DESCRIPTION=$(printf '%s\n' "$TL_DESC_RAW" | sed -E 's|<[^>]+>||g' \
  | tr '\n' ' ' | sed -E 's/[[:space:]]+/ /g; s/^ +//; s/ +$//')
```

If `TASKLIST_DESCRIPTION` is non-empty, render it in Step 4 under a `## Tasklist context` heading.

### Step 3.42 — Expand subtasks (v1.4.0)

Teamwork's data model allows a task to have **subtasks** — child tasks under a
parent task with their own description, acceptance criteria, comments,
attachments, assignees, and stage on the board. A real-world example: a parent
task `[Project bootstrap]` with 8 subtasks (`Set up CI`, `Add login`,
`Wire up DB`, …) where the actual work lives in the subtasks and the parent is
just a container.

Up to v1.3.0 the skill never looked at subtasks. It would happily plan, commit
and time-log against the empty parent, ignoring the 8 real units of work
underneath. v1.4.0 closes that gap: **subtasks are first-class tasks**. After
the initial fetch, the skill expands every parent that has subtasks and
replaces it in the working set with its subtasks. Each subtask then goes
through the rest of the pipeline — filter, planning, implementation, commit,
board move, time log — exactly like a standalone task.

**Decisions baked in (v1.4.0 release):**
- **Parent stays put on the board.** The parent task is **not** moved across
  the workflow and gets **no time log**. It is just a container — the subtasks
  carry the actual work.
- **Per-subtask commits and time logs.** Each subtask gets its own
  `TYPE(scope)[<subtaskId>]: …` commit and its own `POST /tasks/{subtaskId}/time.json`
  entry (sequential, non-overlapping, per Step 5.5 cursor).
- **Per-subtask board moves.** Each subtask moves itself
  *In progress → Done - Local* (fallbacks *Internal testing* → *Testing*) on
  its own board.
- **Filter applies to subtasks.** When Step 3.45 (Tasklist filter) runs after
  this step, it operates on the expanded set — a subtask outside the start
  columns (including one that is not on the board) or assigned to a teammate
  is dropped to `analyse_only` with the same rules as a top-level task.

Skip this step entirely when:
- `config.subtasks.enabled == false` → legacy v1.3.x behaviour (parent stays in
  the working set, subtasks invisible).

> **MANDATORY — do not skip or treat as illustrative.** The v1.4.0/v1.4.1
> builds shipped this step's driver loop as a non-functional stub (fed by
> `< <( true )`), so subtask expansion silently never ran and the parent
> wrongly received the commit, board move and time log. Two things MUST happen
> for real, on every run where a task may have subtasks:
>
> 1. **Materialize the working set to a file.** Before the expansion loop,
>    `WORKING_SET_FILE` must hold one `taskId<TAB>name<TAB>description` row per
>    task fetched in Step 3 — the single parent task for a `URL_KIND=task` run,
>    or every tasklist task for a `URL_KIND=tasklist` run. Since v1.5.0 the
>    Step 3 snippet writes it; the loop below warns when it is empty (an empty
>    file means the expansion is a no-op).
> 2. **Detect via the subtasks endpoint, never a count field.** Always call
>    `GET /tasks/{id}/subtasks.json`. Do **not** trust a `subTasksCount` /
>    `subtaskCount` field from the single-task fetch — Teamwork frequently
>    returns `0` there even when subtasks exist, so relying on it skips the
>    whole expansion. (Step 3.2's single-task fetch deliberately never reads
>    such a field for exactly this reason.)

```bash
# (run preamble — see Step 3)
SUB_ENABLED=$(jq -r    '.subtasks.enabled                | if . == null then true else . end' "$CONFIG_FILE")
SUB_MAX_DEPTH=$(jq -r  '.subtasks.max_depth // 2'                                               "$CONFIG_FILE")
SUB_PARENT_CTX=$(jq -r '.subtasks.include_parent_context | if . == null then true else . end' "$CONFIG_FILE")
SKIP_DONE=$(jq -r      '.skip_completed_tasks            | if . == null then true else . end' "$CONFIG_FILE")
STAGE_NAMES_FILE="$TW_JOB_DIR/stage_names.tsv"      # Step 3.3: stageId<TAB>name
TASK_PROJECT_FILE="$TW_JOB_DIR/task_projects.tsv"   # Step 3: taskId<TAB>projectId
[ -f "$STAGE_NAMES_FILE" ] || : > "$STAGE_NAMES_FILE"

if [ "$SUB_ENABLED" = "true" ]; then
  # The expanded result goes to an auxiliary TSV that Step 3.45's
  # TASK_STAGE_FILE loader merges against.
  #
  # File: /tmp/tw_expanded_${ENTITY_ID}.tsv
  #   format: <taskId>\t<parentTaskId>\t<stageName>\t<assigneesCSV>\t<name>
  #   An empty stage / assignee list is written as "-" (empty TSV fields
  #   collapse under `IFS=$'\t' read`).
  #
  # Every row is a subtask that replaced its parent in the working set.

  EXPANDED_FILE="/tmp/tw_expanded_${ENTITY_ID}.tsv"
  PARENT_CTX_FILE="/tmp/tw_parent_context_${ENTITY_ID}.tsv"
  : > "$EXPANDED_FILE"
  : > "$PARENT_CTX_FILE"

  # Subtasks that count: with skip_completed_tasks=true a completed subtask is
  # ignored, so a parent whose subtasks are all done stays a normal task.
  SUB_COUNT_JQ='[(.tasks // .subtasks // [])[] | select($skip != "true" or .status != "completed")] | length'

  # Recursive expander. Bash 3.2 / zsh-safe — no associative arrays, no
  # `mapfile`, every `local` declared once at the top.
  expand_subtasks() {
    local TID="$1" DEPTH="$2" PARENT_NAME="$3" PARENT_DESC="$4"
    local HTTP="" BODY="" SUB_JSON="" SUB_COUNT="" STID="" S_NAME="" S_DESC=""
    local TMP="${TW_JOB_DIR}/subtasks_${TID}.json"

    if [ "$DEPTH" -gt "$SUB_MAX_DEPTH" ]; then
      return 0
    fi

    # Primary endpoint (v3). Tolerates `.tasks[]` and `.subtasks[]` shapes
    # because v3 has shipped both at different times. This loop fires one GET
    # per task, which can hit Teamwork's rate limit on a big tasklist:
    # `--retry 3` retries HTTP 429 / 5xx with backoff (honouring Retry-After),
    # and the body goes to a file because on stdout curl would concatenate
    # every failed attempt's body in front of the good one.
    HTTP=$(curl -sS --retry 3 --retry-max-time 120 -u "$AUTH" -H "Accept: application/json" \
      -o "$TMP" -w '%{http_code}' \
      "${BASE}/projects/api/v3/tasks/${TID}/subtasks.json?pageSize=100&page=1")
    SUB_JSON=$(cat "$TMP" 2>/dev/null)
    if [ "$HTTP" != "200" ]; then
      echo "  ⚠ GET /projects/api/v3/tasks/${TID}/subtasks.json → HTTP ${HTTP} — trying the parentTaskId fallback" >&2
    elif ! SUB_COUNT=$(jq --arg skip "$SKIP_DONE" "$SUB_COUNT_JQ" <<<"$SUB_JSON"); then
      echo "  ⚠ /tasks/${TID}/subtasks.json is not valid JSON — trying the parentTaskId fallback" >&2
      SUB_COUNT=""
    fi

    # Fallback only when the primary endpoint FAILED (a 200 with 0 rows is a
    # genuine leaf). The filter is the SINGULAR `parentTaskId=`: v3 silently
    # IGNORES the plural `parentTaskIds=` and returns the first 100 tasks of
    # the whole site (verified live: 100 rows, 7 unrelated parent ids,
    # hasMore:true), which the client-side filter below then reduced to a
    # bogus "0 subtasks". The client-side filter to children whose
    # parentTaskId == TID stays as a guard.
    if [ -z "$SUB_COUNT" ]; then
      HTTP=$(curl -sS --retry 3 --retry-max-time 120 -u "$AUTH" -H "Accept: application/json" \
        -o "$TMP" -w '%{http_code}' \
        "${BASE}/projects/api/v3/tasks.json?parentTaskId=${TID}&pageSize=100&page=1")
      BODY=$(cat "$TMP" 2>/dev/null)
      if [ "$HTTP" = "200" ] && SUB_JSON=$(jq --argjson pid "$TID" \
           '.tasks = [((.tasks // .subtasks // [])[] | select((.parentTaskId // 0) == $pid))]' <<<"$BODY"); then
        SUB_COUNT=$(jq --arg skip "$SKIP_DONE" "$SUB_COUNT_JQ" <<<"$SUB_JSON")
      else
        echo "  ⚠ Step 3.42: subtasks of #${TID} could not be read (fallback HTTP ${HTTP}) — #${TID} stays in the working set as a leaf; any subtasks it has are NOT expanded" >&2
        SUB_COUNT=0
      fi
    fi

    if [ "${SUB_COUNT:-0}" = "0" ]; then
      return 0  # leaf (or only completed subtasks) — the parent stays in the working set
    fi

    # Cache parent context (rendered in Step 4 if SUB_PARENT_CTX=true).
    printf "%s\t%s\t%s\n" "$TID" "$PARENT_NAME" "$PARENT_DESC" >> "$PARENT_CTX_FILE"

    # Project id per subtask — v3 has no `projectId` on task objects; it lives
    # in `.tasklist.meta.projectId`. Steps 3.3 / 6.1.5 / 6.8.5 read this file.
    jq -r '(.tasks // .subtasks // [])[]
      | [(.id | tostring), ((.projectId // .tasklist.meta.projectId // "") | tostring)] | @tsv' \
      <<<"$SUB_JSON" >> "$TASK_PROJECT_FILE"

    # One TSV row per subtask. The board column comes from the subtask's own
    # `workflowStages[0].stageId` (0 = not on the board), resolved to a name via
    # Step 3.3's stage table — `?include=cards,stages` returns nothing here.
    # A stage id with no name (Step 3.3 failed or has not seen this project
    # yet) is written as "?<stageId>" — NOT "-": the subtask IS on a board,
    # and Step 3.45 must not mistake an unreadable column for "not on it".
    jq -r --arg PID "$TID" --arg skip "$SKIP_DONE" --rawfile sn "$STAGE_NAMES_FILE" '
      ($sn | split("\n") | map(select(length > 0) | split("\t") | {key: .[0], value: .[1]})
           | from_entries) as $stageNames
      | (.tasks // .subtasks // [])[]
      | select($skip != "true" or .status != "completed")
      | (((.workflowStages // [])[0].stageId // 0) | tostring) as $sid
      | [ (.id | tostring),
          $PID,
          (if $sid == "0" then "-" else ($stageNames[$sid] // ("?" + $sid)) end),
          (((.assigneeUserIds // [(.assignees // [])[]?.id]) | map(tostring) | join(","))
             | if . == "" then "-" else . end),
          (.name // "")
        ] | @tsv
    ' <<<"$SUB_JSON" >> "$EXPANDED_FILE"

    # Recurse into each subtask in case of sub-subtasks. Name + description
    # come from the same response — no extra fetch per child.
    while IFS=$'\t' read -r STID S_NAME S_DESC; do
      [ -z "$STID" ] && continue
      [ "$S_DESC" = "-" ] && S_DESC=""
      expand_subtasks "$STID" $((DEPTH + 1)) "$S_NAME" "$S_DESC"
    done < <(jq -r --arg skip "$SKIP_DONE" '(.tasks // .subtasks // [])[]
      | select($skip != "true" or .status != "completed")
      | [(.id | tostring),
         ((.name // "") | if . == "" then "-" else . end),
         ((.description // "") | gsub("<[^>]+>"; "") | gsub("\\s+"; " ") | .[0:300]
           | if . == "" then "-" else . end)
        ] | @tsv' <<<"$SUB_JSON")
  }

  # Iterate over the working set from Step 3 and expand parents that have
  # subtasks. WORKING_SET_FILE is written by the Step 3 snippet (see the
  # MANDATORY note above): one "taskId<TAB>name<TAB>description" row per task.
  # If the file is missing or empty the expansion is a silent no-op and
  # subtasks are missed (the parent then wrongly keeps the commit / board
  # move / time log).
  WORKING_SET_FILE="/tmp/tw_working_set_${ENTITY_ID}.tsv"
  if [ ! -s "$WORKING_SET_FILE" ]; then
    echo "  ⚠ Step 3.42: WORKING_SET_FILE ($WORKING_SET_FILE) is empty — run the Step 3 fetch first. Subtasks will be MISSED until you do." >&2
  fi

  while IFS=$'\t' read -r TID T_NAME T_DESC; do
    [ -z "$TID" ] && continue
    [ "$T_DESC" = "-" ] && T_DESC=""
    expand_subtasks "$TID" 1 "$T_NAME" "$T_DESC"
  done < "$WORKING_SET_FILE"

  echo "  ℹ Step 3.42: $(cut -f2 "$EXPANDED_FILE" | sort -u | grep -c .) parent(s) expanded into $(grep -c . "$EXPANDED_FILE") subtask(s)" >&2
fi
```

**Working-set replacement (perform every step below for each `EXPANDED_FILE`
row — this is required, not illustrative):**

For each `(subtaskId, parentId, stage, assignees, name)` row in
`EXPANDED_FILE`:

1. **Remove** the row's `parentId` from the implementation working set (the
   parent is no longer treated as work; it stays on the board untouched).
2. **Insert** the row's `subtaskId` as a new working-set task using the same
   fetch + parse pipeline a top-level task would go through (Step 3.5
   comments, Step 3.6 description split, Step 3.7 attachments, Step 3.9 file
   comments, Step 3.10 local discovery — every one of those steps reads from
   the task's own data, so subtasks are picked up automatically once they are
   in the working set).
3. **Carry the stage + assignees** into Step 3.45's `TASK_STAGE_FILE` so the
   filter has the data it needs. The Step 3.45 snippet does this: it drops
   every expanded parent and appends the `EXPANDED_FILE` rows for any subtask
   the tasklist endpoint did not return on its own.
3a. **Carry the project id** — the expander appends `subtaskId<TAB>projectId`
   to `$TW_JOB_DIR/task_projects.tsv` (from `.tasklist.meta.projectId`), so
   Step 3.3's board state and the Step 6.1.5 / 6.8.5 board moves work for
   subtasks too.
4. **Cache parent context** in `PARENT_CTX_FILE`. Step 4 reads it when
   `subtasks.include_parent_context == true` and renders a
   `## Parent context` section for each subtask plan with the parent's name
   plus a truncated description (≤ 300 chars).

**Single-task URL on a parent with subtasks** — `URL_KIND=task` and the user
hands us a parent ID:
- Step 3 fetches just that one task.
- Step 3.42 expands it: the working set becomes the 8 subtasks.
- Step 3.45 is skipped (single-task URLs bypass the filter). Every subtask
  becomes `process_mode=process`.
- The user effectively gets the same behaviour as feeding the skill a tasklist
  URL with 8 tasks in it — minus the tasklist filter.

**Single-task URL on a subtask itself** — user pastes a subtask URL:
- Step 3 fetches that subtask as the single working task.
- Step 3.42 tries to expand it; finds no further subtasks (depth limit + leaf);
  leaves the working set as a single task.
- Pipeline runs as today.

**Edge cases handled:**

1. **`subtasks.enabled = false`** → step is a no-op; v1.3 behaviour.
2. **`SUB_COUNT == 0`** for a task → parent has no (open) subtasks; the
   parent stays in the working set untouched (legacy behaviour preserved per
   task).
3. **API shape variance** — both `.tasks[]` and `.subtasks[]` are accepted.
4. **Endpoint failure** (non-200 or unparsable) on `/tasks/{id}/subtasks.json`
   → fallback to `/tasks.json?parentTaskId={id}` (singular — v3 ignores the
   plural `parentTaskIds`; client-side filtered as well). If
   that fails too, a `⚠` line names the task and it stays a leaf — never a
   silent "0 subtasks".
5. **Runaway recursion** — `max_depth=2` (default) caps at parent →
   subtask → sub-subtask. Increase to 3+ for deeply nested projects.
6. **Subtask in a different project than parent** — Teamwork allows this in
   rare cases. Each subtask carries its own project id (in
   `.tasklist.meta.projectId`) in `task_projects.tsv`; re-run the Step 3.3
   snippet once after this step — it skips resolved projects and fetches only
   the new one. Until then such a subtask's stage is `?<stageId>`; Step 3.45
   resolves it against the refreshed stage table, and one that still does not
   resolve stays `analyse_only` (`stage_unresolved`) — never a silent pass.
7. **`skip_completed_tasks=true`** — applies per subtask: a completed subtask
   is dropped from the expanded set the same way a completed top-level task
   is today. A parent whose subtasks are *all* completed is not expanded.

### Step 3.45 — Tasklist filter: only the start columns + assigned to me

**This step runs only when `URL_KIND=tasklist`. Single-task URLs skip it
entirely** — when a user opens a specific task by ID we trust that intent
and process the task regardless of column or assignee (a completed one only
after Step 3.35 asked).

The motivation is the multi-repo Kanban reality: a single Teamwork project
often contains both a Laravel backend and an Ionic / iOS / Vue frontend, each
owned by a different developer working in a different repository. Without a
filter, running `/teamwork-task <tasklist-url>` from the Laravel repo would
happily pick up the iOS engineer's frontend tasks and start "implementing"
them in the wrong codebase. The filter narrows the *implementation* set to
the tasks the current developer actually owns and has greenlit, while still
fetching the rest for context so the developer can sanity-check teammates'
work in the same plan.

Skip this step entirely when **any** of these hold:
- `URL_KIND != "tasklist"` (single-task URL) — bypass by design.
- `config.tasklist_filter.enabled == false` or `--tasklist-filter=false`.

**v1.4.0 — filter operates on the post-expansion working set.** When
Step 3.42 replaced parent tasks with their subtasks, every subtask runs
through the same stage + assignee rules as a standalone task. A subtask
outside the start columns (off the board included) or assigned to a teammate
is dropped to `analyse_only` exactly like a top-level task would be. Step 3.42 populates
`/tmp/tw_expanded_${ENTITY_ID}.tsv` with per-subtask `stageName` and
`assignees`; if the tasklist endpoint did not return a subtask (typical when
subtasks live outside the parent tasklist), Step 3.45 falls back to that file
when building `TASK_STAGE_FILE` so the filter still has data to work with, and
the expanded parents themselves leave the filter set.

**v1.5.0 — where the column comes from.** Every v3 task object carries its
board column in `workflowStages[0].stageId` (`0` = not on the board). The
filter resolves that id to a name through `$TW_JOB_DIR/stage_names.tsv`
(Step 3.3) and reuses the task list Step 3 already fetched — no second
tasklist call. The 1.3.0–1.4.2 builds read the column from
`?include=cards,stages`, which returns an empty `.included` on this endpoint,
and piped the response through `echo` (broken in zsh), so the stage was
always empty.

**v1.5.0 — start columns; off the board is not a start column.** A task is
implementable when its column is **any of** `tasklist_filter.todo_stages`
(default *Ready for Development* and *To Do* — the two start columns of the
shared WAME board; the older boards simply lack the first one), each name
matched per `todo_stage_match_mode`, plus the assignee rule. Only those
columns are implementable: a task that is **not on the board** (`stageId 0`,
or its project has no workflow) is `analyse_only` with reason `no_card`,
shown in the plan as *"not on the board"* — promote it from the plan when it
is really yours. A column that cannot be named because a GET failed stays
fail-closed `analyse_only` (`stage_unresolved`).

```bash
# (run preamble — see Step 3)
USER_ID="<USER_ID from Step 2.7, empty if unresolved>"
# Booleans: `if . == null` — jq's `// true` would turn an explicit false into true.
TF_ENABLED=$(jq -r     '.tasklist_filter.enabled             | if . == null then true else . end' "$CONFIG_FILE")
TF_ONLY_MINE=$(jq -r   '.tasklist_filter.only_assigned_to_me | if . == null then true else . end' "$CONFIG_FILE")
TF_ANALYZE_ALL=$(jq -r '.tasklist_filter.analyze_all_tasks   | if . == null then true else . end' "$CONFIG_FILE")
TF_TODO_MATCH=$(jq -r  '.tasklist_filter.todo_stage_match_mode // "case_sensitive"'              "$CONFIG_FILE")
# Start columns, one per line — same jq as Step 3.3: `todo_stages` → legacy
# `todo_stage` → default pair; an empty list counts as absent.
TODO_STAGES_CLI=""   # --tasklist-todo-stage for this run (comma-separated); empty = config
TF_TODO_STAGES=$(jq -r '
  (.tasklist_filter.todo_stages
     | if type == "array" then map(select(type == "string" and length > 0))
       elif type == "string" and length > 0 then [.] else [] end
     | if length > 0 then . else null end)
  // (.tasklist_filter.todo_stage | if type == "string" and length > 0 then [.] else null end)
  // ["Ready for Development", "To Do"]
  | .[]' "$CONFIG_FILE")
if [ -n "$TODO_STAGES_CLI" ]; then
  TF_TODO_STAGES=$(printf '%s\n' "$TODO_STAGES_CLI" | tr ',' '\n' \
    | sed -E 's/^[[:space:]]+//; s/[[:space:]]+$//' | grep -v '^$')
fi
# ASCII-only lowering (LC_ALL=C), the same as Step 3.3's case-insensitive lookup.
TF_TODO_STAGES_LC=$(printf '%s\n' "$TF_TODO_STAGES" | LC_ALL=C tr '[:upper:]' '[:lower:]')
# "Ready for Development" / "To Do" — for the messages below and the plan.
TF_TODO_LABEL=$(printf '%s\n' "$TF_TODO_STAGES" | awk 'NF { printf "%s\"%s\"", (n++ ? " / " : ""), $0 }')

if [ "$URL_KIND" != "tasklist" ] || [ "$TF_ENABLED" != "true" ]; then
  # Bypass — every task keeps its default `process_mode=process` from the
  # fetcher in Step 3 and the worker loop runs unchanged.
  echo "  ℹ tasklist filter disabled or single-task URL — processing all fetched tasks." >&2
else
  # Each task's current board column. v3 task objects carry it themselves in
  # `workflowStages[0].stageId` (0 = not on the board); Step 3.3 resolved the
  # project's stage ids to names in stage_names.tsv. (`?include=cards,stages`
  # returns an EMPTY `.included` and `cardId: null` on the tasks endpoints, so
  # the 1.3.0–1.4.2 card lookup gave every task an empty stage.)
  # Build "<taskId>\t<stageName>\t<assigneesCSV>" rows from Step 3's
  # tasks.json — empty stage / assignees are written as "-" so the columns
  # cannot collapse under `IFS=$'\t' read`. A stage id that does not resolve
  # to a name is written as "?<stageId>": the task IS on a board, only its
  # column is unreadable (Step 3.3 failed) — that must not pass as "no card".
  STAGE_NAMES_FILE="$TW_JOB_DIR/stage_names.tsv"
  TASK_STAGE_FILE="/tmp/tw_task_stages_${ENTITY_ID}.tsv"
  [ -s "$STAGE_NAMES_FILE" ] || echo "  ⚠ Step 3.45: ${STAGE_NAMES_FILE} is empty (Step 3.3 not run, or it failed / found no workflow) — tasks on a board become 'stage_unresolved', tasks off the board 'no_card' (both analyse-only)" >&2
  [ -f "$STAGE_NAMES_FILE" ] || : > "$STAGE_NAMES_FILE"

  if ! jq -r --rawfile sn "$STAGE_NAMES_FILE" '
      ($sn | split("\n") | map(select(length > 0) | split("\t") | {key: .[0], value: .[1]})
           | from_entries) as $stageNames
      | .tasks[]
      | (((.workflowStages // [])[0].stageId // 0) | tostring) as $sid
      | [ (.id | tostring),
          (if $sid == "0" then "-" else ($stageNames[$sid] // ("?" + $sid)) end),
          (((.assigneeUserIds // [(.assignees // [])[]?.id]) | map(tostring) | join(","))
             | if . == "" then "-" else . end)
        ] | @tsv' "$TW_JOB_DIR/tasks.json" > "$TASK_STAGE_FILE"; then
    echo "  ⚠ Step 3.45: could not read ${TW_JOB_DIR}/tasks.json — the filter has no data; re-run Step 3 (every task would otherwise be skipped)" >&2
  fi

  # v1.4.0 subtasks: drop every parent that Step 3.42 replaced, then append
  # the subtask rows the tasklist endpoint did not return on its own.
  EXPANDED_FILE="/tmp/tw_expanded_${ENTITY_ID}.tsv"
  if [ -s "$EXPANDED_FILE" ]; then
    awk -F '\t' -v OFS='\t' '
      FNR == NR { parent[$2] = 1; row[$1] = $1 OFS $3 OFS $4; order[++n] = $1; next }
      !($1 in parent) { print; seen[$1] = 1 }
      END { for (i = 1; i <= n; i++) { id = order[i]
              if (!(id in seen) && !(id in parent)) { print row[id]; seen[id] = 1 } } }
    ' "$EXPANDED_FILE" "$TASK_STAGE_FILE" > "${TASK_STAGE_FILE}.merged" \
      && mv "${TASK_STAGE_FILE}.merged" "$TASK_STAGE_FILE"
  fi

  # A "?<stageId>" that Step 3.42 wrote before Step 3.3 knew the subtask's
  # project resolves here once Step 3.3 was re-run. (Guarded by -s: with an
  # empty first file awk's FNR == NR would swallow the second file.)
  if [ -s "$STAGE_NAMES_FILE" ]; then
    awk -F '\t' -v OFS='\t' '
      FNR == NR { name[$1] = $2; next }
      substr($2, 1, 1) == "?" && (substr($2, 2) in name) { $2 = name[substr($2, 2)] }
      { print }
    ' "$STAGE_NAMES_FILE" "$TASK_STAGE_FILE" > "${TASK_STAGE_FILE}.resolved" \
      && mv "${TASK_STAGE_FILE}.resolved" "$TASK_STAGE_FILE"
  fi

  # For each task, set process_mode:
  #   "process"           → fully run (implement, commit, log, board move)
  #   "analyse_only"      → fetch + plan-time analysis, but worker loop skips
  #
  # Reason codes (rendered in the plan as "Skip reason: …"):
  #   "wrong_stage(<name>)" → the column is none of the start columns
  #   "wrong_assignee"    → assignees do not include current user
  #   "wrong_stage+wrong_assignee" → both conditions failed
  #   "no_card"           → task is not on the board (stageId 0) or its
  #                         project has no workflow — it has no start column,
  #                         so it is analyse_only (1.5.0; the plan says "not
  #                         on the board"; promote it there if it is yours)
  #   "stage_unresolved(<id>)" → the task IS on a board but its stage id has
  #                         no name (Step 3.3 failed or never saw the
  #                         project) — fail closed: analyse_only, never a pass

  # PROCESS_FILE rows: taskId<TAB>mode<TAB>detail — detail is the matched
  # start column for `process` (the plan's "stage:"), the reason codes above
  # for `analyse_only` / `drop`.
  PROCESS_FILE="/tmp/tw_tasks_process_${ENTITY_ID}.tsv"
  : > "$PROCESS_FILE"

  while IFS=$'\t' read -r T_ID T_STAGE T_ASSIGNEES; do
    [ -z "$T_ID" ] && continue
    [ "$T_STAGE" = "-" ] && T_STAGE=""
    [ "$T_ASSIGNEES" = "-" ] && T_ASSIGNEES=""

    REASONS=""

    # --- Stage check -------------------------------------------------------
    # Implementable only in one of the start columns. Off the board and an
    # unreadable column both fail (see reason codes above).
    STAGE_OK=0
    if [ -z "$T_STAGE" ]; then
      REASONS="no_card"
    elif [ "${T_STAGE:0:1}" = "?" ]; then
      REASONS="stage_unresolved(${T_STAGE:1})"
    else
      if [ "$TF_TODO_MATCH" = "case_insensitive" ]; then
        T_STAGE_CMP=$(printf '%s' "$T_STAGE" | LC_ALL=C tr '[:upper:]' '[:lower:]')
        STAGE_LIST="$TF_TODO_STAGES_LC"
      else
        # case_sensitive (default) — strict string equality per column
        T_STAGE_CMP="$T_STAGE"
        STAGE_LIST="$TF_TODO_STAGES"
      fi
      while IFS= read -r TS; do
        [ -n "$TS" ] && [ "$T_STAGE_CMP" = "$TS" ] && { STAGE_OK=1; break; }
      done <<<"$STAGE_LIST"
      [ "$STAGE_OK" -eq 0 ] && REASONS="wrong_stage(${T_STAGE})"
    fi

    # --- Assignee check ---------------------------------------------------
    ASSIGNEE_OK=1
    if [ "$TF_ONLY_MINE" = "true" ]; then
      if [ -z "$USER_ID" ]; then
        # Step 2.7 already warned; treat as "skip assignee check" so we don't
        # accidentally lock the user out of their own work.
        ASSIGNEE_OK=1
      else
        # Comma-separated list of numeric IDs; membership check.
        case ",${T_ASSIGNEES}," in
          *",${USER_ID},"*) ASSIGNEE_OK=1 ;;
          *)
            ASSIGNEE_OK=0
            REASONS="${REASONS:+${REASONS}+}wrong_assignee"
            ;;
        esac
      fi
    fi

    # --- Decide mode ------------------------------------------------------
    if [ "$STAGE_OK" -eq 1 ] && [ "$ASSIGNEE_OK" -eq 1 ]; then
      printf "%s\tprocess\t%s\n" "$T_ID" "$T_STAGE" >> "$PROCESS_FILE"
    else
      if [ "$TF_ANALYZE_ALL" = "true" ]; then
        printf "%s\tanalyse_only\t%s\n" "$T_ID" "$REASONS" >> "$PROCESS_FILE"
      else
        # Strict mode — drop the task from the plan entirely.
        printf "%s\tdrop\t%s\n" "$T_ID" "$REASONS" >> "$PROCESS_FILE"
      fi
    fi
  done < "$TASK_STAGE_FILE"

  # Counters for the plan banner and the final summary.
  TF_COUNT_PROCESS=$(awk -F '\t' '$2 == "process"      { c++ } END { print c+0 }' "$PROCESS_FILE")
  TF_COUNT_ANALYSE=$(awk -F '\t' '$2 == "analyse_only" { c++ } END { print c+0 }' "$PROCESS_FILE")
  TF_COUNT_DROP=$(   awk -F '\t' '$2 == "drop"         { c++ } END { print c+0 }' "$PROCESS_FILE")
  TF_COUNT_TOTAL=$(  awk -F '\t' 'END { print NR+0 }'                       "$PROCESS_FILE")

  TF_COUNT_NO_CARD=$(awk -F '\t' '$2 != "process" && $3 ~ /no_card/ { c++ } END { print c+0 }' "$PROCESS_FILE")
  echo "  ℹ tasklist filter (start columns ${TF_TODO_LABEL}): ${TF_COUNT_PROCESS} to implement, ${TF_COUNT_ANALYSE} analyse-only, ${TF_COUNT_DROP} dropped (${TF_COUNT_NO_CARD} of the skipped not on the board)" >&2
  TF_COUNT_UNRESOLVED=$(awk -F '\t' '$3 ~ /stage_unresolved/ { c++ } END { print c+0 }' "$PROCESS_FILE")
  if [ "$TF_COUNT_UNRESOLVED" -gt 0 ]; then
    echo "  ⚠ ${TF_COUNT_UNRESOLVED} task(s) sit in a board column whose name could not be resolved (stage_unresolved) — kept out of the implement set; re-run Step 3.3, then this step" >&2
  fi

  # --- Empty-result handling --------------------------------------------
  # If the filter dropped EVERY task into analyse_only/drop, surface a
  # detailed explanation of WHICH rule disqualified each task so the user
  # can decide whether to widen the filter or pick something to promote.
  # The message itself is rendered here in stderr; Step 4 short-circuits
  # to a dedicated AskUserQuestion when TF_COUNT_PROCESS == 0.
  if [ "$TF_COUNT_PROCESS" = "0" ]; then
    USER_HINT="$USER_ID"
    [ -z "$USER_HINT" ] && USER_HINT="(unresolved — assignee check was skipped)"

    {
      echo ""
      echo "  ⚠ Tasklist filter found 0 tasks to implement out of ${TF_COUNT_TOTAL} fetched."
      echo "    Rules that were applied:"
      echo "      • URL kind        = tasklist                      (single-task URLs bypass the filter)"
      echo "      • Start columns   = ${TF_TODO_LABEL}             (any of them; match mode: ${TF_TODO_MATCH}; off the board = not implementable)"
      echo "      • Assignee check  = $( [ "$TF_ONLY_MINE" = "true" ] && echo "on (must include user ${USER_HINT})" || echo "off" )"
      echo "      • analyze_all     = ${TF_ANALYZE_ALL}             (false would have dropped the rest entirely)"
      echo ""
      echo "    Per-task reasons (top 10):"
      awk -F '\t' '$2 != "process" { printf "      • [#%s] %s\n", $1, $3 }' "$PROCESS_FILE" | head -n 10
      if [ "$TF_COUNT_TOTAL" -gt 10 ]; then
        echo "      • … and $(( TF_COUNT_TOTAL - 10 )) more (see plan section)"
      fi
      echo ""
      echo "    Most common causes:"
      echo "      - the tasklist's tasks are sitting in a different column"
      echo "        (e.g. \"Backlog\", \"In progress\"); pass --tasklist-todo-stage=\"Backlog\""
      echo "      - the tasks are not on the board (no_card) or the project has no"
      echo "        workflow; promote them in Step 4 or pass --tasklist-filter=false"
      echo "      - the tasks are assigned to teammates; pass --tasklist-only-mine=false"
      echo "      - the project workflow uses a different casing"
      echo "        (e.g. \"to do\" vs \"To Do\"); set tasklist_filter.todo_stage_match_mode=case_insensitive"
      echo "      - you simply have nothing assigned in ${TF_TODO_LABEL} right now"
      echo ""
      echo "    Step 4 will ask how to proceed — disable the filter, promote a"
      echo "    specific task from the analyse-only list, change the start columns,"
      echo "    or cancel the run."
      echo ""
    } >&2
  fi
fi
```

**Edge cases the filter has to handle gracefully:**

1. **`USER_ID` unresolved** (Step 2.7 returned empty) → skip the assignee
   check entirely so the user is never silently locked out of their own
   tasklist by a permission glitch. Log one warning line. The stage check
   still applies.
2. **Project has no workflow** (Step 3.3 wrote `disabled 1` and no stage
   names) → every task ends up with empty `T_STAGE`: no task has a start
   column, so the whole tasklist is `analyse_only` (`no_card`) and Step 4.0a
   asks how to proceed (disable the filter for this run, promote tasks, …).
   The plan carries a one-line note that the project has no board. Nothing
   is implemented without that explicit choice.
3. **Task has no card / not on the board** (`workflowStages[0].stageId == 0`)
   → `analyse_only` with `no_card` (or `no_card+wrong_assignee`); the plan
   shows it as *"not on the board"*. Since 1.5.0 only the start columns are
   implementable — a backlog item nobody moved onto the board is not
   greenlit work. Promote it from the plan (Step 4) when it is yours.
   Subtasks follow the same rule.
3a. **Task is on a board, but its column cannot be named**
   (`stageId != 0` with no entry in `stage_names.tsv` — the Step 3.3
   workflows GET failed, e.g. HTTP 429, or never saw the task's project) →
   **fail closed**: `analyse_only` with `stage_unresolved(<stageId>)` and a
   `⚠` line — the column exists and may well be the wrong one; passing it
   would implement tasks from arbitrary columns. Re-run Step 3.3, then this
   step.
3b. **Board without some start columns** — the older per-project boards have
   *To Do* but no *Ready for Development*; the missing name simply never
   matches. A board with **none** of them (Step 3.3 `todo_stage_ids` empty)
   makes every task of that project `wrong_stage` / `no_card` — the plan
   names the project once.
4. **`analyze_all_tasks=false`** (strict mode) → tasks that fail the filter
   are *dropped* from `TASKS_TO_PROCESS` entirely — not even rendered in the
   plan. Use when the user does not want teammates' tasks polluting the
   overview.
5. **The user passes a `tasks` URL but the task sits in a start column
   owned by someone else, in another column, or off the board** → still
   processed (single-task URL bypass from the very top of this step). The
   snippet prints one informational line (*"tasklist filter disabled or
   single-task URL — processing all fetched tasks"*) so the user is aware. A **completed** task is the one
   exception: Step 3.35 asks first (default: skip).

Later steps (Step 3.5 fetch comments, Step 3.7 attachments, Step 3.10 local
discovery) still run for **every** task that survived the filter — including
`analyse_only` ones — because the analysis output in the plan should be
informed by the same context. Only the **implementation side-effects**
(commits, time logs, board moves, attachment cleanup that is per-task) are
gated by `process_mode` in Step 6.

### Step 3.5 — Fetch comments per task (count probe + newest always, full thread when needed)

Read `fetch_comments_mode` from config. The rule this step implements is the
one Step 6.2 relies on: **the last comment is the freshest truth** — a
comment posted after the description can change the spec. So whenever a task
has comments, the newest one is always read; the mode only decides whether
the *full thread* is fetched.

- **`always`** → full thread for every task that has comments.
- **`never`** → skip entirely (no probe either); the plan says *"comments not
  read (fetch_comments_mode=never)"*.
- **`when_needed`** (default) → one cheap probe per task returns the comment
  **count** and the **newest comment** in a single call:
  1. `count == 0` → nothing to read.
  2. `count >= 1` → the newest comment is **always** read (the probe already
     returned it).
  3. `count > 1` → the full thread is fetched as well when at least one of
     these holds (`THIN_DESCRIPTION=1`):
     - `final_summary` is empty (no HR found in description — see Step 3.6), OR
     - `len(strip_html(acceptance_criteria)) < 100`, OR
     - The description matches `(viď|see|viz)[[:space:]]+(komentár|comment|comments|nižšie|below)` (case-insensitive), OR
     - the newest comment itself refers back to earlier ones (*"ako som písal
       vyššie"*, *"see above"*) or contradicts the description. You only know
       this after the first run has read it (`comments_<taskId>.json`), so
       re-run the snippet with `THIN_DESCRIPTION="1"` in that case — the
       probe is one cheap request.

Up to 1.4.2 the mode gated on a comment-count field v3 never returns, so
`when_needed` never fetched anything, and even `always` asked for a sort key
v3 does not know — HTTP 400 `orderBy: unknown comment sort.`, whose error body
was then parsed as an empty list. Valid v3 sorts are `orderBy=date`
(chronological, `orderMode=asc|desc`) and `orderBy=id`; the timestamp field
is `postedDateTime`.

v3 / v1 field map (the v1 endpoint is the fallback when v3 fails):

| Datum | v3 (`/projects/api/v3/tasks/{id}/comments.json`) | v1 (`/tasks/{id}/comments.json`) |
| --- | --- | --- |
| time | `postedDateTime` (`2026-08-14T09:32:28Z`) | `datetime` (`post-date` is null) |
| HTML body | `htmlBody` (plain text: `body`) | `html-body` (plain text: `body`) |
| author | `postedByUserId` / `postedBy` (id only) | `author-id`, `author-firstname`, `author-lastname` |
| files | `files[] = {id, type}`, resolved via `?include=files` → `.included.files` (`fileIds` may be null) | `attachments[]` (`id`, `name`, `size`) |
| count | `.meta.page.count` of the comments list | `."todo-item"."comments-count"` of `GET /tasks/{id}.json` |
| order | `orderBy=date&orderMode=asc` | none guaranteed — always `sort_by(.datetime)` |

Both are normalized into `$TW_JOB_DIR/comments_<taskId>.json` — a
chronological array of `{id, postedDateTime, postedByUserId, author,
htmlBody, body, files: [{id, name, size, downloadURL}]}` (v3 field names) that
Steps 3.8, 6.0 and 6.2 read.

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
THIN_DESCRIPTION="<1 if any when_needed rule above holds, else 0>"
MODE=$(jq -r '.fetch_comments_mode // "when_needed"' "$CONFIG_FILE")
COMMENTS_FILE="$TW_JOB_DIR/comments_${TASK_ID}.json"
RAW_FILE="${COMMENTS_FILE}.jsonl"; : > "$RAW_FILE"
COMMENTS_COUNT=""; COMMENTS_SOURCE="v3"; FULL=0

# jq normalizers → one comment object per line, v3 field names.
V3_NORM='(.included.files // {}) as $f | .comments[]? | {
  id, postedDateTime, postedByUserId: (.postedByUserId // .postedBy), author: null,
  htmlBody: (.htmlBody // ""), body: (.body // ""),
  files: [ (.files // [])[] | ($f[(.id | tostring)] // {}) as $d | {
    id, name: ($d.originalName // $d.displayName // $d.name // ("file_" + (.id | tostring))),
    size: ($d.size // 0), downloadURL: ($d.downloadURL // "") } ] }'
V1_NORM='.comments[]? | {
  id: (.id | tonumber), postedDateTime: .datetime,
  postedByUserId: ((."author-id" // "0") | tonumber),
  author: ([."author-firstname", ."author-lastname"] | map(select(. != null and . != "")) | join(" ")),
  htmlBody: (."html-body" // ""), body: (.body // ""),
  files: [ (.attachments // [])[] | {
    id: (.id | tonumber), name: (.name // .filename // ("file_" + .id)),
    size: ((.size // "0") | tonumber), downloadURL: "" } ] }'

if [ "$MODE" = "never" ]; then
  COMMENTS_STATE="not read (fetch_comments_mode=never)"
else
  # 1) Probe — count + newest comment in ONE call (v3).
  RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
    "${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json?pageSize=1&page=1&orderBy=date&orderMode=desc&include=files")
  HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
  if [ "$HTTP" = "200" ] && COMMENTS_COUNT=$(jq -e '.meta.page.count' <<<"$BODY"); then
    jq -c "$V3_NORM" <<<"$BODY" > "$RAW_FILE"            # the newest comment
  else
    echo "  ⚠ GET /projects/api/v3/tasks/${TASK_ID}/comments.json (count probe) → HTTP ${HTTP}: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — falling back to the v1 endpoints" >&2
    COMMENTS_SOURCE="v1"
    RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' "${BASE}/tasks/${TASK_ID}.json")
    HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
    if [ "$HTTP" = "200" ] && COMMENTS_COUNT=$(jq -e '."todo-item"."comments-count" | tonumber' <<<"$BODY"); then
      :
    else
      echo "  ⚠ GET /tasks/${TASK_ID}.json (v1 comments-count) → HTTP ${HTTP} — count unknown, reading the full thread instead of assuming 0" >&2
      COMMENTS_COUNT="unknown"
    fi
    FULL=1   # v1 has no newest-only probe; read the (small) thread
  fi

  # 2) Decide whether the full thread is needed.
  if [ "$COMMENTS_COUNT" = "unknown" ]; then
    FULL=1
  elif [ "$COMMENTS_COUNT" -eq 0 ]; then
    FULL=0
  elif [ "$MODE" = "always" ] || { [ "$COMMENTS_COUNT" -gt 1 ] && [ "$THIN_DESCRIPTION" = "1" ]; }; then
    FULL=1
  fi

  # 3) Full thread — v3 chronological, paginated; v1 when v3 fails.
  if [ "$FULL" = "1" ] && [ "$COMMENTS_SOURCE" = "v3" ]; then
    : > "$RAW_FILE"
    PAGE=1
    while :; do
      RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
        "${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json?pageSize=100&page=${PAGE}&orderBy=date&orderMode=asc&include=files")
      HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
      if [ "$HTTP" != "200" ] || ! jq -c "$V3_NORM" <<<"$BODY" >> "$RAW_FILE"; then
        echo "  ⚠ GET /projects/api/v3/tasks/${TASK_ID}/comments.json?page=${PAGE} → HTTP ${HTTP} — re-reading the thread from the v1 endpoint" >&2
        COMMENTS_SOURCE="v1"
        break
      fi
      [ "$(jq -r '.meta.page.hasMore // false' <<<"$BODY")" = "true" ] || break
      PAGE=$((PAGE + 1))
    done
  fi
  if [ "$FULL" = "1" ] && [ "$COMMENTS_SOURCE" = "v1" ]; then
    : > "$RAW_FILE"
    PAGE=1
    while :; do
      RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
        "${BASE}/tasks/${TASK_ID}/comments.json?page=${PAGE}&pageSize=100")
      HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
      if [ "$HTTP" != "200" ] || ! jq -c "$V1_NORM" <<<"$BODY" >> "$RAW_FILE"; then
        echo "  ⚠ GET /tasks/${TASK_ID}/comments.json?page=${PAGE} (v1) → HTTP ${HTTP} — comments for #${TASK_ID} are INCOMPLETE; say so in the plan" >&2
        break
      fi
      [ "$(jq '.comments | length' <<<"$BODY")" -lt 100 ] && break
      PAGE=$((PAGE + 1))
    done
  fi

  # 4) Normalize: chronological, deduplicated.
  READ_COUNT=$(grep -c . "$RAW_FILE")
  if [ "$COMMENTS_COUNT" = "0" ]; then
    COMMENTS_STATE="none (0 comments)"
  elif [ "$FULL" = "1" ]; then
    COMMENTS_STATE="${READ_COUNT} of ${COMMENTS_COUNT} comments read — full thread (${COMMENTS_SOURCE})"
  else
    COMMENTS_STATE="newest of ${COMMENTS_COUNT} read — full thread not fetched (description self-contained)"
  fi
fi
jq -s 'unique_by(.id) | sort_by(.postedDateTime)' "$RAW_FILE" > "$COMMENTS_FILE"
echo "  ℹ [#${TASK_ID}] comments: ${COMMENTS_STATE}" >&2
```

`COMMENTS_STATE` is what the plan's **Comments context** line starts with
(Step 4). If `READ_COUNT` is lower than a known `COMMENTS_COUNT` after a full
read, the plan says so — never render a partial thread as the whole story.

Strip HTML from `htmlBody` (or use the plain `body`) for context, keep author +
`postedDateTime`. Keep comments in chronological order — the **last comment is
the freshest truth** when it conflicts with earlier ones or with the
description.

For each fetched comment, `files[]` (name, size, `downloadURL`) feeds Step 3.8.

### Step 3.6 — Parse task description (acceptance criteria + final summary)

Project convention: the task description is split by a horizontal rule (`<hr>`, `<hr/>`, `<hr />` in HTML, or a Markdown `---` / `***` / `___` on its own line). The content **above** the HR is the **acceptance criteria** (what must be true to consider the task done); the content **below** the HR is the **final summary / decision** — treat this as the authoritative goal.

**Canonical WAME format first (v1.5.0).** When the description has an
`## Akceptačné kritériá` / `## Acceptance criteria` heading — the format
`teamwork-task-analyze`, `-from-desk`, `-from-session` and `-from-dnr` write
(`[preamble] → HR → AC → HR → Cieľ → HR → Technický popis`) —
`acceptance_criteria` is the block from that heading to the next HR,
**including** its `### Prierezové požiadavky` / `### Cross-cutting requirements`
sub-block (Step 6.2 takes those items as given requirements for the matching
dimension); `final_summary` is everything below that HR; any text above the
first HR is the reporter's preamble — context, not criteria. The first-HR split
below would hand the preamble (or nothing, when the description starts with the
HR) to Step 6 as the checklist. `/teamwork-task-test` locates the block the
same way. Otherwise split the description on the **first** HR occurrence:
- If exactly one HR is found → `acceptance_criteria = above`, `final_summary = below`.
- If no HR is found → treat the whole description as `acceptance_criteria`, leave `final_summary` empty (warn in the plan that no final summary was provided).
- If multiple HRs → split on the first, ignore the rest (they are likely inside the final summary).

Use the **`final_summary`** (when present) as the primary goal for planning and the commit message body. Use the `acceptance_criteria` to know when to stop and what to verify.

A simple parser sketch (regex-friendly):
```bash
# strips HTML <hr> variants and matches markdown HR on own line
SPLIT_REGEX='(<hr[[:space:]]*/?>|^[[:space:]]*(---|\*\*\*|___)[[:space:]]*$)'
```

### Step 3.7 — Fetch task attachments

If `config.fetch_attachments == true`, fetch the file list for each task and download into a local working folder.

The attachments live on the task object: `.task.attachments` holds
`{id, type: "files"}` references, and `?include=attachments` resolves them in
`.included.files` (keyed by id: `originalName`, `displayName`, `size`,
`downloadURL`). The endpoints used up to 1.4.2 are unusable:
`/projects/api/v3/tasks/{id}/files.json` answers **404**, and the
`/projects/api/v3/files.json?taskIds={id}` "fallback" **ignores the taskIds
filter** and returns the whole workspace's files (100 unrelated files per
page) — never use it. `/projects/api/v3/files/{id}/download` is a 404 as well;
the download URL comes from `downloadURL` (or `GET
/projects/api/v3/files/{id}.json` → `.file.downloadURL`) and accepts the same
Basic auth.

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
ATTACH_DIR="./teamwork-task-${TASK_ID}"
rm -rf "$ATTACH_DIR"          # clean any leftover from a prior failed run
mkdir -p "$ATTACH_DIR"

MAX_MB=$(jq -r '.max_attachment_size_mb // 25' "$CONFIG_FILE")
MAX_BYTES=$(( MAX_MB * 1024 * 1024 ))
FILES_LIST="$TW_JOB_DIR/files_${TASK_ID}.tsv"    # id<TAB>name<TAB>size<TAB>downloadURL
SKIPPED_FILE="$TW_JOB_DIR/skipped_attachments.txt"   # rendered in Step 7

RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}.json?include=attachments")
HTTP=${RESP##*$'\n'}; BODY=${RESP%$'\n'*}
if [ "$HTTP" != "200" ]; then
  echo "  ⚠ GET /projects/api/v3/tasks/${TASK_ID}.json?include=attachments → HTTP ${HTTP}: $(jq -r '.errors[0].detail // .message // empty' <<<"$BODY" 2>/dev/null) — continuing WITHOUT task attachments" >&2
  : > "$FILES_LIST"
elif ! jq -r '(.included.files // {}) as $f
    | (.task.attachments // [])[]
    | ($f[(.id | tostring)] // {}) as $d
    | [ (.id | tostring),
        (($d.originalName // $d.displayName // $d.name // ("file_" + (.id | tostring))) | gsub("/"; "_")),
        (($d.size // 0) | tostring),
        ($d.downloadURL // "") ] | @tsv' <<<"$BODY" > "$FILES_LIST"; then
  echo "  ⚠ attachment list of #${TASK_ID} is not valid JSON — continuing WITHOUT task attachments" >&2
  : > "$FILES_LIST"
fi
echo "  ℹ [#${TASK_ID}] $(grep -c . "$FILES_LIST") task attachment(s)" >&2

# For each file: skip oversized ones; download by ID-prefixed filename to avoid
# collisions. The loop reads a file (not a pipe), so nothing runs in a subshell.
while IFS=$'\t' read -r FID FNM FSZ URL; do
  [ -z "$FID" ] && continue
  if [ "${FSZ:-0}" -gt "$MAX_BYTES" ]; then
    echo "  ⚠ skipped attachment '${FNM}' (${FSZ} bytes > ${MAX_BYTES})" >&2
    printf '%s (%s bytes, task %s)\n' "$FNM" "$FSZ" "$TASK_ID" >> "$SKIPPED_FILE"
    continue
  fi
  if [ -z "$URL" ]; then
    URL=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
      "${BASE}/projects/api/v3/files/${FID}.json" | jq -r '.file.downloadURL // empty')
  fi
  if [ -z "$URL" ]; then
    echo "  ⚠ no download URL for attachment '${FNM}' (file ${FID}) — skipped" >&2
    continue
  fi
  DL=$(curl -sS -L -u "$AUTH" -o "${ATTACH_DIR}/${FID}_${FNM}" -w '%{http_code}' "$URL")
  [ "$DL" = "200" ] || echo "  ⚠ failed to download attachment '${FNM}' (HTTP ${DL})" >&2
done < "$FILES_LIST"
```

If the list cannot be read, the `⚠` line says so and the task continues without attachments (never a silent "0 files").

**Filename hint fall-through.** If the task description (or any fetched
comment) mentions a filename or extension (regex from
`config.readiness_gate.filename_hint_pattern`, e.g. `vzor.docx`,
`prilohy.pdf`, `data.xlsx`) **and** the task ended up with zero downloaded
attachments, set an in-memory flag `FILENAME_HINT_PRESENT=1` for this task.
That flag forces Step 3.10 (local working-tree discovery) to run for this
task **even if `local_context_discovery.enabled` is false** — somebody clearly
referenced a file; either it was attached to the wrong task, lives in the
project folder, or is still on the way. Pretending it doesn't exist is the
worst outcome.

### Step 3.8 — Fetch comment attachments

For each comment normalized in Step 3.5 (`$TW_JOB_DIR/comments_<taskId>.json`),
download its `files[]` to a `comments/` subdirectory inside `$ATTACH_DIR`:

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
ATTACH_DIR="./teamwork-task-${TASK_ID}"
COMMENTS_FILE="$TW_JOB_DIR/comments_${TASK_ID}.json"
SKIPPED_FILE="$TW_JOB_DIR/skipped_attachments.txt"
MAX_BYTES=$(( $(jq -r '.max_attachment_size_mb // 25' "$CONFIG_FILE") * 1024 * 1024 ))
mkdir -p "${ATTACH_DIR}/comments"

while IFS=$'\t' read -r CID FID FNM FSZ URL; do
  [ -z "$FID" ] && continue
  if [ "${FSZ:-0}" -gt "$MAX_BYTES" ]; then
    echo "  ⚠ skipped comment attachment '${FNM}' (${FSZ} bytes > ${MAX_BYTES})" >&2
    printf '%s (%s bytes, comment %s)\n' "$FNM" "$FSZ" "$CID" >> "$SKIPPED_FILE"
    continue
  fi
  # v1-sourced comments carry no URL — resolve it from the v3 file record.
  if [ -z "$URL" ]; then
    URL=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
      "${BASE}/projects/api/v3/files/${FID}.json" | jq -r '.file.downloadURL // empty')
  fi
  if [ -z "$URL" ]; then
    echo "  ⚠ no download URL for comment attachment '${FNM}' (file ${FID}) — skipped" >&2
    continue
  fi
  DL=$(curl -sS -L -u "$AUTH" -o "${ATTACH_DIR}/comments/${CID}_${FID}_${FNM}" -w '%{http_code}' "$URL")
  [ "$DL" = "200" ] || echo "  ⚠ failed to download comment attachment '${FNM}' (HTTP ${DL})" >&2
done < <(jq -r '.[] | .id as $cid | .files[]?
    | [($cid | tostring), (.id | tostring), (.name | gsub("/"; "_")), (.size | tostring), (.downloadURL // "")]
    | @tsv' "$COMMENTS_FILE")
```

Same size limit (`max_attachment_size_mb`) applies if `size` is present on the comment file entry.

### Step 3.9 — Fetch file comments

If `config.fetch_file_comments == true`, for each file already collected in Step 3.7 (the ids in `$TW_JOB_DIR/files_<taskId>.tsv`) fetch its comments and surface a short digest in the per-task plan context:

```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/files/${FID}/comments.json"
```

Render in the plan as *"File comments on <filename>: 2 — <last comment digest>"*. Keep it terse — full comment bodies clutter the plan.

### Step 3.10 — Local working-tree discovery

Tasks frequently reference materials that live in the **project folder**
rather than as Teamwork attachments — client-shared DNR / Q&A docs in a
project-named directory (e.g. `Strečnianska/DNR_Strecnianska_v1.2.docx`),
sample emails dropped in `samples/`, reference exports in `docs/`, SQL dumps
in `dáta db/`. Without this step the skill would happily run with a synthetic
fixture for a task that says *"vzor doručí klient"*, even though
`Strečnianska/Bmail o pohybe na ucte - vzor.docx` was sitting one directory
above the cwd the whole time. v1.1.3 closes that gap.

This step runs **once per session** (not per task) when:
- `config.local_context_discovery.enabled == true` (default), OR
- any task in the list set `FILENAME_HINT_PRESENT=1` in Step 3.7 (the
  fall-through that fires when a description names a file but no attachment
  was downloaded).

```bash
# (run preamble — see Step 3)
LCD_ENABLED=$(jq -r '.local_context_discovery.enabled | if . == null then true else . end' "$CONFIG_FILE")

# Honour the fall-through from Step 3.7 even when explicitly disabled.
if [ "$LCD_ENABLED" != "true" ] && [ "${FILENAME_HINT_PRESENT_ANY:-0}" != "1" ]; then
  echo "  ℹ Local context discovery disabled — skipping working-tree scan." >&2
else
  MAX_DEPTH=$(jq -r '.local_context_discovery.max_depth // 3' "$CONFIG_FILE")
  MIN_SCORE=$(jq -r '.local_context_discovery.min_keyword_score // 1' "$CONFIG_FILE")
  MAX_OFFER=$(jq -r '.local_context_discovery.max_files_to_offer // 12' "$CONFIG_FILE")

  # Keyword set: tasklist name + every task name + every filename hint from
  # descriptions — read from the task list Step 3 wrote.
  TASKS_FILE="$TW_JOB_DIR/tasks.json"
  [ -s "$TASKS_FILE" ] || echo "  ⚠ Step 3.10: ${TASKS_FILE} missing — keyword scoring will find nothing (run Step 3 first)" >&2
  KEYWORDS=$(jq -r '
    [(.tasklist.name // ""), (.tasks[]?.name // ""), (.tasks[]?.description // "")]
    | join(" ")
  ' "$TASKS_FILE" \
    | tr '[:upper:]' '[:lower:]' \
    | tr -dc '[:alnum:]áäčďéíľĺňóôŕšťúýž _\n' \
    | tr ' ' '\n' \
    | awk 'length($0) >= 4' \
    | sort -u)

  # Build the find expression from configured extensions + filename hints.
  # `while read` over the jq output — the 1.4.2 unquoted for-loop over the
  # extension list iterated ONCE in zsh (no word splitting) and produced a
  # single `-iname` term containing every extension, which matched nothing.
  FIND_EXTS=""
  while IFS= read -r E; do
    [ -n "$E" ] && FIND_EXTS="$FIND_EXTS -iname '*.${E}' -o"
  done < <(jq -r '.local_context_discovery.extensions[]?' "$CONFIG_FILE")
  while IFS= read -r H; do
    [ -n "$H" ] && FIND_EXTS="$FIND_EXTS -iname '*${H}*' -o"
  done < <(jq -r '.local_context_discovery.filename_hints[]?' "$CONFIG_FILE")
  FIND_EXTS="${FIND_EXTS% -o}"

  IGNORE_EXPR=""
  while IFS= read -r G; do
    [ -n "$G" ] && IGNORE_EXPR="$IGNORE_EXPR -not -path './${G}'"
  done < <(jq -r '.local_context_discovery.ignore_globs[]' "$CONFIG_FILE")

  # Collect candidates (limit depth, filter out junk paths).
  CANDIDATES=$(eval "find . -maxdepth $MAX_DEPTH -type f $IGNORE_EXPR \\( $FIND_EXTS \\) -print 2>/dev/null" \
    | sort -u)

  # Score each candidate by keyword overlap (filename + parent dir tokens ∩ KEYWORDS).
  SCORED_FILE="$TW_JOB_DIR/local_discovery.tsv"
  : > "$SCORED_FILE"
  while IFS= read -r PATHCAND; do
    [ -z "$PATHCAND" ] && continue
    TOKENS=$(printf '%s\n' "$PATHCAND" \
      | tr '[:upper:]' '[:lower:]' \
      | tr -dc '[:alnum:]áäčďéíľĺňóôŕšťúýž /._\n' \
      | tr '/._' '\n' \
      | awk 'length($0) >= 3' \
      | sort -u)
    SCORE=$(comm -12 <(printf '%s\n' "$TOKENS") <(printf '%s\n' "$KEYWORDS") | wc -l | tr -d ' ')
    if [ "$SCORE" -ge "$MIN_SCORE" ]; then
      MATCHES=$(comm -12 <(printf '%s\n' "$TOKENS") <(printf '%s\n' "$KEYWORDS") | paste -sd , -)
      printf '%s\t%s\t%s\n' "$SCORE" "$PATHCAND" "$MATCHES" >> "$SCORED_FILE"
    fi
  done <<< "$CANDIDATES"

  # Top N, sorted by score desc.
  sort -k1,1nr "$SCORED_FILE" | head -n "$MAX_OFFER" > "${SCORED_FILE}.top"
  TOP_COUNT=$(wc -l < "${SCORED_FILE}.top" | tr -d ' ')
fi
```

If `TOP_COUNT == 0`, log a single line *"Local discovery: no plausible
context files found"* and continue with no extra context. Do **not** ask.

If `TOP_COUNT > 0`, render the list to the user via **AskUserQuestion**:

> "Found local files that look related to this tasklist (not attached in
>  Teamwork). Should I treat any of these as additional context?
>
>  - `Strečnianska/Bmail o pohybe na ucte - vzor.docx` (matches: bmail, vzor)
>  - `Strečnianska/DNR_Strecnianska_v1.2.docx` (matches: DNR, Strečnianska)
>  - `Strečnianska/dáta db/tenant_mysql_db_.sql` (matches: data, tenant)
>  - …"

Use a **multi-select** question (`multiSelect: true`) with one option per
file plus two control options:
- *Use all listed files as context*
- *Use only the ones I pick* (multi-select rest)
- *Ignore all — these are unrelated*
- *Other* (free text — user may paste an absolute path the scan missed)

For each picked file, attempt to inline its content into the per-task
`context_files` array consumed by Step 6.2:

| Extension     | Reader strategy                                                                                  |
| ------------- | ------------------------------------------------------------------------------------------------ |
| `.md`/`.txt`/`.csv`/`.sql`/`.json`/`.yaml`/`.html`/`.xml` | `Read` directly (plain text).                              |
| `.docx`       | `pandoc -t plain "$F"` if installed; else macOS `textutil -convert txt -stdout "$F"`; else fall back to filename-only with a one-line warning *"install pandoc for inline .docx content"*. |
| `.pdf`        | `pdftotext "$F" -` if installed; else filename-only warning.                                     |
| `.xlsx`       | `xlsx2csv "$F" -` if installed; else `in2csv` (csvkit); else filename-only warning.              |
| `.eml`/`.msg` | `Read` raw; treat headers + body as plain text.                                                  |
| Anything else | Add filename to context but warn it cannot be opened inline.                                     |

The set of picked files is then surfaced in the plan (Step 4) under a
`**Context files (local):**` line per task, so the human can see the skill
acknowledged them before approving.

**Bash-3.2 / macOS note.** The `comm` invocation above relies on sorted
input — never pipe `tr | sort -u | comm` without a sort step in between.
The find expression eval'd from a string is intentional (allows the
extension list to be config-driven); if the shell rejects it, fall back to
a simpler hard-coded find over `*.docx *.pdf *.md *.txt` and print a
warning.

---

## Step 4 — Plan overview + approval

Read `plan_mode` from config (or its CLI override). Default is `overview`.

- **`plan_mode = none`** — skip this step entirely, go straight to Step 5.
- **`plan_mode = overview`** (default) — build a single tasklist-wide plan, ask the user to approve it once before any work starts.
- **`plan_mode = per_task`** — skip this step; in the worker loop (Step 6.2) ask for approval **before** each task individually.

For `overview`, render a concise markdown plan to stdout. For tasklist URLs
the plan is **two-section**: first the tasks the skill will actually
implement (`process_mode == "process"`), then the analyse-only ones (and
dropped ones if the user wants to see them via `--show-dropped`):

```
## Tasklist context (if URL_KIND=tasklist and TASKLIST_DESCRIPTION non-empty)
<TASKLIST_DESCRIPTION>

## Plan for tasklist "<name>" (<TF_COUNT_PROCESS> to implement, <TF_COUNT_ANALYSE> analyse-only, <TF_COUNT_DROP> dropped)

> Tasklist filter: implementing only tasks in one of the start columns
> <TF_TODO_LABEL, e.g. "Ready for Development" / "To Do"> (<TF_TODO_MATCH>)
> AND assigned to <USER_DISPLAY_NAME>; tasks not on the board are
> analyse-only. Pass `--tasklist-filter=false` to process everything.
> <only when Step 3.3 found it, one line per project: "Project #X has none
> of the start columns — none of its tasks can pass" | "Project #X has no
> board — its tasks are analyse-only (not on the board)". A board that
> lacks just some of the start columns needs no line.>

---

### To implement (<TF_COUNT_PROCESS>)

#### 1. [#<task-id>] <title>  (est: <X> min, priority: <p>, stage: <its start column from PROCESS_FILE, e.g. Ready for Development | To Do — or, single-task URL: its current column, "not on the board", "<column> — completed task, processing confirmed" (Step 3.35) | promoted: "<column or not on the board> — promoted">, assignee: me)
**Parent context (v1.4.0):** <omitted if not a subtask, or if subtasks.include_parent_context=false | "Project bootstrap" — Container task for initial scaffolding; subtasks split the work by area (CI / auth / DB / …)>  ← from Step 3.42 PARENT_CTX_FILE
**Goal (final summary):** <one-line summary from description below HR>
**Acceptance:** <bullet list condensed from `acceptance_criteria` (Step 3.6)>
**Approach:** <1-3 sentences — what files / modules will likely change, what tests, what risks>
**Comments context:** <COMMENTS_STATE from Step 3.5 + digest — "none (0 comments)" | "newest of 8 read — full thread not fetched: 2026-05-20 client asks for X instead of Y" | "8 of 8 read — user clarified to use X over Y on 2026-05-20" | "not read (fetch_comments_mode=never)">
**Attachments:** <none | 2 files: spec.md (3KB), mockup.png (180KB)>
**File comments:** <skipped | 1 on mockup.png — "use #1A73E8 instead">
**Context files (local):** <none | Strečnianska/DNR_Strecnianska_v1.2.docx, Strečnianska/Bmail o pohybe na ucte - vzor.docx>  ← from Step 3.10
**Missing inputs:** <none | "Real Tatra banka notification sample" (BLOCKER — gating phrase in description) | "Final colour value" (SOFT — referenced in comment)>  ← from Step 3.10 (gap) + Step 6.0 patterns
**Board target:** <In progress → Done - Local | In progress → Internal testing (fallback) | In progress → Testing (fallback) | start only — no done column | disabled — no workflow on this project>  ← `done_resolved_name` of the task's board (Step 3.3)
**Quality dimensions (v1.5.0):** <which of ui_ux / performance / security / reachability / framework the planned change touches, one clause each — e.g. "reachability: new ExportLog screen → menu entry under Faktúry + tab on Invoice detail; security: policy per company; framework: Laravel 12.53 → `casts()` + enum cast for the status" | "backend only: performance, security, framework" | "skipped (--dimensions=none)">  ← refined in Step 6.2 (the framework clause names versions from the Step 6.2 detection, which is cached per run — running it while rendering this plan is free)

#### 2. [#<task-id>] ...

---

### Analyse only — not implemented in this run (<TF_COUNT_ANALYSE>)

> These tasks are in the tasklist but did not pass the filter (not in a
> start column, not on the board, or assigned to someone else).
> Listed here so you can sanity-check teammates' work, but the skill will
> NOT touch them: no commits, no time logs, no board moves. To process any
> of them anyway, pass `--tasklist-filter=false` or re-run with the single
> task URL.

#### 3. [#<task-id>] <title>  (stage: <stage>, assignee: <name>) ⏭ analyse-only
**Why skipped:** <wrong_stage(In Progress) + wrong_assignee | no_card — not on the board | stage_unresolved(300904) — column name could not be read>
**Goal (final summary):** <one-line summary>
**Quick read:** <1-2 sentence opinion / sanity check — "looks correctly scoped" / "watch out for X" / "approach mismatch with our backend convention" / "missing AC for offline behaviour">
**Comments context:** <one-line digest if any non-trivial decisions in comments>

#### 4. …
```

The **`Quick read`** line is the only piece of generative output produced for
analyse-only tasks — keep it to 1–2 sentences, focused on *what would a
reviewer flag if they had two minutes*. Do not draft full implementations, do
not propose tests, do not run the test-strategy detector from Step 6.2.5 — the
worker loop is going to skip them anyway. The skill is acting as a second
pair of eyes on the team's plan, not a ghost-coder for teammates.

The **`Missing inputs`** line is the single most important addition in v1.1.3 —
it makes the plan honest about what the skill cannot guess. Sources:
1. Any task whose description matched a `readiness_gate.patterns` regex.
2. Any task with `FILENAME_HINT_PRESENT=1` but **no** matching file in
   Step 3.10's results.
3. Any comment that says *"⏳ čaká sa na X"* / *"pending X"*.

If at least one task has a non-empty `Missing inputs` row, the plan-approval
question gets one extra option:
- *I have the input — let me paste a path or a short value* — opens a follow-up
  question per blocked task so the user can attach a local path / pasted
  snippet that the skill uses as `context_files` for that task only.

Then ask the user via **AskUserQuestion**:

- **Approve and start** — proceed to Step 5 (process only the "To implement" section; analyse-only output stays on screen for reference).
- **Skip some tasks** — user lists which task IDs to drop from "To implement", then re-render the plan and re-ask.
- **Reorder** — user provides new order, re-render and re-ask.
- **Add context to a task** — user picks a task and pastes extra context; append it to that task's working notes, re-render the plan, re-ask.
- **Promote an analyse-only task to implement** — user picks one or more task IDs from the "Analyse only" section; the skill flips `process_mode` to `process` for those IDs only and re-renders the plan with them moved to the top section. Useful when the user happens to own a teammate's task ad-hoc, or wants a task that is not on the board yet (`no_card`).
- **Disable the tasklist filter for this run** — equivalent to `--tasklist-filter=false`: every fetched task becomes `process` regardless of stage/assignee. Re-renders the plan with everything in "To implement".
- **Cancel** — abort the run, no commits, no time logs, no board moves.

### Step 4.0a — Empty-result short-circuit (v1.3.0)

When `URL_KIND == "tasklist"` AND the tasklist filter produced
`TF_COUNT_PROCESS == 0` (no task survived the start-column + me filter), the
normal plan-approval question above is **replaced** by a focused
"nothing-to-implement" prompt. Step 3.45 has already printed the rule list
+ per-task reasons to stderr, so the user knows *why* the result is empty;
this step's job is to make the next move one click away.

Render the plan with **only** the *Analyse only — not implemented in this
run* section (the *To implement* heading is omitted), preceded by:

```
## Plan for tasklist "<name>"

⚠ The tasklist filter found 0 tasks to implement.

  Rules: start columns = <TF_TODO_LABEL> (<TF_TODO_MATCH>; not on the board = not implementable)
         assignee = <USER_HINT>, analyze_all = <TF_ANALYZE_ALL>

  See the stderr output above for the per-task reason list, then pick one
  of the options below.
```

Then ask via **AskUserQuestion** with this option set instead of the
normal six:

- **Disable the filter for this run** *(recommended)* — equivalent to
  `--tasklist-filter=false`. Flip every analyse-only task to `process`
  and re-render the plan with the full task list, then re-ask the normal
  approval question.
- **Pick tasks from the analyse-only list to implement** — multi-select
  the IDs to promote to `process`. Re-renders the plan + normal approval
  question.
- **Change the start columns** — free-text prompt for one or more column
  names, comma-separated (e.g. `Backlog`, or `Ready for Development,To Do`).
  Re-runs Step 3.45 with the value in its `TODO_STAGES_CLI` line, then
  re-renders. Does **not** persist the change unless the user also opts to
  save it (then it is written to `tasklist_filter.todo_stages`).
- **Toggle the assignee check off** — equivalent to
  `--tasklist-only-mine=false`. Re-runs Step 3.45, then re-renders.
- **Cancel** — abort the run.

If the run had no analyse-only tasks either (somebody pointed at an
empty tasklist), the prompt collapses to just *Disable the filter* /
*Change the start columns* / *Cancel* — the *Pick tasks* and *Toggle
assignee check* options are hidden because they cannot help.

The plan generation time is **not** logged to Teamwork — the per-task timer starts only inside the worker loop (Step 6.1).

If the plan would be very long (>20 tasks), still render it but warn the user that the approach lines for later tasks are speculative (decisions made on early tasks may invalidate them).

---

## Step 5 — Branching setup

### Step 5.0 — `.gitignore` auto-update

Make sure downloaded attachments cannot accidentally land in a commit:

```bash
IGNORE_PATTERN="/teamwork-task-*/"
PROBE_DIR="./teamwork-task-probe"

NEED_APPEND=1
if [ -f .gitignore ]; then
  # If the pattern (any common variant) already ignores our probe path, do nothing
  mkdir -p "$PROBE_DIR" 2>/dev/null || true
  if git check-ignore -q "$PROBE_DIR" 2>/dev/null; then
    NEED_APPEND=0
  fi
  rmdir "$PROBE_DIR" 2>/dev/null || true
fi

if [ "$NEED_APPEND" -eq 1 ]; then
  {
    [ -s .gitignore ] && echo ""
    echo "# Teamwork attachments (managed by /teamwork-task skill)"
    echo "${IGNORE_PATTERN}"
  } >> .gitignore
fi
```

If `.gitignore` was modified, the working tree is no longer clean — under `branching_mode=current_branch` that would normally block work. Ask via **AskUserQuestion**:

- **Commit `.gitignore` first** (recommended default) → `git add .gitignore && git commit -m "CHORE(repo): Ignore /teamwork-task-*/ downloaded by skill"`, then continue.
- **Keep uncommitted** → continue with a dirty `.gitignore`; the user can commit later.
- **Abort** → stop the run, no further changes.

### Step 5 (rest) — Branching mode

Read `branching_mode` from config (or `--branching` override).

- **`current_branch`** (default): verify the working tree is clean.
  ```bash
  if [ -n "$(git status --porcelain)" ]; then
    # ask the user via AskUserQuestion: stash, commit first, or abort
    :
  fi
  ```

- **`new_feature_branch`**: create `feature/teamwork-tasklist-<tasklistId>` (for a tasklist run) or `feature/teamwork-task-<taskId>` (for a single-task run).
  ```bash
  BRANCH="feature/teamwork-tasklist-${ENTITY_ID}"
  git rev-parse --verify "$BRANCH" >/dev/null 2>&1 && git checkout "$BRANCH" || git checkout -b "$BRANCH"
  ```

### Step 5.1 — Detect worktree mode + cache parent branch

When the skill runs from inside a git worktree (typical for `EnterWorktree`
isolation or `.claude/worktrees/<name>` background jobs), every commit in
the worker loop lands on the worktree's branch — not on `main`. Without an
explicit handoff at the end of the run, those commits stay in the worktree
forever; the user has to remember to merge them by hand. Step 9.5 closes
that loop, but it needs three pieces of context resolved up front:

```bash
# Resolve the main repo's working directory (handles being inside a worktree).
MAIN_REPO=$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null \
  | sed -E 's|/\.git$||; s|/\.git/worktrees/[^/]+$||')

# Where we are now.
CURRENT_WT=$(git rev-parse --show-toplevel 2>/dev/null)
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null)

# Are we in the main repo, or in a worktree off it?
if [ -n "$MAIN_REPO" ] && [ "$MAIN_REPO" != "$CURRENT_WT" ]; then
  WT_RUN_IN_WORKTREE=1
else
  WT_RUN_IN_WORKTREE=0
fi

# Capture HEAD before the worker loop so Step 9.5 can compute "commits made
# this run". A user who started with a dirty worktree (uncommitted edits
# committed during plan approval) gets only this-run commits surfaced — not
# the pre-existing ones.
WT_HEAD_BEFORE=$(git rev-parse HEAD 2>/dev/null || echo "")

# Resolve the parent branch — i.e. where this worktree branched off. Best
# effort: prefer the branch the worktree was created from (recorded in
# `git config` for the worktree under `branch.<name>.<parent>` when set by
# Claude Code / EnterWorktree), fall back to origin/HEAD, then main /
# master / trunk.
WT_PARENT_BRANCH=""

if [ "$WT_RUN_IN_WORKTREE" = "1" ]; then
  # 1. Try a sticky config note (Claude Code / EnterWorktree sometimes set this).
  WT_PARENT_BRANCH=$(git -C "$CURRENT_WT" config --get "branch.${CURRENT_BRANCH}.claudeParent" 2>/dev/null)

  # 2. Fall back to origin/HEAD on the main repo.
  if [ -z "$WT_PARENT_BRANCH" ]; then
    WT_PARENT_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null \
                      | sed 's|^origin/||')
  fi

  # 3. Conventional names.
  if [ -z "$WT_PARENT_BRANCH" ]; then
    for B in main master trunk; do
      if git -C "$MAIN_REPO" show-ref --verify --quiet "refs/heads/$B"; then
        WT_PARENT_BRANCH="$B"
        break
      fi
    done
  fi

  # 4. Last resort: current HEAD on the main repo.
  [ -z "$WT_PARENT_BRANCH" ] && \
    WT_PARENT_BRANCH=$(git -C "$MAIN_REPO" rev-parse --abbrev-ref HEAD 2>/dev/null)

  echo "  ℹ running inside a worktree (${CURRENT_WT}); branch=${CURRENT_BRANCH}, parent=${WT_PARENT_BRANCH:-unknown}" >&2
fi
```

Why up here in Step 5 rather than next to Step 10:
- The detection is **cheap** (no Teamwork API calls) but needs the working
  directory + `git` to be reachable, which is exactly the state right
  before the worker loop starts.
- `WT_HEAD_BEFORE` has to be captured **before** any commit, otherwise the
  end-of-run handoff cannot reliably distinguish "commits made by this
  run" from "commits already there when I started".
- If `WT_RUN_IN_WORKTREE=0`, the rest of this section is skipped and Step
  9.5 turns into a no-op.

### Step 5.5 — Initialize the session time cursor

Time logs in Teamwork **must not overlap** even if implementation work overlaps in real time. The plugin maintains a **session cursor** that advances strictly forward as each task is logged, so the resulting time entries are contiguous (no gaps, no overlaps) and every entry's start time is aligned to the rounding step (default 5 minutes).

The cursor's starting position depends on `time_cursor_strategy`:

- **`floor_now`** — start at `floor(now, ROUND)`. Classic v1.0 behavior.
- **`last_teamwork_timelog`** (default) — start at the end of today's most recent timelog by the current user, so the run *picks up where you left off*. Falls back to `floor(now)` when there is no log today.

Initialize the cursor **once**, right before the worker loop starts (after plan approval — plan time is not billed):

```bash
# (run preamble — see Step 3)
USER_ID="<USER_ID from Step 2.7, empty if unresolved>"
ROUND=$(jq -r '.time_rounding_minutes // 5' "$CONFIG_FILE")
ROUND_SECS=$(( ROUND * 60 ))
STRATEGY=$(jq -r '.time_cursor_strategy // "last_teamwork_timelog"' "$CONFIG_FILE")

NOW_TS=$(date +%s)
TODAY=$(date +%Y-%m-%d)

floor_now() { echo $(( NOW_TS - (NOW_TS % ROUND_SECS) )); }

# Cross-platform epoch parser for ISO8601 timestamps.
# Teamwork v3 returns timeLogged in several shapes depending on the endpoint —
# e.g. `2026-05-27T08:15:00Z`, `2026-05-27T08:15:00+00:00`, or with fractional
# seconds. Normalise first (strip `.NNN`, swap `+00:00` for `Z`), then try
# macOS `date -j -f`, GNU `date -d`, and a Python one-liner fallback for
# anything weirder. Returns empty string on total failure (cursor falls back
# to floor_now upstream).
#
# TIMEZONE CONTRACT (critical — do not "fix" by formatting POST times in UTC):
#   * Teamwork RETURNS `timeLogged` in UTC (trailing `Z`). It must be parsed AS
#     UTC into an absolute epoch — hence `date -ju` (the `-u` is mandatory; the
#     `Z` in the BSD format string is a literal, NOT a zone directive, so
#     without `-u` BSD `date` reads the wall-clock as LOCAL and the epoch is
#     wrong by the local offset).
#   * Teamwork ACCEPTS the POST/PATCH `time` field in the user's PROFILE/LOCAL
#     timezone, not UTC. So the cursor epoch (absolute) is later formatted for
#     the POST with LOCAL `date -r "$TS" +%H:%M:%S` (Step 6.8) — NEVER with
#     `date -ju`/`date -u`. Parsing UTC but posting local is intentional and
#     correct; mixing them up shifts every entry by the local offset and makes
#     logs overlap.
parse_iso() {
  local raw norm out
  raw="$1"
  # Strip any millisecond fraction; normalise `+00:00` (and the rare `-00:00`) to `Z`.
  norm=$(printf '%s' "$raw" \
    | sed -E 's/\.[0-9]+(Z|[+-][0-9:]{2,5})?$/\1/' \
    | sed -E 's/[+-]00:?00$/Z/')

  # macOS BSD date — UTC `Z` form. `-u` is REQUIRED: the trailing `Z` is a
  # literal in the format string, so without `-u` the time is read as LOCAL.
  out=$(date -ju -f "%Y-%m-%dT%H:%M:%SZ" "$norm" +%s 2>/dev/null) && \
    { [ -n "$out" ] && echo "$out" && return 0; }
  # macOS BSD date — form with explicit zone `%z`
  out=$(date -j -f "%Y-%m-%dT%H:%M:%S%z" "$raw" +%s 2>/dev/null) && \
    { [ -n "$out" ] && echo "$out" && return 0; }
  # GNU date — accepts both shapes natively
  out=$(date -d "$raw" +%s 2>/dev/null) && \
    { [ -n "$out" ] && echo "$out" && return 0; }
  # Last resort: Python (present on macOS by default as `python3`)
  out=$(python3 - "$raw" <<'PY' 2>/dev/null
import sys, datetime
s = sys.argv[1].replace('Z', '+00:00')
try:
    print(int(datetime.datetime.fromisoformat(s).timestamp()))
except Exception:
    pass
PY
)
  [ -n "$out" ] && echo "$out" && return 0
  echo ""
}

SESSION_CURSOR_TS=""
TIME_CURSOR_SOURCE=""

case "$STRATEGY" in
  last_teamwork_timelog)
    # USER_ID was resolved once in Step 2.7 and cached for the whole run; reuse it.
    # (Pre-1.3.0 this step fetched /me.json itself — kept the call only as a fallback
    # below in case the earlier resolution failed.)
    if [ -z "$USER_ID" ]; then
      USER_ID=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
        "${BASE}/me.json" | jq -r '.person.id // empty')
    fi

    if [ -n "$USER_ID" ]; then
      # 2. Get most recent timelog of TODAY for that user
      RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" -w '\n%{http_code}' \
        "${BASE}/projects/api/v3/time.json?assignedToUserIds=${USER_ID}&pageSize=1&orderBy=date&orderMode=desc&startDate=${TODAY}")
      HTTP=${RESP##*$'\n'}; RESP=${RESP%$'\n'*}

      # Here-strings, not an echo pipe — zsh's echo mangles backslashes in
      # timelog descriptions and jq then rejects the whole response.
      LAST_LOGGED=""; LAST_MIN=0
      if [ "$HTTP" != "200" ]; then
        echo "  ⚠ GET /projects/api/v3/time.json → HTTP ${HTTP} — cursor falls back to floor(now)" >&2
      elif ! LAST_LOGGED=$(jq -r '.timelogs[0].timeLogged // empty' <<<"$RESP") \
         || ! LAST_MIN=$(jq -r '.timelogs[0].minutes // 0' <<<"$RESP"); then
        echo "  ⚠ time.json response is not valid JSON — cursor falls back to floor(now)" >&2
        LAST_LOGGED=""; LAST_MIN=0
      fi

      if [ -n "$LAST_LOGGED" ]; then
        LAST_TS=$(parse_iso "$LAST_LOGGED")
        if [ -n "$LAST_TS" ]; then
          END_TS=$(( LAST_TS + LAST_MIN * 60 ))
          END_DATE=$(date -r "$END_TS" +%Y-%m-%d 2>/dev/null \
                  || date -d "@$END_TS" +%Y-%m-%d 2>/dev/null)

          # Guards: must still be today, must not be in the future
          if [ "$END_DATE" = "$TODAY" ] && [ "$END_TS" -le $((NOW_TS + 60)) ]; then
            # Align UP to rounding step
            REM=$(( END_TS % ROUND_SECS ))
            [ "$REM" -ne 0 ] && END_TS=$(( END_TS + (ROUND_SECS - REM) ))
            if [ "$END_TS" -le "$NOW_TS" ]; then
              SESSION_CURSOR_TS=$END_TS
              TIME_CURSOR_SOURCE="last_timelog @ $(date -r "$END_TS" +%H:%M 2>/dev/null || date -d "@$END_TS" +%H:%M)"
            fi
          fi
        fi
      fi
    fi
    ;;
esac

if [ -z "$SESSION_CURSOR_TS" ]; then
  SESSION_CURSOR_TS=$(floor_now)
  TIME_CURSOR_SOURCE="skill start (first log of day)"
fi
```

Three hard guarantees from this algorithm:
1. **First log of the day** → cursor = `floor(now)` (no carry-over from yesterday).
2. **No future timestamps** — if the computed cursor would be in the future (clock skew, manually edited timelog), fall back to `floor(now)`.
3. **No cross-midnight bleed** — yesterday's last timelog is not considered.

`TIME_CURSOR_SOURCE` is shown in the Step 7 final summary so the user knows exactly where today's billing line started.

---

## Step 6 — Worker loop (per task)

For each task in the (approved and possibly reordered) list, in order:

### Step 6.0a — Process-mode gate (v1.3.0)

**Run this BEFORE Step 6.0 / 6.1 / anything else** — the `0a` suffix marks
that this gate sits in front of the readiness gate (Step 6.0) in execution
order while staying inside the per-task worker loop (so it cannot be hoisted
out to Step 5.x where there is no current task). If the task's
`process_mode` (set in Step 3.45) is `analyse_only`, the worker loop skips
**every** mutation for this task — no timer, no board move, no implementation,
no commit, no time log, no attachment cleanup, no board move to the done
column. The task's `Quick read` line from Step 4 is the entire on-screen
output; the loop continues with the next task.

```bash
PROCESS_MODE=$(awk -F '\t' -v id="$TASK_ID" '$1 == id { print $2; exit }' \
  "/tmp/tw_tasks_process_${ENTITY_ID}.tsv" 2>/dev/null)

# Single-task URLs never write to that file → default to "process" so the
# behaviour matches v1.2.x for /teamwork-task <task-url> runs.
[ -z "$PROCESS_MODE" ] && PROCESS_MODE="process"

if [ "$PROCESS_MODE" = "analyse_only" ]; then
  echo "  ⏭ [#${TASK_ID}] analyse-only (failed tasklist filter) — skipping worker loop." >&2

  # Accounting for the Step 7 summary.
  ANALYSE_ONLY_TASKS+=("${TASK_ID}")

  # No timer, no commit, no log, no board move. Move to the next task.
  continue
fi
```

Initialize `ANALYSE_ONLY_TASKS=()` alongside `SKIPPED_TIMELOGS=()` etc. at
the start of the worker loop so Step 7 always has a value to render.

The `process_mode` gate sits *above* the readiness gate (Step 6.0) on
purpose: an analyse-only task whose description matches a gating phrase
should still be quietly skipped, not turned into an `AskUserQuestion`
prompt the user has to dismiss for someone else's work.

### Step 6.0 — Readiness gate (per task)

**Run this BEFORE Step 6.1 starts the timer.** v1.1.3 catches the case where a
task body explicitly says it needs something that may not be in hand — the
canonical example is *"Pred začatím vyžiadať reálny vzor notifikácie od
klienta. Bez vzorky nemá zmysel písať regex."* The previous behaviour was to
plow through with a synthetic placeholder; the new behaviour is to **ask**.

Skip this step when `config.readiness_gate.enabled == false` or the user passed
`--readiness-gate=false`.

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
RG_ENABLED=$(jq -r '.readiness_gate.enabled | if . == null then true else . end' "$CONFIG_FILE")
[ "$RG_ENABLED" != "true" ] && { echo "  ℹ readiness gate disabled — skipping"; }

if [ "$RG_ENABLED" = "true" ]; then
  # The description comes from Step 3's task list; a subtask added by
  # Step 3.42 is not in it, so fetch that one directly. Shell variables from
  # earlier steps do not survive into this Bash call.
  TASK_DESCRIPTION_TEXT=$(jq -r --arg id "$TASK_ID" \
    '.tasks[] | select((.id | tostring) == $id) | .description // ""' "$TW_JOB_DIR/tasks.json")
  if [ -z "$TASK_DESCRIPTION_TEXT" ]; then
    TASK_DESCRIPTION_TEXT=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
      "${BASE}/projects/api/v3/tasks/${TASK_ID}.json" | jq -r '.task.description // ""')
    [ -z "$TASK_DESCRIPTION_TEXT" ] && echo "  ⚠ [#${TASK_ID}] description empty or GET /projects/api/v3/tasks/${TASK_ID}.json failed — the readiness gate scans the comments only" >&2
  fi
  TASK_DESCRIPTION_TEXT=$(printf '%s\n' "$TASK_DESCRIPTION_TEXT" | sed -E 's/<[^>]+>/ /g')
  # ACCEPTANCE_CRITERIA_TEXT / FINAL_SUMMARY_TEXT: the Step 3.6 split, if you
  # have it at hand — both are substrings of the description, so leaving them
  # empty loses nothing.

  # Every comment read in Step 3.5 (chronological, plain text).
  COMMENTS_CONCAT_TEXT=$(jq -r '.[] | if (.body // "") != "" then .body
      else ((.htmlBody // "") | gsub("<[^>]+>"; " ")) end' "$TW_JOB_DIR/comments_${TASK_ID}.json")

  # Build one big haystack: task description + acceptance_criteria + final_summary
  # + every comment body, lowercased and HTML-stripped.
  HAYSTACK=$(printf '%s\n%s\n%s\n%s\n' \
    "$TASK_DESCRIPTION_TEXT" \
    "$ACCEPTANCE_CRITERIA_TEXT" \
    "$FINAL_SUMMARY_TEXT" \
    "$COMMENTS_CONCAT_TEXT" \
    | tr '[:upper:]' '[:lower:]')

  GATING_HIT=""
  MATCHED_PHRASE=""
  while IFS= read -r PAT; do
    [ -z "$PAT" ] && continue
    # `grep -E` with the configured POSIX-ERE pattern; first match wins.
    # printf, not echo — zsh's echo would expand backslashes in task text.
    if MATCH=$(printf '%s\n' "$HAYSTACK" | grep -oE "$PAT" | head -n1); then
      if [ -n "$MATCH" ]; then
        GATING_HIT=1
        MATCHED_PHRASE="$MATCH"
        break
      fi
    fi
  done < <(jq -r '.readiness_gate.patterns[]' "$CONFIG_FILE")

  if [ -n "$GATING_HIT" ]; then
    # Re-use the local discovery results (Step 3.10) to offer specific files
    # as candidate inputs. If discovery is empty for this task, just ask
    # whether the user has the input as free text.
    OFFERED=""
    [ -s "$TW_JOB_DIR/local_discovery.tsv.top" ] && OFFERED=$(cat "$TW_JOB_DIR/local_discovery.tsv.top")
    : # render AskUserQuestion (see options below)
  fi
fi
```

**The question** (only fires when `GATING_HIT` is set):

> "Task [#<id>] **<title>** contains a gating phrase:
>
>  > *<MATCHED_PHRASE>*  (matched config.readiness_gate.patterns)
>
>  This usually means the task needs an external input (sample file, real
>  data, client decision) before implementation makes sense. How do you want
>  to proceed?"

Options (single-select):

1. **Yes — I have it, point me at it.**
   Opens a follow-up question listing the Step 3.10 local-discovery results
   for this task as multi-select options, plus an *Other* free-text field
   where the user can paste an absolute path, a URL, or a literal snippet.
   The picked content goes into this task's `context_files` array (same
   shape as Step 3.10 output) and the timer in Step 6.1 starts normally.

2. **No — proceed with a synthetic placeholder (mark as needs-verify).**
   Implementation continues, but:
   - The plan entry's `Missing inputs` line is preserved into the per-task
     internal notes.
   - The time-log description (Step 6.8) is **prefixed** with `⚠️
     Implementované so syntetickou náhradou — vyžaduje overenie po doručení
     reálneho vkladu.` (Slovak default; English variant for
     `default_language=en`).
   - The auto-proposed test in Step 6.2.5 must use a fixture clearly named
     `*_synthetic.*` (e.g. `tatra_credit_sample_synthetic.txt`) so a
     reviewer immediately sees what is real and what is filler.
   - A trailer line is appended to the **commit body**:
     `Synthetic-Input: <one-line description of what was faked>`
     so a future grep of `git log --all -i --grep='synthetic-input'`
     surfaces every place that needs revisiting.

3. **No — skip this task entirely.**
   Record the task in the final summary as `⏭️ blocked — gating phrase
   matched, no input provided`. **Do not** commit, **do not** log time,
   **do not** move the board card. Continue to the next task.

4. **Cancel the whole run.**
   Stop the loop, no further commits / logs / board moves.

If `plan_mode == per_task`, Step 6.0 fires **before** the per-task plan render
so the gating decision is part of the same per-task approval — the user does
not have to answer two separate questions back-to-back.

**Why this matters.** A regex against a made-up email format is the kind of
mistake that ships, lives in production for weeks, and only surfaces when a
real bank notification fails to parse and a customer payment goes missing.
The 30-second pause this gate introduces is cheaper than that incident by
several orders of magnitude.

### 6.1 Start timer
```bash
START_TS=$(date +%s)

# Reset per-task state so the final summary always has a value to print, even
# if both board moves fail or board_workflow is disabled.
CURRENT_BOARD_STAGE_FOR_TASK="—"
TIMELOG_OK=0
COMMIT_HASH=""
```

If `plan_mode == per_task`, do **not** start the timer here yet — first render a per-task plan (same shape as the overview entry: Goal / Acceptance / Approach / Comments context / Attachments / Board target / Quality dimensions) and ask the user via **AskUserQuestion** to **Approve / Skip / Add context / Cancel run**. Only **after** approval start the timer.

### 6.1.5 Move task to "In progress" on the board

Immediately after the timer starts (and after per-task approval, if applicable), nudge the card to the in-progress column:

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
# Project id of THIS task (subtasks included) and the board state Step 3.3
# resolved for it — read from files, because no variable survives between
# Bash calls (the 1.4.2 per-project array lookup was empty in every fresh
# shell, and PROJECT_ID itself was null — see Step 3).
PROJECT_ID=$(awk -F '\t' -v id="$TASK_ID" '$1 == id { print $2; exit }' "$TW_JOB_DIR/task_projects.tsv" 2>/dev/null)
BOARD_FILE="$TW_JOB_DIR/board_${PROJECT_ID}.tsv"
WORKFLOW_ID=""; IN_PROGRESS_STAGE_ID=""; BOARD_DISABLED=1
if [ -n "$PROJECT_ID" ] && [ -s "$BOARD_FILE" ]; then
  WORKFLOW_ID=$(awk -F '\t'          '$1 == "workflow_id"          { print $2; exit }' "$BOARD_FILE")
  IN_PROGRESS_STAGE_ID=$(awk -F '\t' '$1 == "in_progress_stage_id" { print $2; exit }' "$BOARD_FILE")
  BOARD_DISABLED=$(awk -F '\t'       '$1 == "disabled"             { print $2; exit }' "$BOARD_FILE")
else
  echo "  ⚠ [#${TASK_ID}] no project id / no Step 3.3 board state (project '${PROJECT_ID}') — board move to 'In progress' skipped" >&2
fi

BW_ENABLED=$(jq -r '.board_workflow.enabled | if . == null then true else . end' "$CONFIG_FILE")
if [ "$BW_ENABLED" = "true" ] \
   && [ "$BOARD_DISABLED" != "1" ] \
   && [ -n "$WORKFLOW_ID" ] && [ -n "$IN_PROGRESS_STAGE_ID" ]; then
  HTTP=$(curl -sS -o /dev/null -w "%{http_code}" -u "$AUTH" \
    -H "Content-Type: application/json" -H "Accept: application/json" \
    -X POST -d "{\"taskIds\":[${TASK_ID}]}" \
    "${BASE}/projects/api/v3/workflows/${WORKFLOW_ID}/stages/${IN_PROGRESS_STAGE_ID}/tasks.json")
  if [ "$HTTP" -lt 200 ] || [ "$HTTP" -ge 300 ]; then
    echo "  ⚠ board move to 'In progress' failed (HTTP $HTTP) — continuing" >&2
  else
    CURRENT_BOARD_STAGE_FOR_TASK="In progress"
  fi
fi
```

Failure here is **non-fatal** — the task itself still runs, and the final summary reports that the board move did not happen. No retries (the most likely cause is a permissions or config mismatch, which retrying will not fix).

### 6.2 Plan implementation (internal)

Re-read the task description (already split into `acceptance_criteria` + `final_summary` in Step 3.6), any fetched comments (Step 3.5), and any text-based files in `$ATTACH_DIR` and `$ATTACH_DIR/comments/`. Use `Read` to inline text files (`.md`, `.txt`, `.json`, `.yaml`, `.yml`, `.csv`, `.log`, `.html`, `.xml`, source files etc.). Binary attachments (images, PDFs, archives) are listed in the plan but not opened.

The `final_summary` (Step 3.6) is the authoritative goal; use `acceptance_criteria` (Step 3.6 — the `## Akceptačné kritériá` block in the canonical format, otherwise the text above the first HR) as the checklist to verify before committing. The **last comment** in chronological order is the freshest source of truth when comments contradict each other.

If anything is genuinely ambiguous (missing acceptance criteria, conflicting requirements with comments, business decision needed) → **AskUserQuestion** with a focused question. Wait for the answer before proceeding. Do **not** guess on business-shaped questions.

For purely technical decisions where there is a reasonable default consistent with the codebase, proceed without asking.

**Build-time quality dimensions (v1.5.0).** Before writing code, decide which
of the five cross-cutting dimensions this task's change touches and plan the
concrete step for each one that applies. The keys are exactly the ones
`/teamwork-task-test` reviews at QA time (its Step 6.6) — building with them
in mind is cheaper than having QA find the gap after the commit. The active
set is `config.build_quality.dimensions` (default all five) or the
`--dimensions=` override; `none` / `[]` skips this block and Step 6.5.5, and
the final summary says `skipped (--dimensions=none)`. In `fast` mode
(Step 2.65) the active set is `[]` unless `--dimensions=` names keys explicitly.

Decide from the **shape of the planned change**, not from the task's wording —
but a `### Prierezové požiadavky` / `### Cross-cutting requirements` block in the
acceptance criteria (Step 3.6) is binding: each item there is an acceptance
criterion, its dimension applies, and the menu section, roles and parent screen
it names win over your own inference. `framework` never appears in that block
(the sibling plugins put it in the *Technický popis* only, because QA treats it
as advisory); a framework note there is guidance — when it names a version or
an API that disagrees with the detected versions below, the detected versions
win and the plan says so.

| Key | Applies when the change … | What to plan |
| --- | --- | --- |
| `ui_ux` | touches a file matching `config.test_visual_file_patterns` (templates, components, views) or a class that feeds one (Nova fields / actions / cards, messages shown to the user) | the sibling screen whose patterns you copy; accessible names and labels; loading / empty / error states; every new string → which module lang file + English key |
| `performance` | adds or changes a query, a migration, a loop over records, a batch / job, an import / export | eager loads, pagination, `chunkById`, indexes on new foreign keys and filtered / sorted columns, the row count at which the chosen approach would start to hurt |
| `security` | adds a route, controller action, Nova action / button, API endpoint, form input, upload, or a raw query | the gate the neighbours use + the object-scoped policy check, FormRequest validation, `$fillable`, tenant / company scope |
| `reachability` | adds, renames or removes a screen — a page, a route rendering a view, a Nova resource / lens / dashboard / tool, an SPA route | the **menu entry** (which menu, which roles) **and the inbound links** from the related screens (relation field or tab on the parent, link on a detail view, action button, breadcrumb) — planned into the **same commit** as the screen |
| `framework` | writes or changes code in a language / framework the project pins a version of — PHP, Laravel, Nova, Livewire, Inertia, Pest, JS / TS, Vue, React, CSS, Tailwind (almost every code task; `not_applicable` for docs / config / data-only changes) | the **installed** versions that bound the change (the cached table below), the current idiom or built-in feature the new code will use instead of a dated or hand-rolled pattern, and the docs page it comes from when the choice is not obvious — inside the Step 6.3 guardrails (project conventions win, no drive-by rewrites) |

A dimension the change does not touch is recorded as `not_applicable(<why>)`
— a pure backend task without UI must not grow UI boilerplate. Add the result
to the per-task internal plan, one line per active key:

```
Build quality (v1.5.0):
  - ui_ux:        applies — new Nova resource ExportLog; labels via lang/sk/export.php, empty state on index
  - performance:  applies — index on export_logs.invoice_id; eager-load invoice.customer on the index
  - security:     applies — ExportLogPolicy::view scoped to the user's company; action behind the same gate as InvoiceActions
  - reachability: applies — menu entry under "Faktúry" (roles admin, accountant) + HasMany tab on the Invoice detail, same commit
  - framework:    applies — Laravel 12.53 / PHP ^8.4 / Nova 5.7: `casts()` + enum cast for ExportLog::status, `Rule::enum` in the FormRequest, `->filterable()` on the Nova status field (docs: eloquent-mutators#enum-casting)
```

**Framework versions — detected once per run, never assumed (v1.5.0).** The
`framework` dimension is only as good as the versions that bound it, and model
memory is stale. The first task that plans with `framework` active runs the
snippet below: it reads the project's lock / manifest files and writes
`$TW_JOB_DIR/framework_versions.tsv` (`name<TAB>version<TAB>source`, `-` when
unknown); every later task of the run reuses that file, and the Step 3 reset
gives the next run a fresh detection. A task whose commit changes
`composer.json` / `composer.lock`, a `package.json` or a JS lock file (an
upgrade, a new package) deletes the file right after its commit
(`rm -f "$TW_JOB_DIR/framework_versions.tsv"`), so the next task detects
again instead of planning against the old versions. PHP packages come from
`composer.lock` (`laravel/framework`, `laravel/nova`, `livewire/livewire`,
`inertiajs/inertia-laravel`, `pestphp/pest`, `laravel/boost`) plus the PHP
constraint (`require.php`, `config.platform.php`); JS packages (`vue`, `react`,
`nuxt`, `vite`, `typescript`, `tailwindcss`, `@inertiajs/*`, `@ionic/*`) from
`node_modules`, then `package-lock.json`, then `yarn.lock`, then the declared
range; plus the `browserslist` target and the Node version (`.nvmrc` /
`.node-version` / `engines.node`).

```bash
# (run preamble — see Step 3)
FW_FILE="$TW_JOB_DIR/framework_versions.tsv"
if [ -s "$FW_FILE" ]; then
  echo "  ℹ framework versions: reusing the table detected earlier in this run" >&2
else
  ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
  : > "$FW_FILE.tmp"

  # PHP side — the app's composer.lock: repo root first, else one level down
  # (never vendor/ or a module package, which has no lock of its own).
  CL=""; [ -f "$ROOT/composer.lock" ] && CL="$ROOT/composer.lock"
  [ -z "$CL" ] && CL=$(find "$ROOT" -mindepth 2 -maxdepth 2 -name composer.lock \
    -not -path '*/vendor/*' -not -path '*/node_modules/*' 2>/dev/null | head -n1)
  CJ="$ROOT/composer.json"; [ -n "$CL" ] && CJ="$(dirname "$CL")/composer.json"
  PHP_PKGS='^(laravel/(framework|nova|boost)|livewire/livewire|inertiajs/inertia-laravel|pestphp/pest)$'
  PHP_ROWS=""
  if [ -n "$CL" ]; then
    PHP_ROWS=$(jq -r --arg re "$PHP_PKGS" --arg src "${CL#"$ROOT"/}" \
      '[(.packages // [])[], (.["packages-dev"] // [])[]] | .[]
       | select(.name | test($re)) | [.name, .version, $src] | @tsv' "$CL") \
      || { PHP_ROWS=""; echo "  ⚠ ${CL#"$ROOT"/} is not valid JSON — PHP package versions fall back to the composer.json constraints" >&2; }
  fi
  if [ -f "$CJ" ]; then
    # No usable lock → the declared constraints are the best bound (the source column says so).
    [ -z "$PHP_ROWS" ] && PHP_ROWS=$(jq -r --arg re "$PHP_PKGS" --arg src "${CJ#"$ROOT"/} (constraint, no lock)" \
      '((.require // {}) + (.["require-dev"] // {})) | to_entries[]
       | select(.key | test($re)) | [.key, .value, $src] | @tsv' "$CJ")
    PHP_ROWS=$(printf '%s\n' "$PHP_ROWS"; jq -r --arg src "${CJ#"$ROOT"/}" \
      '(.require.php // empty | ["php", ., ($src + " require.php")]),
       (.config.platform.php // empty | ["php_platform", ., ($src + " config.platform.php")])
       | @tsv' "$CJ") \
      || echo "  ⚠ ${CJ#"$ROOT"/} is not valid JSON — PHP version constraint unknown" >&2
  fi
  printf '%s\n' "$PHP_ROWS" | grep -v '^$' >> "$FW_FILE.tmp"

  # JS side — the app's package.json (root first, else one level down).
  PJ=""; [ -f "$ROOT/package.json" ] && PJ="$ROOT/package.json"
  [ -z "$PJ" ] && PJ=$(find "$ROOT" -mindepth 2 -maxdepth 2 -name package.json \
    -not -path '*/node_modules/*' -not -path '*/vendor/*' 2>/dev/null | head -n1)
  if [ -n "$PJ" ]; then
    PD=$(dirname "$PJ"); PFX=""; [ "$PD" != "$ROOT" ] && PFX="${PD#"$ROOT"/}/"
    NAMES=$(jq -r '((.dependencies // {}) + (.devDependencies // {})) | keys[]
        | select(test("^(vue|react|nuxt|vite|typescript|tailwindcss|@inertiajs/.+|@ionic/.+)$"))' "$PJ") \
      || echo "  ⚠ ${PFX}package.json is not valid JSON — JS package versions unknown" >&2
    while IFS= read -r N; do
      [ -z "$N" ] && continue
      V=""; SRC=""
      if [ -f "$PD/node_modules/$N/package.json" ]; then
        V=$(jq -r '.version // empty' "$PD/node_modules/$N/package.json"); SRC="${PFX}node_modules"
      fi
      if [ -z "$V" ] && [ -f "$PD/package-lock.json" ]; then
        V=$(jq -r --arg n "$N" '.packages["node_modules/" + $n].version // .dependencies[$n].version // empty' \
          "$PD/package-lock.json"); SRC="${PFX}package-lock.json"
      fi
      if [ -z "$V" ] && [ -f "$PD/yarn.lock" ]; then
        V=$(awk -v p="$N" 'index($0, p "@") == 1 || index($0, "\"" p "@") == 1 { f = 1; next }
                           f && $1 ~ /^version:?$/ { gsub(/[":]/, "", $2); print $2; exit }' "$PD/yarn.lock")
        SRC="${PFX}yarn.lock"
      fi
      if [ -z "$V" ]; then   # pnpm / bun without node_modules: the declared range only
        V=$(jq -r --arg n "$N" '(.dependencies // {})[$n] // (.devDependencies // {})[$n] // empty' "$PJ")
        SRC="${PFX}package.json (range, not installed)"
      fi
      printf '%s\t%s\t%s\n' "$N" "${V:--}" "$SRC" >> "$FW_FILE.tmp"
    done <<<"$NAMES"

    # Browser target for CSS / JS features, and the Node version.
    BL=""; BLSRC=""
    for F in .browserslistrc browserslist; do
      if [ -z "$BL" ] && [ -f "$PD/$F" ]; then
        BL=$(grep -v -e '^[[:space:]]*#' -e '^[[:space:]]*$' "$PD/$F" | paste -sd ',' - | sed 's/,/, /g')
        BLSRC="${PFX}${F}"
      fi
    done
    if [ -z "$BL" ]; then
      BL=$(jq -r '.browserslist // empty
          | if type == "array" then join(", ")
            elif type == "object" then (.production // .defaults // ([.[]] | flatten)) | join(", ")
            else tostring end' "$PJ")
      BLSRC="${PFX}package.json browserslist"
    fi
    [ -z "$BL" ] && { BL="-"; BLSRC="not set — the bundler's default build target applies (check its docs)"; }
    printf '%s\t%s\t%s\n' "browserslist" "$BL" "$BLSRC" >> "$FW_FILE.tmp"

    NODE=""; NSRC=""
    for F in .nvmrc .node-version; do
      if [ -z "$NODE" ] && [ -f "$PD/$F" ]; then
        NODE=$(head -n1 "$PD/$F" | tr -d '[:space:]'); NSRC="${PFX}${F}"
      fi
    done
    [ -z "$NODE" ] && { NODE=$(jq -r '.engines.node // empty' "$PJ"); NSRC="${PFX}package.json engines.node"; }
    [ -n "$NODE" ] && printf '%s\t%s\t%s\n' "node" "$NODE" "$NSRC" >> "$FW_FILE.tmp"
  fi

  [ -s "$FW_FILE.tmp" ] || printf '%s\t%s\t%s\n' "none" "-" "no composer.lock / composer.json / package.json found" > "$FW_FILE.tmp"
  mv "$FW_FILE.tmp" "$FW_FILE"
fi
awk -F '\t' '{ printf "  %-28s %-24s %s\n", $1, $2, $3 }' "$FW_FILE"
```

How to read the table:

- **An installed version beats a constraint.** Rows from `composer.lock`,
  `node_modules`, `package-lock.json` or `yarn.lock` are what runs; a
  `(constraint, no lock)` / `(range, not installed)` row is only a bound —
  plan against the lowest version it admits.
- **PHP language features are bounded by the lowest PHP the project admits,
  not by your local `php -v`:** `php_platform` when set, otherwise the floor
  of `require.php` (`^8.2` → 8.2: no typed class constants from 8.3, no
  property hooks from 8.4).
- **CSS / JS features are bounded by `browserslist`.** When it is not set,
  the bundler's default build target applies — look it up for the installed
  bundler version instead of assuming one; a Tailwind v3 → v4 decision is a
  browser-target decision too.
- **Laravel Boost** (a `laravel/boost` row, and its MCP tools in this
  session): its `application-info` tool reports the versions directly — use it
  to cross-check; the file stays the record the later steps read.
- A missing row is not a missing package: pnpm / bun projects without
  `node_modules` only report the declared range, and a Nova custom component
  under `nova-components/` carries its own `package.json` — read that one
  when the task touches it. In a repository with several apps and nothing at
  the root (e.g. `backend/` + `frontend/`), only the first manifest found one
  level down is read — the source column names it; read the other app's
  manifest the same way when the task touches that app.

**Look the API up in current docs, not in memory.** For every framework the
task touches: once per run and major version, skim that major's upgrade guide
/ release notes (what is new, what is deprecated); per task, look up the
specific API before using it whenever the idiom is not obvious. Source order:
Laravel Boost `search-docs` when the project has `laravel/boost` (it answers
for the installed versions), otherwise the context7 MCP (`resolve-library-id`
→ `query-docs`, naming the installed version), otherwise the official docs via
`WebFetch`. Name the page in the plan line (and in the Step 6.5.5 detail) when
the choice was non-obvious.

### 6.2.5 Test strategy — propose tests when none are specified

If `config.auto_propose_tests == true` (default), the work mode is not
`fast` (Step 2.65 forces it `false`), and the task does **not**
already specify a testing strategy, the skill proposes a concrete test plan
and implements it alongside the code change. This guards against the common
trap where a Teamwork ticket says only *"add PDF export to invoice"* and
nobody describes how the result should be verified.

**Detection — does the task describe tests?**

Classify the task's `acceptance_criteria` + `final_summary` + comments into
one of three states:

1. **`described`** — text mentions any of: `test`, `tests`, `testovanie`,
   `pest`, `phpunit`, `unit test`, `feature test`, `browser test`, `e2e`,
   `dusk`, `selenium`, `playwright`, `cypress`, `acceptance test`,
   `napíš testy`, `cover with tests`, `test cases`, `manual test`. Use the
   described approach as-is.
2. **`opted_out`** — text matches any phrase from
   `config.test_opt_out_keywords` (default includes *"no tests"*,
   *"skip tests"*, *"bez testov"*, *"netreba testy"*). Skip auto-propose for
   this task — log a note in the plan: *"Tests intentionally skipped per
   task description."*
3. **`missing`** — neither of the above. Auto-propose mode kicks in.

**Detection — what does the project use?**

For the `missing` case, sniff the repo once per run (cache results):

```bash
# Language / framework detection
HAS_LARAVEL=0; HAS_PEST=0; HAS_PHPUNIT=0; HAS_DUSK=0
HAS_VITEST=0; HAS_JEST=0; HAS_PLAYWRIGHT=0; HAS_CYPRESS=0; HAS_SELENIUM=0

if [ -f composer.json ]; then
  jq -e '.require."laravel/framework" // .["require-dev"]."laravel/framework"' composer.json >/dev/null 2>&1 && HAS_LARAVEL=1
  jq -e '.require."pestphp/pest" // .["require-dev"]."pestphp/pest"'           composer.json >/dev/null 2>&1 && HAS_PEST=1
  jq -e '.require."phpunit/phpunit" // .["require-dev"]."phpunit/phpunit"'     composer.json >/dev/null 2>&1 && HAS_PHPUNIT=1
  jq -e '.require."laravel/dusk" // .["require-dev"]."laravel/dusk"'           composer.json >/dev/null 2>&1 && HAS_DUSK=1
fi

if [ -f package.json ]; then
  jq -e '.dependencies.vitest      // .devDependencies.vitest'      package.json >/dev/null 2>&1 && HAS_VITEST=1
  jq -e '.dependencies.jest        // .devDependencies.jest'        package.json >/dev/null 2>&1 && HAS_JEST=1
  jq -e '.dependencies["@playwright/test"] // .devDependencies["@playwright/test"]' package.json >/dev/null 2>&1 && HAS_PLAYWRIGHT=1
  jq -e '.dependencies.cypress     // .devDependencies.cypress'     package.json >/dev/null 2>&1 && HAS_CYPRESS=1
  jq -e '.dependencies.selenium    // .devDependencies.selenium // .devDependencies["selenium-webdriver"]' package.json >/dev/null 2>&1 && HAS_SELENIUM=1
fi
```

**Decision — which test type?**

Classify the change by file pattern (reuse `config.test_visual_file_patterns`):

- **Visual change** — diff (or planned diff) touches files matching the
  visual patterns (`.vue`, `.blade.php`, `.tsx`, `resources/views/…`). Prefer
  browser tests.
- **Backend change** — only `.php`, `.ts`, `.js`, `.py`, etc. without UI
  templates. Prefer unit / feature tests.
- **Mixed** — both classes touched. Plan **both** kinds.

Match to the available framework:

| Project signal           | Visual change                                                   | Backend change                                                            |
| ------------------------ | --------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Laravel + Dusk installed | Laravel Dusk (`tests/Browser/…`)                                | Pest if `HAS_PEST`, else PHPUnit (`tests/Feature/…`, `tests/Unit/…`)      |
| Laravel without Dusk     | Ask once to add Dusk permanently, else a manual checklist       | Pest if `HAS_PEST`, else PHPUnit                                          |
| PHP without Laravel      | Selenium standalone PHPUnit test, otherwise skip                | PHPUnit or Pest by signal                                                 |
| JS with Playwright       | Playwright (`tests/e2e/…`)                                      | Vitest if `HAS_VITEST`, Jest if `HAS_JEST`                                |
| JS with Cypress          | Cypress (`cypress/e2e/…`)                                       | Vitest or Jest                                                            |
| JS without any browser   | Ask once to add Playwright permanently, else manual checklist   | Vitest or Jest by signal                                                  |
| Nothing detected         | Plan a manual checklist instead                                 | Plan a manual checklist instead                                           |

`config.test_frameworks.*_preference` lets the user pin a choice (`pest`,
`phpunit`, `dusk`, `playwright`, `cypress`, `vitest`, `jest`,
`selenium`); the default `"auto"` follows the table.

**Missing tool — ask once, install permanently, never on the fly**

Never install or uninstall Playwright, Puppeteer or Laravel Dusk just for this
run. For a visual check prefer the chrome-devtools MCP or the runner the
project already has. If the chosen framework is **not present** in the project
(e.g. visual change in Laravel but no Dusk), use **AskUserQuestion** **once per
run** — the answer holds for every later task — with three options:

- *Add `<package>` to the project permanently* — install it as a committed dev
  dependency (`composer require --dev laravel/dusk && php artisan dusk:install` /
  `npm install -D @playwright/test && npx playwright install`), commit the
  manifest and lock-file changes with the task, and never remove it afterwards.
- *Skip browser tests, write a manual checklist* — emit the checklist into
  the plan and proceed without a browser test file. This is the safe
  default.
- *Abort the task* — leave the working tree as-is and move on.

**Output of this step (added to the per-task internal plan)**

A short block such as:

```
Test strategy (auto-proposed, task did not specify):
  - Backend: tests/Feature/InvoicePdfExporterTest.php (Pest)
      • renders PDF with tenant theme
      • throws when invoice is missing
  - Visual:  tests/Browser/InvoicePdfExportTest.php (Laravel Dusk)
      • clicks "Export PDF" on /invoices/{id}, downloads file, asserts byte signature
```

If the user picks **Add context** in `plan_mode=per_task`, they can override
this block before the timer starts.

### 6.3 Implement
- Use `Read`, `Edit`, `Write`, `Bash`, `Grep`, `Glob` as needed.
- Respect any project conventions found in `CLAUDE.md` at the repo root.
- Write all code comments in **English**.
- **Build-time quality rules (v1.5.0)** — follow the rules of every dimension
  Step 6.2 marked *applies* (and only those):
  - **`ui_ux`** — copy the sibling screen's patterns. Give every interactive
    element an accessible name; a disabled control gets `aria-disabled` (or a
    real `disabled`) **and** a visible reason (`title` / `aria-describedby`).
    Label form controls, give images `alt`, never convey state by colour
    alone. Design the loading / empty / error states, confirm destructive
    actions, avoid layout shift. Every user-facing string goes through the
    translation layer with an English key in the module's own lang file, and
    the new key must resolve to text — no dotted-prefix collision with an
    existing string key (hints under `hint.<thing>`, not `<thing>.hint`).
  - **`performance`** — no query inside a loop; eager-load the relations a
    row touches. Paginate lists; `chunkById` batches and data migrations.
    Index new foreign keys and filtered / sorted columns (check for an
    existing index first). Filter in SQL, not in PHP. Move slow work to a
    queue. Cache only with an invalidation story.
  - **`security`** — put every new route / action / button / endpoint behind
    the same gate as its neighbours **plus** an object-scoped policy check (a
    role check any tenant passes is an IDOR). Never bypass the tenant /
    company scope with raw `DB::` calls or joins. Validate through a
    FormRequest, keep `$fillable` explicit, never `fill($request->all())`, never
    interpolate into raw SQL or a shell. Crafted input gets a handled answer
    (validation error / flash message) — never an unhandled 500, never
    internals in the message (with `APP_DEBUG=false` a thrown message becomes
    a generic 500). No secrets in code, logs or commits. Hiding a menu item is
    not access control.
  - **`reachability`** — a new screen ships **in the same commit** with its
    menu entry (visible to the intended roles) **and** the inbound links from
    the related screens where a user would look for it. A child resource
    reached through its parent's relation tab counts as reachable. A
    deliberately URL-only page (e-mail deep link, landing page) is named as
    such in the commit body and recorded in Step 6.5.5 as an `allow_orphans`
    candidate for QA. Renaming or removing a screen updates or removes every
    menu entry and inbound link that pointed at it. Menu visibility and route
    authorization must agree — visible-but-403 and hidden-but-open are both
    defects.
  - **`framework`** — write the **new or changed** code in the current idiom
    of the versions the project has installed (the Step 6.2 table), looked up
    in current docs — not in patterns remembered from older versions. Prefer
    the framework's built-in feature over hand-rolled code; examples, each
    only when the installed version has it: PHP enums with methods, readonly
    properties / classes, first-class callables, `#[\Override]`; Laravel
    `casts()` + enum casts, `Attribute` accessors, `Rule::enum`,
    `chunkById` / `lazyById`, `whereAny` / `whereAll`, `Http::retry` / pool,
    `once()`, `Str` / `Number` helpers, scoped route model binding; Nova
    `dependsOn()`, `->filterable()`, `Badge` fields, `Nova::mainMenu()`; Vue
    `<script setup>` + `defineModel` / `useTemplateRef`, composables instead
    of mixins; JS optional chaining, `structuredClone`, `Intl.*`,
    `AbortController`; CSS custom properties, `:has()`, container queries,
    logical properties, `clamp()`; Tailwind v4 CSS-first config (`@theme`,
    `@utility`) in a v4 project, `tailwind.config.js` idioms only in a v3 one.
    **Guardrails — consistency beats novelty:** the project's `CLAUDE.md` and
    the sibling code's conventions win over a newer idiom; do not add a second
    pattern next to an established one unless the task *is* the refactor (then
    migrate consistently); **no drive-by rewrites** of code the task does not
    otherwise touch — record the opportunity as a Step 6.5.5 `suggest` row
    instead; never use an API deprecated in the installed version; never use a
    feature newer than the installed version, the PHP floor or the
    browserslist target (it will not run); no new dependency for something
    the framework already ships.
- **Implement the auto-proposed tests from Step 6.2.5 in the same task**, so
  the commit + time log cover both the feature and its verification. Place
  files where the framework conventionally lives (`tests/Feature/…`,
  `tests/Unit/…`, `tests/Browser/…` for Laravel; `tests/e2e/…` or
  `cypress/e2e/…` for JS).

### 6.4 Test
Skipped in `fast` mode (Step 2.65) unless `--test-after=true` or a task
explicitly asks for tests — the deferred list records it instead.
If the project has a test suite and the change is testable:
- Laravel/Pest: `php artisan test --compact --filter=<RelevantTest>`
- Generic JS: `npm test -- --watchAll=false <pattern>` (or whatever the project uses)
- Run only what is relevant — do not run the whole suite per task.

### 6.5 Format
If PHP files changed: `vendor/bin/pint --dirty --format agent`. Skipped in
`fast` mode (Step 2.65) — `/work-mode full` runs the formatter once at the end.

### 6.5.5 Quality self-check (v1.5.0)

Before the timer stops, walk **this task's own diff** against the active
dimensions from Step 6.2 — the same five keys `/teamwork-task-test` audits in
its Step 6.6, so a gap found here costs minutes instead of a QA round trip.
Skip the step (status `skipped(--dimensions)` for every key) when the active
set is empty.

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
# Everything this task changed: tracked edits + new untracked files.
CHANGED=$( { git diff HEAD --name-only; git ls-files --others --exclude-standard; } | sort -u | grep -v '^$')
printf '%s\n' "$CHANGED"
```

For **every** active key record exactly one status:

- `checked` — you read the relevant hunks against the Step 6.3 rules and
  nothing is left open (including things you just fixed).
- `not_applicable(<why>)` — the diff does not touch it (e.g. `no UI files in
  diff`, `no new screen`).
- `skipped(--dimensions)` — excluded by config or flag.
- `open(<item> @ <file:line>)` — a finding you are not fixing in this task
  (too big, needs a business decision, pre-existing code the task walked
  past). One row per item.
- `suggest(<opportunity> @ <file:line>)` — **`framework` only**, in addition
  to its status row: a modernisation opportunity in code this task read but
  did not otherwise change, which the no-drive-by guardrail kept out of the
  diff. Advisory — never a defect, never gates, never counts as open. Only
  with a concrete replacement the installed version supports; at most three
  per task. Something in that code that is a defect under another key (an
  N+1, an unbounded `all()` walked in PHP, a missing policy check, an
  orphaned screen) is an `open` row under **that** key, never a `framework`
  tip — the advisory label must not hide a finding QA is owed.

**Fix what is cheap now** — a missing label or `alt`, an N+1 you can close
with one `with()`, a missing policy call, a missing menu entry or relation
tab, a deprecated call in a line you wrote. The fix belongs to this task and
lands in its commit (Step 6.7).

For **`framework`**, ask of the new and changed lines only:
- Does anything use an API **deprecated** in the installed version, or a
  pattern the installed version **replaced** (e.g. a `$casts` property where
  the siblings already use `casts()`, a mixin where the project uses
  composables)?
- Does anything **hand-roll** what the framework already ships (a manual
  `foreach` + `->save()` batch instead of `chunkById`, string-built number /
  date formatting instead of `Number` / `Intl`, a new package for a built-in)?
- Does anything need a version **newer** than the installed one, the PHP
  floor or the browserslist target? That is a bug, not a style note — it will
  not run. Fix it now; it is never left `open` or `suggest`.
- Did a newer idiom break the sibling convention? Then the convention wins —
  revert to it and note why in the detail.
- For every non-obvious choice, name the version and the docs page in the
  `checked` detail (e.g. `Laravel 12.53: casts() + enum cast — docs
  eloquent-mutators#enum-casting`).

Record every status in `$TW_JOB_DIR/build_quality.tsv`
(`taskId<TAB>dimension<TAB>status<TAB>detail`, detail `-` when there is
none) — Step 6.6.5, Step 7 and Step 8 read it. The **status column holds the
bare keyword** — `checked`, `not_applicable`, `skipped`, `open` or `suggest`
— and the parenthesised part above goes into the detail column
(`--dimensions` for `skipped`, `<item> @ <file:line>` for `open`,
`<opportunity> @ <file:line>` for `suggest`): the Step 6.6.5 gate matches
`open` exactly, so `open(…)` in the status column would slip past it.

```bash
# (run preamble — see Step 3; this is a new Bash call)
TASK_ID="<current task id>"
QUALITY_FILE="$TW_JOB_DIR/build_quality.tsv"
printf '%s\t%s\t%s\t%s\n' "$TASK_ID" "ui_ux"        "checked"        "-"                              >> "$QUALITY_FILE"
printf '%s\t%s\t%s\t%s\n' "$TASK_ID" "performance"  "not_applicable" "no queries or migrations in diff" >> "$QUALITY_FILE"
printf '%s\t%s\t%s\t%s\n' "$TASK_ID" "reachability" "open" \
  "ExportLog is URL-only by design (e-mail deep link) → allow_orphans candidate @ app/Nova/ExportLog.php" >> "$QUALITY_FILE"
printf '%s\t%s\t%s\t%s\n' "$TASK_ID" "framework"    "checked" \
  "Laravel 12.53: casts() + enum cast for ExportLog::status — docs eloquent-mutators#enum-casting" >> "$QUALITY_FILE"
printf '%s\t%s\t%s\t%s\n' "$TASK_ID" "framework"    "suggest" \
  "number_format(\$total, 2, ',', ' ') . ' €' → Number::currency(\$total, 'EUR', 'sk') (untouched helper, Laravel 12.53) @ app/Support/InvoiceFormatter.php:31" >> "$QUALITY_FILE"
```

Hard rules:
- **Never block silently.** An `open` item never aborts the task and is never
  dropped — Step 7 prints it per task and Step 8 hands it to
  `/teamwork-task-test`. An open `security` or `reachability` item also makes
  the Step 6.6.5 safety gate ask before committing (in the default
  `auto_commit_mode=when_safe`; `always` never asks, `never` always does).
  `framework` rows never make the gate ask — an `open` one is listed like any
  other, a `suggest` one only as an advisory note.
- **Never claim a dimension was checked when it was not.** If the diff was too
  large to read, or you ran out of room, the status is
  `open(not reviewed — <reason>)`, not `checked`.

### 6.6 Stop timer + decide minutes

v1.3.0 splits the rounding decision into **two zones** controlled by the new
`round_threshold_minutes` key (default `4`):

- `ELAPSED_MIN >= round_threshold_minutes` → round to the **nearest**
  `time_rounding_minutes` step using **half-up integer rounding**
  (`((n + ROUND/2) / ROUND) * ROUND`). With the defaults (`threshold=4`,
  `ROUND=5`):
  | elapsed | logged |
  | ------- | ------ |
  | 4 min   | 5 min  |
  | 5 min   | 5 min  |
  | 6 min   | 5 min  |
  | 7 min   | 5 min  |
  | 8 min   | 10 min |
  | 9 min   | 10 min |
  | 10 min  | 10 min |
  | 11 min  | 10 min |
  | 12 min  | 10 min |
  | 13 min  | 15 min |
  | 14 min  | 15 min |
  | 15 min  | 15 min |

  The exact tipping point is `n + ROUND/2 >= next ROUND multiple`, i.e.
  the value `2.5`, `7.5`, `12.5`, … rounds *up*. The previous round-up
  policy (every value over the previous step rounded up) systematically
  over-billed by 0–4 min per task; round-to-nearest distributes those
  fractional minutes honestly in both directions.
- `ELAPSED_MIN >= min_log_minutes` but **strictly below** the threshold → log
  the **raw** elapsed minutes (1, 2, or 3 with the default threshold of 4).
  The cursor advances by exactly that many minutes — `TIMELOG_SUB_ROUND=1`
  is set so Step 7 surfaces the entry as a deliberate sub-round, and the
  next log starts off the 5-minute grid.
- `ELAPSED_MIN < min_log_minutes` → either log `min_log_minutes` as a floor
  (when `time_mode=real_rounded_5m`) or, if the headroom guard in Step 6.6.1
  fires, skip the log entirely (`TIMELOG_SKIPPED=1`).

```bash
END_TS=$(date +%s)
ELAPSED_MIN=$(( (END_TS - START_TS + 59) / 60 ))   # ceil to minutes
ROUND=$(jq -r '.time_rounding_minutes // 5'      "$CONFIG_FILE")
THRESHOLD=$(jq -r '.round_threshold_minutes // 4' "$CONFIG_FILE")
MIN_LOG=$(jq -r '.min_log_minutes // 1'          "$CONFIG_FILE")
ROUND_SECS=$(( ROUND * 60 ))

# Defensive: if the user set THRESHOLD > ROUND, the "below threshold but
# rounding applies anyway" zone is incoherent. Treat THRESHOLD > ROUND as
# "round everything once ELAPSED >= ROUND" — never log raw minutes for
# ELAPSED values that are already >= ROUND.
if [ "$THRESHOLD" -gt "$ROUND" ]; then THRESHOLD=$ROUND; fi

DURATION_SOURCE="rounded_nearest"   # rounded_nearest | sub_round_elapsed | floored_min_log

if [ "$ELAPSED_MIN" -ge "$THRESHOLD" ]; then
  # Round to NEAREST ROUND (half-up). Examples for ROUND=5:
  #   4 → 5     (4+2)/5 = 1 → 5
  #   6 → 5     (6+2)/5 = 1 → 5
  #   7 → 5     (7+2)/5 = 1 → 5
  #   8 → 10    (8+2)/5 = 2 → 10
  #   12 → 10   (12+2)/5 = 2 → 10
  #   13 → 15   (13+2)/5 = 3 → 15
  HALF=$(( ROUND / 2 ))
  DURATION_MIN=$(( ((ELAPSED_MIN + HALF) / ROUND) * ROUND ))
  # Never log a 0-minute entry — if rounding collapsed to 0 (only possible
  # when ROUND==1 and ELAPSED==0, which is also caught by the MIN_LOG path),
  # bump up to ROUND.
  if [ "$DURATION_MIN" -lt "$ROUND" ]; then DURATION_MIN=$ROUND; fi
  DURATION_SOURCE="rounded_nearest"
elif [ "$ELAPSED_MIN" -ge "$MIN_LOG" ]; then
  # Below the rounding threshold but worth logging — write the raw minutes.
  # Example with defaults (THRESHOLD=4, ROUND=5, MIN_LOG=1):
  #   1 min →  1 min   (raw, sub-round)
  #   2 min →  2 min   (raw, sub-round)
  #   3 min →  3 min   (raw, sub-round)
  #   4 min →  5 min   (round-to-nearest — handled by the branch above)
  DURATION_MIN=$ELAPSED_MIN
  DURATION_SOURCE="sub_round_elapsed"
  SUB_ROUND_TIMELOGS+=("${TASK_ID}: ${DURATION_MIN}m (elapsed below ${THRESHOLD}m threshold)")
else
  # Floor — implementation finished faster than MIN_LOG; log the floor so the
  # session still produces a trace of this task.
  DURATION_MIN=$MIN_LOG
  DURATION_SOURCE="floored_min_log"
fi
```

Both `rounded_nearest` and `sub_round_elapsed` results may still be
**clamped or skipped** by the future-timestamp guard in Step 6.6.1 below —
the elapsed time is *what the work cost*, the headroom is *what fits
before `now()`* and the smaller of the two wins. The `DURATION_SOURCE`
flag plumbs through to Step 7 so the user can see why an entry ended up
sub-round.

If `time_mode == "ask"` → **AskUserQuestion** with the measured `DURATION_MIN` as the suggested answer; let the user override. The user's override is taken as-is — no extra rounding — because at that point the user has explicitly approved a specific minute count.

### 6.6.1 Future-timestamp guard (hard rule)

The session cursor advances by *logged* minutes, not by *real* minutes — so on a fast run where the model produces work much faster than wall-clock time would allow, the cursor drifts into the future. Teamwork accepts those POSTs without complaint (timesheets can legally hold future entries), but the resulting timesheet is useless for billing and confuses anyone reading it.

Cap `DURATION_MIN` (as computed in Step 6.6) so the resulting log window ends at or before `now()`. v1.3.0 simplifies the clamp logic now that Step 6.6 already produces a fully-decided duration (round-up, sub-round-from-elapsed, or floored-to-min-log). The clamp is "headroom-aware" but **does not re-round** — if Step 6.6 said 3 min (sub-round) and headroom is 10 min, the log stays 3 min. If Step 6.6 said 10 min (round-up) and headroom is 7 min, the log is clamped to 7 min and the entry becomes a sub-round headroom clamp.

```bash
NOW=$(date +%s)
# MIN_LOG was already read in Step 6.6; re-read here defensively in case
# Step 6.6 was skipped (time_mode=ask override).
MIN_LOG=$(jq -r '.min_log_minutes // 1' "$CONFIG_FILE")

# Exact minutes of headroom (NOT rounded down).
HEADROOM_SECS=$(( NOW - SESSION_CURSOR_TS ))
if [ "$HEADROOM_SECS" -lt 0 ]; then HEADROOM_SECS=0; fi
HEADROOM_MIN_RAW=$(( HEADROOM_SECS / 60 ))

TIMELOG_SKIPPED=0
# TIMELOG_SUB_ROUND may already be 1 from Step 6.6 (elapsed below threshold).
TIMELOG_SUB_ROUND=${TIMELOG_SUB_ROUND:-0}

if [ "$HEADROOM_MIN_RAW" -lt "$MIN_LOG" ]; then
  echo "  ⚠ session cursor at $(date -r "$SESSION_CURSOR_TS" +%H:%M) is at/past now ($(date -r "$NOW" +%H:%M)); headroom ${HEADROOM_MIN_RAW}m < min_log_minutes (${MIN_LOG}m) — skipping timelog for task ${TASK_ID}" >&2
  TIMELOG_SKIPPED=1
  SKIPPED_TIMELOGS+=("${TASK_ID} (cursor caught up with real time)")
elif [ "$DURATION_MIN" -gt "$HEADROOM_MIN_RAW" ]; then
  # The Step 6.6 duration (whether round-up or sub-round-from-elapsed) does
  # not fit before now() — clamp to the exact headroom in raw minutes. We do
  # NOT re-round to ROUND here because that could short-change a 7-min-headroom
  # round-up entry by logging only 5 min, or skip a 3-min headroom entirely.
  # Logging the exact headroom is more honest and keeps the cursor in sync.
  echo "  ⚠ measured ${DURATION_MIN}m would overshoot now() — clamping to ${HEADROOM_MIN_RAW}m so the timelog stays in the past" >&2
  CLAMPED_TIMELOGS+=("${TASK_ID}: ${DURATION_MIN}m → ${HEADROOM_MIN_RAW}m")
  DURATION_MIN=$HEADROOM_MIN_RAW
fi

# Final classification: is DURATION_MIN a clean multiple of ROUND? If not,
# mark it as sub-round so Step 7 surfaces the entry. Step 6.6 may have
# already added a SUB_ROUND_TIMELOGS row with the "elapsed below threshold"
# reason — only add the "clamped by headroom guard" row if no row exists for
# this task yet, so the user sees one reason per task.
if [ "${TIMELOG_SKIPPED:-0}" = "0" ] && [ $(( DURATION_MIN % ROUND )) -ne 0 ]; then
  TIMELOG_SUB_ROUND=1
  if ! printf '%s\n' "${SUB_ROUND_TIMELOGS[@]}" | grep -q "^${TASK_ID}:"; then
    SUB_ROUND_TIMELOGS+=("${TASK_ID}: ${DURATION_MIN}m (clamped by headroom guard)")
  fi
fi
```

Why this matters: without the guard, a fast run will silently produce timesheet entries with start/end times that have not happened yet. The PM reads "16:00 — refactor done" at 13:30 and rightly asks how that is possible.

`TIMELOG_SKIPPED=1` plumbs through Step 6.8 — the POST is skipped, the cursor is **not** advanced, and Step 6.8.5 (board move to the done column — *Done - Local* or its fallback) is **also** skipped because the task is not yet considered finished from a billing standpoint. `TIMELOG_SUB_ROUND=1` does NOT skip — the log goes through normally; only the duration is below `ROUND` (or is not a clean multiple of `ROUND`), and Step 7 surfaces `SUB_ROUND_TIMELOGS` alongside the skipped/clamped lists so the user knows which entries broke the 5-min cosmetic alignment and why (`elapsed below threshold` vs `clamped by headroom guard`).

Initialize the accounting arrays once at the start of the worker loop:

```bash
SKIPPED_TIMELOGS=()
CLAMPED_TIMELOGS=()
SUB_ROUND_TIMELOGS=()
```

### 6.6.5 Safety gate — auto-commit or ask?

Before staging and committing, inspect the diff to decide whether the change is trivial enough for an unattended commit or risky enough to deserve a human pass. This step runs **before** `git add` in Step 6.7, so the measurement uses `git diff HEAD` (working tree + index, against the last commit) — that way both freshly modified files and anything that was already staged earlier in the run are counted:

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
MODE=$(jq -r '.auto_commit_mode // "when_safe"' "$CONFIG_FILE")
MAX_LINES=$(jq -r '.auto_commit_max_diff_lines // 100' "$CONFIG_FILE")

PROCEED="auto"        # auto | ask
REASONS=()

case "$MODE" in
  always) PROCEED="auto" ;;
  never)  PROCEED="ask"; REASONS+=("auto_commit_mode=never") ;;
  when_safe)
    # Compare everything (worktree + index) against HEAD so a pre-staged file
    # is not silently ignored by the safety gate.
    CHANGED=$(git diff HEAD --name-only 2>/dev/null)

    PATTERNS=$(jq -r '.auto_commit_risky_patterns[]' "$CONFIG_FILE" | paste -sd '|' -)
    if [ -n "$PATTERNS" ] && printf '%s\n' "$CHANGED" | grep -qiE "$PATTERNS"; then
      MATCHED=$(printf '%s\n' "$CHANGED" | grep -iE "$PATTERNS" | head -3 | tr '\n' ' ')
      REASONS+=("UI/template/styling files changed: ${MATCHED}")
    fi

    # v1.5.0: an open security / reachability item from the Step 6.5.5
    # self-check is worth a human look before it lands in a commit.
    OPEN_QUALITY=$(awk -F '\t' -v id="$TASK_ID" \
      '$1 == id && $3 == "open" && ($2 == "security" || $2 == "reachability") { printf "%s: %s; ", $2, $4 }' \
      "$TW_JOB_DIR/build_quality.tsv" 2>/dev/null)
    [ -n "$OPEN_QUALITY" ] && REASONS+=("open build-quality items: ${OPEN_QUALITY}")

    LINES=$(git diff HEAD --shortstat 2>/dev/null \
      | grep -oE '[0-9]+ (insertion|deletion)' \
      | awk '{s+=$1} END {print s+0}')
    if [ "${LINES:-0}" -gt "$MAX_LINES" ]; then
      REASONS+=("diff is ${LINES} lines (> ${MAX_LINES} threshold)")
    fi

    [ "${#REASONS[@]}" -gt 0 ] && PROCEED="ask"
    ;;
esac

if [ "$PROCEED" = "ask" ]; then
  # AskUserQuestion options:
  #   - Approve commit (and continue to log + board move)
  #   - Inspect first  (pause; user runs `git diff` etc., then re-ask)
  #   - Abort task     (do NOT commit, do NOT log, do NOT move board;
  #                     leave changes uncommitted so the user can finish manually)
  :
fi
```

Trivial textual edits (e.g. a typo fix in `.md`, a copy change in a config string) fall straight through. UI/template work and big rewrites always prompt. If the user picks **Abort task**, do not commit, do not log time for this task, do not move the board card — leave the working tree as-is and move to the next task (or end the loop for a single-task run).

### 6.7 Git commit
Use the project commit convention from `~/.claude/CLAUDE.md`, **extended** with the Teamwork task ID in square brackets right after the scope:
```
TYPE(scope)[<task-id>]: Message

Optional detailed description on next lines.
```
- `TYPE`: `CREATE`, `UPDATE`, `EDIT`, `FIX`, `REMOVE`, `MOVE`, `UPGRADE`, `DELETE` …
- `scope`: module / model / area touched (lowercase, e.g. `user`, `auth`, `invoice`)
- `<task-id>`: numeric Teamwork task ID — exactly as returned by the API (no `#` prefix), so a grep for `[123456]` finds every commit related to that task
- **Do NOT add `Co-Authored-By` lines.**

Stage only files that belong to this task (`git add <paths>` — never `git add -A` / `git add .claude`), make sure the work-mode state files are not in the index (in **every** mode — a list left over from an earlier `fast` stretch must not ride along either), then commit by **piping the body to `git commit -F -`** instead of using an unquoted heredoc — the Teamwork task description / final_summary can legitimately contain `$(...)`, backticks, or `${VAR}` which would otherwise be expanded (or executed) by the shell:

```bash
# Work-mode state is local: unstage it if anything put it in the index.
# A no-op for paths that are not staged or do not exist.
git reset -q -- .claude/work-mode-deferred.local.md .claude/work-mode.local.md \
  .claude/wame-deferred.local.md .claude/wame-mode.local.md

COMMIT_TITLE="UPDATE(${SCOPE})[${TASK_ID}]: ${SHORT_SUMMARY}"

# Build the body in a way that NEVER lets shell expansion touch user-supplied
# text. The task's final_summary is treated as opaque literal content.
{
  printf '%s\n\n' "$COMMIT_TITLE"
  printf '%s\n' "$FINAL_SUMMARY_FOR_COMMIT_BODY"
} | git commit -F -
```

If you must keep a heredoc for clarity, use the **quoted** `'EOF'` form so nothing in the body is expanded:

```bash
git commit -F - <<'EOF'
UPDATE(scope)[TASK_ID_PLACEHOLDER]: Short summary

Final summary text exactly as fetched from Teamwork.
EOF
```

…and then substitute placeholders with `sed`. The piped-printf form above is simpler and equally safe.

**Never** add `Co-Authored-By` lines. Keep the commit body free of any secrets or token strings (the API token must never appear).

Capture the commit hash for the time log and final summary: `COMMIT_HASH=$(git rev-parse --short HEAD)`.

In `fast` mode, now append the task's block to `.claude/work-mode-deferred.local.md`
(Step 2.65 — `mv` the legacy `.claude/wame-deferred.local.md` to that name
first when only the legacy file exists). The append happens **after** the
commit, so the file can never be part of it — never stage it.

Examples:
```
CREATE(user)[123456]: Create user module
UPDATE(invoice)[123789]: Add PDF export action
FIX(auth)[124001]: Resolve session timeout race condition
```

### 6.8 Log time back to Teamwork
`POST ${BASE}/projects/api/v3/tasks/${TASK_ID}/time.json`

**Non-overlapping, sequential time logs (hard rule).** Even though implementation work may overlap in real time (parallel tool calls, interleaved tasks), the time entries written to Teamwork **must be strictly contiguous** — every log's `date + time + minutes` window must end exactly where the next log's window begins. The plugin uses the `SESSION_CURSOR_TS` from Step 5.5 as the single source of truth and never re-reads wall-clock time inside the loop. Both the start time and the duration are aligned to the rounding step (default 5 minutes), so a log will say `start 10:15, duration 20 min` and the next log will say `start 10:35` — never `10:32` or `10:17`.

**Description tone — business, not technical.** Write the description from the perspective of someone reading the timesheet for billing or status (PM, client, accountant), not a developer reading a code review. State **what was delivered for the user/business**. Include a technical detail only when it materially helps identify the work (e.g. a specific module name, a flag/feature key, a migration number) — never variable names, line counts, library versions, or diff stats. Keep it to 1–2 sentences in the configured `default_language` (Slovak by default).

**Commit hash suffix.** If `include_commit_hash_in_log_description == true` (default), append ` (commit: <short-hash>)` to the business description so a timesheet reader can jump straight to the change:

> *"Pridaná možnosť exportu faktúr do PDF s podporou témy nájomcu. (commit: a1b2c3d)"*

**Good (business):**
- *"Pridaná možnosť exportu faktúr do PDF s podporou témy nájomcu. (commit: a1b2c3d)"*
- *"Opravený výpadok prihlasovania pri súbežnom obnovení relácie. (commit: 9f4e2b1)"*
- *"Doplnené akceptačné kritériá pre modul Užívatelia — pripravené na testovanie. (commit: 7c8d9e0)"*

**Bad (technical, avoid):**
- *"Refactored InvoiceController::export() to use new DompdfRenderer, added 3 tests, ran pint."*
- *"Updated 8 files, +142 −37 lines, bumped filament/filament to 5.2."*
- *"Implemented onMounted() lifecycle hook in InvoiceForm.vue."*

**Sequential timestamp logic:**

```bash
# Future-timestamp guard from Step 6.6.1 may have set TIMELOG_SKIPPED=1.
# In that case do NOT POST — leave TIMELOG_OK=0 so Step 6.8.5 skips the board move
# and the cursor is not advanced. The accounting array SKIPPED_TIMELOGS is rendered
# in Step 7.
if [ "${TIMELOG_SKIPPED:-0}" = "1" ]; then
  TIMELOG_OK=0
else

BILLABLE=$(jq -r '.is_billable_by_default | if . == null then true else . end' "$CONFIG_FILE")
INCL_HASH=$(jq -r '.include_commit_hash_in_log_description | if . == null then true else . end' "$CONFIG_FILE")

# Use the session cursor — NOT wall-clock time.
# The `time` field is interpreted by Teamwork in the user's LOCAL/profile
# timezone (it is the inverse of `timeLogged`, which comes back in UTC). So the
# absolute cursor epoch is formatted here with LOCAL `date -r` (no `-u`).
# Do NOT switch this to `date -ju`/`date -u` — that would post the UTC
# wall-clock as if it were local and shift every entry by the local offset,
# making logs land hours early and overlap. See the TIMEZONE CONTRACT note on
# parse_iso in Step 5.5.
LOG_DATE=$(date -r "$SESSION_CURSOR_TS" +%Y-%m-%d 2>/dev/null || date -d "@$SESSION_CURSOR_TS" +%Y-%m-%d)
LOG_TIME=$(date -r "$SESSION_CURSOR_TS" +%H:%M:%S 2>/dev/null || date -d "@$SESSION_CURSOR_TS" +%H:%M:%S)

DESCRIPTION="$DESCRIPTION_BUSINESS"
if [ "$INCL_HASH" = "true" ] && [ -n "$COMMIT_HASH" ]; then
  DESCRIPTION="${DESCRIPTION} (commit: ${COMMIT_HASH})"
fi

PAYLOAD=$(jq -nc \
  --argjson minutes "$DURATION_MIN" \
  --arg desc "$DESCRIPTION" \
  --arg date "$LOG_DATE" \
  --arg time "$LOG_TIME" \
  --argjson billable "$BILLABLE" \
  '{timelog: {hours: 0, minutes: $minutes, description: $desc, date: $date, time: $time, isbillable: $billable}}')

HTTP=$(curl -sS -o /tmp/tw_timelog_resp.json -w "%{http_code}" -u "$AUTH" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST -d "$PAYLOAD" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/time.json")

if [ "$HTTP" -ge 200 ] && [ "$HTTP" -lt 300 ]; then
  # Advance the cursor by exactly the logged duration (already a multiple of ROUND).
  # This guarantees the next log's start time = this log's end time. No gaps, no overlaps.
  SESSION_CURSOR_TS=$(( SESSION_CURSOR_TS + DURATION_MIN * 60 ))
  TIMELOG_OK=1
else
  echo "  ⚠ time log POST failed (HTTP $HTTP) — cursor NOT advanced, board move skipped" >&2
  TIMELOG_OK=0
fi

fi  # end of TIMELOG_SKIPPED guard
```

Verify HTTP status. On non-2xx → report to the user (do not retry blindly; the commit already exists). Do **not** advance `SESSION_CURSOR_TS` if the POST failed — the next successful log should reuse the same start time so the user's timesheet stays contiguous. Also **skip the next "move to the done column"** step (6.8.5), because the task is not yet considered finished from a billing standpoint.

### 6.8.5 Move task to the done column ("Done - Local", fallbacks "Internal testing" → "Testing")

If the time log succeeded and the project has a resolved done stage, post the card to the done column — `board_workflow.done_stage` (default *Done - Local*, the WAME board's column for work finished locally), else the first of `done_stage_fallbacks` (default *Internal testing*, then *Testing*) that exists on the task's board, as resolved in Step 3.3:

```bash
# (run preamble — see Step 3)
TASK_ID="<current task id>"
TIMELOG_OK="<1 if the Step 6.8 POST succeeded, else 0>"
PROJECT_ID=$(awk -F '\t' -v id="$TASK_ID" '$1 == id { print $2; exit }' "$TW_JOB_DIR/task_projects.tsv" 2>/dev/null)
BOARD_FILE="$TW_JOB_DIR/board_${PROJECT_ID}.tsv"
WORKFLOW_ID=""; DONE_STAGE_ID=""; DONE_RESOLVED_NAME=""; BOARD_DISABLED=1
if [ -n "$PROJECT_ID" ] && [ -s "$BOARD_FILE" ]; then
  WORKFLOW_ID=$(awk -F '\t'        '$1 == "workflow_id"        { print $2; exit }' "$BOARD_FILE")
  DONE_STAGE_ID=$(awk -F '\t'      '$1 == "done_stage_id"      { print $2; exit }' "$BOARD_FILE")
  DONE_RESOLVED_NAME=$(awk -F '\t' '$1 == "done_resolved_name" { print $2; exit }' "$BOARD_FILE")
  BOARD_DISABLED=$(awk -F '\t'     '$1 == "disabled"           { print $2; exit }' "$BOARD_FILE")
else
  echo "  ⚠ [#${TASK_ID}] no project id / no Step 3.3 board state (project '${PROJECT_ID}') — board move to the done column skipped" >&2
fi
BW_ENABLED=$(jq -r '.board_workflow.enabled | if . == null then true else . end' "$CONFIG_FILE")

if [ "${TIMELOG_OK:-0}" = "1" ] \
   && [ "$BW_ENABLED" = "true" ] \
   && [ "$BOARD_DISABLED" != "1" ] \
   && [ -n "$WORKFLOW_ID" ] && [ -n "$DONE_STAGE_ID" ]; then
  HTTP=$(curl -sS -o /dev/null -w "%{http_code}" -u "$AUTH" \
    -H "Content-Type: application/json" -H "Accept: application/json" \
    -X POST -d "{\"taskIds\":[${TASK_ID}]}" \
    "${BASE}/projects/api/v3/workflows/${WORKFLOW_ID}/stages/${DONE_STAGE_ID}/tasks.json")
  if [ "$HTTP" -ge 200 ] && [ "$HTTP" -lt 300 ]; then
    CURRENT_BOARD_STAGE_FOR_TASK="${DONE_RESOLVED_NAME:-done column}"
  else
    echo "  ⚠ board move to '${DONE_RESOLVED_NAME:-done column}' failed (HTTP $HTTP) — continuing" >&2
  fi
fi
```

Interaction with `auto_complete_finished_tasks`: both run independently. If the user enabled task completion *and* board moves, the task ends up `completed=true` *and* in the done stage — Teamwork is fine with both flags set.

### 6.9 Cleanup attachments

Once the time log (and the optional board move) is done, remove the attachment working folder according to `attachments_cleanup`:

```bash
CLEAN_MODE=$(jq -r '.attachments_cleanup // "after_timelog"' "$CONFIG_FILE")
case "$CLEAN_MODE" in
  after_commit|after_timelog) rm -rf "$ATTACH_DIR" ;;
  never) : ;;
esac
```

`after_commit` is intentionally honored here (even though we are past 6.7) so power-users who set it explicitly still get the same end state. The recommended default `after_timelog` keeps files available for inspection if a time log POST fails.

### 6.10 Complete the task in Teamwork (optional)

If `config.auto_complete_finished_tasks == true`, or the user opts in at the first task of this session:
```bash
curl -sS -u "$AUTH" -X PUT -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/complete.json"
```

On the first task of the session ask once via **AskUserQuestion**: *"Mark tasks as completed in Teamwork after each commit?"* — and remember the answer for the rest of this run.

### 6.11 Move to next task
Continue the loop.

---

## Step 7 — Final summary

After the loop ends, print a markdown table:

| # | Task ID | Title | Minutes | Commit | TW status | Board stage | Build quality |
|---|---------|-------|---------|--------|-----------|-------------|---------------|

The **Build quality** cell comes from `$TW_JOB_DIR/build_quality.tsv`
(Step 6.5.5), one short token per active key — e.g. `ui_ux ✓ · perf n/a ·
sec ✓ · reach open(1) · fw ✓ +1 tip`, or `skipped (--dimensions=none)`. A
task without rows in that file shows `not self-checked` — never an implied ✓.
`+N tip` counts the task's `framework` `suggest` rows; they never turn the
cell into a failure.

Followed by a short status block:

```
Time cursor: <TIME_CURSOR_SOURCE>           e.g. "last_timelog @ 10:50"
                                            or  "skill start (first log of day)"
Work mode (Step 2.65):                        <full | fast — N task(s) recorded in .claude/work-mode-deferred.local.md; run /work-mode full before pushing>
Tasklist filter:                              <start columns <TF_TODO_LABEL> + me — N implemented, M analyse-only (K not on the board), L dropped> | <disabled / single-task URL>
Analyse-only tasks (not touched):             <list of [#id] title from ANALYSE_ONLY_TASKS or none>
Timelogs skipped (cursor caught up with now): <list from SKIPPED_TIMELOGS or none>
Timelogs below 5-min rounding (sub-round):    <list from SUB_ROUND_TIMELOGS or none>
Timelogs clamped to fit before now:           <list from CLAMPED_TIMELOGS or none>
Attachments skipped (size):                   <lines of $TW_JOB_DIR/skipped_attachments.txt or none>
Board moves disabled for projects:            <projects whose board_<id>.tsv says disabled 1, or none>
Comments read (Step 3.5):                     <per task COMMENTS_STATE, e.g. "[#123] newest of 8 read">
Open build-quality items (handed to QA):      <per task "[#id] <dimension>: <detail>" from build_quality.tsv rows with status open, or none>
Framework versions (Step 6.2, detected once): <one line from framework_versions.tsv, e.g. "laravel/framework v12.53.0 · php ^8.4 · laravel/nova 5.7.7 · tailwindcss 3.4.19 · browserslist not set", or "not detected (framework dimension inactive)">
Framework opportunities (advisory, not changed): <per task "[#id] <opportunity> @ <file:line>" from build_quality.tsv rows with status suggest, or none>
```

Render the open build-quality items verbatim, grouped per task — they are the
part of the run the developer still owes, and Step 8 hands the same list to
`/teamwork-task-test`. The framework opportunities are **not** owed: they sit
in code the task did not otherwise change (the Step 6.3 no-drive-by
guardrail), so list them as candidates for a separate refactor task, not as
defects.

When `SKIPPED_TIMELOGS` is non-empty, also print a short paragraph telling the user **why** those entries did not land in Teamwork (the cursor reached `now()` mid-run, typically because the model produces work faster than wall-clock) and suggest they either re-run later (the cursor will continue from the last successful log) or fill those minutes in manually. The skipped tasks' board cards were **not** moved to the done column (*Done - Local* or its fallback) either — they will be moved on the next run that successfully logs them.

The push reminder is intentionally **not** printed here — it moves to Step 9, after the optional verification handoff, so the user sees test results before they decide whether to push.

---

## Step 8 — Auto-run `/teamwork-task-test` (optional handoff)

**Check the work mode first (v1.6.0).** When `WORK_MODE=fast` (Step 2.65) and the user did not pass `--test-after=true`, skip this whole step: print `ℹ Fast mode — /teamwork-task-test not run; /work-mode full runs the deferred checks once.` and go to Step 9. When the handoff does run in `fast` mode (explicit `--test-after=true`), append `--mode=full` to `TEST_ARGS` in Step 8.2 so the QA pass does not inherit the project's fast mode.

Otherwise, if `config.auto_run_tests_after == true` (default, added in 1.1.1) and the user did not pass `--test-after=false`, hand off to the `/teamwork-task-test` skill for per-criterion verification.

**Why this exists.** The implementation phase produces a passing build and a commit per task, but it does not — by itself — prove that each *acceptance criterion* of the original Teamwork task was actually met. The companion `teamwork-task-test` skill exists precisely for that: it parses each acceptance criterion out of the task description, maps it onto the project's existing tests, runs them, drives a browser via the chrome-devtools MCP for UI-shaped criteria, and writes manual scenarios for whatever cannot be automated. Calling it automatically at the end closes the implement → verify loop in a single command.

### 8.1 — Detect whether the test skill is available

The `teamwork-task-test` skill is published on the same WAME marketplace. Two signals tell us if it is currently installed in this session:

1. **The Skill tool's available-skills list** — when invoked via the `Skill` tool, only listed skill names succeed. The model can see this list in the session's system-reminder messages. If `teamwork-task-test` appears there, it is installed.
2. **Filesystem fallback** — check for an installed skill folder under the plugins cache
   (`~/.claude/plugins/cache/<marketplace>/teamwork-task-test/<version>/skills/teamwork-task-test/SKILL.md`).
   Use `find`, not a glob: the 1.4.2 `ls …/*/teamwork-task-test/SKILL.md`
   never matched that depth, and in zsh an unmatched glob is a hard
   `no matches found` error.
   ```bash
   if find "$HOME/.claude/plugins" -path '*teamwork-task-test*' -name SKILL.md -print 2>/dev/null | grep -q .; then
     TEST_SKILL_INSTALLED=1
   else
     TEST_SKILL_INSTALLED=0
   fi
   ```

If neither signal confirms availability, print a single-line tip and end the run:

```
ℹ Tip: install /teamwork-task-test to automatically verify acceptance criteria after each implementation.
   /plugin install teamwork-task-test@wame
```

Do **not** treat this as an error — the user may have intentionally chosen not to install the verification skill.

### 8.2 — Skip when nothing to verify

If every task in the run ended in *aborted*, *skipped*, or had a failed time log (`TIMELOG_OK == 0`), there is nothing meaningful to verify. Print one line ("No completed tasks in this run — skipping verification.") and proceed to Step 9.

Otherwise, build the argument string from the originally-parsed URL plus a few sensible overrides:

```
ORIG_URL="<the URL the user passed to /teamwork-task>"
TEST_ARGS="$ORIG_URL --time-log=true --language=${DEFAULT_LANG:-sk}"
```

We pass `--time-log=true` explicitly so QA work continues the same sequential, non-overlapping time cursor that the implementation phase just advanced — both skills resolve the cursor via the **last Teamwork timelog of the day**, so the test skill's first log automatically starts where the implementation phase's last log ended (no gaps, no overlaps, no manual handover).

### 8.2.5 — Hand the build-time quality notes to QA (v1.5.0)

`/teamwork-task-test` reviews the same five keys (`ui_ux`, `performance`,
`security`, `reachability`, `framework`) in its Step 6.6 — `framework` there
is **advisory**: the tester only recommends, never edits code, never fails or
downgrades an acceptance criterion and never blocks on it (only a feature
newer than the installed version, the PHP floor or the browserslist target —
code that will not run — is a real finding, filed under the other keys).
Immediately **before** the `Skill` call, render the Step 6.5.5 results so they
sit in the context the QA pass runs in — open items first, one line each with
its location, then the `framework` rows marked as advisory:

```
Build-time quality notes for /teamwork-task-test (its Step 6.6 reviews the same keys):
  [#123456] reachability — open: ExportLog is URL-only by design (e-mail deep link) → allow_orphans candidate @ app/Nova/ExportLog.php
  [#123456] performance  — open: invoice export walks all rows in PHP; fine at 2 000, a full scan at 100 000 @ app/Actions/ExportInvoices.php:41
  [#123457] ui_ux        — not_applicable (no UI files in diff)
  [#123456] framework    — advisory, checked: Laravel 12.53: casts() + enum cast for ExportLog::status — docs eloquent-mutators#enum-casting
  [#123456] framework    — advisory, suggest (not changed): number_format(…) . ' €' → Number::currency($total, 'EUR', 'sk') @ app/Support/InvoiceFormatter.php:31
  Files: /tmp/tw_job_<ENTITY_ID>/build_quality.tsv, /tmp/tw_job_<ENTITY_ID>/framework_versions.tsv
```

The `framework` lines tell QA which versions and idioms the build already
settled on, so its recommendations do not re-propose them; an installed
`teamwork-task-test` older than 1.2.0 does not know the key and simply reads
the lines as context.

Do **not** narrow the test skill's `--dimensions` to what was checked here and
do **not** add flags it does not know — QA runs its own configured set; this
block and the TSV are the whole handoff. A task whose rows say
`skipped(--dimensions)` is listed as such, so QA knows nobody looked.

### 8.3 — Invoke the test skill

Use the `Skill` tool:

```
Skill(skill: "teamwork-task-test", args: "<TEST_ARGS>")
```

The invocation is **synchronous** in the model's perspective — the test skill's full per-task report (its Step 7) and tasklist summary (its Step 8) are rendered inline before control returns. There is no need to re-print anything in this skill — the user sees the verification output as the closing section of the run.

### 8.4 — Failure handling

The verification skill can finish with any of these outcomes:

- **All ✅** — everything verified, nothing to do.
- **Some ❌** — one or more acceptance criteria failed under a real test. The implementation phase already committed; do **not** auto-revert. Surface the failures clearly (the test skill already does this) and let the user decide whether to write a fix-up commit before pushing.
- **Some 📋 / 📝** — manual scenarios were written or criteria were proposed. No action needed — the user reads them and runs the manual checks themselves.

If the `Skill` tool call itself errors (transient MCP issue, skill not loadable), report it on one line and continue to Step 9 — the user has not lost anything; they can re-run `/teamwork-task-test <url>` manually.

---

## Step 9 — Final push reminder

After Step 7's summary and Step 8's optional verification handoff:

> **Push manually when ready:** `git push` (or `git push -u origin <branch>` for a new feature branch).
> Time logs were written to Teamwork for each task; cards were moved on the board per project configuration.
> If the verification step reported any ❌ failures, consider fixing those first before pushing.

When running in a worktree, the push reminder above is followed by the
worktree handoff in Step 9.5 — which may fully handle the merge / push for
the user.

---

## Step 9.5 — Worktree handoff (v1.3.0)

The classic failure mode this step closes: the skill ran in a worktree
(`.claude/worktrees/<name>`), committed N tasks to a worktree-local branch,
and ended cleanly. The commits **never reach `main`** unless the user
remembers to merge them by hand. With Claude Code launching background
runs in worktrees more aggressively, this loop has to close inside the
skill itself.

**Skip this step entirely when any of these hold:**
- `config.worktree_handoff.enabled == false`
- The user passed `--worktree-handoff=leave`
- `WT_RUN_IN_WORKTREE != 1` (from Step 5.1 — we are in the main checkout,
  nothing to merge)
- `config.worktree_handoff.skip_if_no_commits == true` AND
  `git rev-parse HEAD == WT_HEAD_BEFORE` (no new commits this run)
- The run was cancelled at plan approval (Step 4) — nothing to wrap up
- Every implemented task ended in aborted / skipped / `TIMELOG_OK=0` (no
  commits actually landed)

### Step 9.5.1 — Compute new commits

```bash
WT_NEW_HEAD=$(git rev-parse HEAD 2>/dev/null || echo "")
WT_NEW_COMMITS=""
WT_NEW_COUNT=0

if [ -n "$WT_HEAD_BEFORE" ] && [ -n "$WT_NEW_HEAD" ] && [ "$WT_HEAD_BEFORE" != "$WT_NEW_HEAD" ]; then
  WT_NEW_COMMITS=$(git log --oneline "${WT_HEAD_BEFORE}..${WT_NEW_HEAD}" 2>/dev/null)
  # `echo "" | wc -l` returns 1 on macOS/Linux (the trailing newline counts).
  # Guard the empty case explicitly so WT_NEW_COUNT is genuinely 0 when
  # `git log` produced no rows.
  if [ -z "$WT_NEW_COMMITS" ]; then
    WT_NEW_COUNT=0
  else
    WT_NEW_COUNT=$(printf '%s\n' "$WT_NEW_COMMITS" | grep -c .)
  fi
fi

# Bail if no new commits and skip_if_no_commits is on (the default).
# WT_HANDOFF_DONE acts as a short-circuit flag — every subsequent sub-step
# checks it at the top and returns early so we never render the action
# prompt for an empty commit set.
WT_HANDOFF_DONE=0
SKIP_IF_EMPTY=$(jq -r '.worktree_handoff.skip_if_no_commits | if . == null then true else . end' "$CONFIG_FILE")
if [ "$WT_NEW_COUNT" = "0" ] && [ "$SKIP_IF_EMPTY" = "true" ]; then
  echo "  ℹ worktree has no new commits since the run started — skipping handoff." >&2
  WT_HANDOFF_RESULT="skipped_no_commits"
  WT_HANDOFF_DONE=1
fi
```

### Step 9.5.2 — Decide the action

**Short-circuit:** Steps 9.5.2 through 9.5.7 all run inside an outer
`if [ "${WT_HANDOFF_DONE:-0}" != "1" ]; then … fi` block so a Step 9.5.1
"no new commits — skip" decision flows through to Step 10 without
rendering any prompt. The model treats `WT_HANDOFF_DONE=1` as the
single early-exit flag for the whole 9.5 section.


Read `worktree_handoff.default_action` (default `ask`). CLI flag
`--worktree-handoff=…` overrides config. Possible values:

- `ask` (default) → render an **AskUserQuestion** with the four options
  below.
- `merge` → skip the action question, go straight to target selection in
  Step 9.5.3.
- `push` → skip the action question, push the current branch to
  `worktree_handoff.push_remote` (default `origin`), do not merge.
- `leave` → no-op, fall through to Step 10.

When `default_action=ask`, the question is:

> "Run finished in worktree `<CURRENT_WT>` on branch `<CURRENT_BRANCH>`
>  with **<WT_NEW_COUNT>** new commits this session:
>
>  ```
>  <first 5 lines of WT_NEW_COMMITS, ellipsis if more>
>  ```
>
>  What do you want to do with them?"

Options (single-select):

1. **Merge into `<WT_PARENT_BRANCH>`** *(default when one is detected)* —
   fast-forward if possible, else create a merge commit, then delete the
   worktree's branch + the worktree itself.
2. **Merge into a different branch…** — opens a follow-up
   AskUserQuestion in Step 9.5.3 with `main`, `master`, `<WT_PARENT_BRANCH>`,
   *Other (free text)* as options.
3. **Push the branch to `<push_remote>` for a PR** — `git push -u
   <remote> <branch>`, leave both the branch and the worktree alone so
   the user can open a PR manually. Step 10 cleanup will still ask about
   the worktree afterwards.
4. **Leave as-is — I will handle the handoff manually.** — no-op.
5. **Cherry-pick specific commits into a target branch** — power option,
   triggers Step 9.5.5.

### Step 9.5.3 — Target selection (only for `merge` actions)

Read `worktree_handoff.default_target` (default `ask`).

- `ask` → **AskUserQuestion** with options:
  - *Parent branch (`<WT_PARENT_BRANCH>`)* — default-highlighted when
    detected.
  - *Default branch (`<DEFAULT_BRANCH>` from `origin/HEAD` / `main`)* —
    only if different from parent.
  - *Other branch* (free text) — user types a branch name; the skill
    validates it exists locally before continuing.
- `parent` → use `WT_PARENT_BRANCH` without asking.
- `main` → use the resolved default branch.
- `<any branch name>` → use that exact name (validate it exists; if not,
  fall back to `ask`).

Store the resolved target in `WT_MERGE_TARGET`.

### Step 9.5.4 — Execute the merge

```bash
STRATEGY=$(jq -r '.worktree_handoff.merge_strategy // "ff_else_merge"' "$CONFIG_FILE")
DELETE_BRANCH=$(jq -r '.worktree_handoff.delete_branch_after_merge | if . == null then true else . end' "$CONFIG_FILE")
DELETE_WT=$(jq -r     '.worktree_handoff.delete_worktree_after_merge | if . == null then true else . end' "$CONFIG_FILE")

# All git operations target the MAIN repo. We can run `git -C "$MAIN_REPO"`
# because the worktree branch is visible there too (worktrees share the
# object database).
TARGET_BRANCH="$WT_MERGE_TARGET"

# Refuse merge if the target's working tree (in the main checkout) is dirty —
# we cannot safely change the main checkout's HEAD without losing the user's
# uncommitted work. Surface and fall back to "push branch instead".
if [ -n "$(git -C "$MAIN_REPO" status --porcelain 2>/dev/null)" ]; then
  echo "  ⚠ main checkout has uncommitted changes — cannot merge safely. Falling back to push only." >&2
  WT_HANDOFF_RESULT="push_fallback_dirty_main"
  git -C "$CURRENT_WT" push -u "$(jq -r '.worktree_handoff.push_remote // "origin"' "$CONFIG_FILE")" "$CURRENT_BRANCH" 2>/dev/null
else
  # Switch the main repo to the target branch and merge.
  git -C "$MAIN_REPO" checkout "$TARGET_BRANCH" 2>/tmp/wth_err || {
    echo "  ⚠ failed to checkout '${TARGET_BRANCH}' in main repo: $(cat /tmp/wth_err)" >&2
    WT_HANDOFF_RESULT="merge_failed_checkout"
  }

  if [ "${WT_HANDOFF_RESULT:-}" != "merge_failed_checkout" ]; then
    # Capture each branch's exit code into MERGE_RC explicitly. We do NOT
    # test `$?` after `esac` because the exit code that bubbles up depends
    # on which branch ran AND whether that branch ended in an `if/fi`
    # block (whose `fi` reports the last inner command). Setting MERGE_RC
    # inside each arm keeps the contract local and unambiguous.
    MERGE_RC=1
    case "$STRATEGY" in
      ff_only)
        git -C "$MAIN_REPO" merge --ff-only "$CURRENT_BRANCH" 2>/tmp/wth_err
        MERGE_RC=$?
        ;;
      no_ff)
        git -C "$MAIN_REPO" merge --no-ff --no-edit "$CURRENT_BRANCH" 2>/tmp/wth_err
        MERGE_RC=$?
        ;;
      squash)
        if git -C "$MAIN_REPO" merge --squash "$CURRENT_BRANCH" 2>/tmp/wth_err; then
          git -C "$MAIN_REPO" commit --no-edit -m "Squashed merge of ${CURRENT_BRANCH} via /teamwork-task" 2>>/tmp/wth_err
          MERGE_RC=$?
        else
          MERGE_RC=$?
        fi
        ;;
      ff_else_merge|*)
        # Default — try fast-forward, fall back to a no-edit merge commit.
        if git -C "$MAIN_REPO" merge --ff-only "$CURRENT_BRANCH" 2>/dev/null; then
          MERGE_RC=0  # FF succeeded
        else
          git -C "$MAIN_REPO" merge --no-edit "$CURRENT_BRANCH" 2>/tmp/wth_err
          MERGE_RC=$?
        fi
        ;;
    esac

    if [ "$MERGE_RC" -eq 0 ]; then
      WT_HANDOFF_RESULT="merged"
      WT_HANDOFF_TARGET="$TARGET_BRANCH"

      # Delete the worktree branch + the worktree itself if configured.
      if [ "$DELETE_WT" = "true" ]; then
        git -C "$MAIN_REPO" worktree remove "$CURRENT_WT" 2>/tmp/wth_err \
          && WT_HANDOFF_DELETED_WORKTREE=1 \
          || echo "  ⚠ could not remove worktree '${CURRENT_WT}' (you may still be inside it): $(cat /tmp/wth_err)" >&2
      fi

      if [ "$DELETE_BRANCH" = "true" ]; then
        # Use -d so we refuse if the branch has unmerged commits (sanity belt).
        git -C "$MAIN_REPO" branch -d "$CURRENT_BRANCH" 2>/dev/null \
          && WT_HANDOFF_DELETED_BRANCH=1
      fi
    else
      WT_HANDOFF_RESULT="merge_failed"
      echo "  ⚠ merge failed: $(cat /tmp/wth_err)" >&2
      echo "  ℹ leaving the worktree branch in place so you can resolve manually." >&2
    fi
  fi
fi
```

If the merge fails (conflicts, non-FF when `ff_only`, etc.), **do not**
auto-resolve. Surface the error, leave the worktree intact, and let the
user finish manually. Step 10 will still offer to clean up other
worktrees but will skip this one.

### Step 9.5.5 — Cherry-pick path (power option)

When the user picks "Cherry-pick specific commits", render
**AskUserQuestion** with a multi-select listing every commit from
`WT_NEW_COMMITS` (short hash + first 60 chars of subject). Picked commits
are cherry-picked into the user-selected target branch in chronological
order:

```bash
# WT_PICKED_SHAS: one short hash per line, oldest first. `while read`, not
# an unquoted for-loop — zsh does not word-split an unquoted variable, so
# the 1.4.2 loop handed git ONE argument made of every hash.
git -C "$MAIN_REPO" checkout "$TARGET_BRANCH"
while IFS= read -r SHA; do
  [ -z "$SHA" ] && continue
  git -C "$MAIN_REPO" cherry-pick "$SHA" || {
    echo "  ⚠ cherry-pick of $SHA failed — pausing." >&2
    break
  }
done <<<"$WT_PICKED_SHAS"
```

Cherry-pick conflicts are non-recoverable inside the skill — surface the
state (`git status` output), abort, and let the user resolve.

### Step 9.5.6 — Push-only path

When the user picks "Push the branch", run:

```bash
REMOTE=$(jq -r '.worktree_handoff.push_remote // "origin"' "$CONFIG_FILE")
git -C "$CURRENT_WT" push -u "$REMOTE" "$CURRENT_BRANCH"
WT_HANDOFF_RESULT="pushed"
WT_HANDOFF_PUSH_REMOTE="$REMOTE"
```

Print the remote URL of the pushed branch so the user can open the PR
page in one click (GitHub / GitLab / Bitbucket all surface a "compare &
PR" link on the next visit). Do **not** delete the branch or the
worktree on this path — the user explicitly asked to keep them.

### Step 9.5.7 — Plumb result into Step 10 and Step 7

`WT_HANDOFF_RESULT` is consumed by:

- **Step 10 cleanup** — if `merged` and `WT_HANDOFF_DELETED_WORKTREE=1`,
  the current worktree is already gone; Step 10 still scans for the
  rest. If `pushed` or `leave`, Step 10 lists the current worktree under
  the *current — never auto-removed* category, same as today.
- **Step 7 final summary** — append a `Worktree handoff:` line with the
  resolved result (`merged into main (FF)`, `merged into feature/foo
  (merge commit, 3 commits)`, `pushed to origin`, `left as-is`,
  `merge_failed: <reason>`, `skipped_no_commits`). The user sees in one
  glance whether the run's commits are safely in their final home.

### Step 9.5.8 — Edge cases handled

- **No new commits this run** → skipped silently (with the default
  `skip_if_no_commits=true`).
- **Main checkout is dirty** → cannot safely change its HEAD, fall back
  to `push_fallback_dirty_main` (push the branch, do not merge). The
  user gets a clear warning.
- **`WT_PARENT_BRANCH` could not be detected** → the *Merge into parent*
  option is hidden; the user has to pick *Merge into a different branch…*
  with explicit input.
- **The current worktree's branch is also checked out somewhere else**
  (Claude Code multi-window) → `git branch -d` will refuse. The error is
  caught and surfaced; the worktree itself is still removed. The branch
  is left for the user to clean up.
- **CI / unattended run** → set `default_action=merge` and
  `default_target=parent` in config (no `AskUserQuestion` fires). Pair
  with `delete_branch_after_merge=true` and
  `delete_worktree_after_merge=true` for a fully hands-off pipeline.

---

## Step 10 — Worktree cleanup (end-of-run housekeeping)

After Step 7's summary, the optional Step 8 verification handoff, and the Step 9 push reminder, the skill scans the current repository for other git worktrees and offers to delete the ones that are stale. This catches the gradual accumulation of `.claude/worktrees/<name>` directories from prior background runs that nobody ever explicitly removed (worktrees are never auto-deleted by git, by Claude Code, or by this skill's earlier steps).

Skip this step when any of these hold:
- `config.worktree_cleanup.enabled == false`
- The user passed `--worktree-cleanup=false`
- The run was cancelled at plan approval (Step 4) — nothing to wrap up
- The repo has only the main worktree and no other entries (and `config.worktree_cleanup.report_when_empty == false`)
- Step 9.5 already merged + deleted the current worktree (`WT_HANDOFF_RESULT=merged` AND `WT_HANDOFF_DELETED_WORKTREE=1`) **and** there are no other worktrees in the repo — the only worktree we care about was just removed, no need to re-ask.

### Step 10.1 — Discover and classify worktrees

```bash
WTC_ENABLED=$(jq -r '.worktree_cleanup.enabled | if . == null then true else . end'                "$CONFIG_FILE")
WTC_AUTO=$(jq    -r '.worktree_cleanup.auto_remove_merged_clean // false' "$CONFIG_FILE")
WTC_STALE_DAYS=$(jq -r '.worktree_cleanup.stale_age_days // 14'        "$CONFIG_FILE")
WTC_REPORT_EMPTY=$(jq -r '.worktree_cleanup.report_when_empty // false' "$CONFIG_FILE")

if [ "$WTC_ENABLED" != "true" ] || [ "${WTC_CLI_OVERRIDE:-}" = "false" ]; then
  echo "  ℹ Worktree cleanup disabled — skipping Step 10." >&2
  WTC_RUN=0
else
  WTC_RUN=1
fi

# Resolve the main repo's working directory (handles being run from inside a worktree).
MAIN_REPO=$(git rev-parse --path-format=absolute --git-common-dir 2>/dev/null \
  | sed -E 's|/\.git$||; s|/\.git/worktrees/[^/]+$||')
CURRENT_WT=$(git rev-parse --show-toplevel 2>/dev/null)

# Best-effort default branch resolution (origin/HEAD when set, else "main", "master", "trunk").
DEFAULT_BRANCH=$(git -C "$MAIN_REPO" symbolic-ref --quiet --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||')
[ -z "$DEFAULT_BRANCH" ] && for B in main master trunk; do
  git -C "$MAIN_REPO" show-ref --verify --quiet "refs/heads/$B" && DEFAULT_BRANCH="$B" && break
done
[ -z "$DEFAULT_BRANCH" ] && DEFAULT_BRANCH=$(git -C "$MAIN_REPO" rev-parse --abbrev-ref HEAD 2>/dev/null)

# Bash 3.2-safe parsing of `git worktree list --porcelain` into TSV:
# path<TAB>branch<TAB>head<TAB>category<TAB>age_days<TAB>size_human
WT_TSV="/tmp/tw_worktrees_${$}.tsv"
: > "$WT_TSV"

git -C "$MAIN_REPO" worktree list --porcelain | awk -v RS= '
  {
    path = ""; branch = ""; head = "";
    n = split($0, lines, "\n");
    for (i = 1; i <= n; i++) {
      if (lines[i] ~ /^worktree /) { path   = substr(lines[i], 10) }
      if (lines[i] ~ /^HEAD /)     { head   = substr(lines[i], 6)  }
      if (lines[i] ~ /^branch /)   { branch = substr(lines[i], 8); sub(/^refs\/heads\//, "", branch) }
    }
    if (path != "") print path "\t" branch "\t" head
  }
' | while IFS=$'\t' read -r WT_PATH WT_BRANCH WT_HEAD; do
  CATEGORY=""
  if [ "$WT_PATH" = "$MAIN_REPO" ];   then CATEGORY="main";    fi
  if [ "$WT_PATH" = "$CURRENT_WT" ];  then CATEGORY="current"; fi
  if [ -z "$CATEGORY" ]; then
    if [ ! -d "$WT_PATH" ]; then
      CATEGORY="ghost"   # registered but directory was deleted manually
    elif [ -n "$(git -C "$WT_PATH" status --porcelain 2>/dev/null)" ]; then
      CATEGORY="dirty"
    elif [ -n "$WT_BRANCH" ] && git -C "$MAIN_REPO" merge-base --is-ancestor "$WT_BRANCH" "$DEFAULT_BRANCH" 2>/dev/null; then
      CATEGORY="merged-clean"
    else
      CATEGORY="unmerged-clean"
    fi
  fi

  if [ -d "$WT_PATH" ]; then
    LAST_TS=$(git -C "$WT_PATH" log -1 --format=%ct 2>/dev/null || echo 0)
    NOW_TS=$(date +%s)
    AGE_DAYS=$(( (NOW_TS - LAST_TS) / 86400 ))
    SIZE=$(du -sh "$WT_PATH" 2>/dev/null | awk '{print $1}')
  else
    AGE_DAYS=0
    SIZE="-"
  fi

  printf "%s\t%s\t%s\t%s\t%s\t%s\n" "$WT_PATH" "$WT_BRANCH" "$WT_HEAD" "$CATEGORY" "$AGE_DAYS" "$SIZE" >> "$WT_TSV"
done
```

### Step 10.2 — Decide what to offer

If `WT_HANDOFF_RESULT == "merged"` and `WT_HANDOFF_DELETED_WORKTREE == 1`,
the entry for `CURRENT_WT` is filtered out of `$WT_TSV` before
categorisation — git will still list it for a moment until the next
`git worktree list --porcelain` refresh, but we know it is already gone and
the user has been asked about it once already in Step 9.5. Avoid a
double-question.

Categorize entries from `$WT_TSV`:

| Category | Meaning | Default action |
|---|---|---|
| `main` | The primary checkout | Never touch. |
| `current` | The worktree this session is running in | Never touch (cannot `git worktree remove` ourselves). |
| `merged-clean` | Clean tree, branch is ancestor of default branch | **Safe to remove.** With `auto_remove_merged_clean=true`, removed without asking. |
| `unmerged-clean` | Clean tree but branch is NOT merged into default | Offer with explicit warning ("deleting branch loses unmerged commits"). |
| `dirty` | Working tree has uncommitted changes | Never auto-remove; list with a warning so the user is aware. |
| `ghost` | Registered with git but the directory was deleted manually | Offer to prune the bookkeeping (`git worktree prune` equivalent). |

If the only non-skip categories are empty (no candidates) and `report_when_empty == false`, end Step 10 silently. Otherwise render the **AskUserQuestion** prompt.

### Step 10.3 — Ask the user

Build a multi-select question listing each candidate with its category and size, plus convenience options:

> "Found {N} other git worktrees in this repo. Which (if any) should I remove?"
>
> Options (`multiSelect: true`):
> - `Remove all merged-clean ({K} worktrees, {SIZE} MB)` — bundle option, shows up only when ≥1 merged-clean exists
> - `{path}  (merged-clean, {age}d, {size})`
> - `{path}  (unmerged-clean — branch has {N} unique commits, {age}d, {size})`
> - `{path}  (dirty — has uncommitted changes, will NOT delete, just remind)`
> - `{path}  (ghost — only the bookkeeping entry remains)`
> - `Skip cleanup — keep all worktrees`

For `unmerged-clean` picks, follow up with a single-select confirmation: *"Branch X has unmerged commits — really delete?"* with options `Yes, force-delete (git branch -D)` / `No, keep this one`. Skip the confirmation when `auto_remove_merged_clean=true` is set AND the entry is merged.

### Step 10.4 — Execute

For each user-confirmed entry:

```bash
# Clean removal (refuses if dirty; the categorization already excluded dirty).
git -C "$MAIN_REPO" worktree remove "$WT_PATH" 2>/tmp/wtc_err \
  && echo "  ✓ removed worktree $WT_PATH" \
  || echo "  ⚠ git worktree remove failed: $(cat /tmp/wtc_err)" >&2

# Delete the branch. -d for merged, -D when user explicitly confirmed force-delete above.
if [ "$FORCE_BRANCH_DELETE" = "1" ]; then
  git -C "$MAIN_REPO" branch -D "$WT_BRANCH" 2>/dev/null \
    && echo "  ✓ force-deleted branch $WT_BRANCH" \
    || echo "  ⚠ could not force-delete branch $WT_BRANCH" >&2
else
  git -C "$MAIN_REPO" branch -d "$WT_BRANCH" 2>/dev/null \
    && echo "  ✓ deleted branch $WT_BRANCH" \
    || echo "  ⚠ could not delete branch $WT_BRANCH (may still be reachable)" >&2
fi
```

For `ghost` entries, run `git -C "$MAIN_REPO" worktree prune` once at the end.

### Step 10.5 — Report

Append to the Step 7 final summary block:

```
Worktree cleanup:
  Removed:    <list of paths> (<total MB> freed)
  Kept:       <list of paths with reasons — dirty / user declined>
  Skipped:    <list of paths with reasons — current / main>
```

Failure to remove a worktree (e.g. permission error, lockfile still held) is **NOT a blocker** — log a warning and continue. The user can always finish cleanup manually with `git worktree remove --force` later.

### Step 10.6 — Edge cases handled

- **We are inside a worktree** that's also in the discovery list → categorized as `current`, never touched.
- **Branch checked out in another worktree** has the `+` prefix in `git branch` output → `merge-base --is-ancestor` accepts the raw branch name (without `+`), so the prefix issue does not affect classification.
- **Repo has only the main worktree** → if `report_when_empty=false` (default) end silently; otherwise render "No other worktrees to clean up — nothing to do.".
- **`origin/HEAD` not set** → fall back to `main`, `master`, `trunk`, then HEAD's current branch — whichever exists first.
- **Plan was cancelled** (Step 4) → Step 10 does not run. The user may have intentionally left worktrees alone.

---

## Blocker handling — non-negotiable

If at any step you genuinely do not have enough information to proceed (ambiguous spec, missing file, environmental issue, conflicting requirements), **stop and ask via AskUserQuestion**. Do not fabricate decisions for business-shaped questions. Do not skip the task silently.

Board move failures, attachment download failures, and file-comments fetch failures are **NOT blockers** — they degrade silently with a warning, the task still runs end-to-end.

---

## Security

- The API token lives only in `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (chmod 600).
- Never `echo` or paste the token into commit messages, logs, or `git` outputs.
- Pass auth to `curl` via `-u "$TOKEN:xxx"`, never via URL query string.
- The plugin's `.gitignore` excludes `config.json` from the repo itself as a belt-and-braces safety net.
- Attachments downloaded into `./teamwork-task-<id>/` may contain confidential customer material — the auto-added `.gitignore` entry prevents accidental commits, and the cleanup in Step 6.9 removes them once the work is logged.
