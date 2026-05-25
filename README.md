# teamwork-task

A Claude Code plugin that fetches Teamwork.com tasks (single task or whole tasklist) by URL, implements each task in the current repository, commits per task using the `TYPE(scope): Message` convention, and logs spent time back to Teamwork. Push to remote is intentionally left to the user.

Part of the [`wame`](https://github.com/wamesk/claude-code) Claude Code plugin marketplace.

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

The config lives **outside** the plugin cache, so reinstalling or updating the plugin will **not** wipe your API token.

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
```

Optional flags (override the saved config **for this run only**, not persisted):

- `--plan-mode=overview` — generate a single tasklist-wide plan and ask for approval before any work (default).
- `--plan-mode=per_task` — render and approve a plan **before each individual task**.
- `--plan-mode=none` — skip planning approval entirely.
- `--time-mode=real_rounded_5m` — measure actual elapsed time and round up to the nearest 5 minutes (default).
- `--time-mode=ask` — ask after each task how many minutes to log; the measured value is suggested as the default.
- `--branching=current_branch` — commit on the current branch (default).
- `--branching=new_feature_branch` — create `feature/teamwork-tasklist-<id>` (or `feature/teamwork-task-<id>`) and commit there.

## What the plugin does, step by step

1. **Parses** the Teamwork URL → workspace, kind (task/tasklist), entity ID.
2. **Loads or creates** the persistent config; prompts for credentials on first run.
3. **Fetches** the task(s) via Teamwork REST API v3, including:
   - all **comments** per task (for decision history and clarifications) — controlled by `fetch_comments`,
   - the description **split on the first horizontal rule** into *acceptance criteria* (above HR) and *final summary* (below HR — treated as the authoritative goal).
4. **Renders a plan** (per `plan_mode`):
   - `overview` (default) — single tasklist-wide plan with goal / acceptance / approach / comments digest per task → one approval.
   - `per_task` — same content but rendered and approved one task at a time inside the loop.
   - `none` — no plan approval, jump straight to work.
   You can **Approve / Skip tasks / Reorder / Add context / Cancel** before the work starts. Plan time is **not** logged to Teamwork.
5. For each task (in the approved order):
   - Starts a timer.
   - Plans the implementation internally using the final summary + acceptance criteria + comments. If a business decision or missing context is required, it **stops and asks** via `AskUserQuestion`.
   - Implements the change in the current repo.
   - Runs relevant tests, if any.
   - Runs Pint formatting for PHP changes.
   - Stops the timer, rounds up to the configured minute step (default 5).
   - Commits using the `TYPE(scope)[<task-id>]: Message` convention (no `Co-Authored-By` lines).
   - Logs time back to Teamwork via the API with a Slovak description of what was actually done.
   - Optionally marks the task as completed in Teamwork.
6. **Prints a summary table** (task ID, title, minutes, commit hash, TW status) and reminds you to `git push` manually.

## Configuration reference

File: `~/.claude/plugins/data/teamwork-task-wamesk/config.json`

| Key                              | Default                                | Notes                                                                                       |
| -------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------- |
| `teamwork.base_url`              | `https://<workspace>.teamwork.com`     | Your Teamwork workspace URL. Asked on first run.                                            |
| `teamwork.api_token`             | (empty)                                | Personal API key. Asked on first run. Stored at chmod 0600.                                 |
| `plan_mode`                      | `overview`                             | `overview` (one approval for the whole tasklist), `per_task` (approval before each task), or `none`. |
| `fetch_comments`                 | `true`                                 | If `true`, fetches all comments for each task to enrich context for planning + implementation. |
| `time_mode`                      | `real_rounded_5m`                      | `real_rounded_5m` measures actual time; `ask` prompts after every task.                     |
| `time_rounding_minutes`          | `5`                                    | Rounding step (minutes). Applies to `real_rounded_5m`.                                      |
| `branching_mode`                 | `current_branch`                       | `current_branch` or `new_feature_branch`.                                                   |
| `default_language`               | `sk`                                   | Language for the Teamwork time-log description (Slovak by default).                         |
| `auto_complete_finished_tasks`   | `false`                                | If `true`, the plugin marks each task as completed in TW after the commit + time log.        |
| `skip_completed_tasks`           | `true`                                 | If `true`, tasks already marked as completed in TW are skipped when iterating a tasklist.   |
| `is_billable_by_default`         | `true`                                 | Sets `isbillable` on every logged time entry.                                               |

To change a setting, edit the file directly and rerun the command.

## Plan modes

- **`overview` (default)** — after fetching tasks and comments, the plugin prints a single markdown plan covering every task in the run (goal, acceptance criteria, planned approach, comments digest) and asks for one approval. You can **Approve / Skip tasks / Reorder / Add context / Cancel** before any code is touched.
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

All **comments** on the task are fetched too (when `fetch_comments=true`) and surfaced in the plan as a short digest so prior decisions and clarifications carry into the work.

## Time tracking modes

- **`real_rounded_5m` (default)** — the plugin measures wall-clock time between starting the task and the commit, then rounds **up** to the nearest `time_rounding_minutes`. Best for billable work where you trust the measurement.
- **`ask`** — after each commit, the plugin asks you how many minutes to log, with the measured value prefilled. Use this when you frequently take long breaks, multitask, or want explicit control.

## Branching modes

- **`current_branch` (default)** — every commit lands on whatever branch is currently checked out. The plugin verifies the working tree is clean before starting.
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

## Security

- The Teamwork API token is stored in plaintext at `~/.claude/plugins/data/teamwork-task-wamesk/config.json` with file permissions `0600`.
- The token is never echoed to stdout, never written to commit messages, and never passed to `curl` via the URL — only via `-u "$TOKEN:xxx"`.
- The plugin's own `.gitignore` excludes `config.json` from the repo itself as a safety net.

If you suspect your token has leaked, revoke it in Teamwork (`/launchpad/apikey/manage`) and rerun the plugin — it will re-prompt.

## Roadmap — out of scope for v1.0.0

- Download task attachments and feed them into the implementation context.
- Branch per task (currently: tasklist-level branch or current branch).
- Automatic `git push` after the run.
- Open a Pull Request per task or per tasklist.
- Multi-repo orchestration (when a tasklist spans multiple repositories).

## License

MIT — see [`LICENSE`](LICENSE).
