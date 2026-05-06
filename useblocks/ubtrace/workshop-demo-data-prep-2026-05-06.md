# ubTrace workshop demo data prep (2026-05-06 Mercedes workshop)

Captured 2026-05-06 from the session that prepped demo data for the ubTrace version-trend feature. Source repo: `useblocks/sphinx-needs-demo`. Result: four new releases (v0.1.2 through v0.1.5) on top of the existing v0.1.0 / v0.1.1, giving the trend chart visible movement.

## Goal

ubTrace's version-trend feature was being demoed at the Mercedes workshop. Before this session, only three versions existed in the dashboard (`main`, `v0.1.0`, `v0.1.1`). The diff between v0.1.0 and v0.1.1 was a single feature trace by Max Pabinger (PR #47, "traffic sign recognition v-model trace"). Marco asked for "3-4 more" tags so the chart would actually move.

## What was added

| Tag | PR | Type | What |
| --- | --- | --- | --- |
| v0.1.2 | #52 | New V-model trace | Blind Spot Monitoring (NEED_006) |
| v0.1.3 | #53 | Field churn only | Status flips on existing items + one regression |
| v0.1.4 | #56 (replaced #54) | New V-model trace | Driver Drowsiness Detection (NEED_007) |
| v0.1.5 | #55 | New V-model trace | Automated Parking Assist (NEED_008) |

Two design choices to make the dashboard story richer:

1. **Author rotation** across PRs: PR 1 used SARAH/ALFRED, PR 3 used STEVEN, PR 4 used SARAH again, so the "who" dimension also moves between versions. The personas come from `docs/automotive-adas/persons.rst` (PETER, ALFRED, ROBERT, SARAH, STEVEN, THOMAS).
2. **One regression in v0.1.3**: TEST_INT_007 went `passed -> failed` to demonstrate that ubTrace can show negative trends, not just monotonic growth.

## Recurring lessons

### ubTrace sorts versions alphabetically, not by semver

Marco's note: as of 2026-05-06 the dashboard sorts version tags alphabetically. For the four new tags this is fine because v0.1.0 through v0.1.5 are equal-length strings and alphabetic order matches semver order.

The trap kicks in the moment a single-digit boundary is crossed:

- `v0.1.10` sorts BEFORE `v0.1.2` alphabetically.
- `v0.10.0` sorts BEFORE `v0.2.0`.

For demo data, either stay under 10 in each component, OR zero-pad (`v0.01.02`, etc.). Better long-term fix is on the ubTrace side (parse semver, not strings).

### Stacked PRs: do NOT delete the base branch before retargeting

This bit us with PR #54.

Setup: PRs #54 and #55 were stacked. #54 had base `feat/blind-spot-monitoring` (PR #52's branch). #55 had base `feat/driver-drowsiness-detection` (PR #54's branch). When #52 merged into main, GitHub did not auto-retarget #54 to main, even though `feat/blind-spot-monitoring` was now redundant.

Cleanup attempt: I deleted the remote `feat/blind-spot-monitoring` branch via `git push origin --delete`. Result: GitHub **closed** PR #54 (because its base branch no longer existed) instead of auto-retargeting it. Recovery from a closed PR with a deleted base is hard: you cannot reopen it, and you cannot edit the base of a closed PR. We had to abandon #54 and open #56 with the same branch and base `main`.

Correct order for stacked PR cleanup after the parent merges:

1. Retarget the dependent PR's base to `main` first (`gh pr edit <num> --base main`, or via REST PATCH on `/pulls/<num>` if GraphQL is flaking).
2. Rebase the dependent branch onto main locally (`git rebase main`). Git skips already-applied commits, so only the new commit remains.
3. Force-push with `--force-with-lease`.
4. Only THEN delete the now-stale parent branch.

### gh GraphQL flake fallback

`gh pr edit --base` failed with transient GraphQL errors during the session ("Something went wrong while executing your query"). The REST API equivalent worked first try:

```sh
curl -s -X PATCH \
  -H "Authorization: token $(gh auth token)" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/useblocks/sphinx-needs-demo/pulls/<num> \
  -d '{"base":"main"}'
```

## Pattern for adding a V-model trace in sphinx-needs-demo

Mirrors PR #47. One feature trace touches 11 files:

```
docs/automotive-adas/sys_1_req_elicitation.rst   # NEED_xxx        (author: ROBERT)
docs/automotive-adas/sys_2_req_analysis.rst      # REQ_xxx         (author: ROBERT)
docs/automotive-adas/sys_3_sys_arch.rst          # ARCH_xxx + UML  (author: ALFRED)
docs/automotive-adas/sys_4_sys_integation_tests.rst  # TEST_SYS_INT_xxx
docs/automotive-adas/sys_5_sys_quali_test.rst    # TEST_SYS_QUAL_xxx
docs/automotive-adas/swe_1_sw_req_analysis.rst   # SWREQ_xxx       (author: SARAH or STEVEN)
docs/automotive-adas/swe_2_sw_arch.rst           # SWARCH_xxx + UML
docs/automotive-adas/swe_5_sw_integration_tests.rst  # TEST_INT_xxx (author: THOMAS)
docs/automotive-adas/swe_6_sw_quali_tests.rst    # TEST_QUAL_xxx
src/automotive_adas.py                           # IMPL_xxx (Python class with .. impl:: docstrings)
src/automotive_adas_tests.py                     # TEST_xxx (unit tests with .. test:: docstrings)
```

ID counters as of v0.1.5 (next free numbers):

| Series | Last used | Next |
| --- | --- | --- |
| NEED_ | 008 | 009 |
| REQ_  | 016 | 017 |
| ARCH_ | 010 | 011 |
| TEST_SYS_INT_ | 010 | 011 |
| TEST_SYS_QUAL_ | 009 | 010 |
| SWREQ_ | 027 | 028 |
| SWARCH_ | 010 | 011 |
| TEST_INT_ | 010 | 011 |
| TEST_QUAL_ | 009 | 010 |
| IMPL_ | 024 | 025 |
| TEST_ (unit) | 024 | 025 |

Total LOC per trace: ~240 insertions.

## Release workflow that worked

After each PR merges to main:

```sh
gh release create vX.Y.Z --target main --title vX.Y.Z \
  --generate-notes --repo useblocks/sphinx-needs-demo
```

The tag lands on the merge commit. ubTrace's pipeline picks it up on its next sync.

Repo settings observed: no branch protection on `main`, all merge methods allowed (`allow_merge_commit/squash/rebase` all null = repo defaults), `delete_branch_on_merge` not set. PR #47 used a regular merge commit, so this session followed that style for consistency.
