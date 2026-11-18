# AGENTS.md

NBA Player Value Analyzer: a local Streamlit app that finds the NBA players who best fit a role and who give the most of that role per salary dollar. What it is and how it fits together: [CLAUDE.md](CLAUDE.md), [docs/architecture.md](docs/architecture.md). Current state and next task: [docs/handoff.md](docs/handoff.md).

## Project rules

The project's rules live in [CLAUDE.md](CLAUDE.md), from "Project purpose" to "Environment notes". **They apply to every agent, including Codex.** Read them before starting. The most important ones:

- Never invent statistics or salaries, label every value, and keep mock data obviously fictional ("Data-integrity rules").
- Never commit real data (`data/raw/`, `data/processed/`, `data/salaries/`). Tests use mock data only and never call NBA.com or salary sites.
- Keep the MVP small: no features, dependencies or infrastructure beyond the current build step without Nathan's approval.
- Commit only on your own task branch, as the dual-conductor workflow below allows. Never push `main`, and never merge.

Checks: `pytest`, `ruff check .` and `ruff format --check .` (CLAUDE.md, "Commands"). Use uv with Python 3.13; the Mac's default `python3` is too old.

## Dual-conductor workflow (Orca)

Codex is a conductor and reviewer in this repo's dual-conductor workflow ([docs/ENGINEERING.md](docs/ENGINEERING.md#dual-conductor-workflow-orca)). A Claude coordinator in the main checkout starts you as an Orca worker; your prompt then has a Task/Dispatch preamble. Do only that task, in the workspace you were started in, and report with `worker_done` in the JSON shape the task asks for. If you cannot run `orca` for `worker_done`, end your turn with that JSON as your final reply; the coordinator reads it from your transcript.

- **Plan reviewer:** read the plan file the task names, check it against the plan section of [docs/workflow/review-template.md](docs/workflow/review-template.md), and do not edit any file.
- **Conductor:** build the task on your workspace's branch. For implement and test you may start Orca workers in your own workspace, one writer at a time (`orca orchestration worker-start --agent codex --model gpt-6.1-terra --effort medium --worktree current --spec-file <task.md>`; research on `gpt-6-luna` / `low` and security on `gpt-6.1-sol` / `high` only when needed, and they never edit files), or do the roles yourself in order: implement, then test. Commit when the tests pass; never push, open PRs or merge.
- **Commit reviewer:** review the exact SHA in the author's workspace against the commit section of the review checklist. Never edit, stage, commit, stash or switch branches, and never review your own commit.
