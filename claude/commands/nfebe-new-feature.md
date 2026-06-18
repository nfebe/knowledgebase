---
description: Prepare one or more repos to start a new feature: land on the updated default and cut a properly named branch. No commits, no push.
argument-hint: [prefix/issue-number/short-name] [repos...]
allowed-tools: Bash(git:*), Bash(gh:*), Read
---

Get a repo (or several) ready for new work: confirmed on the freshly updated default branch, with a clean, well-named feature branch checked out. This command only positions the branch. It does not stage, commit, or push. $ARGUMENTS

## Where the branch comes from (same rules as `/nfebe-commit`)

The feature branch must be cut from the freshly updated default branch, never started on a shared branch and never silently on whatever happens to be checked out.

1. **Find the default branch:** `git symbolic-ref --short refs/remotes/origin/HEAD` (fall back to `gh repo view --json defaultBranchRef`). Shared branches: `main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, and the remote default.
2. **Look at the current branch and `git status --porcelain`, then branch by case:**
   - **On the default branch, clean tree:** the happy path. Update it (`git pull --ff-only`), then cut the feature branch (name format below).
   - **On the default branch with uncommitted changes:** PAUSE and ask. Do not stash or discard on your own. Offer to (a) carry the changes onto the new branch (cut the branch first, work continues there), or (b) stash, update default, cut the branch, pop. Wait for the answer.
   - **On a feature branch already (not the default):** PAUSE and ask. Do not assume the current branch is the right base. Offer to (a) switch to the default, `pull --ff-only`, and cut a fresh branch from there, or (b) start the new branch off the current branch. Wait for the answer.
   - **Default branch can't fast-forward, or detached HEAD:** stop and report. Do not merge, rebase, reset, or force.
3. **Branch name format:** `prefix/issue-number-if-known/short-name`, for example `feat/142/windows-cache` or, with no known issue, `enh/transactions-empty-state`. The prefix is the change type (`fix`, `enh`, `refactor`, `feat`, `docs`, `test`, or another fitting one). Use the name from the args if given; otherwise propose one from the feature description and confirm before creating.

## Multiple repos

- The args may name several repos, or the feature may obviously span more than one. Resolve each repo independently: its own default-branch lookup, its own clean-tree check, its own branch-off decision. Do the same-named branch in each so the set stays consistent, and report each repo's result separately. If one repo is in a state that needs a question (case above), ask for that repo and continue preparing the others meanwhile.

## What this command does not do

- No `git add`, no commit, no push, no PR. Cutting the branch is the end of the job. After it runs, `/nfebe-commit` handles staging and committing on the branch this prepared.
- Do not create the branch on the remote. Local branch only; the first push happens later through the normal flow.

## Report

- Per repo: the default branch it updated, whether a fast-forward pulled anything new, and the feature branch now checked out. If a repo is waiting on an answer, say which one and what the choice is.
