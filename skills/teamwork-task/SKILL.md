---
name: teamwork-task
version: 1.1.3
description: "Use when the user provides a Teamwork.com URL (tasklist or task) and asks to 'work on these tasks', 'urob tasky z teamworku', 'spracuj tasky z teamwork', 'vypracuj tasky z teamworku', or invokes '/teamwork-task'. Fetches tasks via the Teamwork REST API (v3), pulls task description, attachments, comments (when needed), and file comments for context, **scans the local working tree for unattached specs / samples / DNR docs that match the task keywords and asks the user whether to use them**, **detects gating phrases in the task body (e.g. 'Bez vzorky nemá zmysel písať regex') and pauses with a question before implementing instead of barreling through with synthetic data**, implements tasks one by one in the current repository, moves the task on the board (In progress → Internal testing, with fallback to Testing), commits per task using the TYPE(scope)[<task-id>]: Message convention, and logs time back to Teamwork as sequential, non-overlapping 5-min-aligned entries that pick up from your last timelog of the day. Configurable safety gate asks for review when the diff touches UI/template files or grows beyond 100 lines. When the companion `teamwork-task-test` skill is installed, hands off to it at the very end so each task's acceptance criteria get individually verified before the user pushes. Pauses and asks the user via AskUserQuestion on blockers."
argument-hint: "<teamwork-url> [--time-mode=real_rounded_5m|ask] [--branching=current_branch|new_feature_branch] [--plan-mode=overview|per_task|none] [--auto-commit=always|when_safe|never] [--test-after=true|false]"
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
- `--test-after=true|false` — hand off to `/teamwork-task-test` after a clean finish (default `true`)

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
ENTITY_ID=$(echo "$URL" | sed -nE 's|.*/(tasks|tasklists)/([0-9]+).*|\2|p')
URL_KIND=$(echo "$URL" | sed -nE 's|.*/(tasks|tasklists)/[0-9]+.*|\1|p' | sed 's/s$//')
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

6. **Apply CLI flag overrides** to in-memory config (`--time-mode`, `--branching`, `--plan-mode`, `--auto-commit`) — do not persist them.

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
  (.fetch_comments_mode //= "when_needed") |
  (.fetch_attachments //= true) |
  (.fetch_file_comments //= true) |
  (.max_attachment_size_mb //= 25) |
  (.attachments_cleanup //= "after_timelog") |
  (.auto_commit_mode //= "when_safe") |
  (.auto_commit_risky_patterns //= ["\\.(vue|jsx|tsx|svelte|blade\\.php|css|scss|sass|less|html)$"]) |
  (.auto_commit_max_diff_lines //= 100) |
  (.auto_propose_tests //= true) |
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
  (.include_commit_hash_in_log_description //= true) |
  (.time_mode //= "real_rounded_5m") |
  (.time_rounding_minutes //= 5) |
  (.branching_mode //= "current_branch") |
  (.default_language //= "sk") |
  (.auto_complete_finished_tasks //= false) |
  (.skip_completed_tasks //= true) |
  (.is_billable_by_default //= true) |
  (.board_workflow //= {
    "enabled": true,
    "in_progress_stage": "In progress",
    "done_stage": "Internal testing",
    "done_stage_fallbacks": ["Testing"],
    "match_mode": "case_insensitive"
  }) |
  (.auto_run_tests_after //= true) |
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
  })
' "$CONFIG_FILE" > "$CONFIG_FILE.tmp" && mv "$CONFIG_FILE.tmp" "$CONFIG_FILE"
chmod 600 "$CONFIG_FILE"
```

The migration is idempotent — running it twice produces the same file. Do not print the config to stdout; only mention "config migrated to 1.1.3 schema" once if any change was made.

The `auto_run_tests_after` key (added in 1.1.1) controls whether this skill, on a clean finish, hands off to `/teamwork-task-test` to verify the acceptance criteria of every implemented task. Default is `true`. Disable per run with `--test-after=false`.

The `local_context_discovery` key (added in 1.1.3) controls Step 3.10 — scanning the working tree for unattached specs / samples that the user dropped into the project folder but did not attach to the Teamwork task. Default is `true`. Disable per run with `--local-discovery=false`.

The `readiness_gate` key (added in 1.1.3) controls Step 6.0 — detecting phrases in the task description that mark an external blocker (e.g. *"Bez vzorky nemá zmysel písať regex"*) and pausing with **AskUserQuestion** before implementing, so the skill does not silently produce code against synthetic placeholders. Default is `true`. Disable per run with `--readiness-gate=false`.

---

## Step 3 — Fetch tasks via Teamwork REST API v3

Authentication: HTTP Basic, username = API token, password = any string (Teamwork convention: use `xxx`).

```bash
TOKEN=$(jq -r '.teamwork.api_token' "$CONFIG_FILE")
BASE=$(jq -r '.teamwork.base_url' "$CONFIG_FILE")
AUTH="${TOKEN}:xxx"
```

Endpoints (Teamwork API v3 — see `https://apidocs.teamwork.com/docs/teamwork/v3/`):

**Single task** (when `URL_KIND=task`):
```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${ENTITY_ID}.json"
```

**Tasklist** (when `URL_KIND=tasklist`):
```bash
# Tasklist metadata (description lives here too — captured in Step 3.4)
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"

# Tasks in the tasklist (paginated; loop pages if .meta.page.hasMore == true)
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=1"
```

For each task, extract via `jq`:
- `.task.id` (single response) or `.tasks[] | {id, name, description, projectId, estimateMinutes, priority, status, dueAt, commentsCount}` (list response). Field names follow v3 schema; if the response uses different keys (legacy v1 lived at `/tasks.json` with snake_case), inspect with `jq 'keys'` and adapt.
- **`projectId`** is required for board workflow resolution (Step 3.3).
- **`commentsCount`** is used by the `when_needed` comments heuristic (Step 3.5).

Filtering:
- If `config.skip_completed_tasks == true` → drop tasks whose status is `completed`.
- Sort tasks by priority then id ascending (stable processing order).

If the API returns HTTP 401 → token is invalid. Re-prompt the user for a new token (re-run the first-run flow in Step 2 / Step 2.6 to overwrite `.teamwork.api_token`), save, retry. If 403/404 → report to the user and stop (cannot recover automatically).

### Step 3.3 — Resolve board workflow stages

If `config.board_workflow.enabled == true`, fetch the workflow + stages for each unique `projectId` in the task list (one call per project, cache results):

Bash 4+ associative arrays are not available on macOS's default `/bin/bash` 3.2, so this skill performs name→id lookup via a temp lookup function over a `name<TAB>id` text table. The table is built once per project from the API response and reused for both the start and done stage resolution.

```bash
IN_PROGRESS_NAME=$(jq -r '.board_workflow.in_progress_stage' "$CONFIG_FILE")
DONE_NAME=$(jq -r       '.board_workflow.done_stage'         "$CONFIG_FILE")

# Build FALLBACKS array (bash 3.2-safe: no `mapfile`)
FALLBACKS=()
while IFS= read -r FB; do
  [ -n "$FB" ] && FALLBACKS+=("$FB")
done < <(jq -r '.board_workflow.done_stage_fallbacks[]?' "$CONFIG_FILE")

# Per project (cache by $PROJECT_ID):
WF_RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/workflows.json?projectIds=${PROJECT_ID}&include=stages")

WORKFLOW_ID=$(echo "$WF_RESP" | jq -r '.workflows[0].id // empty')

# Build per-project stage lookup file: lowercase_name<TAB>id
# Teamwork v3 returns `.included.stages` as an OBJECT keyed by string ID; the
# stage's own `.value.id` may or may not duplicate the key, so we prefer `.key`
# as the canonical ID. We also tolerate an array shape just in case.
STAGE_TABLE_FILE="/tmp/tw_stages_${PROJECT_ID}.tsv"
echo "$WF_RESP" | jq -r '
  if (.included.stages | type) == "object" then
    .included.stages | to_entries[] | [(.value.name // ""), (.key // (.value.id|tostring))]
  elif (.included.stages | type) == "array" then
    .included.stages[]                  | [(.name // ""), (.id|tostring)]
  else empty end
  | "\(.[0] | ascii_downcase)\t\(.[1])"
' > "$STAGE_TABLE_FILE"

# Lookup helper (bash 3.2 compatible)
lookup_stage_id() {
  local needle
  needle=$(echo "$1" | tr '[:upper:]' '[:lower:]')
  awk -F '\t' -v n="$needle" '$1 == n { print $2; exit }' "$STAGE_TABLE_FILE"
}

# Resolve target stages (case-insensitive via lookup_stage_id)
IN_PROGRESS_STAGE_ID=$(lookup_stage_id "$IN_PROGRESS_NAME")

DONE_STAGE_ID=$(lookup_stage_id "$DONE_NAME")
DONE_RESOLVED_NAME=""
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
```

Outcomes (per project):
- **Both stages found** → board moves enabled, store `WORKFLOW_ID`, `IN_PROGRESS_STAGE_ID`, `DONE_STAGE_ID`.
- **Only `IN_PROGRESS_STAGE_ID` found** → start move enabled, done move skipped with a warning in the plan ("Project #X has no 'Internal testing' or 'Testing' column — task will stay in 'In progress' after completion.").
- **Only `DONE_STAGE_ID` found** → start move skipped, done move enabled.
- **Neither / no workflow** → set `BOARD_MOVE_DISABLED[$projectId]=1`, plan warning *"Project #X has no workflow — board moves disabled."*

Silent degradation: never block the run, never `AskUserQuestion` here. Report state in the plan and the final summary.

### Step 3.4 — Tasklist description (when `URL_KIND=tasklist`)

Pull the tasklist's own description and surface it at the top of the plan:

```bash
TL_DESC_RAW=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json" \
  | jq -r '.tasklist.description // empty')

# Strip HTML tags down to plain text (best-effort sed; sufficient for plan rendering)
TASKLIST_DESCRIPTION=$(echo "$TL_DESC_RAW" | sed -E 's|<[^>]+>||g' | sed -E 's/[[:space:]]+/ /g; s/^ +//; s/ +$//')
```

If `TASKLIST_DESCRIPTION` is non-empty, render it in Step 4 under a `## Tasklist context` heading.

### Step 3.5 — Fetch comments per task (conditional)

Read `fetch_comments_mode` from config. Behavior:

- **`always`** → fetch comments for every task (legacy v1.0 behavior).
- **`never`** → skip entirely.
- **`when_needed`** (default) → fetch only if **both** of these hold:
  1. `task.commentsCount > 0` (cheap short-circuit; no comments to read anyway).
  2. At least one of:
     - `final_summary` is empty (no HR found in description — see Step 3.6), OR
     - `len(strip_html(acceptance_criteria)) < 100`, OR
     - The description matches `(viď|see|viz)[[:space:]]+(komentár|comment|comments|nižšie|below)` (case-insensitive).

When fetching:
```bash
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/comments.json?pageSize=100&page=1&orderBy=postedAt&orderMode=asc"
```

Paginate while `.meta.page.hasMore == true`. Strip HTML tags from `htmlBody` to a plain-text body for context (keep author + posted timestamp). Keep comments in chronological order — the **last comment is the freshest truth** when it conflicts with earlier ones.

For each fetched comment, retain the list of `comment.files` / `comment.attachments` for Step 3.8.

### Step 3.6 — Parse task description (acceptance criteria + final summary)

Project convention: the task description is split by a horizontal rule (`<hr>`, `<hr/>`, `<hr />` in HTML, or a Markdown `---` / `***` / `___` on its own line). The content **above** the HR is the **acceptance criteria** (what must be true to consider the task done); the content **below** the HR is the **final summary / decision** — treat this as the authoritative goal.

For each task, split the description on the **first** HR occurrence:
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

If `config.fetch_attachments == true`, fetch the file list for each task and download into a local working folder:

```bash
ATTACH_DIR="./teamwork-task-${TASK_ID}"
rm -rf "$ATTACH_DIR"          # clean any leftover from a prior failed run
mkdir -p "$ATTACH_DIR"

MAX_MB=$(jq -r '.max_attachment_size_mb // 25' "$CONFIG_FILE")
MAX_BYTES=$(( MAX_MB * 1024 * 1024 ))

# Primary endpoint
FILES_JSON=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/files.json?pageSize=100&page=1")

# Fallback if 404 / shape unexpected
if ! echo "$FILES_JSON" | jq -e '.files // .data // empty' >/dev/null 2>&1; then
  FILES_JSON=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
    "${BASE}/projects/api/v3/files.json?taskIds=${TASK_ID}&pageSize=100&page=1")
fi

# For each file: skip oversized ones; download by ID-prefixed filename to avoid collisions.
# Use process substitution so SKIPPED_ATTACHMENTS+=() mutations survive the loop
# (a piped `while read` would run in a subshell and discard them).
while IFS= read -r FILE; do
  [ -z "$FILE" ] && continue
  FID=$(echo "$FILE"  | jq -r '.id')
  FNM=$(echo "$FILE"  | jq -r '.name // .displayName // ("file_" + (.id|tostring))')
  FSZ=$(echo "$FILE"  | jq -r '.size // 0')
  URL=$(echo "$FILE"  | jq -r '.downloadUrl // .downloadURL // empty')

  if [ "${FSZ:-0}" -gt "$MAX_BYTES" ]; then
    echo "  ⚠ skipped attachment '${FNM}' (${FSZ} bytes > ${MAX_BYTES})" >&2
    SKIPPED_ATTACHMENTS+=("${FNM} (${FSZ} bytes)")
    continue
  fi
  if [ -z "$URL" ]; then
    # Some Teamwork responses expose a per-file download endpoint instead
    URL="${BASE}/projects/api/v3/files/${FID}/download"
  fi
  curl -sS -L -u "$AUTH" -o "${ATTACH_DIR}/${FID}_${FNM}" "$URL" || \
    echo "  ⚠ failed to download attachment '${FNM}'" >&2
done < <(echo "$FILES_JSON" | jq -c '.files[]? // .data[]? // empty')
```

If neither endpoint shape returns a file list, log a single line warning and continue without attachments.

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

For each comment fetched in Step 3.5, iterate its `files` / `attachments` array and download to a `comments/` subdirectory inside `$ATTACH_DIR`:

```bash
mkdir -p "${ATTACH_DIR}/comments"

# Same subshell trick as Step 3.7 — use process substitution so any accounting
# variables (e.g. SKIPPED_ATTACHMENTS) updated inside the loop survive afterwards.
while IFS= read -r CMT; do
  [ -z "$CMT" ] && continue
  CID=$(echo "$CMT" | jq -r '.id')
  while IFS= read -r CF; do
    [ -z "$CF" ] && continue
    FID=$(echo "$CF" | jq -r '.id')
    FNM=$(echo "$CF" | jq -r '.name // .displayName // ("file_" + (.id|tostring))')
    FSZ=$(echo "$CF" | jq -r '.size // 0')
    URL=$(echo "$CF" | jq -r '.downloadUrl // .downloadURL // empty')

    if [ "${FSZ:-0}" -gt "$MAX_BYTES" ]; then
      echo "  ⚠ skipped comment attachment '${FNM}' (${FSZ} bytes > ${MAX_BYTES})" >&2
      SKIPPED_ATTACHMENTS+=("${FNM} (${FSZ} bytes, comment ${CID})")
      continue
    fi
    [ -z "$URL" ] && URL="${BASE}/projects/api/v3/files/${FID}/download"
    curl -sS -L -u "$AUTH" -o "${ATTACH_DIR}/comments/${CID}_${FID}_${FNM}" "$URL" \
      || echo "  ⚠ failed to download comment attachment '${FNM}'" >&2
  done < <(echo "$CMT" | jq -c '.files[]? // .attachments[]? // empty')
done < <(echo "$COMMENTS_JSON" | jq -c '.comments[]? // empty')
```

Same size limit (`max_attachment_size_mb`) applies if `size` is present on the comment file entry.

### Step 3.9 — Fetch file comments

If `config.fetch_file_comments == true`, for each file already collected in Step 3.7 fetch its comments and surface a short digest in the per-task plan context:

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
LCD_ENABLED=$(jq -r '.local_context_discovery.enabled // true' "$CONFIG_FILE")

# Honour the fall-through from Step 3.7 even when explicitly disabled.
if [ "$LCD_ENABLED" != "true" ] && [ "${FILENAME_HINT_PRESENT_ANY:-0}" != "1" ]; then
  echo "  ℹ Local context discovery disabled — skipping working-tree scan." >&2
else
  MAX_DEPTH=$(jq -r '.local_context_discovery.max_depth // 3' "$CONFIG_FILE")
  MIN_SCORE=$(jq -r '.local_context_discovery.min_keyword_score // 1' "$CONFIG_FILE")
  MAX_OFFER=$(jq -r '.local_context_discovery.max_files_to_offer // 12' "$CONFIG_FILE")

  # Keyword set: tasklist name + every task name + every filename hint from descriptions.
  KEYWORDS=$(jq -r '
    [.tasklist.name, (.tasks[]?.name), (.tasks[]?.description // "")]
    | join(" ")
  ' "$CLAUDE_JOB_DIR/tasks.json" 2>/dev/null \
    | tr '[:upper:]' '[:lower:]' \
    | tr -dc '[:alnum:]áäčďéíľĺňóôŕšťúýž _\n' \
    | tr ' ' '\n' \
    | awk 'length($0) >= 4' \
    | sort -u)

  # Build the find expression from configured extensions + filename hints.
  EXTS=$(jq -r '.local_context_discovery.extensions[]' "$CONFIG_FILE")
  HINTS=$(jq -r '.local_context_discovery.filename_hints[]' "$CONFIG_FILE")

  FIND_EXTS=""
  for E in $EXTS; do FIND_EXTS="$FIND_EXTS -iname '*.${E}' -o"; done
  for H in $HINTS; do FIND_EXTS="$FIND_EXTS -iname '*${H}*' -o"; done
  FIND_EXTS="${FIND_EXTS% -o}"

  IGNORE_EXPR=""
  while IFS= read -r G; do
    [ -n "$G" ] && IGNORE_EXPR="$IGNORE_EXPR -not -path './${G}'"
  done < <(jq -r '.local_context_discovery.ignore_globs[]' "$CONFIG_FILE")

  # Collect candidates (limit depth, filter out junk paths).
  CANDIDATES=$(eval "find . -maxdepth $MAX_DEPTH -type f $IGNORE_EXPR \\( $FIND_EXTS \\) -print 2>/dev/null" \
    | sort -u)

  # Score each candidate by keyword overlap (filename + parent dir tokens ∩ KEYWORDS).
  SCORED_FILE="$CLAUDE_JOB_DIR/local_discovery.tsv"
  : > "$SCORED_FILE"
  while IFS= read -r PATHCAND; do
    [ -z "$PATHCAND" ] && continue
    TOKENS=$(echo "$PATHCAND" \
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

For `overview`, render a concise markdown plan to stdout:

```
## Tasklist context (if URL_KIND=tasklist and TASKLIST_DESCRIPTION non-empty)
<TASKLIST_DESCRIPTION>

## Plan for tasklist "<name>" (<N> tasks)

### 1. [#<task-id>] <title>  (est: <X> min, priority: <p>)
**Goal (final summary):** <one-line summary from description below HR>
**Acceptance:** <bullet list condensed from description above HR>
**Approach:** <1-3 sentences — what files / modules will likely change, what tests, what risks>
**Comments context:** <skipped (description sufficient) | 3 comments — user clarified to use X over Y on 2026-05-20>
**Attachments:** <none | 2 files: spec.md (3KB), mockup.png (180KB)>
**File comments:** <skipped | 1 on mockup.png — "use #1A73E8 instead">
**Context files (local):** <none | Strečnianska/DNR_Strecnianska_v1.2.docx, Strečnianska/Bmail o pohybe na ucte - vzor.docx>  ← from Step 3.10
**Missing inputs:** <none | "Real Tatra banka notification sample" (BLOCKER — gating phrase in description) | "Final colour value" (SOFT — referenced in comment)>  ← from Step 3.10 (gap) + Step 6.0 patterns
**Board target:** <In progress → Internal testing | In progress → Testing (fallback) | start only — no testing column | disabled — no workflow on this project>

### 2. [#<task-id>] ...
```

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

- **Approve and start** — proceed to Step 5.
- **Skip some tasks** — user lists which task IDs to drop, then re-render the plan and re-ask.
- **Reorder** — user provides new order, re-render and re-ask.
- **Add context to a task** — user picks a task and pastes extra context; append it to that task's working notes, re-render the plan, re-ask.
- **Cancel** — abort the run, no commits, no time logs, no board moves.

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

### Step 5.5 — Initialize the session time cursor

Time logs in Teamwork **must not overlap** even if implementation work overlaps in real time. The plugin maintains a **session cursor** that advances strictly forward as each task is logged, so the resulting time entries are contiguous (no gaps, no overlaps) and every entry's start time is aligned to the rounding step (default 5 minutes).

The cursor's starting position depends on `time_cursor_strategy`:

- **`floor_now`** — start at `floor(now, ROUND)`. Classic v1.0 behavior.
- **`last_teamwork_timelog`** (default) — start at the end of today's most recent timelog by the current user, so the run *picks up where you left off*. Falls back to `floor(now)` when there is no log today.

Initialize the cursor **once**, right before the worker loop starts (after plan approval — plan time is not billed):

```bash
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
parse_iso() {
  local raw norm out
  raw="$1"
  # Strip any millisecond fraction; normalise `+00:00` (and the rare `-00:00`) to `Z`.
  norm=$(printf '%s' "$raw" \
    | sed -E 's/\.[0-9]+(Z|[+-][0-9:]{2,5})?$/\1/' \
    | sed -E 's/[+-]00:?00$/Z/')

  # macOS BSD date — UTC `Z` form
  out=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$norm" +%s 2>/dev/null) && \
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
    # 1. Resolve current user via V1 /me.json (V3 has no me endpoint)
    USER_ID=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
      "${BASE}/me.json" | jq -r '.person.id // empty')

    if [ -n "$USER_ID" ]; then
      # 2. Get most recent timelog of TODAY for that user
      RESP=$(curl -sS -u "$AUTH" -H "Accept: application/json" \
        "${BASE}/projects/api/v3/time.json?assignedToUserIds=${USER_ID}&pageSize=1&orderBy=date&orderMode=desc&startDate=${TODAY}")

      LAST_LOGGED=$(echo "$RESP" | jq -r '.timelogs[0].timeLogged // empty')
      LAST_MIN=$(echo    "$RESP" | jq -r '.timelogs[0].minutes    // 0')

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

### Step 6.0 — Readiness gate (per task)

**Run this BEFORE Step 6.1 starts the timer.** v1.1.3 catches the case where a
task body explicitly says it needs something that may not be in hand — the
canonical example is *"Pred začatím vyžiadať reálny vzor notifikácie od
klienta. Bez vzorky nemá zmysel písať regex."* The previous behaviour was to
plow through with a synthetic placeholder; the new behaviour is to **ask**.

Skip this step when `config.readiness_gate.enabled == false` or the user passed
`--readiness-gate=false`.

```bash
RG_ENABLED=$(jq -r '.readiness_gate.enabled // true' "$CONFIG_FILE")
[ "$RG_ENABLED" != "true" ] && { echo "  ℹ readiness gate disabled — skipping"; }

if [ "$RG_ENABLED" = "true" ]; then
  # Build one big haystack: task description + acceptance_criteria + final_summary
  # + every comment body, lowercased and HTML-stripped.
  HAYSTACK=$(printf '%s\n%s\n%s\n' \
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
    if MATCH=$(echo "$HAYSTACK" | grep -oE "$PAT" | head -n1); then
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
    OFFERED=$(awk -F '\t' '{print }' "$CLAUDE_JOB_DIR/local_discovery.tsv.top" 2>/dev/null)
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

If `plan_mode == per_task`, do **not** start the timer here yet — first render a per-task plan (same shape as the overview entry: Goal / Acceptance / Approach / Comments context / Attachments / Board target) and ask the user via **AskUserQuestion** to **Approve / Skip / Add context / Cancel run**. Only **after** approval start the timer.

### 6.1.5 Move task to "In progress" on the board

Immediately after the timer starts (and after per-task approval, if applicable), nudge the card to the in-progress column:

```bash
BW_ENABLED=$(jq -r '.board_workflow.enabled // true' "$CONFIG_FILE")
if [ "$BW_ENABLED" = "true" ] \
   && [ -z "${BOARD_MOVE_DISABLED[$PROJECT_ID]}" ] \
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

The `final_summary` (below HR) is the authoritative goal; use `acceptance_criteria` (above HR) as the checklist to verify before committing. The **last comment** in chronological order is the freshest source of truth when comments contradict each other.

If anything is genuinely ambiguous (missing acceptance criteria, conflicting requirements with comments, business decision needed) → **AskUserQuestion** with a focused question. Wait for the answer before proceeding. Do **not** guess on business-shaped questions.

For purely technical decisions where there is a reasonable default consistent with the codebase, proceed without asking.

### 6.2.5 Test strategy — propose tests when none are specified

If `config.auto_propose_tests == true` (default) and the task does **not**
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
| Laravel without Dusk     | Propose Dusk install, otherwise skip browser tests (see prompt) | Pest if `HAS_PEST`, else PHPUnit                                          |
| PHP without Laravel      | Selenium standalone PHPUnit test, otherwise skip                | PHPUnit or Pest by signal                                                 |
| JS with Playwright       | Playwright (`tests/e2e/…`)                                      | Vitest if `HAS_VITEST`, Jest if `HAS_JEST`                                |
| JS with Cypress          | Cypress (`cypress/e2e/…`)                                       | Vitest or Jest                                                            |
| JS without any browser   | Propose Playwright install, otherwise skip                      | Vitest or Jest by signal                                                  |
| Nothing detected         | Plan a manual checklist instead                                 | Plan a manual checklist instead                                           |

`config.test_frameworks.*_preference` lets the user pin a choice (`pest`,
`phpunit`, `dusk`, `playwright`, `cypress`, `vitest`, `jest`,
`selenium`); the default `"auto"` follows the table.

**Missing tool — ask, don't auto-install**

If the chosen framework is **not present** in the project (e.g. visual change
in Laravel but no Dusk), use **AskUserQuestion** with three options:

- *Install `<package>` now* — run the install command and continue
  (`composer require --dev laravel/dusk && php artisan dusk:install` /
  `npm install -D @playwright/test && npx playwright install --with-deps`).
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
- **Implement the auto-proposed tests from Step 6.2.5 in the same task**, so
  the commit + time log cover both the feature and its verification. Place
  files where the framework conventionally lives (`tests/Feature/…`,
  `tests/Unit/…`, `tests/Browser/…` for Laravel; `tests/e2e/…` or
  `cypress/e2e/…` for JS).

### 6.4 Test
If the project has a test suite and the change is testable:
- Laravel/Pest: `php artisan test --compact --filter=<RelevantTest>`
- Generic JS: `npm test -- --watchAll=false <pattern>` (or whatever the project uses)
- Run only what is relevant — do not run the whole suite per task.

### 6.5 Format
If PHP files changed: `vendor/bin/pint --dirty --format agent`.

### 6.6 Stop timer + decide minutes
```bash
END_TS=$(date +%s)
ELAPSED_MIN=$(( (END_TS - START_TS + 59) / 60 ))   # ceil to minutes
ROUND=$(jq -r '.time_rounding_minutes // 5' "$CONFIG_FILE")
ROUND_SECS=$(( ROUND * 60 ))
DURATION_MIN=$(( ((ELAPSED_MIN + ROUND - 1) / ROUND) * ROUND ))   # round UP to nearest ROUND
# Minimum 5 minutes (or ROUND) so trivial tasks still log something
if [ "$DURATION_MIN" -lt "$ROUND" ]; then DURATION_MIN=$ROUND; fi
```

If `time_mode == "ask"` → **AskUserQuestion** with the measured `DURATION_MIN` as the suggested answer; let the user override.

### 6.6.1 Future-timestamp guard (hard rule)

The session cursor advances by *logged* minutes, not by *real* minutes — so on a fast run where the model produces work much faster than wall-clock time would allow, the cursor drifts into the future. Teamwork accepts those POSTs without complaint (timesheets can legally hold future entries), but the resulting timesheet is useless for billing and confuses anyone reading it.

Cap `DURATION_MIN` so the resulting log window ends at or before `now()`:

```bash
NOW=$(date +%s)
# How much real-time space is still ahead of the cursor, floored to a ROUND multiple.
HEADROOM_SECS=$(( NOW - SESSION_CURSOR_TS ))
if [ "$HEADROOM_SECS" -lt 0 ]; then HEADROOM_SECS=0; fi
HEADROOM_MIN=$(( (HEADROOM_SECS / 60 / ROUND) * ROUND ))

TIMELOG_SKIPPED=0
if [ "$HEADROOM_MIN" -lt "$ROUND" ]; then
  echo "  ⚠ session cursor at $(date -r "$SESSION_CURSOR_TS" +%H:%M) is at/past now ($(date -r "$NOW" +%H:%M)) — skipping timelog for task ${TASK_ID}" >&2
  TIMELOG_SKIPPED=1
  SKIPPED_TIMELOGS+=("${TASK_ID} (cursor caught up with real time)")
elif [ "$DURATION_MIN" -gt "$HEADROOM_MIN" ]; then
  echo "  ⚠ measured ${DURATION_MIN}m would overshoot now() — clamping to ${HEADROOM_MIN}m so the timelog stays in the past" >&2
  CLAMPED_TIMELOGS+=("${TASK_ID}: ${DURATION_MIN}m → ${HEADROOM_MIN}m")
  DURATION_MIN=$HEADROOM_MIN
fi
```

Why this matters: without the guard, a fast run will silently produce timesheet entries with start/end times that have not happened yet. The PM reads "16:00 — refactor done" at 13:30 and rightly asks how that is possible.

`TIMELOG_SKIPPED=1` plumbs through Step 6.8 — the POST is skipped, the cursor is **not** advanced, and Step 6.8.5 (board move to *Internal testing*) is **also** skipped because the task is not yet considered finished from a billing standpoint. Step 7 surfaces `SKIPPED_TIMELOGS` and `CLAMPED_TIMELOGS` so the user can decide whether to fill in the missing time manually.

Initialize the accounting arrays once at the start of the worker loop:

```bash
SKIPPED_TIMELOGS=()
CLAMPED_TIMELOGS=()
```

### 6.6.5 Safety gate — auto-commit or ask?

Before staging and committing, inspect the diff to decide whether the change is trivial enough for an unattended commit or risky enough to deserve a human pass. This step runs **before** `git add` in Step 6.7, so the measurement uses `git diff HEAD` (working tree + index, against the last commit) — that way both freshly modified files and anything that was already staged earlier in the run are counted:

```bash
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
    if [ -n "$PATTERNS" ] && echo "$CHANGED" | grep -qiE "$PATTERNS"; then
      MATCHED=$(echo "$CHANGED" | grep -iE "$PATTERNS" | head -3 | tr '\n' ' ')
      REASONS+=("UI/template/styling files changed: ${MATCHED}")
    fi

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

Stage only files that belong to this task (`git add <paths>`), then commit by **piping the body to `git commit -F -`** instead of using an unquoted heredoc — the Teamwork task description / final_summary can legitimately contain `$(...)`, backticks, or `${VAR}` which would otherwise be expanded (or executed) by the shell:

```bash
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

BILLABLE=$(jq -r '.is_billable_by_default // true' "$CONFIG_FILE")
INCL_HASH=$(jq -r '.include_commit_hash_in_log_description // true' "$CONFIG_FILE")

# Use the session cursor — NOT wall-clock time
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

Verify HTTP status. On non-2xx → report to the user (do not retry blindly; the commit already exists). Do **not** advance `SESSION_CURSOR_TS` if the POST failed — the next successful log should reuse the same start time so the user's timesheet stays contiguous. Also **skip the next "move to Internal testing"** step, because the task is not yet considered finished from a billing standpoint.

### 6.8.5 Move task to "Internal testing" (or fallback) on the board

If the time log succeeded and the project has a resolved done stage, post the card to the done column:

```bash
if [ "${TIMELOG_OK:-0}" = "1" ] \
   && [ "$BW_ENABLED" = "true" ] \
   && [ -z "${BOARD_MOVE_DISABLED[$PROJECT_ID]}" ] \
   && [ -n "$WORKFLOW_ID" ] && [ -n "$DONE_STAGE_ID" ]; then
  HTTP=$(curl -sS -o /dev/null -w "%{http_code}" -u "$AUTH" \
    -H "Content-Type: application/json" -H "Accept: application/json" \
    -X POST -d "{\"taskIds\":[${TASK_ID}]}" \
    "${BASE}/projects/api/v3/workflows/${WORKFLOW_ID}/stages/${DONE_STAGE_ID}/tasks.json")
  if [ "$HTTP" -ge 200 ] && [ "$HTTP" -lt 300 ]; then
    CURRENT_BOARD_STAGE_FOR_TASK="${DONE_RESOLVED_NAME:-Internal testing}"
  else
    echo "  ⚠ board move to '${DONE_RESOLVED_NAME:-Internal testing}' failed (HTTP $HTTP) — continuing" >&2
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

| # | Task ID | Title | Minutes | Commit | TW status | Board stage |
|---|---------|-------|---------|--------|-----------|-------------|

Followed by a short status block:

```
Time cursor: <TIME_CURSOR_SOURCE>           e.g. "last_timelog @ 10:50"
                                            or  "skill start (first log of day)"
Timelogs skipped (cursor caught up with now): <list from SKIPPED_TIMELOGS or none>
Timelogs clamped to fit before now:           <list from CLAMPED_TIMELOGS or none>
Attachments skipped (size):                   <list or none>
Board moves disabled for projects:            <list or none>
```

When `SKIPPED_TIMELOGS` is non-empty, also print a short paragraph telling the user **why** those entries did not land in Teamwork (the cursor reached `now()` mid-run, typically because the model produces work faster than wall-clock) and suggest they either re-run later (the cursor will continue from the last successful log) or fill those minutes in manually. The skipped tasks' board cards were **not** moved to *Internal testing* either — they will be moved on the next run that successfully logs them.

The push reminder is intentionally **not** printed here — it moves to Step 9, after the optional verification handoff, so the user sees test results before they decide whether to push.

---

## Step 8 — Auto-run `/teamwork-task-test` (optional handoff)

If `config.auto_run_tests_after == true` (default, added in 1.1.1) and the user did not pass `--test-after=false`, hand off to the `/teamwork-task-test` skill for per-criterion verification.

**Why this exists.** The implementation phase produces a passing build and a commit per task, but it does not — by itself — prove that each *acceptance criterion* of the original Teamwork task was actually met. The companion `teamwork-task-test` skill exists precisely for that: it parses each acceptance criterion out of the task description, maps it onto the project's existing tests, runs them, drives a browser via the chrome-devtools MCP for UI-shaped criteria, and writes manual scenarios for whatever cannot be automated. Calling it automatically at the end closes the implement → verify loop in a single command.

### 8.1 — Detect whether the test skill is available

The `teamwork-task-test` skill is published on the same WAME marketplace. Two signals tell us if it is currently installed in this session:

1. **The Skill tool's available-skills list** — when invoked via the `Skill` tool, only listed skill names succeed. The model can see this list in the session's system-reminder messages. If `teamwork-task-test` appears there, it is installed.
2. **Filesystem fallback** — check for an installed skill folder under the plugins cache:
   ```bash
   if ls "$HOME/.claude/plugins"/*/teamwork-task-test/SKILL.md >/dev/null 2>&1 \
      || ls "$HOME/.claude/plugins"/*/skills/teamwork-task-test/SKILL.md >/dev/null 2>&1; then
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
