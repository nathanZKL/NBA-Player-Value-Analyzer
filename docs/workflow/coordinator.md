# Coordinator playbook

You are the **coordinator** when you are Claude Code in the main checkout, you were not started as an Orca worker (your prompt has no Task/Dispatch preamble), and Nathan gives you a feature idea or asks you to use the dual-conductor workflow. You plan and you run the loop through Orca orchestration. You never write product code yourself.

Everything below runs through Orca: workers are started with `orca orchestration worker-start`, you talk to them with `orca orchestration send`, and Orca wakes you with their reports. Before the first Orca command in a session, load the version-matched guide once: `orca skills get orchestration`. Never poll, never run watch scripts, never steer a worker through its terminal.

## Who does what

| Role | Agent | `--model` / `--effort` | Workspace |
|---|---|---|---|
| Coordinator (you) | Claude Code | opus | main checkout |
| Plan reviewer | Codex | `gpt-6.1-sol` / `high` | `--worktree current` (read-only) |
| Claude conductor | Claude Code | `opus` / `high` | `--worktree new-child --name <plan-id>-<task>` |
| Codex conductor | Codex | `gpt-6.1-sol` / `high` | `--worktree new-child --name <plan-id>-<task>` |
| Commit reviewer | the *other* conductor's agent and model | as above | the author's task workspace (read-only) |

A `new-child` workspace is created under `~/orca/workspaces/NBA-Player-Value-Analyzer/<name>` on a new branch (Orca names it, for example `nathanZKL/<name>`) from the current `main`, or from `--base-branch`. It has no `.venv`.

Conductor subagents: Claude uses its `.claude/agents/` subagents (`implementer`, `tester` by default; `researcher`, `security-reviewer` on demand). Codex starts Orca workers (implement and test on `gpt-6.1-terra` / `medium`, research on `gpt-6-luna` / `low`, security on `gpt-6.1-sol` / `high`) or does the roles itself in order.

Test command, run from any workspace (it uses the main checkout's `.venv`):
`/Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/python -m pytest -q && /Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/ruff check . && /Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/ruff format --check .`

## 1. Plan

1. Ask only the questions you truly need answered. Read the code you need; do not change it.
2. Write `docs/workflow/plans/<plan-id>.md` from [plan-template.md](plan-template.md). Short kebab-case id.
3. Split the work into small tasks with exactly one owner each: `claude` (architecture, frontend, integration, UX by default) or `codex` (backend, data, reliability, tests by default). Two tasks never edit the same files unless one depends on the other.
4. Each task gets its files, the interfaces it provides or uses, test-checkable acceptance criteria, its dependencies, and any on-demand subagents (research, security) with the reason.
5. Give Nathan a short summary and stop. Wait until Nathan says the plan is approved (for example "send it").

## 2. Plan review loop (automatic)

1. Start the plan reviewer with a spec file (`--spec-file`, so quotes need no escaping):
   `orca orchestration worker-start --agent codex --model gpt-6.1-sol --effort high --worktree current --task-title "<plan-id>: plan review" --spec-file <spec>`
   The spec: the plan's absolute path; review it against the plan section of [review-template.md](review-template.md); do not edit, create or delete any file; report with `worker_done` whose body is only `{"verdict": "GO" | "NO-GO", "findings": ["<problem> -> <expected fix>", ...]}`.
2. End your turn. Orca wakes you with the report.
3. **GO:** set the plan's status to GO and go to step 3.
4. **NO-GO:** revise the plan to address every finding. If you disagree with one, keep your version and explain why in the plan's "Review notes". Then send the same reviewer the next round, so it keeps its context:
   `orca orchestration send --to dispatch:<dispatch-id> --subject "Plan revised: review again" --body "<what changed, finding by finding>. Review the same file again and report the same way."`
   Do not ask Nathan between rounds.
5. After **3 NO-GO rounds**, stop and give Nathan the open findings.

## 3. Build

For every task whose dependencies are done, start its conductor. Start all independent tasks together, then end your turn.

`orca orchestration worker-start --agent <claude|codex> --model <opus|gpt-6.1-sol> --effort high --worktree new-child --name <plan-id>-<task> --task-title "<plan-id> <task>: build" --spec-file <spec>`

A task that depends on another adds `--base-branch <that task's branch>`.

The build spec contains:

- The task section of the plan, word for word, and the plan's absolute path for context.
- "Work on this workspace's current branch. Change only the files the task lists."
- The subagents to use (Claude: `implementer` and `tester`, plus any the task names; Codex: Orca workers with `--worktree current`, one writer at a time, or the roles yourself in order).
- Skills to follow: `superpowers:test-driven-development` and `superpowers:verification-before-completion`.
- The test command above. "Commit on this branch when the acceptance criteria are met and the tests pass. Do not push, open a PR or merge."
- "Do not change `CLAUDE.md`, `AGENTS.md`, `docs/ENGINEERING.md`, `docs/workflow/`, `.claude/`, `.github/workflows/` or `.env`."
- Report with `worker_done`, body only `{"branch": "...", "sha": "<HEAD>", "workspace": "<absolute path>", "tests": "<command>: <result>", "summary": "..."}`.

When a build report arrives, confirm it: `git -C <workspace> rev-parse HEAD` equals `sha`, `git -C <workspace> status --porcelain` is empty, and `git -C <workspace> diff --name-only <base>...<sha>` touches none of the files above. If not, `send` the conductor what is wrong.

## 4. Commit review loop (automatic)

1. Start the reviewer: the other agent, in the author's workspace:
   `orca orchestration worker-start --agent <other agent> --model <its conductor model> --effort high --worktree path:<workspace> --task-title "<plan-id> <task>: review <short sha>" --spec-file <spec>`
   The spec: review commit `<sha>` on `<branch>` (`git diff <base>...<sha>`) against the task's acceptance criteria and the commit section of [review-template.md](review-template.md); run the test command; do not edit, stage, commit, stash or switch branches; report with `worker_done`, body only `{"verdict": "GO" | "NO-GO", "sha": "<sha>", "findings": ["<file>:<line> <problem> -> <expected fix>", ...], "tests": "<command>: <result>"}`.
2. End your turn. On the report, confirm the workspace is still clean and still at `sha`; if the reviewer changed anything, discard that review and start it again.
3. **NO-GO:** send the findings to the **author's** dispatch: "Fix these findings, run the tests, commit, and report the same way." On its new report, send the **reviewer's** dispatch: "The author committed `<new sha>` for your findings. Review it the same way." Repeat.
4. After **3 NO-GO rounds** on one task, stop and give Nathan the open findings.
5. **GO** counts only when the verdict's `sha` equals the branch's current head.

## 5. Publish

On GO, from the main checkout:

1. Ask Nathan to approve the push of `<branch>` at `<sha>`, and wait for a yes. An approval for one branch does not cover another. Then `git push -u origin <branch>`.
2. `gh pr create --draft --base <main, or the dependency's branch> --head <branch> --title "<task title> (<plan-id> <task>)" --body-file <file>`, with the body filled from `.github/pull_request_template.md`: plan path, task, author, tests, reviewer, verdict and reviewed SHA.

Then start any tasks that were waiting on this one (step 3).

## 6. Report

When every task is published or escalated, tell Nathan per task: PR link, reviewed SHA, tests, and anything still open. Nathan merges. Leave the task workspaces; Nathan removes them after merging.

## When Orca wakes you with something else

- **`question`:** answer it with `orca orchestration reply --id <message-id> --body "..."` when the plan or Nathan's instructions answer it; otherwise ask Nathan.
- **`escalation` with `stopReason` `finished`** (a worker ended its turn without `worker_done`): read its last reply with `orca orchestration worker-read --dispatch <id> --source transcript`. If that reply is the report, use it; otherwise `send` it a reminder to finish and report.
- **`lost` or `exited`:** start it again once with `--retry-of <dispatch-id>` and the same placement and agent; if it fails again, tell Nathan.

## Never

- Write product code, commit to `main`, push `main`, force-push, merge, or delete branches or workspaces.
- Let a conductor review its own commit, or count a GO for any SHA other than the branch head.
- Fetch from NBA.com or salary sites, commit real data, or add features, dependencies or infrastructure beyond the plan without Nathan's approval ([CLAUDE.md](../../CLAUDE.md), "Working with Nathan" and "Data-integrity rules").
