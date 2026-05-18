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

## Pull Requests
- NEVER run `gh pr create` or any equivalent without explicit per-PR approval. Even when I say "open a PR", "PR it", "create the PR", or similar -- treat that as a request for preparation, not for the actual `gh pr create` call. Branch off, edit, commit, push the branch, then stop. State the branch URL on the remote, draft the PR title and body, and ask one yes/no: "Open it now?" Wait for an explicit "yes" before running `gh pr create`.
- The same gate applies to publishing changes on PRs: marking ready for review, converting from draft, requesting reviewers. Confirm first.
- NEVER close, reopen, or otherwise change the state of a PR (mine or anyone else's) without me asking. "Closing" is not a step I expect you to take on your own initiative, even to clean up after a PR you mistakenly opened. Surface the mistake, propose closing, and wait.
- If I tell you to "add a rule" or save feedback, that is a memory/rule operation and nothing else -- it is not implicit permission to undo prior actions like closing a PR. Treat the rule update as the entire ask unless I explicitly chain another action.

## Concision (chat replies, PR comments, commit bodies, drafted messages)
- Lead with the point. One sentence stating what is wrong, what to do, or what you want.
- Cut hedging ("worth confirming", "in practice", "mostly a theoretical"), throat-clearing ("LGTM in spirit"), and meta-narration about the analysis.
- Skip restating the diff. The reader has it open.
- Concrete words over abstract ones. "Falls back to /api" not "violates the invariant".
- If a comment runs past 5 lines, cut it in half before posting.
- Listing possibilities is fine; pad each option with no more than a sentence.
- Apply this everywhere prose is written: chat, PR review comments, issue bodies, commit messages, drafted Slack/Teams messages.

## Writing style
- Avoid dashes as a punctuation device in prose. This covers em dashes (`—`, U+2014), en dashes (`–`, U+2013) used as punctuation, and the `--` (two-hyphen) substitute. Applies to every prose output: code comments, commit messages, PR descriptions, chat replies, user-visible copy in apps/sites, and documentation.
  - Default: rewrite the sentence so no dash is needed. Use a colon, semicolon, period, comma, or parentheses, whichever preserves the meaning most naturally. Common substitutions: parenthetical aside -> parentheses or commas; elaboration / list introduction -> colon; abrupt break -> period or semicolon.
  - `--` (two hyphens) is a last-resort fallback for **Markdown or plain-text documents** (READMEs, notes, commit bodies, PR descriptions) only when rewriting would genuinely hurt clarity. Prefer rewriting first; reach for `--` only if the alternative is worse. Never use `--` in rendered prose (HTML/JSX body copy, marketing pages, blog posts, UI microcopy).
  - Em dashes (`—`) are never acceptable, in any context, regardless of how natural they feel. No exceptions for "stylistic" use.
- En dashes (`–`) are fine for **numeric ranges** (e.g. `pp. 45–52`, `2024–2026`). They are not acceptable as a punctuation dash in prose; convert those the same way as em dashes.
- When editing an existing document that contains em dashes, en-dash punctuation, or stylistic `--`, fix them as part of the change. Do not leave them in place "because they weren't mine".

## Explanations
- Explain from first principles. Start at the simplest, most basic building block and build up from there. Shortest path, fewest words, newbie-friendly. Treat me as someone who has never seen the topic before unless the conversation has already shown otherwise.
- Do NOT jump straight to the "core" / advanced explanation. Before going deeper, check what foundation is already established:
  - If the conversation (current messages, files I have open, code I've written, questions I've asked) shows I already know a prerequisite, skip it and say so in one line ("skipping X since you already used it above").
  - If there's no evidence I know the prerequisite, explain the prerequisite first, briefly, before the thing I asked about.
  - If you're unsure, ask one short question ("do you already know how X works?") instead of guessing.
- Order: smallest concept first, then the next layer, then the next. No leapfrogging. No "this assumes you know Y" without then explaining Y or confirming I know it.
- Keep each step short: one idea per sentence, plain words, concrete examples over abstractions. A two-line example beats a paragraph of theory.
- No jargon without a plain-English gloss the first time it appears. "Idempotent (running it twice gives the same result as running it once)" -- not just "idempotent".
- Length: shortest explanation that actually lands. If a one-liner works, stop there. Do not pad with history, caveats, or "fun facts".
- This applies to chat replies, code comments meant to teach, PR review explanations, and any walkthrough I ask for.

## No assumptions, verify first
- I hate assumptions. Verify every claim before presenting it. If you are about to state a fact, propose a fix, name an API, cite a flag, reference a file path, describe how something behaves, or attribute a cause: confirm it first by reading the code, running the command, or fetching the doc. Then include the evidence inline (file:line, command output, quoted snippet, doc URL).
- Before each claim, ask yourself: "Is this true? How do I know?" If the answer is "I think" or "usually" or "it should", that is not good enough. Go check.
- Prefer non-destructive verification: read files, `grep`, `--dry-run`, `--help`, `git status`, `git diff`, `git log`, fetch docs. These never need to be gated.
- Destructive or side-effectful verification steps (writes to disk outside the working tree, schema or DB mutations, network calls that change remote state, package installs/removals, killing processes, `rm`, force-pushes, anything irreversible) require explicit per-operation permission. Do not run them just to confirm a hunch.
  - Before running one, state what you want to run and why, ask, and wait.
  - When the operation touches data that could be lost (files about to be overwritten, DB rows about to change, branches about to be reset), create a backup first (copy the file, dump the table, tag the commit) and tell me where the backup is.
- If verification is not possible without a destructive step I have not approved, say so explicitly and label the statement as unverified rather than presenting it as fact. Do not paper over uncertainty with confident phrasing.
- This applies to root-cause diagnoses, library behavior, framework conventions, version-specific features, error meanings, and any "I remember that..." recall. Memory and training data are not evidence.

## Tests
- Drive tests through the same boundary the user uses. If a feature is reachable via HTTP, set up state and take actions via HTTP, not by calling internal models/services that bypass the controller, validation, allowlist, or middleware. Same principle for CLI commands, queue jobs, UI events, etc.
  - Why: bypassing the user's path hides allowlist mismatches, validation gaps, serialization bugs, and middleware behavior. A test that goes around the controller stays green while real users hit 422s or 500s.
  - For pure unit tests of internal collaborators (functions, parsers, calculators) with no user-facing path, test directly.
  - For state only ever written by an internal code path (a counter the server increments, never the user), seed it the same way production writes it. The principle is "match the production write path", not "always go through HTTP".
  - When fixing a user-reported bug, the regression test should reproduce the user's actual call. If you can't reproduce it through the user's path, you haven't proven the fix.
