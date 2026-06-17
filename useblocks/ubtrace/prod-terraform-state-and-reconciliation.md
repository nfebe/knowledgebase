# Prod Terraform state and reconciliation (2026-06-15)

Where the production Terraform lives, why it looked broken, and how it was brought back to a clean plan. Context for anyone running prod IaC on `useblocks/ubtrace` `feat/844-eks-templates`.

## The tooling is OpenTofu, not Terraform

The repo lockfile uses `registry.opentofu.org`. Run `tofu`, not `terraform`. Running HashiCorp `terraform` rewrites the lockfile to the HashiCorp registry and bumps provider patches; do not commit that. The state was written by tofu.

## The state was never lost

The earlier "403 / lost state" was a wrong backend filename, not a missing state.

- Real backend (in the prod account `809338888439`, readable):
  - bucket `ubtrace-terraform-state-prod`
  - key `prod/terraform.tfstate`
  - lock table `ubtrace-terraform-locks-prod`
- The committed `environments/backend-production.hcl` had pointed at `ubtrace-terraform-state` / `production/terraform.tfstate` / `ubtrace-terraform-locks` (no `-prod`). Those names live in a different account and 403 from prod. Fixed in the reconciliation commit.

Init with: `tofu init -reconfigure -backend-config=environments/backend-production.hcl`. Plans need `-var-file=environments/production.tfvars` (not auto-loaded).

## The environment-name gotcha (a careless apply would have rebuilt prod)

Deployed prod was created with `environment = "prod"` (resources named `ubtrace-prod-*`). The branch had drifted `production.tfvars` to `environment = "production"`, and `locals.tf` builds `name_prefix = "${project_name}-${environment}"`. That renames every resource, and names are immutable, so a plan proposed 76 destroys to rebuild the whole stack. Setting it back to `prod` (plus enabling the bastion and ALB toggles that the running cluster has) collapsed the plan to near zero.

## Prod runs a modest tier, not the documented HA spec

The PRD/#936 specifies production as HA/multi-AZ, but the running cluster is the cheaper tier, and the config was reconciled down to match it:

- RDS app and keycloak: `db.t3.small`, single-AZ, 3-day backups (not `db.r6g.large` multi-AZ / 14-day).
- Redis: `cache.t3.micro`, 1 node, no failover (not `cache.r6g.large` x2).
- OpenSearch: off (`enable_opensearch` defaults false; the cluster uses in-cluster Elasticsearch).

Bringing prod up to the HA spec is ~+$1,100/mo and a deliberate, scheduled decision for Marco. Recorded as deferred, not silently dropped.

## The TLS certificate must never be deleted

`module.loadbalancer.aws_acm_certificate.main` is a multi-domain cert (`team` + `api.team` + `admin.team` + `auth.team`.useblocks.com) and is in use by the four `k8s-ubtrace-*` ALBs the Kubernetes ALB controller creates (referenced by ARN). The branch had dropped the domain config, so Terraform wanted to delete it.

Fix: set `domain_name` and `subject_alternative_names` to match the live cert exactly (a no-SAN cert forces replacement), and add `lifecycle { prevent_destroy = true }` so a future config change hard-errors instead of deleting it. A stale deposed cert from a prior failed apply was unused (`InUseBy` empty) and cleaned up.

## EFS is on elastic throughput

Switched `module.storage.aws_efs_file_system.main` from bursting to elastic via a new `efs_throughput_mode` variable. Removes the burst-credit exhaustion that throttled the filesystem (credits were 0 for ~10 days and made the catalogue walk read 149s). Un-throttled the same walk is 11s cold / 6s warm across 6,354 dirs, still multi-second, so the catalogue materialization work (#1526) is still required.

## Current plan state

After reconciliation the plan is clean except one in-place diff on the audit-logs bucket encryption config, a provider-version artifact under tofu/aws 6.38 (absent under newer providers), unrelated to the reconciliation. Benign; reconcile when convenient.
