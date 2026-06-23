---
description: Triage a Nextcloud support ticket - verify context, find the root cause in code or research the product question, write a structured ticket note, draft a customer response, then commit and push
argument-hint: <ticket# and description, or pasted ticket text>
allowed-tools: Bash(git:*), Bash(gh:*), Bash(rg:*), Bash(grep:*), Bash(ls:*), Bash(find:*), Bash(cat:*), Bash(basename:*), Read, Write, Edit, WebSearch, WebFetch
---

Triage the Nextcloud support ticket described in $ARGUMENTS: investigate it, record a structured note in the `nc-tickets` repo, draft a customer response, then commit and push. The ticket may be a code bug or a product/feature/availability question; handle both.

The ticket notes repo is `nc-tickets` (git remote `nfebe/nc-tickets`, branch `master`), a sibling of the Nextcloud `server` checkout under the workspace. Find it rather than assuming its path.

## 1. Locate the repos and confirm the target

- Find the `nc-tickets` checkout. Run `git remote get-url origin` in it and confirm it ends with `nfebe/nc-tickets.git`. If it does not, STOP: this is not the tickets repo, do not write or push.
- Find the `nextcloud/server` checkout (often a `server/` subdir of the workspace) and note the `docker-dev` workspace, for code investigation.
- `git -C <nc-tickets> status --porcelain` and `git branch --show-current` so you know the starting state.

## 2. Parse the ticket

Extract: ticket number, severity (use the customer's number if given; the scale runs 1 highest to 4 lowest, where 4 is a question / advice / documentation / feature request), customer name and contact if present, the affected component/topic, the reported Nextcloud version, and the actual problem or question. If any of these is missing, leave it blank in the note rather than inventing it.

## 3. Investigate - verify, never assume

This is the core. Every fact in the note must be backed by evidence gathered this session, not memory.

- **Code bug**: reproduce or trace it in the `server` codebase. `rg`/`grep` for the symptom, read the responsible files, and identify the root cause with concrete `file:line` evidence. Quote the relevant lines in the note. Do not propose a fix you have not located in the code.
- **Product / feature / version / availability question**: research official Nextcloud sources and cite every claim with a URL and a quoted line. Prefer `nextcloud.com/blog`, `docs.nextcloud.com`, the release-schedule wiki (`github.com/nextcloud/server/wiki/Maintenance-and-Release-Schedule`), and the relevant app's GitHub releases (`github.com/nextcloud-releases/<app>`). Map any seasonal Hub release name to its server major version and date from the schedule wiki.
- When the search space is wide, delegate the sweep to an Explore agent and have it return findings with source URLs.
- Never fabricate an external shape: a version number, changelog entry, API response, config key, error string, or release date must come from a source you actually read this session. If you cannot verify something, say so in the note and mark it unverified rather than stating it as fact.

## 4. Classify

Record severity, topic/component, and type (bug, question, or feature request). For a bug, note rough fix risk and whether it is version-specific.

## 5. Write the ticket note

Create `<nc-tickets>/<number>-<short-slug>.md`, matching the format of existing notes in the repo (read one first). Use this skeleton:

- A metadata block: Created (today's date), Severity, Topic, Customer, Assignee, NC Version context, and (for bugs, once they exist) Fix branch / Fix commit / PR.
- `## Summary`: the problem or question and the short answer.
- For a bug: `## Reproduction`, `## Root Cause` (with `file:line` and quoted code), `## Fix`. For a question: `## Findings` with each fact and its source URL.
- `## Verification`: how the answer or fix was confirmed.
- `## Resolution`: the outcome that resolves the ticket and the current state of the conversation (answered and awaiting review, pending customer info, fix in progress, escalated).
- `## Customer Response Draft` (optional, keep short): the key points the human should convey, stated as verified facts with sources where useful and no fabricated versions or dates. This is a starting draft to send, not an archive. Do not store a transcript of every message exchanged; the note's value is the resolution and findings, not a copy of the replies.
- `## Action Items` (checkbox list) and/or `## Status`.
- `## References`: the source URLs and any code paths.

The note records what resolved the ticket and the findings behind it, plus where the conversation stands. It is not a log of the exact text sent to the customer.

Follow the writing rules: avoid em dashes (use one only when genuinely clearest), no en-dash punctuation, no `--` in rendered prose; no AI/Claude attribution anywhere; no contextual comments referencing this conversation, issues, or colleagues; plain direct facts, no process narration.

## 6. Verify before committing (`/nfebe-verify`)

Run `/nfebe-verify` over the finished note and the customer response draft before staging anything. Every `file:line`, quoted snippet, version, release date, config key, and behavior claim must be backed by evidence gathered this session, not memory. Tighten any over-broad statement, drop tangential claims, name the source behind any "fixed in version N" type fact, and label anything you could not confirm as unverified. Apply the corrections to the note and draft, then continue.

## 7. Commit and push

- Stage only the new ticket file (and its attachments dir if you created one). Do not sweep up other untracked notes in the repo.
- Commit in the repo's format: `docs: <Title>` (capitalized, under 80 chars, describing what was recorded). No attribution.
- `nc-tickets` is a personal repo whose normal flow is direct to `master`, and recording-and-publishing is this command's purpose, so pushing to `master` here is authorized. Still print `git log origin/master..HEAD --oneline` before pushing.
- `git push`. If the output shows any protected-branch warning ("Bypassed rule violations", "changes must be made through a pull request", or similar), STOP and surface it.

## 8. Report

One short summary: the note path, the classification (severity / topic / type), the root cause or answer in a line, that the verify pass ran (and what it corrected, if anything), the commit subject, and that the push succeeded. Then show the Customer Response Draft so it can be reviewed before sending.
