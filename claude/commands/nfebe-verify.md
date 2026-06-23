---
description: Verify drafted claims against evidence gathered this session, flag fabrications, assumptions, and unverified statements, then correct them
argument-hint: [a note/draft/file | "this session's output" | pasted claims]
allowed-tools: Bash(git:*), Bash(rg:*), Bash(grep:*), Bash(ls:*), Bash(find:*), Bash(gh:*), Read, Edit, WebFetch
---

Fact-check the target and scrub out anything not backed by evidence gathered THIS session. $ARGUMENTS

Default target, if none given, is the substantive claims in the output I just produced (a ticket note, an issue/PR body, a chat answer, a diagnosis). The aim is narrow and absolute: catch hallucinations, false claims, over-broad statements, and assumptions before they ship. This command may read, grep, fetch, and edit to correct; it does not commit, push, or post.

## The gate (apply to every claim)

For each factual claim in the target, ask: "Is this true, and how do I know?" If the honest answer is "I think", "usually", "it should", or "from memory", that is not evidence. Go confirm it from the real source this session, or mark it unverified. Memory and training data are not evidence.

Claims that need a source include: a file path, a `file:line`, quoted code, a function or method signature, a version number, a release date, a changelog entry, an API or error-response shape, a config or env key, an enum or constant, a CLI flag, a behavior ("X happens when Y"), and any root-cause attribution.

## What to do per claim

1. **Verify from the real thing.** Read the file, run the command, fetch the doc. Cite where: `file:line`, command output, or a doc URL with the quoted line. For external contracts (a library's type/error, an API response, a returned shape), read the installed source (`node_modules/...`) or trigger a real instance. Never infer the shape from "what it probably looks like".
2. **Check scope.** Is the statement broader than the evidence supports? "Never re-syncs" when a separate value does re-sync, "always", "every", "none" are red flags. Tighten the claim to exactly what the evidence shows.
3. **Check relevance.** Does the claim actually bear on the question being answered? Drop tangential facts that pad the answer without changing the conclusion.
4. **Check attribution.** When a fact comes from a person, a commit, or a doc rather than from the behavior itself (for example "fixed in version N"), name that source and treat it as a reported claim, not an independently traced fact.

## Verdicts

Assign each claim one of:

- **Verified** with the evidence inline (`file:line`, command, or URL).
- **Corrected**: the claim was wrong or over-broad; state the accurate version and the evidence, then fix it in the target.
- **Unverified**: could not confirm without a step not taken (or a destructive step not approved). Label it as such in the target rather than asserting it. Phrase it explicitly, for example: "Unverified: the guard landed in 32.0.10. I haven't confirmed this against the tags yet."
- **Remove**: fabricated, irrelevant, or unsupportable; cut it.

## Output

1. A scannable table, one row per claim: `Claim | Verdict | Evidence`. Highest-risk claims first.
2. Apply the corrections and removals to the target (when it is a file or draft I can edit), and report each change.
3. A one-line bottom line: what held, what was corrected, what stays unverified.

Do not soften a fabrication into "roughly right". If it was not verified, say so plainly.
