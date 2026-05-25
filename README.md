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

- `--time-mode=real_rounded_5m` — measure actual elapsed time and round up to the nearest 5 minutes (default).
- `--time-mode=ask` — ask after each task how many minutes to log; the measured value is suggested as the default.
- `--branching=current_branch` — commit on the current branch (default).
- `--branching=new_feature_branch` — create `feature/teamwork-tasklist-<id>` (or `feature/teamwork-task-<id>`) and commit there.

## What the plugin does, step by step

1. **Parses** the Teamwork URL → workspace, kind (task/tasklist), entity ID.
2. **Loads or creates** the persistent config; prompts for credentials on first run.
3. **Fetches** the task(s) via Teamwork REST API v3.
4. **Confirms** the task list with you before any work starts.
5. For each task:
   - Starts a timer.
   - Plans the implementation. If a business decision or missing context is required, it **stops and asks** via `AskUserQuestion`.
   - Implements the change in the current repo.
   - Runs relevant tests, if any.
   - Runs Pint formatting for PHP changes.
   - Stops the timer, rounds up to the configured minute step (default 5).
   - Commits using the `TYPE(scope): Message` convention (no `Co-Authored-By` lines).
   - Logs time back to Teamwork via the API with a Slovak description of what was actually done.
   - Optionally marks the task as completed in Teamwork.
6. **Prints a summary table** (task ID, title, minutes, commit hash, TW status) and reminds you to `git push` manually.

## Configuration reference

File: `~/.claude/plugins/data/teamwork-task-wamesk/config.json`

| Key                              | Default                                | Notes                                                                                       |
| -------------------------------- | -------------------------------------- | ------------------------------------------------------------------------------------------- |
| `teamwork.base_url`              | `https://<workspace>.teamwork.com`     | Your Teamwork workspace URL. Asked on first run.                                            |
| `teamwork.api_token`             | (empty)                                | Personal API key. Asked on first run. Stored at chmod 0600.                                 |
| `time_mode`                      | `real_rounded_5m`                      | `real_rounded_5m` measures actual time; `ask` prompts after every task.                     |
| `time_rounding_minutes`          | `5`                                    | Rounding step (minutes). Applies to `real_rounded_5m`.                                      |
| `branching_mode`                 | `current_branch`                       | `current_branch` or `new_feature_branch`.                                                   |
| `default_language`               | `sk`                                   | Language for the Teamwork time-log description (Slovak by default).                         |
| `auto_complete_finished_tasks`   | `false`                                | If `true`, the plugin marks each task as completed in TW after the commit + time log.        |
| `skip_completed_tasks`           | `true`                                 | If `true`, tasks already marked as completed in TW are skipped when iterating a tasklist.   |
| `is_billable_by_default`         | `true`                                 | Sets `isbillable` on every logged time entry.                                               |

To change a setting, edit the file directly and rerun the command.

## Time tracking modes

- **`real_rounded_5m` (default)** — the plugin measures wall-clock time between starting the task and the commit, then rounds **up** to the nearest `time_rounding_minutes`. Best for billable work where you trust the measurement.
- **`ask`** — after each commit, the plugin asks you how many minutes to log, with the measured value prefilled. Use this when you frequently take long breaks, multitask, or want explicit control.

## Branching modes

- **`current_branch` (default)** — every commit lands on whatever branch is currently checked out. The plugin verifies the working tree is clean before starting.
- **`new_feature_branch`** — the plugin creates (or switches to) `feature/teamwork-tasklist-<tasklistId>` for tasklist runs, or `feature/teamwork-task-<taskId>` for single-task runs, and commits there.

## Commit format

Per the conventions in your `~/.claude/CLAUDE.md`:

```
TYPE(scope): Short summary

Optional detailed description on next lines.
```

- `TYPE` ∈ `CREATE`, `UPDATE`, `EDIT`, `FIX`, `REMOVE`, `MOVE`, `UPGRADE`, `DELETE`, …
- `scope` is the module / model / area touched (lowercase, e.g. `user`, `auth`, `invoice`).
- The plugin **never** adds `Co-Authored-By` lines.

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
