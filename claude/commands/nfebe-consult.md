---
description: Search my knowledge base for notes on a topic and summarize what it records
argument-hint: <search terms or question>
allowed-tools: Bash(rg:*), Bash(grep:*), Bash(ls:*), Bash(find:*), Read
---

Search my knowledge base for $ARGUMENTS and report what it already records.

The knowledge base lives at `$HOME/work/knowledgebase`. It is private, local-only notes, not a procedure to cite back elsewhere.

## How to search
- Get the layout first (`ls`/`find` the tree) rather than assuming folder names; the structure is organized by org and topic and changes over time.
- Search the whole tree with ripgrep, case-insensitive, across content and filenames. Run the literal terms first, then obvious synonyms and abbreviations if the first pass is thin.
- Open the most relevant files with Read rather than trusting a single grep line out of context.
- Skip the tracked Claude config and any billing/time-log folder unless the question is actually about those.

## How to report
- Lead with the answer the notes give, then cite each source as `path:line`.
- Quote the relevant lines. Do not paraphrase a fact into something the note does not say.
- If nothing matches, say so plainly. Do not fill the gap with a guess.
- Notes are point-in-time, not live state. Flag anything (resource IDs, versions, file paths, behavior) that must be verified against the live system before acting on it.
