# Claude Code custom commands

Canonical, version-controlled home for my Claude Code slash commands. Each `.md` file here becomes a `/<filename>` command. The files are symlinked into `~/.claude/commands/` so Claude Code loads them while the source of truth stays tracked in this repo (and syncs across machines via `~/work`).

## Naming convention

- `nc-*`: Nextcloud work commands (operate on `nextcloud/server` etc.). Example: `/nc-finish-backports`.
- `nfebe-*`: my personal, cross-project commands. Example: `/nfebe-pr`.

## Use on another machine

From a checkout of this repo, link every command into the Claude Code commands dir:

```sh
mkdir -p ~/.claude/commands
for f in ~/work/knowledgebase/claude/commands/*.md; do
  [ "$(basename "$f")" = "README.md" ] && continue
  ln -sf "$f" ~/.claude/commands/"$(basename "$f")"
done
```

Edit a command here, commit, and the change is live everywhere the symlinks point.

## Commands

### Nextcloud (`nc-`)

- `nc-finish-backports`: clean up and finish the bot-created backport PRs for a merged `nextcloud/server` PR (rebase onto latest stable, drop empty commits, de-mangle the message, remove `[skip ci]`, verify the source matches the original, force-push).

### Personal (`nfebe-`)

- `nfebe-pr`: prepare a PR to my standards (branch off shared branches, commit in my format, push the feature branch, draft title and body) and stop before creating it.
- `nfebe-commit`: commit staged changes in my format after a pre-commit scrub for secrets, attribution, dashes, and contextual comments; refuses shared branches.
- `nfebe-scrub`: clean a diff, message, or drafted text to my writing and hygiene rules. No git side effects.
- `nfebe-secret-scan`: read-only scan of changes and example/config files for secret-shaped strings before commit.
- `nfebe-test`: write or run tests that exercise the same boundary a real user hits, with no fabricated external shapes.
- `nfebe-review`: review a diff against my global rules, citing only files actually opened this session.
- `nfebe-git-clean`: clean merged feature branches and refresh the default branch in one or more related repos (safe-delete only, protects shared branches); optionally start a fresh feature branch.
- `nfebe-update-knowledge`: record a note into the knowledge base, sync the live global Claude config (`CLAUDE.md` and command symlinks) into the repo, then commit and push to `main`.
