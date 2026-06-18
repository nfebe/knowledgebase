---
description: Draft or clean a GitHub issue to my style (conventional-commit title, terse Task/Action/Acceptance)
argument-hint: [topic or rough notes | <path to draft> | pasted draft]
allowed-tools: Bash(gh:*), Bash(git:*), Read, Edit
---

Draft a GitHub issue (or clean the one given) to my style. $ARGUMENTS

This command writes the draft and shows it. It does not run `gh issue create` unless I explicitly say to.

## Main rules

1. **Title: conventional-commit form**, like my commits: `prefix(scope): Capitalized title`, under 80 chars, no `[Tag]` prefix. Prefixes: `fix`, `refactor`, `docs`, `feat`, `ui`, `test`; for a maintenance/dependency/infra upgrade use `chore` or `build`. Scope is the component (e.g. `runs-on`). Example: `chore(runs-on): Upgrade from v2 to v3`.
2. **Body is terse and factual, not an article or a conversation.** Structure, matching what the target repo already uses:
   - `## Task` (or `## Problem`): one tight paragraph leading with the concrete point. Add `Constraint: ...` on its own line for the one fact that shapes the work.
   - `## Action`: plain `- [ ]` checkboxes, one line each.
   - `## Acceptance criteria`: how we know it is done.
   - `## Verification` (optional): how to prove it end to end.
3. **Minimal but complete.** Every line is information needed to start work. Cut anything that is not. Do not pad to look thorough.
4. **Leave out (these are not issue content):**
   - Colleague names or quoted messages ("Marco said...", a pasted Slack line). State the task, not who reported it or how.
   - The originating outage/incident/conversation when the task itself is the deliverable. "Upgrade X to v3" does not need the outage story.
   - Process/correctness metadata: no "Verified facts", no "(verified this session)", no notes on how I checked. Verifying is for my correctness, not the reader. See the global rule "Process metadata stays out of deliverables".
   - Narration: state a constraint as `Constraint: ...`, never "the one non-obvious thing driving the whole plan is...".
5. **Writing hygiene** (global rules): no em dashes or en-dash/`--` punctuation in prose, never the acronyms "DAG" or "DoD" (spell them out), no AI/Claude attribution anywhere.

## How to work

- Before drafting, look at 1-2 recent issues in the target repo (`gh issue view <n> --json title,body`) and match their structure and voice.
- If the input is rough notes, turn them into the structure above. If the input is an existing draft, edit it in place to these rules and report each change.
- Show the final title and body. If I then say to create it, run `gh issue create --title ... --body-file ...` against the named repo.
