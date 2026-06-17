---
description: Commit staged changes in my format after a pre-commit scrub
argument-hint: [scope or message hint]
allowed-tools: Bash(git:*), Read, Edit
---

Commit the staged changes following my rules. $ARGUMENTS

## Guardrails

1. **Refuse shared branches.** If on `main`/`master`/`dev`/`develop`/`trunk`/`production`/`staging` or the remote default, stop and offer to branch off first. Do not commit there.
2. **Inspect, do not blind-add.** Show `git status` and `git diff --staged`. If nothing is staged, ask what to stage rather than running `git add -A` yourself.

## Pre-commit scrub of the staged diff

- **Secrets.** Scan for secret-shaped strings: `AIza` (Google), `sk-` (OpenAI), `ghp_`/`gho_` (GitHub), `xox` (Slack), `AKIA` (AWS), and any long hex/base64 blob under a key named `api_key`/`token`/`password`/`secret`/`*_key`. If one is in a tracked file, STOP and surface it. Do not commit. Recommend a placeholder and tell me to rotate the real key. Do not name the leak in the commit message.
- **Attribution.** Remove any AI/Claude attribution, `Co-Authored-By`, "Generated with" marker, or banned variant from the message and the diff.
- **Dashes.** In added prose and comments, rewrite em dashes (never allowed), en-dash punctuation, and stylistic `--` using a colon, parentheses, comma, or period. En dash is fine only in numeric ranges.
- **Contextual comments.** Flag and remove code comments that reference where the change came from (my requests, colleague names, conversations, ticket/PR/issue IDs, "Step 2", or unrelated file references).

## Commit message format

- Subject: `prefix(scope): Title` or `prefix: Title`. Prefixes: `fix`, `refactor`, `docs`, `feat`, `ui`, `test`. First letter after `:` capitalized. Max 80 chars.
- Body: WHAT changed behaviourally and WHY, not HOW. No file paths, function names, fields, routes, config keys.

Re-scan the final message for attribution and dashes immediately before running `git commit`.
