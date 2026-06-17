# GitHub activity — May 2026 (useblocks)

Scope: nfebe activity across the useblocks org for May 2026. Most work was deployment and issue-driven, not commit-heavy; the deployment record lives in the ubinfra `values-team.yaml` overlay bumps.

## Prod deployments to team.useblocks.com

Each version bump to the prod overlay corresponds to a Helm upgrade of the live `ubtrace` release.

| Date | Version | Notes |
| --- | --- | --- |
| 2026-05-20 | 1.4.0-dev.35 | First prod overlay + helm upgrade runbook; EKS Terraform stack and bastion tunnel landed |
| 2026-05-22 | 1.4.0-dev.38 | Deploy automation (deploy.sh, manual workflow, OCI chart support) |
| 2026-05-25 | 1.4.0-dev.39 | Deploy + ~21 min outage during investigation; startup-probe budget raised; recovered (see outage-2026-05-25.md) |
| 2026-05-26 | 1.4.0-dev.40, then 1.4.0 | dev.40, then stable 1.4.0 |
| 2026-05-29 | 1.5.0-dev.0 | Pre-release bump |

(For reference, 1.5.0 stable shipped 2026-06-03, outside May.)

## Issues filed

| Date | Repo | # | Title |
| --- | --- | --- | --- |
| 2026-05-13 | ubtrace | #1346 | Worker ingestion loops forever for projects with only an AspiceReadinessConfig row |
| 2026-05-15 | ubinfra | #148 | Migrate PyPI stack from SST to Terraform |
| 2026-05-18 | ubtrace | #1379 | Worker scheduler loops re-ingesting versions when NDJSON format_version is below expected |
| 2026-05-20 | ubt-sphinx | #51 | v0.8.0 wheel missing sphinx.builders entry point for ubtrace builder |
| 2026-05-25 | ubtrace | #1418 | api startup blocks on EFS scan, exceeds startup probe budget |

The three worker issues (#1346, #1379, and the later #1419/#1423 line) trace the same prod ingestion-loop regression that later degraded the org/project selector.

## Pull requests

| Date | Repo | # | State | Title |
| --- | --- | --- | --- | --- |
| 2026-05-05 | sphinx-needs-demo | #52, #53, #55, #56 | merged | v-model trace content + status-field sweep |
| 2026-05-13 | ubinfra | #147 | merged 05-15 | PyPI proxy returns 404 for unknown packages so uv falls through to pypi.org |
| 2026-05-20 | ubinfra | #156 | open | ubtrace instance config, runbooks, and deploy automation |
| 2026-05-20 | ubtrace | #1402 | open | Chart + Terraform + publish workflow for the ubtrace deploy surface |
| 2026-05-29 | ubinfra | #61 | merged | runs-on custom AMI pipeline, TISAX hardening, OIDC IAM |

## Deployment-config commits (ubinfra)

The deploy surface itself (overlay, deploy.sh, runbooks, workflows) was built across 05-20 to 05-24: EKS Terraform stack, bastion tunnel script, kubectl/RDS access runbooks, deploy.sh with tunnel/template/diff-live/snapshot/apply/smoke subcommands, OCI chart support, manual deploy workflow, and the cross-repo chart checkout via SSH key.
