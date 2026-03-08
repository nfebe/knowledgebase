# RunsOn Custom AMI Image Resolution

## Problem

When migrating GitHub Actions workflows to use [RunsOn](https://runs-on.com) self-hosted runners with custom AMI images (e.g. `ub-runner`), jobs fail at the "Set up runner" step with:

```
Failed to launch runner: failed to resolve image spec: image ub-runner not found in default images or repo config
```

This happened on [needs-large-scale PR #25](https://github.com/useblocks/needs-large-scale/pull/25) — the workflow used `image=ub-runner` in the `runs-on` label but RunsOn couldn't resolve the image.

## Root Cause

RunsOn resolves custom image names from a `.github/runs-on.yml` file in each repository. The custom AMIs (`ub-runner`, `ub-runner-windows`) are built via Packer and defined in [ubinfra PR #61](https://github.com/useblocks/ubinfra/pull/61), but each repo that uses them must have its own `.github/runs-on.yml` mapping image names to AMIs.

The `ubtrace` repo had this file and worked fine. The `needs-large-scale` repo was missing it.

## Fix

Add `.github/runs-on.yml` to the repository:

```yaml
images:
  ub-runner:
    platform: linux
    arch: x64
    owner: '595319716151'
    name: 'ub-runner-linux-*'
  ub-runner-windows:
    platform: windows
    arch: x64
    owner: '595319716151'
    name: 'ub-runner-windows-*'
```

## Key Takeaways

- Every repo using custom RunsOn images needs its own `.github/runs-on.yml`.
- The `owner` field is the AWS account ID that owns the AMIs.
- The `name` field uses a glob pattern to match the latest AMI (e.g. `ub-runner-linux-*`).
- RunsOn launches the EC2 instance first, then fails during runner setup if the image can't be resolved — so you'll see the runner spin up but the job still fails.

## Related

- ubinfra PR #61: Adds custom AMI config (Packer builds, Terraform, docs)
- Repos that need this file: ubdocs, ubtrace, ubcode, ubconnect, needs-large-scale, and any future repo using RunsOn custom images
