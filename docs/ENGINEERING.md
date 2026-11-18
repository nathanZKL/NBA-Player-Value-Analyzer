# Engineering workflow

How Nathan and the AI agents (Claude Code: [CLAUDE.md](../CLAUDE.md); OpenAI Codex: [AGENTS.md](../AGENTS.md)) work on this repo together. The project's rules (purpose, data integrity, conventions, commands) live in CLAUDE.md and apply to every agent, including Codex. If two instructions ever seem to conflict, follow the stricter one and tell Nathan.

## Checks and task workspaces

Orca task workspaces (`~/orca/workspaces/NBA-Player-Value-Analyzer/<name>`) have no `.venv`; the workflow runs checks with the main checkout's:
`/Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/python -m pytest -q && /Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/ruff check . && /Users/atala/Desktop/Projects/NBA-Player-Value-Analyzer/.venv/bin/ruff format --check .`

The agent instruction and guardrail files (`CLAUDE.md`, `AGENTS.md`, `docs/ENGINEERING.md`, `docs/workflow/`, `.claude/settings.json`, `.claude/agents/`) are tracked, so every workspace gets them. Agents must not change them in a task commit; the coordinator checks every build for that.

`.claude/settings.json` allows `orca orchestration` and `orca skills get` without a prompt, so workflow agents can dispatch and report. Its deny rules block force pushes, hard resets, `git branch -D`, worktree removal, `gh pr merge` and reading `.env` files. Everything else is policy, not a sandbox.

## Dual-conductor workflow (Orca)

Feature work runs as one conversation: Nathan talks to the coordinator, and the coordinator drives everyone else through Orca orchestration (`orca orchestration worker-start`, `send`, `worker_done`). There is no other tool and no hand-off file; workers report to the coordinator, and Orca wakes it. Run the coordinator as a Claude Code agent in Orca, in the main checkout.

### Roles

| Role | Agent and model | Where it works |
|---|---|---|
| Nathan (owner) | Human | Gives the idea, edits and approves the plan, approves pushes, merges PRs. Final say on scope, security, spend and destructive actions. |
| Coordinator | Claude Code, opus | Main checkout. Plans and runs the loop; writes no product code. Steps: [workflow/coordinator.md](workflow/coordinator.md). |
| Plan reviewer | Codex, gpt-6.1-sol (high) | Main checkout, read-only. |
| Claude conductor | Claude Code, opus | Its own Orca workspace per task (`~/orca/workspaces/NBA-Player-Value-Analyzer/<plan-id>-<task>`). |
| Codex conductor | Codex, gpt-6.1-sol (high) | Its own Orca workspace per task. |
| Commit reviewer | The other conductor | The author's task workspace, read-only. |

### Loop

1. **Idea.** Nathan gives the coordinator a feature idea.
2. **Plan.** The coordinator writes `docs/workflow/plans/<plan-id>.md` (gitignored; plans stay local to the main checkout) from [workflow/plan-template.md](workflow/plan-template.md): tasks, one owner each (`claude` or `codex`), interfaces, acceptance criteria, dependencies, optional subagents.
3. **Nathan edits.** Nathan edits the plan and approves it.
4. **Plan review (automatic).** The coordinator starts Codex as the plan reviewer. On NO-GO it revises the plan and sends it back to the same reviewer, until GO. After 3 NO-GO rounds it stops and asks Nathan.
5. **Build (parallel).** The coordinator starts each task's conductor in a new Orca workspace on its own branch. Conductors use their subagents and commit when the tests pass.
6. **Commit review (automatic).** The other conductor reviews that exact SHA in the author's workspace. On NO-GO the coordinator sends the findings to the author, then the new SHA to the reviewer, until GO. After 3 NO-GO rounds it stops and asks Nathan.
7. **Publish.** On GO for the branch head, the coordinator asks Nathan to approve the push, pushes the branch, and opens a **draft** PR from [the PR template](../.github/pull_request_template.md).
8. **Merge.** Only Nathan merges.

### Subagents

Each conductor runs at most 4 at once. Implement and test run by default; research and security only when the plan asks or the conductor finds a concrete need (secrets, dependencies, untrusted input such as fetched data or salary CSVs).

| Role | Claude conductor (`.claude/agents/`) | Codex conductor (Orca workers in its workspace) |
|---|---|---|
| implement | `implementer` (sonnet) | gpt-6.1-terra (medium) |
| test | `tester` (sonnet) | gpt-6.1-terra (medium) |
| research | `researcher` (haiku), read-only | gpt-6-luna (low), read-only |
| security | `security-reviewer` (opus), read-only | gpt-6.1-sol (high), read-only |

Builds follow `superpowers:test-driven-development` and `superpowers:verification-before-completion`; the Claude `implementer` and `tester` load them at startup.

### Rules

- **No self-review, ever.** The conductor that wrote a commit never reviews it.
- **A GO covers one SHA.** Any new commit needs a new review before it is pushed.
- **Reviewers are read-only.** They never edit, stage, commit, stash or switch branches; the coordinator discards a review that changed the workspace.
- **Workers never push, open PRs or merge.** Conductors commit only on their own task branch.
- Task commits never touch the workflow and guardrail files (`CLAUDE.md`, `AGENTS.md`, `docs/ENGINEERING.md`, `docs/workflow/`, `.claude/`, `.github/workflows/`, `.env`). If a task changes a command CLAUDE.md lists, the conductor says so in its report and the coordinator asks Nathan to update CLAUDE.md separately.
- Everything CLAUDE.md says needs Nathan ("Working with Nathan", "Data-integrity rules") still needs Nathan, whichever agent asks.
- Never put secrets, tokens or credentials in code, plans, reviews or PRs.
