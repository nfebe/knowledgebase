# Does PR #1050 cover prod upgrades?

PR #1050 (`feat/844-eks-google-idp`) is the EKS-on-prod PR. As of branch tip `56d9ee82` it is **115+ commits behind develop** and needs rebasing before any new commits land.

## Files PR #1050 touches under `docs/` and `deploy/`

```
deploy/helm/ubtrace/files/ubtrace-realm.json       # +37 / -2
deploy/helm/ubtrace/templates/keycloak-configmap.yaml  # +2 / -2
deploy/helm/ubtrace/values-internal.yaml           # +51 / -0
deploy/helm/ubtrace/values.yaml                    # +8 / -0
docs/source/cloud-deployment/eks-internal.rst      # +279 / -0  (NEW)
docs/source/cloud-deployment/index.rst             # +1 / -0
```

`eks-internal.rst` is the only new doc. It is a 279-line runbook for first-time deployment of `*.internal.useblocks.com` (the dev environment in account `610014107205`).

## What it covers

- Initial Terraform apply (`tofu apply -var-file=...`).
- Initial helm install with values overlay.
- Google IdP / restricted-domain identity provider setup.
- Manual smoke tests (try logging in with a non-`@useblocks.com` Google account, confirm rejection).
- A "Rollback" section: `helm uninstall` + `tofu destroy`.
- A small "partial rollback" snippet (`helm upgrade ... --set keycloak.google.enabled=false`).
- Troubleshooting for 3 specific failure modes around Google IdP.

## What it does NOT cover

This is a **first-deployment** doc, not an **upgrade** doc. It does not address:

| Topic | Mentioned? |
| --- | --- |
| How to upgrade chart version | No (only the `--set` partial-rollback example) |
| How to upgrade app images / `image.tag` | No |
| Pre-flight checks before `helm upgrade` | No |
| Backing up the `ubtrace` Secret | No |
| `helm.sh/resource-policy: keep` annotation | No |
| `global.existingSecret` cutover hazard | No |
| PVC / StatefulSet immutability gotchas | No |
| What ingress fields are mandatory in the overlay | No |
| ALB recreation -> Route53 update | No |
| API ingress healthcheck path | No |
| RDS TLS workaround (`?sslmode=no-verify`) | No |
| Smoke tests after upgrade (curl, target group health) | No |
| Rollback for a failed chart upgrade | Only `helm uninstall` + `tofu destroy`, which is teardown |

The minimal "Helm Upgrades" section in `docs/source/cloud-deployment/operations.rst` (which exists outside PR #1050, on develop) shows exactly one helm upgrade command and a `helm rollback`, with none of the above caveats.

## Conclusion

**PR #1050 does not document the upgrade procedure.** Adding a routine-upgrade runbook is one of the gaps in `deploy-gaps.md` (gap #8). It should be added either by extending `eks-internal.rst` with an "Upgrades" section, or by promoting `operations.rst` into a fuller runbook (preferred, because operations apply to both dev and prod environments).

The full upgrade recipe we developed during the 2026-05-01 outage is captured in `upgrade-procedure.md` in this folder. That can be lifted directly into the docs (after stripping prod-specific identifiers) when the PR is rebased and the gap fixes start landing.
