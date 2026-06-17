---
description: Prepare a pull request to my standards and stop before creating it
argument-hint: [base-branch] [extra context]
allowed-tools: Bash(git:*), Bash(gh:*), Read, Edit
---

Prepare a PR for the current work, following my rules exactly. **Do NOT create the PR.** $ARGUMENTS

## Steps

1. **Determine branches.** Current branch (`git branch --show-current`) and the repo's default/shared branch (`git symbolic-ref --short refs/remotes/origin/HEAD` or `gh repo view --json defaultBranchRef`). Shared branches include `main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, and whatever the remote default is.
2. **Never work on a shared branch.** If the current branch is a shared one, create a feature branch from HEAD first and switch to it. Never commit or push on the shared branch. Name the branch from the change, not from anything sensitive that leaked.
3. **Commit pending work** (if any) using my format:
   - Subject: `prefix(scope): Title` or `prefix: Title`. Prefixes: `fix`, `refactor`, `docs`, `feat`, `ui`, `test`. First letter after `:` capitalized. Max 80 chars.
   - Body: WHAT changed behaviourally and WHY, not HOW. No file paths, function names, fields, routes, or config keys. Describe the user/system outcome.
   - No AI/Claude attribution. No `Co-Authored-By`. No dashes as punctuation (em dash never; rewrite with colon/parentheses/period).
4. **Push the feature branch** to `origin`. If the push output shows protected-branch warnings ("Bypassed rule violations", "must be made through a pull request"), STOP and surface it: the push went somewhere it should not have.
5. **Draft the PR title and body.** Title in the same format as the commit subject. Body: concise, lead with the point, WHAT/WHY, no diff restatement. No attribution footer, no robot emoji, no tool credit. No dashes as punctuation. End on the last line of real content.
6. **STOP and ask.** Print the pushed branch URL, the drafted title, and the drafted body. Ask one yes/no: "Open it now?" Do NOT run `gh pr create` (or any equivalent) until I explicitly say yes. Treat "open a PR" / "PR it" as a request to prepare, not to create.

## Before you finish
Re-scan the commit message(s) and drafted PR text for any attribution line or dash punctuation and strip it. Report the branch, the title, the body, and the single yes/no question.
