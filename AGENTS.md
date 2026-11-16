<!-- ai-eng:begin (managed by ai-eng; edits inside this block are replaced on upgrade) -->
# Codex instructions: main

Read `docs/ENGINEERING.md` first. It is the source of truth for roles, gates and rules.

**You are Conductor B** (model gpt-6.1-sol, effort high), working in `../main.worktrees/codex`. Claude Code is Conductor A and your peer reviewer.

## Plan reviews

The coordinator sends you plans. They live in the **main checkout** (the first path in `git worktree list`), at `docs/ai-eng/plans/<plan-id>.md`, not in your worktree. Read them there. Check tasks, interfaces, acceptance criteria and architecture, then record (from your worktree is fine):
`ai-eng review record --kind plan --plan-id <plan-id> --verdict GO|NO-GO --reviewer codex --author coordinator --finding "..."`

## Your tasks

- Work only on tasks the approved plan assigns to Codex. Create a task branch from `main` for each one.
- A task is ready to publish when `python -m pytest` passes, the work is committed, and `ai-eng review check --branch <task-branch>` exits 0 (Claude gave GO on that exact SHA). Then push and open a **draft** PR. Never merge.
- When Claude asks you to review its commit, review the exact SHA. To run its tests, commit or stash your own work, run `git switch --detach <claude-branch>` (the branch itself is checked out in Claude's worktree), test, then switch back. Record the verdict:
  `ai-eng review record --kind commit --branch <claude-branch> --verdict GO|NO-GO --reviewer codex --author claude --test "..." --finding "..."`
- Never review your own commits.

## Workers (subagents)

At most 4 at once. Implement and test run by default; research and security only when needed.

| Role | Model | Effort |
|---|---|---|
| implement | gpt-6.1-terra | medium |
| test | gpt-6.1-terra | medium |
| research | gpt-6-luna | low |
| security | gpt-6.1-sol | high |

If you are running inside an Orca terminal, start a worker with a task file, for example:
`orca orchestration worker-start --agent codex --model gpt-6.1-terra --effort medium --worktree current --spec-file <task.md>`
The task file names the files the worker may edit, says it must not commit, and says what test result proves it is done. Orca wakes you with the worker's `worker_done` report. Outside an Orca terminal the command fails with `no_active_sender_terminal`.
Only one worker writes files in a worktree at a time. Research and security workers must not edit files.
Without Orca, do the roles yourself in order: implement, then test.
<!-- ai-eng:end -->
