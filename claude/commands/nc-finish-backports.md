---
description: Clean up and finish the auto-created backport PRs for a merged nextcloud/server PR
argument-hint: <original-PR or backport-PR number or URL>
allowed-tools: Bash(git:*), Bash(gh:*), Bash(diff:*), Bash(cat:*), Bash(ls:*), Read, Edit, Write
---

You are finishing bot-created backport PRs in `nextcloud/server`, and deleting the merged local branch when one exists.

`$1` may be a PR number or a URL, and it may point at either the **original merged PR** or a **single backport PR**. Resolve which one before acting (see Context step 2):
- If `$1` is the **original** PR, finish **all** of its OPEN backports.
- If `$1` is a **single backport** PR (head `backport/<orig>/<stableXX>`), finish **only that one PR**. Do not touch its sibling backports.

Throughout, `<orig>` is the original PR number: it equals `$1` when `$1` is the original, or the number parsed from the backport head when `$1` is a backport. Backport branches are always named `backport/<orig>/<stableXX>`.

The backport bot leaves these PRs in a rough state: mangled commit messages, a `[skip ci]` line, empty placeholder "Recompile assets" commits, and sometimes based on a stale stable tip. Finishing one is mostly **verification**: confirm it is a faithful replay of the original, based on a current stable tip, with a clean message and no placeholder commits. Bring it up to standard and force-push only if you actually changed something. The repair needed depends on the situation, so the steps below are conditional, not a fixed recipe.

## Context you must gather first (verify, do not assume)

1. Find the `nextcloud/server` checkout. The current dir may be a different repo (e.g. docker-dev). Confirm with `git remote -v`; if origin is not `nextcloud/server`, locate the server checkout (commonly a `server/` subdir of the workspace) and `cd` there for all git work.
2. **Resolve the scope of `$1`.** `gh pr view $1 --repo nextcloud/server --json number,state,title,headRefName,baseRefName`. If `headRefName` matches `backport/<orig>/<stableXX>`, then `$1` is a single backport PR: set `<orig>` to that parsed number and finish ONLY this PR. Otherwise `$1` is the original PR: set `<orig>` to `$1` and finish all of its OPEN backports.
3. `git fetch origin master stable33 stable34 ...` and the relevant backport branch refs. Use `gh pr view <orig> --repo nextcloud/server --json ...` to get the original head branch, base branch, and **the exact commit list (oids + full messages)** of the merged PR. The original source commit message is your source of truth for rewording.
4. Determine the backport set. If `$1` was a single backport PR, the set is just that one. Otherwise find them all: `gh pr list --repo nextcloud/server --search "<orig> in:body" --state all --json number,baseRefName,headRefName,state,url,body`. Backport branches are named `backport/<orig>/<stableXX>`. Only act on OPEN ones.
5. Identify the original PR's **source commit(s)** (the real code change; there may be more than one, for example a fix plus a test) and any **assets commit** (`chore(assets): Recompile assets`, touches only `dist/`). The source commit's message and patch are the reference.

## Delete the merged local branch

Only if the original head branch actually exists locally (`git branch --list <original-head-branch>`). It usually does not, in which case skip this section and say so.

- Switch off it first (`git checkout <default-branch>`).
- Fast-forward the local default branch to `origin/<default>` so the safe delete recognises it as merged.
- `git branch -d <original-head-branch>` (safe delete; never `-D` unless you have proven it is merged and explain why).

## For each OPEN backport PR in scope

Work on a local branch tracking the remote backport branch: `git checkout -B backport/<orig>/<stable> origin/backport/<orig>/<stable>`. Record the current remote tip SHA; you need it for the lease on push.

1. **Rebase only if the backport is behind its target.** Check first: if `git merge-base origin/<stable> HEAD` already equals `git rev-parse origin/<stable>`, the branch is on the current stable tip and no rebase is needed. Only when it is behind, `git rebase --empty=drop origin/<stable>`, resolving conflicts so the result reproduces the original change; never invent code. (A no-op rebase does not drop an empty placeholder commit, so drop it explicitly per step 2.)
2. **Drop empty placeholder commits.** A `chore(assets): Recompile assets` commit whose `git diff --stat HEAD^ HEAD` is empty is noise; remove it (e.g. `git reset --hard <source-commit>` when it is the tip, or drop it in the rebase todo). Do NOT manually recompile assets.
3. **Fix the commit message(s).** Reword the source commit to **exactly** the original source commit's full message (`git show -s --format=%B <original-source-oid>`). This removes the bot's duplicated headline, double-spaced body lines, and any trailing `[skip ci]` line. Use `git commit --amend -F <tmpfile>` (write the message to a temp file outside the repo).
4. **Verify the backport matches the original.** Compare the actual changed lines, not a raw diff: `index` and `@@` header lines legitimately differ across stable branches (blob hashes and line numbers shift), so a plain `diff` of two `git show` outputs reports spurious differences. Compare only the `+`/`-` content lines:
   ```
   pl() { git show --format= "$1" -- apps/ lib/ core/ | grep -E '^[-+]' | grep -vE '^[-+][-+][-+]'; }
   diff <(pl <original-source-oid>) <(pl HEAD)
   ```
   This must be empty (adjust the path list to whatever the original touched). It catches dropped or adapted hunks: in one case the bot's backport silently dropped one of several identical `setTime` changes, which this check surfaced. If it is not identical, stop and report; do not push a backport that diverges from the original. If a hunk was dropped, the fix is to reproduce the original change exactly (never invent code), then re-verify.
5. **Force-push only if you changed the branch.** If verification passed and you made no changes, there is nothing to push; say so. When you did rewrite history, force-push. These are PR feature branches (`backport/...`), not shared/default branches, so this is in scope. Use `--force-with-lease` with the **explicit** expected SHA, because the tracking ref created by an explicit refspec fetch often lacks a reflog and a bare `--force-with-lease` fails with "stale info":
   `git push --force-with-lease=backport/<orig>/<stable>:<recorded-remote-tip-sha> origin backport/<orig>/<stable>`

## Rules and gotchas

- The strategy depends on the situation; finishing is verification first, mechanics second. If the bot's branch is already on the current stable tip, you may only need to fix the message and verify (no rebase). If it is behind its stable target, rebase onto the tip. If no backport branch exists yet (starting from scratch, the bot did not create one), cherry-pick the original commits onto the stable base and resolve conflicts to reproduce the original. Do not reflexively cherry-pick the master originals to rebuild a branch the bot already adapted, and do not rebase when the branch is already current.
- NEVER recompile assets manually. Removing `[skip ci]` lets CI's asset-staleness check run; if `dist/` is stale, tell the user to comment `/compile amend` on the PR. State clearly that assets were not recompiled.
- NEVER touch the `3rdparty` submodule pointer if it shows as modified; that is pre-existing working-tree state, unrelated to the backport.
- NEVER force-push or otherwise write to a default/shared branch (master/stableXX). Only the `backport/...` PR branches.
- NEVER add AI/Claude attribution to any commit message; the reworded message is the original's verbatim message and nothing else.
- Leave the repo checked out on its default branch with a clean working tree at the end.

## Report at the end

For each backport in scope: the PR number/URL, the final single-line commit summary, what you did (rebased, message-only fix, or nothing needed), confirmation that no `[skip ci]` remains and the source is identical to the original, and whether assets still need a `/compile amend`. Confirm the merged local branch was deleted, or note it did not exist locally.
