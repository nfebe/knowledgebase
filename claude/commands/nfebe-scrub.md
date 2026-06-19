---
description: Clean a diff, message, or drafted text to my writing and hygiene rules
argument-hint: [working | staged | <file path> | pasted text]
allowed-tools: Bash(git:*), Read, Edit
---

Clean the target to my rules. Default target is the working-tree diff if none is given. $ARGUMENTS

This command only cleans text and files. It does not commit, push, or post anything.

## What to find and fix

1. **AI/Claude attribution.** Remove any signature, "Generated with" / "via Claude" / "with Claude" / "Co-Authored-By" marker, robot emoji, or anthropic/claude.com link, in any casing or reworded variant. Applies to commit messages, PR/issue text, comments, docs, READMEs, drafted Slack/Teams messages.
2. **Dashes as punctuation.** Em dashes (`—`): avoid, rewrite by default; leave one only where it is genuinely the clearest option. En-dash (`–`) punctuation: rewrite (en dash stays only in numeric ranges like `2024-2026`). Stylistic `--`: rewrite. Prefer recasting the sentence with a colon, semicolon, comma, period, or parentheses. In Markdown/plain text, `--` is a last resort only if rewriting genuinely hurts clarity; never in rendered prose.
3. **Contextual code comments.** Remove comments that reference origin: my requests/feedback, people's names, conversations, ticket/PR/issue IDs, plan steps ("Step 2"), or references to unrelated files. Keep only comments that teach a reader with zero outside context something non-obvious about that line.
4. **Concision** (for prose targets). Lead with the point. Cut hedging, throat-clearing, meta-narration, and diff restatement. If a passage runs long, halve it.

## When editing an existing document
Fix every offending dash and attribution line you encounter, not only the ones I added. Report each change you made (what and where).
