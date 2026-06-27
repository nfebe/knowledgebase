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
5. **Draft the PR title and body.** Title in the same format as the commit subject.
   - **Lead with a short overview of the change**: what it is and the context a reviewer needs, where it came from (a bug, an issue or ticket, an alert, a request), and anything load-bearing about why it exists. For a small or one-line change that overview is the entire body: a sentence or two, no headings. Match the body's weight to the change's weight.
   - **Add further sections only when the specific change warrants them**, and let the change decide which: screenshots or a recording for UI, repro and root cause for a bug, apply or rollout notes for infra, risk or follow-ups when there are any. Do NOT impose a fixed What/Why/Changes scaffold, and do NOT wrap a trivial change in headed sections; that reads as ceremony, not communication.
   - The body renders as GitHub-flavored markdown: use bullets and headings where they aid scanning, plain sentences where they do not. Do NOT hard-wrap to a fixed column width (the ~72-column rule is commit-message only; a hard-wrapped body reads like a pasted commit). No diff restatement, no attribution footer, no robot emoji, no tool credit. Avoid em dashes (use one only if it is genuinely the clearest option). End on the last line of real content.
   - Then run `/nfebe-verify` over the drafted body: any factual claim about what the change does, why, or how it was confirmed must match the actual diff and evidence from this session, not a remembered or assumed shape. Correct or drop anything unsupported.
6. **STOP and ask.** Print the pushed branch URL, the drafted title, and the drafted body. Ask one yes/no: "Open it now?" Do NOT run `gh pr create` (or any equivalent) until I explicitly say yes. Treat "open a PR" / "PR it" as a request to prepare, not to create.

## After the PR is open
Once I approve and the PR exists, confirm CI actually goes green (`gh pr checks <branch> --watch`, or watch the Actions run) and check for review feedback. If any check fails or a review requests changes, fix it, recommit, repush, and re-verify. Do not treat the PR as ready while checks are red.

## Before you finish
Re-scan the committed diff and the drafted PR text and strip three things:
- **Unnecessary / obvious comments** in the diff. The `/nfebe-commit` scrub covers this, but fixes you make while getting CI green (step 3) can add new ones, so check again. Remove comments that only restate what the code does or narrate that the task I described is done ("Add the export button", "Now stream the rows"). Keep a comment only when it explains a non-obvious decision a reader must know to change the line safely (invariant, workaround, surprising default, optimization, tricky math). When in doubt, cut it.
- **Attribution** in any commit message or PR text.
- **Dash punctuation** in commit messages and PR text.

Report the branch, the title, the body, and the single yes/no question.
