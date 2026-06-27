# runs-on v2 to v3 migration and v2 teardown

Captured 2026-06-27. CI for the useblocks GitHub org runs on the third-party runs-on service in AWS account `595319716151` (CI). The v2 stack was replaced by a Terraform-managed v3 stack, and v2 was torn down on 2026-06-27.

## Accounts and profiles

- CI account `595319716151`. Two SSO profiles: `ci` (the `ub-dev` role, blocked from `iam:*`) and `ci-admin` (the `ub-admin` role). Stack changes that create IAM/ECS/API Gateway need `ci-admin`. `ubtrace-prod-admin` is a different account (`809338888439`) and cannot reach CI.

## v3 stack (current, live)

- Terraform at `ubinfra/deploy/terraform/runs-on-v3`, the runs-on `runs-on/runs-on/aws//modules/flex` module v3.1.1, Flex (webhook-driven) model. VPC `10.2.0.0/16`, tag `stack=runs-on-v3`. State backend S3 `ubinfra-tfstate-595319716151`, key `ubinfra/runs-on-v3/terraform.tfstate`, native S3 locking. Use `terraform` (1.14.x), not tofu, for this stack.
- The AMI rebuild pipeline and the SG/NACL hardening discover targets by tag `stack=runs-on-v3`, no hardcoded IDs.

### Applying the stack

The `runs-on-v3.yml` workflow (manual dispatch, plan/apply) is **not usable yet**: it needs repo settings that are not provisioned (`RUNS_ON_V3_DEPLOY_ROLE_ARN` var, `RUNS_ON_LICENSE_KEY` secret, `RUNS_ON_ALERT_EMAIL` var). Until those exist, apply locally:

```
cd ubinfra/deploy/terraform/runs-on-v3
aws sso login --profile ci-admin
AWS_PROFILE=ci-admin terraform init -backend-config=backend.hcl
TF_VAR_license_key="$(cat ~/.runs-on-license)" TF_VAR_email="security@useblocks.com" \
  AWS_PROFILE=ci-admin terraform plan   # then apply
```

The license key lives at `~/.runs-on-license` on the operator machine; the budget/alert email is `security@useblocks.com`. Verify these against the live stack before relying on them.

## Two gotchas worth knowing

1. **The stack fights the SG hardening over public SSH.** The Flex module defaults `ssh_allowed = true`, which creates a `0.0.0.0/0:22` ingress on the runner security group. The SG hardening removes that rule for Vanta/ISO. So every untargeted `terraform apply` re-opened public SSH until hardening re-ran. Fix: set `ssh_allowed = false` on the module block (PR #188). Then the stack matches the hardened state. To debug-SSH later, set `ssh_cidr_range` to a bastion/office CIDR, never public. A budget-only change had to be applied with `-target` to avoid this re-opening.

2. **Daily cost alert threshold.** `app_budget_daily_usd` was `10`, which tripped on normal CI usage. Raised to `50` (PR #187, applied live 2026-06-26). The value flows into the published stack config, so changing it does a brief control-plane redeploy (replaces the stack-config materializer + ECS task def, updates the lambda + ECS service).

## v2 teardown (2026-06-27)

- v2 was a single console/CloudFormation stack named `runs-on` (App Runner entrypoint, VPC `10.1.0.0/16`, 11 SQS queues, 2 DynamoDB tables, IAM, networking). Not in Terraform.
- Confirmed safe first: all 65 stack resources were v2-scoped (no v3 overlap), and CI runs on v3.
- Deleted via `cloudformation delete-stack`. The VPC delete first failed on a leftover **Packer security group** (`packer_<uuid>`, orphaned from a pre-cutover AMI build in the v2 VPC); deleting that SG unblocked it. The AMI pipeline builds in v3's subnet, so it was unaffected.
- The 3 S3 buckets (`runs-on-s3bucket-*`, `-cache-*`, `-logging-*`) are **versioned and were auto-retained** (CFN `DELETE_SKIPPED`) as a safety net. They are now orphaned (in no stack); purge when confident nothing is needed.
- Backups of the v2 stack template and resource list were saved locally at teardown time.
