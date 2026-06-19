# Nextcloud PR workflow

Notes on preparing pull requests and backports for `nextcloud/server`.

## Disregard the `dist/` folder

When preparing a PR or a backport, never commit local `npm run build` output. The compiled assets under `dist/` (and the bundled CSS) must be left to the project's `/compile` bot.

Running the build locally produces hundreds of churned files: chunk-hash renames and sourcemap drift across unrelated apps (for example `user_ldap`, `workflowengine`, `weather_status`), because the local toolchain does not match the bot's controlled environment. That noise is not a real recompile and must not land on the branch.

Correct flow:

1. Commit only the source change.
2. Push the branch.
3. Comment `/compile` (or `/compile rebase`) on the PR. The bot regenerates the correct minimal asset diff and commits it as `chore(assets): Recompile assets`.

If a rebase drops the bot's recompile commit, that is fine: the assets are stale only until the next `/compile`, and the bot owns regenerating them.

To discard an accidental local build, restore tracked files with `git checkout -- .` rather than `git clean`, so untracked source is never deleted.
