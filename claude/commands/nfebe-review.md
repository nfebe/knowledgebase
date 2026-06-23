---
description: Review a diff against my global rules, citing only what I actually opened
argument-hint: [working | staged | <base>..<head> | PR#]
allowed-tools: Bash(git:*), Bash(gh:*), Read, Grep
---

Review the target diff. Default to the working-tree diff if none is given. $ARGUMENTS

This is a read-only review. No commits, pushes, or PR state changes (no merging, closing, requesting reviewers).

## Evidence discipline (hard rule)

- Cite only files you actually opened and commands you actually ran in THIS session. No fabricated file paths, recalled APIs, or "it probably looks like" shapes.
- When a finding depends on an external contract (a library error/type, an API response, an enum, a function signature), verify it from the real source (`node_modules`, the live API, its docs) and cite where. If you cannot verify, label the finding unverified rather than asserting it.
- Before presenting the findings, run them through `/nfebe-verify`: it is the formal pass for this discipline. Every cited `file:line`, claimed behavior, and external shape gets confirmed against what was actually opened or run this session; tighten over-broad findings and drop or label any that cannot be backed.

## What to check

1. **Correctness.** Real bugs, broken edge cases, wrong assumptions about external shapes.
2. **Tests.** Do they go through the user's boundary (HTTP/CLI/queue/UI), not around the controller? Does a bug fix have a regression test reproducing the user's actual call?
3. **Secrets.** Secret-shaped strings (`AIza`, `sk-`, `ghp_`/`gho_`, `xox`, `AKIA`, long hex/base64 under `api_key`/`token`/`password`/`secret`/`*_key`), especially in example/config files.
4. **Attribution.** Any AI/Claude attribution, `Co-Authored-By`, "Generated with" marker, or robot emoji anywhere in the diff or messages.
5. **Dashes.** Gratuitous em dashes (avoid by default, fine only when clearly the best choice), en-dash punctuation, stylistic `--` in added prose/comments.
6. **Code comments.** Only non-obvious, context-free technical notes survive. Flag comments that reference requests, names, conversations, ticket/PR IDs, plan steps, or unrelated files.
7. **Commit format** (if reviewing commits): `prefix(scope): Title`, allowed prefixes, capitalized, <= 80 chars, body describes WHAT/WHY not HOW.

## Output

Concise findings: lead with the point, `file:line`, severity. Keep each to a few lines. List the highest-impact items first. State what you verified and what you could not.
