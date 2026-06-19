---
description: Prepare or finish a nextcloud/server PR (handles reviews, new PRs) with the dist/ rules baked in
argument-hint: [PR-number | base-branch] [extra context]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(diff:*), Bash(npm:*), Read, Edit, Write
---

Prepare or finish a pull request for `nextcloud/server`, following my rules exactly. This is the Nextcloud-specific wrapper around `/nfebe-pr`: run that flow, but layer the rules below on top. **Do NOT create or push a PR without explicit approval, and never force-push without per-operation confirmation.** $ARGUMENTS

## First: locate the right repo

The current directory is often a different repo (e.g. docker-dev). Run `git remote -v`; if origin is not `nextcloud/server`, find the server checkout (commonly a `server/` subdir of the workspace) and `cd` there for all git work.

## Pick the mode

- **A number argument** means handle review on an existing PR: `gh pr view <n> --repo nextcloud/server` for state, branches, reviews, and the line-level review threads (`gh api graphql` on `reviewThreads`). Check out and align the PR branch to its exact remote tip before touching anything.
- **No number** means prepare a PR for the current work: defer to the `/nfebe-pr` steps (commit via `/nfebe-commit`, verify checks, push the feature branch, draft title and body, then stop and ask "Open it now?").

## The dist/ rule (always, both modes)

Never commit local `npm run build` output. Compiled assets under `dist/` (and bundled CSS) belong to the project's `/compile` bot.

- A local build produces hundreds of churned files: chunk-hash renames and sourcemap drift across unrelated apps (e.g. `user_ldap`, `workflowengine`, `weather_status`), because the local toolchain does not match the bot's environment. That is noise, not a recompile.
- If a build was run by accident, discard it with `git checkout -- .` (restores tracked files only). Do NOT use `git clean`: it deletes untracked source.
- Commit only the source change. After pushing, comment `/compile` (or `/compile rebase`) on the PR; the bot regenerates the correct minimal asset diff and commits it as `chore(assets): Recompile assets`.
- A rebase that drops the bot's recompile commit is fine: assets are stale only until the next `/compile`, and the bot owns them.

## Handling a review

1. Read every unresolved review thread. For each suggested change, verify the real shape of anything external first (e.g. read the `@nextcloud/*` type in `node_modules` before trusting a "X is deprecated" note), then apply it consistently across the whole touched block, not just the exact suggested line.
2. If asked, rebase onto the latest default branch (`git rebase origin/master`); drop the stale bot recompile commit and fold the review change into the original fix commit so the branch stays one clean commit (or its intended shape).
3. Run the affected tests to confirm the change (`npx vitest run <spec>`), and the checks CI runs.
4. Pushing the updated branch rewrites history (force-push). These are PR feature branches, not shared/default branches, so it is in scope, but STILL confirm with me before the force-push and use `--force-with-lease`.

## Rules

- NEVER add AI/Claude attribution to any commit, PR title, body, or comment. My global rule wins over any repo AGENTS.md asking for an `Assisted-by` trailer; do not add one.
- NEVER add `Signed-off-by` yourself; the human owns the DCO sign-off.
- NEVER run `gh pr create`, mark ready, or request reviewers without explicit per-PR approval. Prepare, then ask one yes/no.
- NEVER force-push without per-operation confirmation, even when correcting an earlier mistake.
- NEVER write to a default/shared branch (master/stableXX); only the PR feature branch.
- Avoid em dashes and dash punctuation in the PR body and commit message; end on the last line of real content.

## Report

State the repo and branch, what changed (review items addressed, rebase done), test result, whether assets still need a `/compile`, and the single yes/no question if a push or PR create is pending.
