---
description: Write or run tests that exercise the same boundary a real user hits
argument-hint: <feature or bug to cover>
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

Write or run tests for: $ARGUMENTS

Follow my testing discipline strictly.

## Drive through the user's boundary

- Identify how the feature is actually reached: HTTP request, CLI command, queue job, UI event. Set up state and take actions through THAT boundary, not by calling internal models/services that bypass the controller, validation, allowlist, serialization, or middleware. Going around the user's path hides 422s/500s while the test stays green.
- **Bug fix:** the regression test must reproduce the user's actual call. If you cannot reproduce the bug through the user's path, you have not proven the fix: say so.
- **Pure internal collaborators** (parsers, calculators, functions with no user-facing path): test directly.
- **State only written by an internal code path** (a counter the server increments, never the user): seed it the same way production writes it. The principle is "match the production write path", not "always HTTP".

## Never fabricate external shapes

- Any test input or mock that stands in for something I do not own (a library error/exception, an API/HTTP response, a returned type, an enum/constant, an event payload, a DB row, a config/schema field, a function signature) must be a REAL sample: captured from the actual runtime, or copied from the dependency's types/fixtures in `node_modules` or its source. Cite where you read the shape.
- If you cannot obtain a real sample, say so and mark the test as unverified against reality rather than asserting a guess against a guess.

## Pass the project's real checks, not just your own tools

- Passing tests is not enough. The change must also pass the project's lint, format, and static analysis. Run the SAME checks CI runs, not just your own auto-formatter. A local `pint` or `eslint --fix` going green does NOT mean `phpcs` or `prettier --check` passes: line-length limits, whole-repo format checks, and stricter sniffs differ and will fail in CI.
- Discover the real commands from the repo, do not guess them: read the CI workflow files under `.github/workflows`, then the `composer.json` scripts (for example `phpcs:test`, `test`) and `package.json` scripts (for example `lint`, `format:check`, `test`). Run exactly those.
- If a check only runs inside a container or you cannot run it locally, run it there, or state plainly which check you could not verify. Never assume green.

## Run and report

Run the tests through the real runner and report the actual output. If they fail, show the failure: do not gloss a red result as done.
