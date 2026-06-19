---
description: Prepare a pull request to my standards and stop before creating it
argument-hint: [base-branch] [extra context]
allowed-tools: Bash(git:*), Bash(gh:*), Read, Edit
---

Prepare a PR for the current work, following my rules exactly. **Do NOT create the PR.** $ARGUMENTS

## Steps

1. **Determine branches.** Current branch (`git branch --show-current`) and the repo's default/shared branch (`git symbolic-ref --short refs/remotes/origin/HEAD` or `gh repo view --json defaultBranchRef`). Shared branches include `main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, and whatever the remote default is.
2. **Get the work committed via `/nfebe-commit`.** Run the `/nfebe-commit` flow to land the pending work on a properly named feature branch cut from the updated default. It handles the branch-off, refuses shared branches, scrubs the diff, conforms the message to the repo's commit validation job, and never pushes. If the work is already committed cleanly, skip. Come back here once the commit exists, then continue with push and PR draft below.
3. **Verify the project's checks pass before pushing.** Run the SAME checks CI runs, not just your own auto-formatter: lint, format check, static analysis, and tests. Discover the exact commands from the repo, do not guess them: read the CI workflow files under `.github/workflows`, then the `composer.json` scripts (for example `phpcs:test`, `test`) and `package.json` scripts (for example `lint`, `format:check`, `test`). A local `pint` or `eslint --fix` going green does NOT prove CI passes: `phpcs` line limits and `prettier --check` across the repo can still fail. Fix and amend the commit until they pass. If a check only runs inside a container, run it there, or state plainly which one you could not verify. Do not push red.
4. **Push the feature branch** to `origin`. If the push output shows protected-branch warnings ("Bypassed rule violations", "must be made through a pull request"), STOP and surface it: the push went somewhere it should not have.
5. **Draft the PR title and body.** Title in the same format as the commit subject. The body renders as GitHub-flavored markdown, so structure it: short headings, bullet lists, tight one-line points. Not prose paragraphs and not a wall of text. Do NOT hard-wrap body lines to a fixed column width: the ~72-column wrapping is a commit-message rule only, never apply it to a PR body (a hard-wrapped body reads like a pasted commit). Each point is its own line; let the renderer handle width. Lead with the point, WHAT and WHY, no diff restatement. No attribution footer, no robot emoji, no tool credit. Avoid em dashes (use one only if it is genuinely the clearest option). End on the last line of real content.
6. **STOP and ask.** Print the pushed branch URL, the drafted title, and the drafted body. Ask one yes/no: "Open it now?" Do NOT run `gh pr create` (or any equivalent) until I explicitly say yes. Treat "open a PR" / "PR it" as a request to prepare, not to create.

## After the PR is open
Once I approve and the PR exists, confirm CI actually goes green (`gh pr checks <branch> --watch`, or watch the Actions run) and check for review feedback. If any check fails or a review requests changes, fix it, recommit, repush, and re-verify. Do not treat the PR as ready while checks are red.

## Before you finish
Re-scan the commit message(s) and drafted PR text for any attribution line or dash punctuation and strip it. Report the branch, the title, the body, and the single yes/no question.
