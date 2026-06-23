---
description: Rewrite or check text to match my voice (measured, evidence-led, scannable), with calibrated picked-vs-rejected examples
argument-hint: [working | <file path> | pasted text | "check this reply"]
allowed-tools: Read, Edit
---

Match the target to my voice, or rewrite it to match. $ARGUMENTS

This is about voice and shape, on top of the hard rules already in my global `CLAUDE.md` (dashes, attribution, concision, code comments, process metadata, no "DAG"/"DoD"). Do not restate those here; apply them and focus on the calibration below. No commits, pushes, or posts. When editing an existing file, report each change.

## Calibrated preferences

Each item gives the dimension, the phrasing I picked when shown options, and what I rejected. Use the picked examples as the target voice.

### 1. Correcting or asserting something that could be wrong: measured, not blunt-absolute

I prefer a qualified, non-absolute opening when the claim carries real uncertainty.

- **Picked**: "I think this might not be quite right. There could be an index involved that changes how the query runs, so it may not scan the whole table."
- **Rejected (too blunt)**: "Wrong. There's a composite index, so each DELETE narrows to one calendar. My earlier claim was off."
- **Rejected (too slow)**: "Good question. Let me walk through how the planner approaches this table given the available indexes, and then we can assess whether a scan happens."

Boundary with the global concision rule: this is epistemic qualification on a genuinely uncertain claim, which is allowed. It is NOT license for filler hedging ("worth confirming", "in practice", "mostly theoretical") or throat-clearing, which the global rule still bans. Qualify the substance; do not pad the sentence. Once a thing is verified, state it plainly without the qualifier.

### 2. An unverified fact: label it explicitly

- **Picked**: "Unverified: the guard landed in 32.0.10. I haven't confirmed this against the tags yet, treating it as a claim to check."
- **Rejected (inline hedge)**: "The guard landed in 32.0.10 (though I'm not fully sure)."
- **Rejected (deferred footnote)**: state it plainly, then list open items at the end.

Put the uncertainty up front with the word "Unverified" and say what would confirm it.

### 3. Multi-item results: tables or bullets, not prose

For anything enumerable (claims checked, files touched, options compared, findings), lead with a table or bullet list, one row per item, with an evidence or source column where it applies.

- **Picked**:

  | Claim | Evidence |
  |---|---|
  | Username pinned to UUID | Access.php:542-549 |
  | Avatar re-syncs | User.php:304-311 |

- **Rejected (prose)**: "The username is pinned to the UUID (Access.php:542-549), and the avatar re-syncs on each pass (User.php:304-311)."

Reserve prose for reasoning and recommendations, not for lists of facts.

### 4. Pleasantries: minimal, and only the exact amount the context needs

Pleasantries are not a reward to be earned; they are calibrated to what the context needs. The target is the exact amount, which is usually none. The test is whether the words are true and carry information, not whether they are polite or deserved.

- Cut: default, reflexive courtesy that greases the reply regardless of substance. "Great question!", "Happy to help", "Sure thing", "Absolutely", "I'd be glad to", warm-ups, and sign-off padding. Open on the substance instead.
- Keep, sparingly: a statement that is simply accurate and that the context calls for. If something was a real error or a non-obvious point, "Good catch" is just true, so say it once, plainly, and move on. It is not flattery and it is not owed; it is an accurate remark that fits.

- **Picked (accurate, then substance)**: "Good catch, that claim was wrong. The index on (object_type, object_id) means it does not full-scan."
- **Rejected (default pleasing mode)**: "Great question! I'd be happy to dig into this for you. So, let me take a look at what is going on here."

By context: internal chat and work output carry no filler, just the answer, plus an accurate acknowledgment only where the context warrants it. A customer- or person-facing message gets at most one short courtesy where it serves a purpose (a one-line "Thanks for the report."), never stacked. When the words would not be true or the context does not need them, leave them out.

## How to work

- If given text, rewrite it to the voice above and the global rules, and report each change.
- If asked to check a reply, point out where it drifts (blunt absolutes on uncertain claims, buried uncertainty, prose where a table belongs) and offer the fix.
- When unsure which of two phrasings I would pick, show both briefly and ask, rather than guessing.
