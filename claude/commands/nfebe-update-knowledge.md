---
description: Record a knowledge-base note, sync the global Claude config, then commit and push
argument-hint: [what to record... | --sync-only]
allowed-tools: Bash(git:*), Bash(cp:*), Bash(ln:*), Bash(ls:*), Bash(diff:*), Bash(readlink:*), Bash(find:*), Bash(basename:*), Read, Write, Edit
---

Capture a note into my knowledge base, make sure the global Claude config tracked in the repo matches what is live on this machine, then commit and push. The knowledge base lives at `~/work/knowledgebase` (git remote `nfebe/knowledgebase`, branch `main`). $ARGUMENTS

If the only argument is `--sync-only`, skip the note step and just run the sync, commit, and push.

## 1. Confirm the target repo

- `cd ~/work/knowledgebase`. Run `git remote get-url origin` and confirm it ends with `nfebe/knowledgebase.git` (or `.git`). If it does not, STOP: this is not the knowledge base, do not write or push anything.
- Run `git status --porcelain` and `git branch --show-current` so you know the starting state and branch.

## 2. Record the note

- Decide what to capture: use the argument text, or if it is vague, the relevant facts established in this session. Confirm the gist in one line before writing if it is ambiguous.
- Pick the right location under the existing structure. Do not invent a new top-level folder unless the note clearly belongs to a new org or topic:
  - `useblocks/` for useblocks work, with the existing sub-folders (`ubinfra/`, `ubtrace/`, ...). Create a new sub-folder only for a genuinely new repo or topic.
  - `mac/` for machine setup notes.
  - `claude/` only for Claude config itself (handled by the sync step below, not by hand).
  - A new top-level folder named after the org for a new company.
- Keep billing out of technical notes: `useblocks/billing/` is invoice-grade time logs only. Never drop technical notes there, and never put billing detail into a technical note.
- If the note introduces something major (a new top-level org folder or a new major sub-folder), update the index to match: the `Structure` tree in the top-level `README.md` lists every org and major folder. Add the new entry with a one-line description and keep the tree accurate. The index tracks folders, not individual note files.
- Append to the most relevant existing file rather than creating a near-duplicate; create a new file only when no existing one fits. Match the surrounding note style. Follow my writing rules: no em dashes, no en-dash punctuation, no `--` in rendered prose, and never use the acronyms "DAG" or "DoD" (spell them out).
- Remember this is a private, local-only knowledge base: it is notes, not a procedure to cite back to me elsewhere.

## 3. Sync the global Claude config into the repo

The repo is the version-controlled mirror of my live Claude config. Two things can drift and must be reconciled before committing.

### 3a. Global `CLAUDE.md`

- The live file is `~/.claude/CLAUDE.md` (a real file, edited in place by the harness). The tracked copy is `~/work/knowledgebase/claude/CLAUDE.md`.
- `diff ~/.claude/CLAUDE.md ~/work/knowledgebase/claude/CLAUDE.md`.
  - Identical: nothing to do.
  - They differ: the live file is normally the source of truth, so copy live into the repo: `cp ~/.claude/CLAUDE.md ~/work/knowledgebase/claude/CLAUDE.md`. But first show me the diff and which direction you intend. If the repo copy is clearly the newer/intended one (for example I just edited it here), copy the other way instead. Never silently discard the side with content I want; when unsure, show the diff and ask.

### 3b. Command files

- The canonical commands live in `~/work/knowledgebase/claude/commands/*.md` and are symlinked into `~/.claude/commands/`. Editing a tracked command flows to live automatically, so no copy is needed for those.
- Adopt any command that exists live but is not yet tracked: for each real (non-symlink) `*.md` in `~/.claude/commands/` (skip `README.md`), move it into `~/work/knowledgebase/claude/commands/` and replace the live file with a symlink:
  - `mv ~/.claude/commands/<name>.md ~/work/knowledgebase/claude/commands/<name>.md`
  - `ln -s ~/work/knowledgebase/claude/commands/<name>.md ~/.claude/commands/<name>.md`
- Ensure every tracked command is linked live: for each `*.md` in the repo commands dir (skip `README.md`), if `~/.claude/commands/<name>.md` is missing, create the symlink with `ln -sf`.
- Report any dangling symlink in `~/.claude/commands/` (points at a repo file that no longer exists) but do not delete it without asking.
- If you added or removed a command, update the command list in `~/work/knowledgebase/claude/commands/README.md` to match.

## 4. Commit and push

- `git add -A` in the knowledge base, then show `git status --short` and a short summary of what changed (the note, and any config sync).
- Commit in my format: `docs: <Title>` (capitalized first letter, title under 80 chars, describing what was recorded or synced). No AI/Claude attribution anywhere in the message or body. Body, if any, says what and why, not file paths.
- This command's whole purpose is to publish, and this personal repo's normal flow is direct to `main`, so pushing to `main` here is authorized. Still print the target branch and `git log origin/main..HEAD --oneline` before pushing so I see exactly what goes up.
- `git push`. If the output shows any protected-branch warning ("Bypassed rule violations", "changes must be made through a pull request", or similar), STOP and surface it: the push was not the simple personal-repo push expected.

## 5. Report

- One short summary: what note was recorded and where, whether `CLAUDE.md` or any command was synced, the commit subject, and that the push succeeded (or what blocked it).
