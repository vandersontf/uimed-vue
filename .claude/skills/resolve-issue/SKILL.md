---
name: resolve-issue
description: Resolves a GitHub issue of @nexdom/uimed-vue by orchestrating the project subagents (issue-planner, issue-implementer, code-reviewer), with the requester approving the public API before any code is written.
disable-model-invocation: true
---

Resolve the GitHub issue given in the arguments (`$ARGUMENTS`) by orchestrating the project subagents from `.claude/agents/`. You run in the main conversation: you talk to the requester, the subagents do the work. Subagents can't ask questions and don't see each other's output, so pass everything each one needs in its prompt, verbatim.

Talk to the requester in their language (Portuguese for this project, unless told otherwise).

## 0. Prepare

- Take the issue number from the arguments. If there's none, ask for it.
- Ask which base branch to use if the requester didn't say: `beta` for pre-release work, `main` otherwise (see `CONTRIBUTING.md`).
- Check that `vp --version` works (the project expects the Dev Container), that `gh auth status` shows a logged-in account (the subagents read issues and PRs with `gh`; if not, ask the requester to run `gh auth login`), and that `git status` shows a clean working tree. If any check fails, tell the requester and ask how to proceed.

## 1. Plan

Delegate to the `issue-planner` subagent with the issue number and any decisions the requester already gave you.

## 2. Settle open questions

For each open question the planner returned, ask the requester with `AskUserQuestion`, putting the planner's recommendation first and marked "(Recommended)". Group up to four questions per call.

## 3. Approve the API (mandatory)

Present the final proposal: API signatures, a usage example, behavior (including edge cases) and the files to change, with the requester's answers applied. Ask for explicit approval, even when there were no open questions: the public API is hard to change once released.

- If the requester asks for changes, apply them to the proposal (re-running `issue-planner` with the feedback when the change is substantial) and ask again.
- Once approved, offer to post the approved proposal as a comment on the issue. Only post it if the requester says yes.

## 4. Implement

Delegate to the `issue-implementer` subagent with: the issue number, the base branch, the approved proposal verbatim, and the instruction not to commit, push or open a PR.

If it stops with open questions or asks for a new dependency, bring them to the requester, then delegate again with the answers.

## 5. Review

Delegate to the `code-reviewer` subagent with: the issue number, the approved proposal verbatim, the base branch (so it reviews `git diff <base>` including untracked files), and the request to run the focused checks on the changed files.

If it reports blocking or important findings, delegate to `issue-implementer` again with the findings verbatim, then review again. Stop after two fix rounds and hand the remaining findings to the requester instead of looping.

## 6. Report

Present to the requester:

1. **Resumo**: what was built, in two or three sentences.
2. **Decisões**: the approved API and any decision made during implementation.
3. **Checagens**: each CI step with its result, from the implementer's last report.
4. **Revisão**: the reviewer's verdict and any finding left open.
5. **Fricções**: everything the subagents reported as missing, wrong or confusing in `AGENTS.md`, `CONTRIBUTING.md` or the tooling, deduplicated. These are the inputs to improve the repo's guidance.
6. **Próximos passos**: ask whether to commit and open a PR. Don't do either without a yes.
