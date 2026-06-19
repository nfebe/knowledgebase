---
description: Branch off the default and commit my work in my format, conforming to the repo's commit validation. No PR, no push.
argument-hint: [prefix/issue/short-name] [message hint]
allowed-tools: Bash(git:*), Read, Edit
---

Commit my work as one or more clean commits. Never open a PR. Never push. $ARGUMENTS

## Where the commit lands (decide this before staging)

The commit must land on a feature branch cut from the freshly updated default branch, never directly on a shared branch and never silently on whatever branch happens to be checked out.

1. **Find the default branch:** `git symbolic-ref --short refs/remotes/origin/HEAD` (fall back to `gh repo view --json defaultBranchRef`). Shared branches: `main`, `master`, `dev`, `develop`, `trunk`, `production`, `staging`, and the remote default.
2. **Look at the current branch and `git status --porcelain`, then branch by case:**
   - **On the default branch with changes:** the happy path. Update it (`git pull --ff-only`), branch off into a properly named feature branch (format below), and commit there. The commit never lands on the default branch.
   - **On a feature branch already (not the default):** PAUSE and ask what to do. Do not assume the current branch is the right base. Offer the choices: (a) move the work onto a fresh branch cut from the updated default (stash, switch to default, `pull --ff-only`, branch off, pop), or (b) commit on the current branch as is. Wait for the answer.
   - **Default branch can't fast-forward, or detached HEAD:** stop and report. Do not merge, rebase, reset, or force.
3. **Branch name format:** `prefix/issue-number-if-known/short-name`, for example `fix/53/exchange-rate-margin`, or with no known issue `enh/transactions-empty-state`. The prefix is the change type (`fix`, `enh`, `refactor`, `feat`, `docs`, `test`, or another fitting one). Use the name from the args if given.

## Multiple commits, multiple repos

- The work may split into several commits or span several repos. Do not guess the grouping. Show what changed, propose a split (or a single commit), and ask before committing. Each repo gets its own default-branch check and branch-off decision; resolve them one repo at a time.

## Pre-commit scrub of the staged diff

- **Inspect, do not blind-add.** Show `git status` and `git diff --staged`. If nothing is staged, ask what to stage rather than running `git add -A` yourself.
- **Secrets.** Scan for secret-shaped strings: `AIza` (Google), `sk-` (OpenAI), `ghp_`/`gho_` (GitHub), `xox` (Slack), `AKIA` (AWS), and any long hex/base64 blob under a key named `api_key`/`token`/`password`/`secret`/`*_key`. If one is in a tracked file, STOP and surface it. Do not commit. Recommend a placeholder and tell me to rotate the real key. Do not name the leak in the message.
- **Attribution.** Remove any AI/Claude attribution, `Co-Authored-By`, "Generated with" marker, or banned variant from the message and the diff.
- **Contextual comments (do this pass first, it matters more than dashes).** Flag and remove code comments that reference where the change came from (my requests, colleague names, conversations, ticket/PR/issue IDs, "Step 2", or unrelated file references). An irrelevant comment rots the code and misleads future readers; a stray dash is cosmetic. Prioritize this.
- **Dashes (lower priority).** In added prose and comments, rewrite gratuitous em dashes, en-dash punctuation, and stylistic `--` using a colon, parentheses, comma, or period; leave an em dash only where it is genuinely the clearest option. En dash is fine only in numeric ranges. Worth doing, but never let dash scanning crowd out the comment pass above.

## Commit message: conform to the repo's commit validation

A commit that fails the repo's CI commit linter is worse than no commit, so match the repo's rules before writing the message.

1. **Detect the validator.** Look for a commit-message check: a workflow under `.github/workflows/*` that runs `commitlint` (often a "Commit message validation" / `validate` job), or a `commitlint.config.*` / `.commitlintrc*` file. Read the rules it enforces.
2. **If a commitlint config exists, it wins over my defaults.** Satisfy its rules, which for `@commitlint/config-conventional` typically means:
   - **type-enum:** use only an allowed type. config-conventional allows `feat, fix, docs, style, refactor, test, chore, build, ci, revert`; repos often add `enh, enhance, tweak, imp, improve`. My `ui` prefix is usually NOT allowed: fall back to `feat`/`fix`/`refactor`.
   - **subject-case = sentence-case:** only the first character of the subject is uppercase, the rest lowercase. No Title Case, no mid-subject capitals, no uppercase acronyms (reword, or lowercase the acronym, if it trips the rule).
   - **header-max-length:** the whole `type(scope): subject` header must fit. config-conventional defaults to 72, stricter than my usual 80. Honor the configured number.
   - **subject-full-stop:** no trailing period on the subject.
   - **type-case:** lowercase type.
   - **body and footer:** a blank line before the body, and wrap body and footer lines to the configured max (config-conventional 100).
3. **If no validator exists, use my default format:** `prefix(scope): Title` or `prefix: Title`, prefixes `fix, refactor, docs, feat, ui, test`, first letter after `:` capitalized, subject under 80.
4. **Body in both cases:** WHAT changed behaviourally and WHY, not HOW. No file paths, function names, fields, routes, or config keys.

Re-scan the final message for attribution and dash punctuation immediately before `git commit`. Commit only: do not push, do not open a PR.
