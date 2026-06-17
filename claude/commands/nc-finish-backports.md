---
description: Clean up and finish the auto-created backport PRs for a merged nextcloud/server PR
argument-hint: <original-merged-PR-number>
allowed-tools: Bash(git:*), Bash(gh:*), Bash(diff:*), Bash(cat:*), Bash(ls:*), Read, Edit, Write
---

You are finishing the bot-created backport PRs for the already-merged PR **#$1** in `nextcloud/server`, and deleting the merged local branch.

The backport bot leaves these PRs in a rough state: mangled commit messages, a `[skip ci]` line, empty placeholder "Recompile assets" commits, and sometimes based on a stale stable tip. Your job is to make each backport a clean replay of the original on top of the latest stable branch, then force-push it.

## Context you must gather first (verify, do not assume)

1. Find the `nextcloud/server` checkout. The current dir may be a different repo (e.g. docker-dev). Confirm with `git remote -v`; if origin is not `nextcloud/server`, locate the server checkout (commonly a `server/` subdir of the workspace) and `cd` there for all git work.
2. `git fetch origin master stable33 stable34 ...` and also fetch the original PR's branch refs. Use `gh pr view $1 --repo nextcloud/server --json ...` to get the original head branch, base branch, and **the exact commit list (oids + full messages)** of the merged PR. The original source commit message is your source of truth for rewording.
3. Find the backports: `gh pr list --repo nextcloud/server --search "$1 in:body" --state all --json number,baseRefName,headRefName,state,url,body`. Backport branches are named `backport/$1/<stableXX>`. Only act on OPEN ones.
4. Identify the original PR's **source commit** (the real code change) and any **assets commit** (`chore(assets): Recompile assets`, touches only `dist/`). The source commit's message and patch are the reference.

## Delete the merged local branch

- Switch off it first (`git checkout <default-branch>`).
- Fast-forward the local default branch to `origin/<default>` so the safe delete recognises it as merged.
- `git branch -d <original-head-branch>` (safe delete; never `-D` unless you have proven it is merged and explain why).

## For each OPEN backport PR

Work on a local branch tracking the remote backport branch: `git checkout -B backport/$1/<stable> origin/backport/$1/<stable>`. Record the current remote tip SHA — you need it for the lease on push.

1. **Rebase onto the latest stable.** `git rebase --empty=drop origin/<stable>`. If it reports "up to date", the branch is already on the latest tip; the empty commit will NOT have been dropped by the no-op rebase, so drop it explicitly (see step 2). Resolve any conflicts so the result reproduces the original change; never invent code.
2. **Drop empty placeholder commits.** A `chore(assets): Recompile assets` commit whose `git diff --stat HEAD^ HEAD` is empty is noise — remove it (e.g. `git reset --hard <source-commit>` when it is the tip, or drop it in the rebase todo). Do NOT manually recompile assets.
3. **Fix the commit message(s).** Reword the source commit to **exactly** the original source commit's full message (`git show -s --format=%B <original-source-oid>`). This removes the bot's duplicated headline, double-spaced body lines, and any trailing `[skip ci]` line. Use `git commit --amend -F <tmpfile>` (write the message to a temp file outside the repo).
4. **Verify the backport matches the original.** The source patch must be byte-identical to the original (compiled `dist/` differs per branch and is expected to differ):
   `diff <(git show <original-source-oid> -- apps/ lib/ core/ | tail -n +2) <(git show HEAD -- apps/ lib/ core/ | tail -n +2)` should be empty. Adjust the path list to whatever the original touched. If it is not identical, stop and report — do not push a backport that diverges from the original.
5. **Force-push.** History was rewritten, so a force-push is required. These are PR feature branches (`backport/...`), not shared/default branches, so this is in-scope for the task — but still use `--force-with-lease` with the **explicit** expected SHA, because the tracking ref created by an explicit refspec fetch often lacks a reflog and a bare `--force-with-lease` fails with "stale info":
   `git push --force-with-lease=backport/$1/<stable>:<recorded-remote-tip-sha> origin backport/$1/<stable>`

## Rules and gotchas

- NEVER recompile assets manually. Removing `[skip ci]` lets CI's asset-staleness check run; if `dist/` is stale, tell the user to comment `/compile amend` on the PR. State clearly that assets were not recompiled.
- NEVER touch the `3rdparty` submodule pointer if it shows as modified — that is pre-existing working-tree state, unrelated to the backport.
- NEVER force-push or otherwise write to a default/shared branch (master/stableXX). Only the `backport/...` PR branches.
- NEVER add AI/Claude attribution to any commit message; the reworded message is the original's verbatim message and nothing else.
- Leave the repo checked out on its default branch with a clean working tree at the end.

## Report at the end

For each backport: the PR number/URL, the final single-line commit summary, confirmation that no `[skip ci]` remains and the source is identical to the original, and whether assets still need a `/compile amend`. Confirm the merged local branch was deleted.
