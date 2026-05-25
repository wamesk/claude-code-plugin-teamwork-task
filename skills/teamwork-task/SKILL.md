---
name: teamwork-task
description: "Use when the user provides a Teamwork.com URL (tasklist or task) and asks to 'work on these tasks', 'urob tasky z teamworku', 'spracuj tasky z teamwork', 'vypracuj tasky z teamworku', or invokes '/teamwork-task'. Fetches tasks via the Teamwork REST API (v3), implements them one by one in the current repository, commits per task using the TYPE(scope): Message convention, and logs time back to Teamwork. Configurable time tracking (real time rounded to 5 minutes, or ask after each task) and branching strategy (current branch or new feature branch). Pauses and asks the user via AskUserQuestion on blockers."
argument-hint: "<teamwork-url> [--time-mode=real_rounded_5m|ask] [--branching=current_branch|new_feature_branch]"
allowed-tools: [Bash, Read, Write, Edit, Grep, Glob, AskUserQuestion]
---

# Teamwork Task Worker

Fetch tasks from Teamwork.com (single task or whole tasklist), implement them one by one in the current repository, commit per task and log spent time back to Teamwork. The user pushes to remote manually.

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

6. **Apply CLI flag overrides** to in-memory config (`--time-mode`, `--branching`) — do not persist them.

7. **Never echo the API token** in shell output. When invoking `curl`, pass auth via `-u` to keep it out of `ps`.

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
# Tasklist metadata
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}.json"

# Tasks in the tasklist (paginated; loop pages if .meta.page.hasMore == true)
curl -sS -u "$AUTH" -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasklists/${ENTITY_ID}/tasks.json?pageSize=100&page=1"
```

For each task, extract via `jq`:
- `.task.id` (single response) or `.tasks[] | {id, name, description, estimateMinutes, priority, status, dueAt}` (list response). Field names follow v3 schema; if the response uses different keys (legacy v1 lived at `/tasks.json` with snake_case), inspect with `jq 'keys'` and adapt.

Filtering:
- If `config.skip_completed_tasks == true` → drop tasks whose status is `completed`.
- Sort tasks by priority then id ascending (stable processing order).

If the API returns HTTP 401 → token is invalid. Re-prompt the user for a new token (Step 2.5), save, retry. If 403/404 → report to the user and stop (cannot recover automatically).

Print a short table of fetched tasks (id, title, estimate) and ask the user via **AskUserQuestion** to confirm before starting the worker loop (single yes/no).

---

## Step 4 — Branching setup

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

---

## Step 5 — Worker loop (per task)

For each task in the filtered list, in order:

### 5.1 Start timer
```bash
START_TS=$(date +%s)
```

### 5.2 Plan implementation
Read the task title + description. If anything is genuinely ambiguous (missing acceptance criteria, conflicting requirements, business decision needed) → **AskUserQuestion** with a focused question. Wait for the answer before proceeding. Do **not** guess on business-shaped questions.

For purely technical decisions where there is a reasonable default consistent with the codebase, proceed without asking.

### 5.3 Implement
- Use `Read`, `Edit`, `Write`, `Bash`, `Grep`, `Glob` as needed.
- Respect any project conventions found in `CLAUDE.md` at the repo root.
- Write all code comments in **English**.

### 5.4 Test
If the project has a test suite and the change is testable:
- Laravel/Pest: `php artisan test --compact --filter=<RelevantTest>`
- Generic JS: `npm test -- --watchAll=false <pattern>` (or whatever the project uses)
- Run only what is relevant — do not run the whole suite per task.

### 5.5 Format
If PHP files changed: `vendor/bin/pint --dirty --format agent`.

### 5.6 Stop timer + decide minutes
```bash
END_TS=$(date +%s)
ELAPSED_MIN=$(( (END_TS - START_TS + 59) / 60 ))   # ceil to minutes
ROUND=$(jq -r '.time_rounding_minutes // 5' "$CONFIG_FILE")
DURATION_MIN=$(( ((ELAPSED_MIN + ROUND - 1) / ROUND) * ROUND ))   # round UP to nearest ROUND
# Minimum 5 minutes (or ROUND) so trivial tasks still log something
if [ "$DURATION_MIN" -lt "$ROUND" ]; then DURATION_MIN=$ROUND; fi
```

If `time_mode == "ask"` → **AskUserQuestion** with the measured `DURATION_MIN` as the suggested answer; let the user override.

### 5.7 Git commit
Use the convention from `~/.claude/CLAUDE.md`:
```
TYPE(scope): Message

Optional detailed description on next lines.
```
- `TYPE`: `CREATE`, `UPDATE`, `EDIT`, `FIX`, `REMOVE`, `MOVE`, `UPGRADE`, `DELETE` …
- `scope`: module / model / area touched (lowercase, e.g. `user`, `auth`, `invoice`)
- **Do NOT add `Co-Authored-By` lines.**

Stage only files that belong to this task (`git add <paths>`), then commit using a heredoc:
```bash
git commit -m "$(cat <<'EOF'
UPDATE(scope): Short summary

Optional detailed body.
EOF
)"
```

Capture the commit hash for the final summary: `COMMIT_HASH=$(git rev-parse --short HEAD)`.

### 5.8 Log time back to Teamwork
`POST ${BASE}/projects/api/v3/tasks/${TASK_ID}/time.json`

The Slovak description for the time log should be written in Slovak (per project default `default_language=sk`) and describe **what was actually done**, not just the task title.

```bash
TODAY=$(date +%Y-%m-%d)
NOW=$(date +%H:%M:%S)
BILLABLE=$(jq -r '.is_billable_by_default // true' "$CONFIG_FILE")

PAYLOAD=$(jq -nc \
  --argjson minutes "$DURATION_MIN" \
  --arg desc "$DESCRIPTION_SK" \
  --arg date "$TODAY" \
  --arg time "$NOW" \
  --argjson billable "$BILLABLE" \
  '{timelog: {hours: 0, minutes: $minutes, description: $desc, date: $date, time: $time, isbillable: $billable}}')

curl -sS -u "$AUTH" -H "Content-Type: application/json" -H "Accept: application/json" \
  -X POST -d "$PAYLOAD" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/time.json"
```

Verify HTTP status. On non-2xx → report to the user (do not retry blindly; the commit already exists).

### 5.9 Complete the task in Teamwork (optional)
If `config.auto_complete_finished_tasks == true`, or the user opts in at the first task of this session:
```bash
curl -sS -u "$AUTH" -X PUT -H "Accept: application/json" \
  "${BASE}/projects/api/v3/tasks/${TASK_ID}/complete.json"
```
On the first task of the session ask once via **AskUserQuestion**: *"Mark tasks as completed in Teamwork after each commit?"* — and remember the answer for the rest of this run.

### 5.10 Move to next task
Continue the loop.

---

## Step 6 — Final summary

After the loop ends, print a markdown table:

| # | Task ID | Title | Minutes | Commit | TW status |
|---|---------|-------|---------|--------|-----------|

Then a short reminder:
> **Push manually when ready:** `git push` (or `git push -u origin <branch>` for a new feature branch).
> Time logs were written to Teamwork for each task.

---

## Blocker handling — non-negotiable

If at any step you genuinely do not have enough information to proceed (ambiguous spec, missing file, environmental issue, conflicting requirements), **stop and ask via AskUserQuestion**. Do not fabricate decisions for business-shaped questions. Do not skip the task silently.

---

## Security

- The API token lives only in `~/.claude/plugins/data/teamwork-task-wamesk/config.json` (chmod 600).
- Never `echo` or paste the token into commit messages, logs, or `git` outputs.
- Pass auth to `curl` via `-u "$TOKEN:xxx"`, never via URL query string.
- The plugin's `.gitignore` excludes `config.json` from the repo itself as a belt-and-braces safety net.
