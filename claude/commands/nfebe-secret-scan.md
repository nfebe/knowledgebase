---
description: Scan changes and example/config files for secret-shaped strings before commit
argument-hint: [staged | working | <paths...>]
allowed-tools: Bash(git:*), Bash(grep:*), Read
---

Scan for leaked secrets before anything is committed. Default target is the staged diff plus the working-tree diff if none is given. $ARGUMENTS

This is read-only: grep and read only. Never write, commit, or rewrite history on your own.

## Scan scope

- The target diff (added lines especially).
- Every `*.example.*`, `*.sample.*`, and template/default config or env file touched. These are meant to be published, so a real value here is the worst case.

## Patterns

- `AIza` (Google), `sk-` (OpenAI), `ghp_` / `gho_` (GitHub), `xox` (Slack), `AKIA` (AWS).
- Any long hex or base64 blob assigned to a key named `api_key` / `token` / `password` / `secret` / `*_key` / connection string.

## On a hit

1. Report `file:line` and which pattern matched. Do not echo the full secret value beyond what is needed to locate it.
2. If it is in a tracked file, STOP and surface it before any commit.
3. Recommend: replace with an obvious placeholder (`your-api-key-here`, `<REDACTED>`) in the working tree, and tell me to rotate/revoke the real key (that is what actually stops the damage).
4. Do not telegraph the leak in branch names, commit messages, or PR text. Use neutral hygiene framing ("Use placeholder values in example config"); keep "leaked"/"live"/"exposed" out of anything pushed. Explain the real reason to me in chat.
