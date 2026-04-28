# Global Claude Code Preferences

## Git Commits
- NEVER add `Co-Authored-By` lines to commits
- Commit message format: `prefix(scope): Title` or `prefix: Title` (scope is optional)
  - Allowed prefixes: fix, refactor, docs, feat, ui, test
  - First letter after `:` must be capitalized
  - Title must not exceed 80 characters
  - Example: `fix(ingester): Resolve issue with ingester function`
  - Example: `feat: Add user authentication`
- Commit body describes WHAT changed behaviorally and WHY, not HOW
  - Do not mention file paths, function names, struct fields, route strings, or config keys -- those are visible in the diff
  - Describe the user-facing or system-level outcome, not the code-level mechanism
  - Bad: "Adds RenewCertificate(domain) in ssl.Manager and POST /certificates/:domain/renew"
  - Good: "Certificates can now be renewed individually by domain"

## Git Push
- NEVER push to a default / shared integration branch (`main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, or any branch configured as the repo's default on the remote) without explicit, per-push instructions from the user -- not just "push", but "push to <that branch>". Even if the current local branch has that name. Even if the user approved committing. A prior approval to commit or push does not extend to shared branches on later turns.
- Default behavior when asked to "push" on a default-branch checkout: create a feature branch from HEAD, push that instead, and tell the user. Do not push directly.
- If the remote push output contains warnings like "Bypassed rule violations", "changes must be made through a pull request", or similar protected-branch warnings, STOP -- assume the push was wrong and surface it to the user immediately.
- `--force` / `--force-with-lease` on any shared branch always requires explicit per-operation confirmation, even if it's correcting an earlier mistake.

## Concision (chat replies, PR comments, commit bodies, drafted messages)
- Lead with the point. One sentence stating what is wrong, what to do, or what you want.
- Cut hedging ("worth confirming", "in practice", "mostly a theoretical"), throat-clearing ("LGTM in spirit"), and meta-narration about the analysis.
- Skip restating the diff. The reader has it open.
- Concrete words over abstract ones. "Falls back to /api" not "violates the invariant".
- If a comment runs past 5 lines, cut it in half before posting.
- Listing possibilities is fine; pad each option with no more than a sentence.
- Apply this everywhere prose is written: chat, PR review comments, issue bodies, commit messages, drafted Slack/Teams messages.

## Writing style
- NEVER use em dashes (`—`, U+2014) in any prose output: code comments, commit messages, PR descriptions, chat replies, user-visible copy in apps/sites, or documentation.
  - In **Markdown or plain-text documents** (READMEs, notes, commit bodies, PR descriptions), prefer `--` (two hyphens) where an em dash would otherwise go.
  - In **rendered prose** (HTML/JSX body copy, marketing pages, blog posts, UI microcopy), do NOT use `--` either. Rewrite the sentence with a colon, semicolon, period, or parentheses, whichever preserves the meaning most naturally. Common substitutions: parenthetical aside -> parentheses; elaboration/list introduction -> colon; abrupt break -> period or semicolon.
- En dashes (`–`) are fine for numeric ranges (e.g. `pp. 45–52`, `2024–2026`) but should not be used as a punctuation dash in prose. Convert prose en dashes the same way as em dashes.
- When editing an existing document that contains em dashes, replace them as part of the change. Do not leave them in place "because they weren't mine".
