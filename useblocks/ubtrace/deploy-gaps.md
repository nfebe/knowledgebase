# Documentation, chart, and runbook gaps

The nine gaps that, if any one of them had been closed, would have prevented or shortened the 2026-05-01 outage. Source for the follow-up commits to PR #1050 once it has been rebased on `develop`.

## 1. No `values-team.yaml` in the chart

The prod overlay only exists in `/tmp/values-prod.yaml`. There is no source of truth in the repo. `values-eks-terraform.yaml` is the closest thing, but it targets the dev cluster (`*.internal.useblocks.com`) and is missing pieces (gap #3).

**Fix:** Commit the working prod overlay as `deploy/helm/ubtrace/values-team.yaml` (or similar). Strip secrets. Update `operations.rst` to reference it.

## 2. Chart defaults silently break upgrades on real clusters

Three chart defaults conflict with how prod is deployed today. None of them are flagged in the chart README:

- `ingress.enabled: false` (causes ALBs to be deleted if the overlay forgets the override)
- `persistence.*.accessMode: ReadWriteOnce` and `storageClass: ""` (causes PVC immutability errors; any RWX/EFS-backed deployment must override)
- API ingress `healthcheck-path: /health` (api returns 404; correct path is `/api/v1/health/readiness`)

**Fix:** Either change the chart defaults to safe-for-cloud values, or add an "EKS upgrade pre-flight checklist" section in the chart README listing the values that **must** be set in the overlay.

## 3. `values-eks-terraform.yaml` is missing the `admin:` block

The chart's reference EKS overlay does not configure the admin app. Anyone copying it as a starting point gets a deployment with no admin ingress.

**Fix:** Add the `admin:` block with `enabled: true` and matching annotations, plus an entry in the host-table in the doc.

## 4. API healthcheck path is wrong in chart and docs

`values.yaml` and `values-eks-terraform.yaml` both have `alb.ingress.kubernetes.io/healthcheck-path: /health`. The api on `1.3.0-dev.22+` does not expose `/health`; the readiness endpoint is `/api/v1/health/readiness`. Wrong path -> ALB target group permanently unhealthy -> 502.

**Fix:** Change the chart default to `/api/v1/health/readiness`. (Or expose `/health` in the api as a permanent alias.)

## 5. Cert covers 4 hosts but only `admin` ingress configured for HTTPS

The ACM cert `aac8197b-...` has all four SANs (`team`, `api`, `admin`, `auth`). Only `admin` had the HTTPS:443 listener wired into its ingress annotations. The other three ingresses listed only HTTP:80, so HTTPS access failed at the ALB layer (HSTS-upgraded browser hits no listener).

**Fix:** All four ingress blocks must include:

```yaml
alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:eu-central-1:809338888439:certificate/aac8197b-13ad-4314-985a-d7a2941014dd
alb.ingress.kubernetes.io/ssl-redirect: '443'
```

Document this in `operations.rst` under "Helm Upgrades" and in `configuration.rst` under TLS.

## 6. `global.existingSecret` cutover deletes the live secret

Without the `helm.sh/resource-policy: keep` annotation on the existing `ubtrace` Secret, the first upgrade that flips to `global.existingSecret: <name>` causes Helm to delete the secret. (Helm decides from the previous release manifest, not from live cluster annotations.)

**Fix:** The chart README (or a dedicated `MIGRATING.md`) needs a "Switching to existingSecret" section with the explicit pre-step:

```bash
kubectl annotate secret <name> helm.sh/resource-policy=keep --overwrite
```

## 7. `?sslmode=no-verify` workaround is undocumented

`docs/source/cloud-deployment/configuration.rst` says nothing about RDS TLS. Issue #1122 documents the workaround. PR #1198/#1199 will fix this properly. Until then operators have to discover the symptom (Prisma `P1011`) and the workaround themselves.

**Fix:** Add a "RDS TLS gotcha" sub-section to `configuration.rst` referencing #1122. Mention the `?sslmode=no-verify` suffix and the `global.existingSecret` route as the only way to persist it.

## 8. Runbook still targets `*.internal.useblocks.com`

`docs/source/cloud-deployment/eks-internal.rst` is the only EKS runbook in the repo. It targets the dev environment. There is no equivalent runbook for the prod (`team.useblocks.com`) deployment, even though it has its own AWS account, cluster, certs, and Route53 zone.

**Fix:** Either generalise `eks-internal.rst` (parameterise host/account) or add `eks-team.rst`. At minimum add a "Targets and accounts" preamble explaining which file is for which environment.

## 9. ALB recreation forces Route53 update

The chart has no documentation of the fact that significant ingress annotation changes (scheme, listen-ports, cert ARN flip) cause AWS LB Controller to recreate the ALB with a new DNS name, leaving Route53 alias records pointing at a deleted ALB. The recovery uses `aws route53 change-resource-record-sets` with a UPSERT change-batch.

**Fix:** Add a "When ALBs are recreated" sub-section to `operations.rst` with the change-batch JSON template and the eu-central-1 ALB hosted zone ID (`Z215JYRZR1TBD5`).

---

## Sequencing for PR #1050

PR #1050 is currently 115+ commits behind `develop`. Suggested sequencing:

1. Rebase `feat/844-eks-google-idp` onto latest `develop`.
2. Open user review of this gap list and pick which to close.
3. Add small commits per gap (1 commit per gap, conventional-commit prefix `docs(deploy):` or `fix(helm):` as appropriate).
4. Re-run CI, request review.

Gaps 1, 2, 4, 5, 6 are the highest-leverage (each one is a separate root cause).
