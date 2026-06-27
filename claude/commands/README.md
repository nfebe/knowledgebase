# Claude Code custom commands

Canonical, version-controlled home for my Claude Code slash commands. Each `.md` file here becomes a `/<filename>` command. The files are symlinked into `~/.claude/commands/` so Claude Code loads them while the source of truth stays tracked in this repo (and syncs across machines via `~/work`).

## Naming convention

- `nc-*`: Nextcloud work commands (operate on `nextcloud/server` etc.). Example: `/nc-finish-backports`.
- `nfebe-*`: my personal, cross-project commands. Example: `/nfebe-pr`.
- `ws-*`: WhileSmart PHP/Laravel package work. Example: `/ws-php-make-package`.

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
- `nc-pr`: prepare or finish a `nextcloud/server` PR. Wraps `/nfebe-pr` and adds the Nextcloud rules: locate the server checkout, handle review threads (verify external shapes, rebase, fold into the fix commit), and always disregard `dist/` (never commit local `npm run build` output; let the `/compile` bot regenerate assets). Confirms before any push or PR create.
- `nc-triage`: triage a Nextcloud support ticket end to end: verify context, find the root cause in code (or research a product/feature/availability question with cited sources), classify it, write a structured note into the `nc-tickets` repo with a customer response draft, then commit and push to `master`.

### Personal (`nfebe-`)

- `nfebe-pr`: prepare a PR to my standards (commit via `/nfebe-commit`, verify the repo's CI checks, push the feature branch, draft title and body) and stop before creating it.
- `nfebe-commit`: branch off the updated default (name format `prefix/issue/short-name`) and commit my work after a pre-commit scrub for secrets, attribution, dashes, and contextual comments; conforms the message to the repo's commit-validation job (commitlint) so it does not fail CI; pauses and asks when already on a feature branch; no push, no PR.
- `nfebe-scrub`: clean a diff, message, or drafted text to my writing and hygiene rules. No git side effects.
- `nfebe-issue`: draft or clean a GitHub issue to my style (conventional-commit title, terse Task/Action/Acceptance, no names/quotes/process-metadata). Stops before `gh issue create`.
- `nfebe-consult`: search the knowledge base for notes on a topic and summarize what it records, citing sources. Read-only.
- `nfebe-secret-scan`: read-only scan of changes and example/config files for secret-shaped strings before commit.
- `nfebe-test`: write or run tests that exercise the same boundary a real user hits, with no fabricated external shapes.
- `nfebe-review`: review a diff against my global rules, citing only files actually opened this session.
- `nfebe-verify`: fact-check a note, draft, or answer against evidence gathered this session, flag fabrications, over-broad statements, and assumptions, label what is unverified, and apply the corrections. Wired in as a final pass in `nc-triage`, `nfebe-issue`, `nfebe-pr`, and `nfebe-review`. Read-only plus edits to the target; no commit or push.
- `nfebe-writing-style`: rewrite or check text to my voice (measured on uncertain claims, explicit "Unverified" labels, tables/bullets for enumerable facts), on top of the hard rules in the global `CLAUDE.md`. Carries calibrated picked-vs-rejected examples.
- `nfebe-new-task`: prepare one or more repos to start any task (fix, feature, refactor, chore, docs, investigation): land on the freshly updated default branch and cut a properly named branch (same branch rules as `nfebe-commit`). The general form of `nfebe-new-feature`. No staging, commit, or push.
- `nfebe-new-feature`: prepare one or more repos to start a new feature: land on the freshly updated default branch and cut a properly named feature branch (same branch rules as `nfebe-commit`). No staging, commit, or push.
- `nfebe-git-clean`: clean merged feature branches and refresh the default branch in one or more related repos (safe-delete only, protects shared branches); optionally start a fresh feature branch.
- `nfebe-update-knowledge`: record a note into the knowledge base, sync the live global Claude config (`CLAUDE.md` and command symlinks) into the repo, then commit and push to `main`.
- `nfebe-design-google`: move a UI toward a Google-style design language (clean, colorful, creative, illustration- and vector-rich) while keeping the project's own brand. Brand-adaptive, works on any project.

### WhileSmart (`ws-`)

- `ws-php-make-package`: scaffold a new `whilesmart/eloquent-<noun>` Laravel package to the WhileSmart conventions: polymorphic owner via `eloquent-owner-access`, config-gated routes, any integration abstracted behind a contract plus a host binding (never a vendor hard-coded in the schema), Testbench tests, and the meta and workflow files. Verifies in a `laravel-dev` container until Pint and tests pass. No commit or publish.
