---
description: Clean merged feature branches and refresh the default branch in related repos
argument-hint: [repo paths or parent dir...] [--branch <new-feature-name>]
allowed-tools: Bash(git:*), Bash(gh:*), Read
---

Get one or more related repos into a clean state: prune stale branches, delete merged feature branches, and update each default branch to the latest. Optionally start a fresh feature branch at the end. $ARGUMENTS

## Resolve the target repos

- If repo paths are given, use them. If a parent directory is given, treat each immediate subdirectory that is a git repo as a target. If nothing is given, use the current repo only.
- Print the resolved repo list. For anything beyond the current repo, confirm the list before deleting any branches.

## Per repo (nothing destructive until the working tree is clean)

1. **Refuse a dirty tree.** Run `git status --porcelain`. If there are staged, unstaged, or untracked changes, STOP for this repo, report it, and skip. Never stash, reset, `clean`, or discard to force it clean.
2. **Identify the default branch.** `git symbolic-ref --short refs/remotes/origin/HEAD`, falling back to `gh repo view --json defaultBranchRef`.
3. **Fetch and prune.** `git fetch --all --prune` so branches whose remote was deleted get marked `[gone]`.
4. **Update the default branch.** Switch to it and fast-forward only: `git switch <default>` then `git pull --ff-only`. If it cannot fast-forward, STOP and report. Do not merge, rebase, or reset the default branch.
5. **Delete merged feature branches.** For every local branch that is neither the default nor protected, try `git branch -d <branch>` (safe delete: it refuses anything not merged into the current default). **Protected branches you must never delete:** `main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, anything matching `release/*` or `stable*`, and the remote default.
6. **Surface, never auto-force.** A branch that `-d` refused is either a squash-merged PR or genuinely unmerged work. Do not `git branch -D` it on your own. Collect those branches, cross-check `git branch -vv` for a `[gone]` upstream (a strong sign the PR was squash-merged and the branch is safe to drop), list them with that signal, and ask before any force delete. Unmerged work without a `[gone]` upstream stays untouched.

## Finish

- Leave each repo on its updated default branch, clean.
- If a new feature branch name was given (`--branch <name>`, or clearly stated in the args), create it off the updated default in the primary repo and switch to it. Do not push it.
- Report per repo: the default branch and that it is up to date, branches deleted, branches kept (with the reason), and any repo skipped for being dirty.
