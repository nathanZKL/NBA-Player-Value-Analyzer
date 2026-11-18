# Review checklist

Reviewers report their verdict to the coordinator with Orca's `worker_done`, in the JSON shape the task asks for. A GO applies to one plan version (plan reviews) or one exact SHA (commit reviews). Reviewers never edit, stage, commit, stash or switch branches.

## Plan review (Codex reviews the coordinator's plan)

- Every task has one owner, listed files, interfaces and checkable acceptance criteria.
- Tasks do not edit the same files, or the order between them is explicit.
- The architecture fits the existing code and [docs/architecture.md](../architecture.md), and the scope stays inside [docs/mvp-plan.md](../mvp-plan.md).
- Risky choices, security needs and research needs are called out.
- Anything that needs Nathan's approval (features, dependencies or infrastructure beyond the current build step; real data fetches) is marked so.

## Commit review (the other conductor reviews the author's commit)

- You are not the author. If you are, stop and report that the gate is blocked.
- Review the exact SHA: `git show <sha>` and `git diff <base>...<sha>`.
- The task's acceptance criteria are met.
- Run the tests yourself with the test command the task gives, and report the command and result.
- The data-integrity rules in [CLAUDE.md](../../CLAUDE.md) hold: no invented values, every value labelled, mock data obviously fictional, no real data committed, no network calls from tests.
- No secrets, no unrelated changes, no weakened or skipped tests, no disabled checks, no changes to the workflow and guardrail files.
- Each finding: file, line, problem, expected fix. Use NO-GO for anything that must change before merge.

## Verdicts

- **GO:** ready for a draft PR at this SHA.
- **NO-GO:** the author fixes, commits and reports again; you review the new SHA.
- After 3 NO-GO rounds on the same plan or task, the coordinator stops and asks Nathan.
