# ubTrace prod deploy process (current working mode)

Authoritative as of 2026-06-23. This supersedes any "helm-only, prod has no terraform state" wording in older notes here (that was true before the 2026-06-15 state reconciliation, not now). Read this first when asked to deploy ubTrace to `team.useblocks.com`.

## The arrangement (both EKS PRs now merged)

Both branches merged on 2026-06-23: ubtrace #1402 (chart + Terraform modules) into `develop`, ubinfra #156 (overlay + `deploy.sh` + runbooks) into `main`. The earlier "deploy from unmerged branches" mode is over.

- **Chart source: the published OCI chart**, `oci://ghcr.io/useblocks/charts/ubtrace`, pinned by version. The chart with the keycloak-config-cli templates and the keycloak.userProfile/google fields exists only on develop-line tags (`1.6.0-dev.28` and later); the rc.0/rc.1 tags predate #1402. The dry-run and deploy were pinned to the OCI chart in ubinfra #185 (PR #186); before that, the dry-run rendered against a moving `develop` and was false-green. Run `helm registry login ghcr.io` before pulling; the ghcr `charts/ubtrace` package must grant the ubinfra repo read access for CI.
- **Instance config: in ubinfra.** The prod overlay `deploy/helm/ubtrace/values-team.yaml`, the `deploy.sh` wrapper, the runbooks under `ubtrace/`, and the Terraform instance config (`production.tfvars` + `backend-production.hcl`) under `deploy/terraform/ubtrace-prod/`. The ubtrace repo ships only `*.example` instance files and excludes real ones from the published Terraform tarball.

The canonical step-by-step runbook lives in ubinfra at `ubtrace/prod-helm-upgrade.md`; `deploy.sh` implements it. This note is the context around it, not a copy.

## Terraform state exists (this is the part that confused us)

Prod has real, usable Terraform state since the 2026-06-15 reconciliation. See [prod-terraform-state-and-reconciliation.md](prod-terraform-state-and-reconciliation.md). Backend (prod account `809338888439`):

- bucket `ubtrace-terraform-state-prod`, key `prod/terraform.tfstate`, lock table `ubtrace-terraform-locks-prod`
- init: `tofu init -reconfigure -backend-config=environments/backend-production.hcl`
- plan: `tofu plan -var-file=environments/production.tfvars` (the tfvars is not auto-loaded)
- Use `tofu`, never `terraform` (HashiCorp rewrites the lockfile to the wrong registry).

What this means for releases: a routine version bump is **helm only**. The terraform plan is clean (one benign in-place audit-logs SSE diff, a tofu/aws 6.38 provider artifact, ignore it). You only run `tofu apply` when actually changing infra (the cost-cutting work: EKS extended-support upgrade, sizing reconciled down to the cheaper tier). Do not bundle that into a version-bump deploy.

## Connecting to the private EKS API

The API server is private. Reach it through the SSM tunnel via the bastion (`i-0be1fa4eff999b159`):

- `AWS_PROFILE=ubtrace-prod-admin bash deploy.sh tunnel` (binds local `6443`, foreground, leave running).
- `/etc/hosts` maps the EKS hostname to `127.0.0.1`. The kubeconfig `server:` line for the `ubtrace-prod` cluster must end in `:6443` to match the tunnel. If it points at `:443`, patch it (back up `~/.kube/config` first):
  `kubectl config set-cluster <arn> --server=https://<eks-hostname>:6443`
- Confirm with `kubectl get ns`.

## Release procedure (version bump)

1. **Target tag.** Image tags drop the leading `v`: git tag `v1.6.0-rc.0` becomes image tag `1.6.0-rc.0`. Confirm the `publish-images.yml` run for that git tag is green; the images are private on ghcr, so you cannot pull-test from a laptop, the workflow success is the signal. A stable tag is cut by the `release.yml` workflow (`workflow_dispatch`, `version=1.6.0`, empty prerelease) or by merging `release/1.6.0` into `main`; cutting a stable that does not exist yet is a separate, outward-facing step.
2. **Verify live.** `helm history ubtrace -n ubtrace` (latest must be `deployed`, not `failed`/`pending`), current image tags, pod readiness.
3. **Bump the overlay.** In `values-team.yaml`, change `image.tag` on the five ubTrace components (`api`, `frontend`, `admin`, `worker`, `builder`) to the target. Leave `keycloak` (the upstream server, `quay.io/keycloak/keycloak`) and `elasticsearch` alone. `keycloak-theme` is pinned by the chart default, not the overlay, so it tracks the chart, this is why it can shift independently of the release tag and is not a keycloak change.
4. **Dry-run.** `deploy.sh diff-live <path-to-ubtrace-chart>`. Expect image tags plus intentional chart fields. Stop if ConfigMaps disappear, Secrets change, or Ingresses lose annotations.
5. **Snapshot both DBs.** `deploy.sh snapshot <tag>` (or the runbook's `aws rds create-db-snapshot` for `ubtrace-prod-app` and `ubtrace-prod-keycloak`). Snapshot identifiers cannot contain dots, so `1.6.0-rc.0` becomes `1-6-0-rc-0` in the id. Wait until both are `available`.
6. **Apply.** `deploy.sh apply <path-to-ubtrace-chart>` (`helm upgrade --atomic --timeout 10m`). `--atomic` rolls back on failure. Watch `kubectl get pods -n ubtrace -w`.
7. **Smoke.** `deploy.sh smoke`: four hosts (`team` 200, `admin` 200, `auth` 302, `api` 404) plus `/api/v1/health/readiness` returning `status: ok` with idProvider, cache, search, database all `up`. The 1.5.0 selector incident showed as `search` flapping, so check that field specifically.
8. **Point the branch at the release.** Commit the bumped `values-team.yaml` on the ubinfra branch so the file mirrors live.

## Known fixes that ride along on the current chart

Deploying from the `feat/844-eks-templates` chart tip also lands, beyond image tags:

- a `scratch` emptyDir (4Gi) on the builder at `/scratch`: the #1446 ingest-504 extraction fix.
- worker env `WORKER_RETRY_NEW_VERSION=3` and `WORKER_RETRY_CONTENT_CHANGED=3`: bounds the ingestion-loop retries (#1419 / #1423).

## Last deploy

2026-06-22: `1.6.0-rc.0` to prod, helm rev 35 `deployed`, all five components on rc.0, smoke green (search up, no flapping). Previous: `1.5.0` at rev 34 (2026-06-03). Stable `1.6.0` was not cut at the time; rc.0 was deployed by choice.
